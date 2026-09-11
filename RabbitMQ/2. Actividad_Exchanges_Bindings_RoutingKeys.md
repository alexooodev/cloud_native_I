# Actividad — Exchanges, bindings y routing keys aplicados al caso

**Sesión 3 del bloque de mensajería.** Requisito: haber hecho la Sesión 2 (RabbitMQ corriendo en Docker local, con la consola web en `http://localhost:15672` accesible).

## El caso

Van a construir un sistema de logs realista donde los mensajes se enrutan de forma inteligente. El escenario: una aplicación genera logs con 3 niveles de severidad — `INFO`, `WARNING` y `ERROR`. Necesitamos:

- Una cola que reciba **todos** los logs, para monitoreo general.
- Una cola que reciba **sólo** los `ERROR`, para un sistema de alertas críticas.

Esto es el mismo problema que van a encontrar cada vez que necesiten que un evento dispare varias acciones distintas según su tipo — hoy lo resolvemos con logs, pero el patrón sirve para cualquier cosa que se enrute por categoría (pedidos por estado, notificaciones por canal, etc.).

**Qué van a aprender:** usar un `DirectExchange` para enrutar mensajes según una routing key, crear varias colas con propósitos distintos, usar múltiples bindings para conectar esas colas al exchange, y levantar un frontend simple para disparar los eventos.

---

## Backend — Spring Boot

### Paso 1 — Dependencias

Si vienen de la Sesión 2 pueden reusar ese mismo proyecto. Si arrancan uno nuevo, en Spring Initializr agreguen las mismas dos dependencias de siempre:

- Spring Web
- Spring for RabbitMQ

`pom.xml` debe tener:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Nada nuevo respecto a la Sesión 2 — mismas dos dependencias.

### Paso 2 — Configurar la topología (Exchange, Queues, Bindings)

Esta es la clase completa que define **todo** lo que va a existir en RabbitMQ para este caso. Créala en `RabbitMQConfig.java`:

```java
package com.example.loggingsystem;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    public static final String EXCHANGE_NAME = "logs_direct_exchange";
    public static final String ALL_LOGS_QUEUE = "all_logs_queue";
    public static final String ERRORS_ONLY_QUEUE = "errors_only_queue";

    // 1. Declarar el Exchange
    @Bean
    public DirectExchange directExchange() {
        return new DirectExchange(EXCHANGE_NAME);
    }

    // 2. Declarar las Queues
    @Bean
    public Queue allLogsQueue() {
        return new Queue(ALL_LOGS_QUEUE, true); // durable
    }

    @Bean
    public Queue errorsOnlyQueue() {
        return new Queue(ERRORS_ONLY_QUEUE, true); // durable
    }

    // 3. Declarar los Bindings
    // Cada binding une una cola a un exchange con una routing key.

    @Bean
    public Binding bindAllLogsForInfo(DirectExchange exchange, Queue allLogsQueue) {
        return BindingBuilder.bind(allLogsQueue).to(exchange).with("INFO");
    }

    @Bean
    public Binding bindAllLogsForWarning(DirectExchange exchange, Queue allLogsQueue) {
        return BindingBuilder.bind(allLogsQueue).to(exchange).with("WARNING");
    }

    @Bean
    public Binding bindAllLogsForError(DirectExchange exchange, Queue allLogsQueue) {
        return BindingBuilder.bind(allLogsQueue).to(exchange).with("ERROR");
    }

    @Bean
    public Binding bindErrorsOnly(DirectExchange exchange, Queue errorsOnlyQueue) {
        return BindingBuilder.bind(errorsOnlyQueue).to(exchange).with("ERROR");
    }
}
```

**Qué estamos haciendo y por qué:** un `DirectExchange` reenvía cada mensaje a las colas cuyo binding tenga exactamente la misma routing key que trae el mensaje. `all_logs_queue` está enlazada 3 veces — una por cada nivel — así que le llega todo. `errors_only_queue` está enlazada una sola vez, solo a `"ERROR"`, así que solo le llegan esos. Antes de seguir, dibuja esto en un papel como un diagrama de flechas (exchange al centro, 4 flechas de binding saliendo hacia las 2 colas) — se entiende mucho más rápido así que leyendo el código.

