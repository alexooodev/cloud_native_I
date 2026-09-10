# Levantar RabbitMQ sin Docker Desktop — usando GitHub Codespaces

Si tu laptop no puede correr Docker Desktop porque la virtualización por hardware (VT-x/AMD-V) está bloqueada en la BIOS, esta guía es para ti. Vas a hacer exactamente la misma actividad de RabbitMQ (Sesión 2 y 3), con los mismos comandos de Docker, solo que corriendo en un Codespace — una máquina Linux en la nube de GitHub — en vez de en tu laptop.

**Requisito:** cuenta de GitHub (la misma que ya usas para entregar tus repos).

---

## Paso 0 — Activar el GitHub Student Developer Pack (recomendado, no obligatorio)

Te da más horas gratis de Codespaces al mes.

1. Ir a [education.github.com/pack](https://education.github.com/pack).
2. "Get student benefits" → verificar con tu correo institucional (`@duocuc.cl` o el que pida GitHub).
3. Puede tardar un par de días en aprobarse — no es bloqueante: con la cuenta gratis normal ya alcanza para esta actividad (GitHub Free da 120 core-hours al mes, suficiente para varias sesiones de clase).

---

## Paso 1 — Preparar tu repo con el `devcontainer.json`

Este archivo le dice a GitHub "cuando abra un Codespace acá, instala Docker adentro".

1. Abre el repo de GitHub donde tienes tu proyecto Spring Boot de la actividad (el mismo que vas a usar para entregar el encargo).
2. Crea la carpeta `.devcontainer` en la raíz del repo, con un archivo `devcontainer.json` dentro:

```json
{
  "name": "rabbitmq-demo",
  "image": "mcr.microsoft.com/devcontainers/java:17",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  },
  "forwardPorts": [5672, 15672, 8080, 5173],
  "postCreateCommand": "docker compose version"
}
```

**Qué hace cada parte:**
- `image: java:17` — la máquina ya viene con Java 17 y Maven instalados, lo que necesitas para Spring Boot.
- `features: docker-in-docker` — esto es lo que habilita `docker` y `docker compose` adentro del Codespace.
- `forwardPorts` — deja pre-anunciados los puertos que vas a usar: 5672 (AMQP), 15672 (consola de RabbitMQ), 8080 (backend Spring Boot), 5173 (frontend Vite, para la Sesión 3).

3. Haz commit y push de ese archivo a tu repo (puedes crearlo directo en la web de GitHub con "Add file" → "Create new file", o desde tu Git local como siempre).

---

## Paso 2 — Abrir el Codespace

1. En la página de tu repo en GitHub, botón verde **Code**.
2. Pestaña **Codespaces** → **Create codespace on main**.
3. Se abre una pestaña nueva con VS Code corriendo en el navegador. La primera vez tarda 2-4 minutos en construirse (está instalando Docker-in-Docker) — las siguientes veces que lo abras es casi instantáneo.

---

## Paso 3 — Verificar que Docker funciona

En la terminal integrada de VS Code (`Terminal` → `New Terminal`, o `` Ctrl+` ``):

```bash
docker --version
docker compose version
```

Si ambos responden con una versión, está listo.

---

## Paso 4 — Levantar RabbitMQ (mismo docker-compose.yml de la guía original)

Crea el archivo en la raíz del repo (con el explorador de archivos de VS Code, o por terminal):

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

Levantar:

```bash
docker compose up -d
docker ps
docker logs rabbitmq
```

Es idéntico a lo que harías en tu propio laptop — el archivo no cambia en absoluto.

---

## Paso 5 — Abrir la consola web de RabbitMQ

Codespaces detecta automáticamente que algo quedó escuchando en el puerto 15672 y muestra una notificación abajo a la derecha ("Your application running on port 15672..."). Si no aparece:

1. Pestaña **Ports** (al lado de "Terminal", en el panel inferior de VS Code).
2. Busca la fila del puerto `15672` → clic en el ícono del globo (Open in Browser).
3. Se abre una pestaña nueva con la consola. Login: `guest` / `guest`.

**Nota de seguridad:** por defecto el puerto se comparte como **Private** (solo tú, con tu sesión de GitHub, puedes abrirlo) — no hace falta cambiarlo a Public para esta actividad.

---

## Paso 6 — Correr el backend Spring Boot (dentro del mismo Codespace)

Como RabbitMQ corre en la misma máquina que tu Codespace, **no necesitas tocar `application.yml`** — `host: localhost` sigue siendo correcto.

```bash
mvn clean package
mvn spring-boot:run
```

Si necesitas probar el endpoint REST desde fuera del Codespace (por ejemplo, con Postman en tu laptop), usa el puerto 8080 igual que en el Paso 5: pestaña Ports → abrir en navegador, o copiar la URL forwarded.

---

## Paso 7 — Sesión 3: frontend Vite + React

Mismo patrón. Dentro del Codespace:

```bash
npm install
npm run dev
```

Vite va a avisar que está en el puerto 5173 — se forwardea igual que los demás, aparece en la pestaña Ports con un link para abrirlo en una pestaña del navegador. Ahí pruebas los 3 botones (INFO/WARNING/ERROR) exactamente como en la guía.

---

## Paso 8 — Al terminar

Los Codespaces se pausan solos tras ~30 minutos sin actividad, pero **siguen contando horas del plan gratuito mientras estén corriendo** (no mientras están pausados por inactividad, pero sí mientras figuran como "Running"). Para no gastar horas de más:

1. Ir a [github.com/codespaces](https://github.com/codespaces).
2. Ubicar el Codespace de la actividad → menú `...` → **Stop codespace**.

El Codespace no se borra al detenerlo — la próxima vez que lo abras desde el mismo lugar, todo lo que instalaste/hiciste sigue ahí.

---

## Troubleshooting

| Síntoma | Causa típica | Solución |
|---|---|---|
| El Codespace tarda mucho en crearse la primera vez | Está descargando la imagen base + instalando Docker-in-Docker | Normal, 2-4 min la primera vez; después es rápido |
| `docker: command not found` | El `devcontainer.json` no tiene el feature docker-in-docker, o el Codespace se creó antes de agregarlo | Revisar que el archivo esté bien escrito y hacer **Rebuild Container** (`Ctrl+Shift+P` → "Codespaces: Rebuild Container") |
| No aparece la pestaña "Ports" | Está colapsada | Junto a "Terminal" en el panel inferior, o `Ctrl+Shift+P` → "Ports: Focus on Ports View" |
| Te quedaste sin horas gratis del mes | Codespaces quedaron corriendo sin usarlos | Revisar github.com/codespaces y detener los que no uses; activar el Student Pack si no lo tienes |
| El puerto forwarded pide iniciar sesión en GitHub | Visibilidad en Private (default) | Es lo esperado y lo más seguro para esta actividad — no hace falta cambiarlo |
