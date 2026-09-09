# Demo — Levantar RabbitMQ con Docker y explorar la consola web (Windows)

**Para qué es esto:** la demo de 10 minutos de la Sesión 1 (sin código) — levantar RabbitMQ y mostrarles la consola web en `http://localhost:15672` antes de que toquen una línea de Java. Pensado para ejecutarlo en un PC con Windows.

---

## Paso 0 — Prerrequisito: Docker Desktop instalado

Si ya lo tienes instalado y corriendo, salta al Paso 1.

1. Descarga Docker Desktop desde [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) (botón "Download for Windows").
2. Ejecuta el instalador. Cuando pregunte, deja marcada la opción **"Use WSL 2 instead of Hyper-V"** (es la recomendada; Hyper-V también funciona pero WSL2 es más liviano y es lo que la mayoría de las guías asume).
3. Si Windows te pide reiniciar, reinicia.
4. Abre **Docker Desktop** desde el menú de inicio. La primera vez tarda un poco en arrancar el motor — espera a que el ícono de la ballena (abajo a la izquierda de la ventana, o en la bandeja del sistema) diga **"Engine running"** en verde.

**Problema típico en este paso:** si Docker Desktop se queja de que "WSL 2 no está instalado" o "virtualización deshabilitada":
- Virtualización: hay que habilitarla en la BIOS/UEFI del equipo (varía por fabricante — se busca como "Intel VT-x", "AMD-V" o "Virtualization Technology"). Si el laptop es de la universidad y no tienen acceso a la BIOS, este es un buen punto para avisar con anticipación.
- WSL2 faltante: abrir PowerShell **como administrador** y correr `wsl --install`, luego reiniciar.

---

## Paso 1 — Crear una carpeta para la demo

Abre **Git Bash** (recomendado, así los comandos son iguales a los de la guía) o PowerShell, y crea una carpeta cualquiera, por ejemplo:

```bash
mkdir demo-rabbitmq
cd demo-rabbitmq
```

---

## Paso 2 — Crear el archivo `docker-compose.yml`

Dentro de esa carpeta, crea un archivo llamado exactamente `docker-compose.yml` (con Notepad, VS Code, o el editor que uses) con este contenido:

```yaml
services:
  rabbitmq:
    image: rabbitmq:4.2-management
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
```

**Qué es cada parte, para explicarlo en vivo si preguntan:**
- `rabbitmq:4.2-management` — la imagen oficial de RabbitMQ, en su variante `-management` (trae la consola web incluida; la imagen "pelada" sin ese sufijo no la tiene).
- `5672` — el puerto que usan las aplicaciones (Spring Boot, etc.) para conectarse al broker.
- `15672` — el puerto de la consola web que vamos a abrir en el navegador.
- `guest` / `guest` — usuario y clave por defecto, válidos solo para uso local (nunca en producción).

**Nota Windows:** si usas VS Code, ojo con que el archivo no se guarde como `docker-compose.yml.txt` — Windows a veces oculta la extensión real. En el explorador de archivos, activa "Mostrar extensiones de archivo" para verificarlo.

---

## Paso 3 — Levantar el contenedor

Desde la misma carpeta (donde está el `docker-compose.yml`):

```bash
docker-compose up -d
```

El flag `-d` es "detached" — corre en segundo plano y te devuelve la terminal.

**Nota Windows:** si te da error `'docker-compose' is not recognized`, es porque tienes una versión reciente de Docker Desktop que usa el comando integrado (sin guion). Prueba:

```bash
docker compose up -d
```

Ambas formas hacen lo mismo — solo cambia si tu versión de Docker trae el plugin nuevo o el binario viejo.

---

## Paso 4 — Verificar que está arriba

Tres formas rápidas, de más a menos visual:

```bash
docker ps
```
Debe aparecer una fila con `rabbitmq` y el estado `Up`.

```bash
docker logs rabbitmq
```
Buscar en el output la línea `Server startup complete` o `ready to accept connections` (puede tardar 10-20 segundos la primera vez que descarga la imagen).

```bash
docker exec rabbitmq rabbitmq-diagnostics ping
```
Si responde `Ping succeeded`, está listo.

---

## Paso 5 — Abrir la consola web

En el navegador, ir a:

```
http://localhost:15672
```

Login:
- **Usuario:** `guest`
- **Contraseña:** `guest`

---

## Paso 6 — Qué mostrar en la consola (para la demo en vivo)

Con la clase mirando la pantalla proyectada, recorre estas 3 secciones — es lo que arma el puente conceptual antes de escribir código:

1. **Overview** (pestaña por defecto) — mensajes publicados, tasa de mensajes por segundo, memoria usada. Punto a decir: "esto es el tablero de control de todo lo que pasa por el broker, en tiempo real".
2. **Connections** — vacío por ahora (todavía no hay ninguna app conectada). Punto a decir: "acá vamos a ver aparecer nuestra app de Spring Boot en la próxima sesión, apenas se conecte".
3. **Queues and Streams** — también vacío. Punto a decir: "esto es literalmente lo mismo que van a crear con `@Bean Queue` en Java — la consola te deja crear una cola manualmente para probar, sin escribir código". Si quieres, puedes crear una cola de prueba a mano desde acá (botón "Add a new queue", nombre `demo`, click "Add queue") solo para mostrar que aparece al instante en la lista — y borrarla después con el botón "Delete".

---

## Paso 7 — Apagar todo al terminar la demo

Para dejar el equipo limpio (o antes de la Sesión 2, para que los alumnos lo levanten ellos mismos desde cero):

```bash
docker-compose down
```

Esto detiene y elimina el contenedor (no borra la imagen descargada, así que la próxima vez que hagan `up -d` arranca mucho más rápido).

---

## Troubleshooting rápido

| Síntoma | Causa típica | Solución |
|---|---|---|
| `docker-compose up` se queda colgado o da timeout | Docker Desktop no terminó de iniciar el motor | Esperar a que el ícono de la ballena diga "Engine running", reintentar |
| `Bind for 0.0.0.0:5672 failed: port is already allocated` | Ya hay otro RabbitMQ corriendo (de una sesión anterior no cerrada) | `docker ps` para verlo, luego `docker stop rabbitmq` o `docker-compose down` en esa carpeta |
| La consola web no carga (`localhost:15672` no responde) | El contenedor sigue iniciando, o el puerto 15672 está ocupado por otra app | Esperar 15-20s más; si persiste, revisar `docker logs rabbitmq` |
| Login guest/guest falla | Poco común, pero puede pasar si quedó un volumen viejo con otra configuración | `docker-compose down -v` (el `-v` borra también los datos) y volver a `up -d` |

---

## Enlaces oficiales

- Docker Desktop para Windows: https://docs.docker.com/desktop/setup/install/windows-install/
- RabbitMQ — imagen oficial en Docker Hub: https://hub.docker.com/_/rabbitmq
- RabbitMQ — Management Plugin: https://www.rabbitmq.com/docs/management
