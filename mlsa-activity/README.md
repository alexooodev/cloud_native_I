# Guía paso a paso — Configurar MSAL en el Frontend (Angular)

**Objetivo:** que un proyecto Angular pueda mostrar un botón "Iniciar sesión", loguear al usuario contra Microsoft Entra ID, y guardar el token para usarlo después.

Esta guía toma el tutorial oficial de Duoc y le agrega el "por qué" de cada paso, el código completo y qué revisar si algo no funciona.

## Antes de empezar — checklist de prerrequisitos

No avances sin tener esto listo, porque MSAL necesita datos que solo existen después de registrar la app:

- [ ] Completaste el tutorial de **registro de app en Microsoft Entra ID** ("Creando una aplicación para usuarios externos").
- [ ] Tienes anotados estos 3 datos (portal de Azure → "App registrations" → tu app → "Overview"):
  - **Application (client) ID** → un UUID
  - **Directory (tenant) ID** → otro UUID
  - **Redirect URI** configurado (normalmente `http://localhost:4200` en desarrollo)

Esta guía asume que partes desde cero, sin proyecto entregado — el Paso 1 muestra cómo crearlo.

## Conceptos que vas a tocar

- **`@azure/msal-browser`**: la librería base que habla con Microsoft Entra ID.
- **`@azure/msal-angular`**: el adaptador que integra esa librería con Angular (guards, interceptors, servicios inyectables).
- **Interceptor**: código que se ejecuta automáticamente en cada petición HTTP saliente, para agregarle el token sin hacerlo a mano cada vez.
- **Guard**: código que Angular ejecuta antes de dejarte entrar a una ruta, para bloquear el acceso si no estás logueado.

---

## Paso 1 — Crear el proyecto Angular desde cero

### 1a. Verificar Node.js y npm

```bash
node -v
npm -v
```