`durable=true` (a diferencia de la cola `hello` de la Sesión 2, que era `false`) significa que estas colas sobreviven un reinicio de RabbitMQ — tiene sentido para un sistema de logs que no quieres perder.

### Paso 3 — Los consumidores

```java
package com.example.loggingsystem;

import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Service;

@Service
public class LogConsumer {

    @RabbitListener(queues = RabbitMQConfig.ALL_LOGS_QUEUE)
    public void receiveAllLogs(String message) {
        System.out.println("[MONITOR GENERAL] Log recibido: " + message);
    }

    @RabbitListener(queues = RabbitMQConfig.ERRORS_ONLY_QUEUE)
    public void receiveErrorLogs(String message) {
        System.out.println("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!");
        System.out.println("[ALERTA CRÍTICA] Error detectado: " + message);
        System.out.println("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!");
    }
}
```

**Punto clave:** cada método solo sabe de su cola — ninguno sabe que existe el exchange ni los bindings. Esa ignorancia es a propósito: es lo que permite agregar mañana un tercer consumidor (por ejemplo, uno que solo escuche `WARNING` para un panel de auditoría) sin tocar nada de lo que ya existe.

### Paso 4 — El productor (REST Controller)

Este es el punto de entrada para el frontend:

```java
package com.example.loggingsystem;

import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@CrossOrigin(origins = "http://localhost:5173") // permite peticiones desde Vite
public class LogProducerController {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostMapping("/log")
    public String sendLog(@RequestBody LogMessage logMessage) {
        // La routing key es el nivel del log (INFO, WARNING, ERROR)
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.EXCHANGE_NAME,
            logMessage.getLevel(),
            logMessage.getMessage()
        );
        return "Log enviado: " + logMessage.getMessage();
    }

    // DTO para el cuerpo de la petición
    public static class LogMessage {
        private String level;
        private String message;

        public String getLevel() { return level; }
        public void setLevel(String level) { this.level = level; }
        public String getMessage() { return message; }
        public void setMessage(String message) { this.message = message; }
    }
}
```

**El "aha" de esta clase:** `logMessage.getLevel()` — la routing key **no es un campo técnico aparte que inventamos**, es un dato de negocio que ya viene en la petición (la severidad del log). En cualquier sistema real, la routing key casi siempre va a ser algo que el dominio ya te da: un estado, una categoría, un tipo de evento.

`@CrossOrigin` — es lo mismo que ya vieron en el módulo de zapatillas: permite que el frontend (puerto 5173) le hable al backend (puerto 8080) sin que el navegador lo bloquee.

---

## Frontend — Vite + React

### Paso 5 — Crear el proyecto

Si es un proyecto nuevo:

```bash
npm create vite@latest
# Seleccionar: React → JavaScript
cd <nombre-del-proyecto>
npm install
```

### Paso 6 — La interfaz

Reemplazar `src/App.jsx`:

```jsx
import React from 'react';
import './App.css';

function App() {

  const sendLog = async (level, message) => {
    const backendUrl = 'http://localhost:8080/log';
    try {
      const response = await fetch(backendUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ level, message }),
      });
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      const result = await response.text();
      console.log(result);
      alert(`Log '${level}' enviado!`);
    } catch (error) {
      console.error("Error al enviar el log:", error);
      alert("Error al enviar el log. Revisa la consola.");
    }
  };

  return (
    <div className="App">
      <h1>Sistema de Logging RabbitMQ</h1>
      <div className="button-container">
        <button className="info" onClick={() => sendLog('INFO', 'El usuario ha iniciado sesión.')}>
          Enviar Log INFO
        </button>
        <button className="warning" onClick={() => sendLog('WARNING', 'El uso de CPU está al 85%.')}>
          Enviar Log WARNING
        </button>
        <button className="error" onClick={() => sendLog('ERROR', 'No se pudo conectar a la base de datos.')}>
          Enviar Log ERROR
        </button>
      </div>
    </div>
  );
}

export default App;
```

`src/App.css`:

```css
.button-container { display: flex; gap: 1rem; justify-content: center; margin-top: 2rem; }
button { padding: 1rem; border-radius: 8px; border: none; cursor: pointer; font-size: 1rem; }
.info { background-color: #3498db; color: white; }
.warning { background-color: #f1c40f; color: white; }
.error { background-color: #e74c3c; color: white; }
```

No hay nada nuevo de React acá — es el mismo patrón de `fetch` que ya usaron en zapatillas.

---

## Prueba completa del sistema

1. Confirmar que RabbitMQ sigue corriendo: `docker ps` (o revisar `http://localhost:15672`).
2. Iniciar el backend: `mvn spring-boot:run`. Debe conectarse a RabbitMQ sin errores.
3. Iniciar el frontend: `npm run dev`, abrir `http://localhost:5173`.
4. **Clic en "Enviar Log INFO"** — en la consola de Spring Boot debe aparecer solo:
   ```
   [MONITOR GENERAL] Log recibido: El usuario ha iniciado sesión.
   ```
5. **Clic en "Enviar Log WARNING"** — mismo resultado, solo en MONITOR GENERAL.
6. **Clic en "Enviar Log ERROR"** — este es el momento importante: deben aparecer **ambos** consumidores reaccionando al mismo mensaje:
   ```
   [MONITOR GENERAL] Log recibido: No se pudo conectar a la base de datos.
   !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
   [ALERTA CRÍTICA] Error detectado: No se pudo conectar a la base de datos.
   !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
   ```

Si ven esto, el mismo mensaje llegó a dos colas distintas a través de un solo exchange — el objetivo de la clase está cumplido.

---

## Cierre — 3 ideas que tienen que quedar claras

1. Un exchange nunca almacena mensajes, solo decide a dónde reenviarlos.
2. Una cola puede tener varios bindings (recibir por varias routing keys).
3. Varias colas pueden compartir la misma routing key — el mismo mensaje llega a todas.

---

## Troubleshooting

| Síntoma | Causa típica | Solución |
|---|---|---|
| El backend no arranca / error de conexión a RabbitMQ | RabbitMQ no está corriendo | `docker ps`; si no aparece, `docker-compose up -d` en la carpeta de la Sesión 2 |
| El botón ERROR solo activa un consumidor | Falta el binding de `errors_only_queue` a `"ERROR"`, o falta uno de los 3 bindings de `all_logs_queue` | Revisar `RabbitMQConfig.java` — deben existir 4 `@Bean Binding` en total |
| El frontend tira error de CORS en la consola del navegador | Falta o está mal escrito `@CrossOrigin` en el controller, o el puerto de Vite no es 5173 | Confirmar el puerto real que muestra `npm run dev` y que coincida con `@CrossOrigin(origins = "http://localhost:XXXX")` |
| `404` al hacer POST a `/log` | El backend no se recompiló tras el último cambio | `mvn clean package` y reiniciar |

---

## Enlaces oficiales

- Spring AMQP — Broker configuration: https://docs.spring.io/spring-amqp/reference/amqp/broker-configuration.html
- Spring AMQP — Sending messages: https://docs.spring.io/spring-amqp/reference/amqp/sending-messages.html
- RabbitMQ Tutorial 4 — Routing (Java): https://www.rabbitmq.com/tutorials/tutorial-four-java
- CORS Support in Spring Framework: https://docs.spring.io/spring-framework/reference/web/webmvc-cors.html
- Vite — Getting Started: https://vitejs.dev/guide/
- React — Fetch API (MDN): https://developer.mozilla.org/es/docs/Web/API/Fetch_API/Using_Fetch