Deberían mostrar versiones (Angular 16+ necesita Node 18 o superior). Si no tienes Node, instálalo desde [nodejs.org](https://nodejs.org) (versión LTS).

### 1b. Instalar Angular CLI globalmente

```bash
npm install -g @angular/cli
```

Verifica con:

```bash
ng version
```

### 1c. Crear el proyecto

```bash
ng new msal-frontend --routing --style=css --standalone
```

Cuando pregunte por Server-Side Rendering (SSR), responde `No`.

| Flag           | Qué hace                                                    |
| -------------- | ----------------------------------------------------------- |
| `--routing`    | genera `app.routes.ts`, necesario para `MsalGuard` (Paso 7) |
| `--style=css`  | CSS plano (puedes usar `scss` si prefieres)                 |
| `--standalone` | componentes sin `NgModule`, formato que asume esta guía     |

### 1d. Verificar que el proyecto base funciona

```bash
cd msal-frontend
ng serve
```

Abre `http://localhost:4200` — deberías ver la pantalla de bienvenida por defecto de Angular. Este checkpoint descarta que un error posterior sea de tu instalación base.

---

## Paso 2 — Instalar las librerías de MSAL

```bash
npm install @azure/msal-browser @azure/msal-angular
```

**Por qué dos paquetes:** `msal-browser` hace la autenticación en el navegador; `msal-angular` lo conecta con Angular.

**Verificación:** revisa `package.json` — deberían aparecer ambas dependencias.

> **Si `npm install` falla** con un error tipo `Cannot read properties of null (reading 'matches')`: es un bug conocido de npm, no un error tuyo.
>
> ```bash
> npm cache clean --force
> npm install @azure/msal-browser @azure/msal-angular
> ```
>
> Si persiste:
>
> ```bash
> rm -rf node_modules package-lock.json
> npm install
> ```

---

## Paso 3 — Cargar tus datos de Entra ID en `environment.ts`

Verifica que el archivo existe:

```bash
ls src/environments
```

**Desde Angular 15, `ng new` ya NO genera esta carpeta por defecto.** Si no existe, créala:

```bash
ng generate environments
```

Esto genera `environment.ts` (producción) y `environment.development.ts` (desarrollo). Para esta actividad, edita **`environment.development.ts`** (es el que usa `ng serve`).

```ts
export const environment = {
  production: false,
  msalConfig: {
    clientId: "TU_APPLICATION_CLIENT_ID",
    authority: "https://login.microsoftonline.com/TU_TENANT_ID",
    redirectUri: "http://localhost:4200",
  },
};
```

**Error común:** invertir `clientId` y `tenantId` entre sí — revisa que cada UUID esté en su campo correcto.

---

## Paso 4 — Entender la instancia de MSAL (concepto)

MSAL necesita un objeto `PublicClientApplication` configurado una sola vez, antes de que arranque la app. En Angular standalone esto se hace con una **factory function** que se registra como provider global en `main.ts` — el código completo está en el Paso 6b.

---

## Paso 5 — Entender el Interceptor (concepto)

El `MsalInterceptor` agrega automáticamente `Authorization: Bearer <token>` a cada petición HTTP hacia tu API. Se configura indicando **a qué URLs** debe agregarle el token (no quieres que le agregue tu token de Microsoft a una API pública externa). El código completo está en el Paso 6d.

**Verificación (una vez configurado):** en las DevTools del navegador (pestaña Network), vas a ver el header `Authorization: Bearer ...` en las peticiones a tu API, pero NO en peticiones a otros dominios.

---

## Paso 6 — Registrar todo en el bootstrap (`main.ts`)

En un proyecto standalone, todo se registra como `providers` dentro de `bootstrapApplication()`.

### 6a. Importar lo necesario

```ts
import { bootstrapApplication } from "@angular/platform-browser";
import { provideRouter } from "@angular/router";
import { provideHttpClient, withInterceptorsFromDi, HTTP_INTERCEPTORS } from "@angular/common/http";
import {
  MsalGuard,
  MsalInterceptor,
  MsalService,
  MsalBroadcastService,
  MSAL_INSTANCE,
  MSAL_GUARD_CONFIG,
  MSAL_INTERCEPTOR_CONFIG,
  MsalGuardConfiguration,
  MsalInterceptorConfiguration,
} from "@azure/msal-angular";
import { PublicClientApplication, InteractionType } from "@azure/msal-browser";
import { environment } from "./environments/environment.development";
import { AppComponent } from "./app/app.component";
import { routes } from "./app/app.routes";
```

### 6b. Función que construye la instancia de MSAL

```ts
function MSALInstanceFactory(): PublicClientApplication {
  return new PublicClientApplication({
    auth: {
      clientId: environment.msalConfig.clientId,
      authority: environment.msalConfig.authority,
      redirectUri: environment.msalConfig.redirectUri,
    },
    cache: { cacheLocation: "localStorage" },
  });
}
```

### 6c. Función de configuración del Guard

Le dice a `MsalGuard` **cómo** pedir el login si el usuario no está autenticado:

```ts
function MSALGuardConfigFactory(): MsalGuardConfiguration {
  return { interactionType: InteractionType.Redirect };
}
```

### 6d. Función de configuración del Interceptor

Define **a qué URLs** agregar el token, y con qué scopes:

```ts
function MSALInterceptorConfigFactory(): MsalInterceptorConfiguration {
  const protectedResourceMap = new Map<string, Array<string>>();
  protectedResourceMap.set("http://localhost:8080/api", ["api://myapi/read"]);
  return { interactionType: InteractionType.Redirect, protectedResourceMap };
}
```

Cambia la URL por la de tu propio backend cuando lo tengas.

### 6e. Registrar todo en `bootstrapApplication`

```ts
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptorsFromDi()),
    { provide: HTTP_INTERCEPTORS, useClass: MsalInterceptor, multi: true },
    { provide: MSAL_INSTANCE, useFactory: MSALInstanceFactory },
    { provide: MSAL_GUARD_CONFIG, useFactory: MSALGuardConfigFactory },
    { provide: MSAL_INTERCEPTOR_CONFIG, useFactory: MSALInterceptorConfigFactory },
    MsalService,
    MsalGuard,
    MsalBroadcastService,
  ],
}).catch((err) => console.error(err));
```

**Por qué cada provider:**

| Provider               | Para qué sirve                                                 |
| ---------------------- | -------------------------------------------------------------- |
| `MsalGuard`            | protege rutas, exige login antes de entrar                     |
| `MsalInterceptor`      | agrega el token a las peticiones HTTP                          |
| `MsalService`          | permite llamar `login()`/`logout()` desde cualquier componente |
| `MsalBroadcastService` | notifica eventos de MSAL (login completado, error, etc.)       |

### 6f. Verificación

Corre `ng serve`. Si ves este error en consola:

```
NullInjectorError: No provider for MSAL_INSTANCE!
```

Falta uno de los `provide:` del paso 6e — revisa que los 3 (`MSAL_INSTANCE`, `MSAL_GUARD_CONFIG`, `MSAL_INTERCEPTOR_CONFIG`) estén con su `useFactory` correcto.

Si no hay errores y la app carga igual que en el checkpoint del Paso 1d, quedó bien registrado.

---

## Paso 7 — Botones de Login/Logout

En `app.component.ts`, inyecta `MsalService`:

```ts
import { MsalService } from '@azure/msal-angular';

constructor(private msalService: MsalService) {}

login() {
  this.msalService.loginRedirect();
}

logout() {
  this.msalService.logoutRedirect();
}
```

En `app.component.html`:

```html
<button (click)="login()">Iniciar sesión</button> <button (click)="logout()">Cerrar sesión</button>
```

**Recomendación para tu primera vez:** usa `loginRedirect` (redirige a la pantalla de Microsoft y vuelve) en vez de `loginPopup` — tiene menos problemas de bloqueadores de pop-ups y es más fácil de depurar.

---

## Paso 8 — Probar todo

1. Ejecuta `ng serve` (si no está corriendo).
2. Abre `http://localhost:4200`.
3. Haz clic en "Iniciar sesión".
4. Deberías ser redirigido a la pantalla de login de Microsoft Entra ID.
5. Al loguearte, vuelves al frontend ya autenticado.
6. Verifica que puedes acceder a rutas protegidas por `MsalGuard`.

### Troubleshooting

| Síntoma                                             | Causa probable                                                                                                  |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `ng: command not found`                             | El Angular CLI no quedó instalado globalmente, o falta reabrir la terminal                                      |
| Pantalla de bienvenida no carga (Paso 1d)           | No estás en la carpeta del proyecto, o el puerto 4200 está ocupado                                              |
| Pantalla en blanco o error al cargar la app         | `clientId` o `authority` mal copiados en `environment.development.ts`                                           |
| `NullInjectorError: No provider for MSAL_INSTANCE!` | Falta un provider en el Paso 6e                                                                                 |
| "Redirect URI mismatch" en la pantalla de Microsoft | El Redirect URI no coincide exactamente (con/sin `/`) con el registrado en Azure                                |
| El login funciona pero el token no llega al backend | El `protectedResourceMap` del Paso 6d no incluye la URL de tu API                                               |
| Código `AADSTS...` de error                         | Búscalo en la documentación de Microsoft — casi siempre apunta a un dato mal configurado en el App Registration |

---

## Qué sigue

Con el frontend ya obteniendo tokens, el siguiente paso es configurar el **backend** (Spring Security) para que valide esos tokens antes de aceptar cualquier petición.

## Enlaces oficiales

- [Registro de app para SPA](https://learn.microsoft.com/azure/active-directory/develop/scenario-spa-app-registration)
- [MSAL Angular — documentación principal](https://learn.microsoft.com/azure/active-directory/develop/msal-angular)
- [MsalGuard](https://learn.microsoft.com/azure/active-directory/develop/msal-angular#protect-routes-using-the-msalguard)
- [MsalInterceptor](https://learn.microsoft.com/azure/active-directory/develop/msal-angular#protect-web-api-calls-using-the-msalinterceptor)
