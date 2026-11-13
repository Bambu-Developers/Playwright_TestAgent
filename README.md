# Playwright QA Template

Plantilla reutilizable de pruebas E2E con [Playwright](https://playwright.dev). Los planes y los tests se generan con los agentes de Playwright (planner, generator y healer) desde Claude Code. Sirve para cualquier aplicación web: solo apunta el seed a tu app.

## 🛠️ 1. Requerimientos previos

- **Node.js** LTS (probado con v24.18.0) y **npm**
- **Claude Code**, para usar los agentes de Playwright (`.claude/agents/`)
- Acceso a la aplicación web que vas a probar (URL pública, staging o local)

## 🔧 2. Instalación

```bash
# Instalar dependencias (@playwright/test, @types/node)
npm install

# Descargar los navegadores de Playwright (Chromium, Firefox, WebKit)
npx playwright install --with-deps

# Crear tu archivo de variables de entorno
cp .env.example .env
```

### Variables de entorno (`.env`)

`playwright.config.ts` carga el archivo `.env` con [dotenv](https://github.com/motdotla/dotenv). El `.env` está en `.gitignore`, así que no se sube al repo. Solo se versiona `.env.example`.

| Variable | Uso |
|---|---|
| `BASE_URL` | URL base de la app. Se asigna a `use.baseURL`, así puedes usar `page.goto('/')` |
| `APP_USER` / `APP_PASSWORD` | Credenciales de prueba (opcional), se leen con `process.env.APP_USER` |

Puedes agregar las variables que necesites. Recuerda documentarlas también en `.env.example`.

El servidor MCP `playwright-test` ya está configurado en `.mcp.json`. Claude Code lo levanta solo (`npx playwright run-test-mcp-server`) al abrir el proyecto. Si te pide aprobarlo, acéptalo.

### Adaptar la plantilla a un nuevo proyecto

1. Cambia `name` y `description` en `package.json`.
2. Define `BASE_URL` (y credenciales si aplica) en tu `.env`.
3. Configura el seed (`tests/seed.spec.ts`) con el estado inicial de tu app (ver abajo).
4. Ajusta los navegadores en `projects` de `playwright.config.ts` según lo que necesites cubrir.

## 🚀 3. Uso

El flujo es: **seed → planner → specs → generator → tests → healer**.

### Seed (`tests/seed.spec.ts`)

El seed es el test base que los agentes ejecutan para dejar la página en su estado inicial antes de explorar o generar código. Todo lo que pongas ahí (navegación, login, fixtures) lo heredan los tests generados.

Ejemplo genérico (la URL viene de `BASE_URL` en `.env`; ajusta los selectores a tu app):

```ts
import { test, expect } from '@playwright/test';

test.describe('Test group', () => {
  test('seed', async ({ page }) => {
    await page.goto('/');

    // Opcional: login u otro estado inicial
    // await page.getByLabel('Usuario').fill(process.env.APP_USER!);
    // await page.getByLabel('Contraseña').fill(process.env.APP_PASSWORD!);
    // await page.getByRole('button', { name: 'Iniciar sesión' }).click();
  });
});
```

> Guarda las credenciales en `.env` en lugar de escribirlas en el código.

#### Generar el seed automáticamente: `/test-seed`

El proyecto incluye el comando de Claude Code `/test-seed` (`.claude/commands/test-seed.md`). Analiza la página con el navegador, hace login si le pasas credenciales, escribe `tests/seed.spec.ts` y lo ejecuta para verificarlo.

```text
/test-seed https://tu-app.com/login usuario contraseña   # con login
/test-seed https://tu-app.com                            # sin login
```

Guarda `BASE_URL`, `APP_USER` y `APP_PASSWORD` en `.env`. El seed generado lee las credenciales con `process.env`, así que nunca quedan en el código.

### Agents (`.claude/agents/`)

Desde Claude Code, pídele al agente lo que necesitas en lenguaje natural:

| Agente | Para qué sirve | Ejemplo de prompt |
|---|---|---|
| `playwright-test-planner` | Explora la app a partir del seed y escribe un plan de pruebas en `specs/` | *"Usa el planner para crear un plan de pruebas del módulo de login"* |
| `playwright-test-generator` | Toma un escenario del plan y genera el `.spec.ts` en `tests/` | *"Genera los tests del plan `specs/login.md`"* |
| `playwright-test-healer` | Ejecuta los tests que fallan, los depura y corrige locators o esperas | *"Usa el healer para arreglar los tests que fallan"* |

#### Crear un plan de pruebas: `/test-planner`

El comando `/test-planner` (`.claude/commands/test-planner.md`) orquesta el agente `playwright-test-planner`:

1. Valida que el seed exista y no esté vacío.
2. Revisa los planes que ya hay en `specs/` para no duplicarlos.
3. Delega al agente con tu solicitud y tus restricciones.
4. Verifica que el plan las cumpla.

```text
/test-planner Crea un plan de pruebas del flujo de compra de la tienda: login, agregar productos al carrito,
revisar el carrito y completar el checkout. El seed test está en tests/seed.spec.ts. Limita el plan a
4 escenarios, incluyendo al menos una validación de error en el formulario de checkout.
```

Si no indicas un seed, usa `tests/seed.spec.ts`.

#### Generar los tests desde el plan: `/test-generator`

El comando `/test-generator` (`.claude/commands/test-generator.md`) orquesta el agente `playwright-test-generator`:

1. Lee el plan de `specs/` y extrae cada escenario.
2. Lanza el agente una vez por escenario, en secuencia, porque todos comparten el mismo navegador.
3. Guarda cada test en `tests/<suite>/<escenario>.spec.ts`.
4. Ejecuta los tests generados y te muestra el resultado de cada uno.

```text
/test-generator                                  # usa el único plan en specs/
/test-generator specs/flujo-de-compra.md         # un plan específico
/test-generator specs/flujo-de-compra.md 1.1 1.3 # solo algunos escenarios
```

Si un escenario ya tiene su test, no lo regenera a menos que se lo pidas. Si algún test falla, el siguiente paso es `/test-healer`.

#### Arreglar un test que falla: `/test-healer`

El comando `/test-healer` (`.claude/commands/test-healer.md`) orquesta el agente `playwright-test-healer`:

1. Confirma que el test falla.
2. Delega al agente, que clasifica la causa:
   - **Problema del test** (selector, timing, datos dinámicos): lo corrige hasta que pase.
   - **Bug funcional de la app**: **no modifica la aserción**. Marca el test con `test.skip` y le pone un comentario `// BUG:` con lo esperado y lo real.
3. Revisa el diff para asegurar que no se tocaron aserciones en caso de bug.
4. Vuelve a ejecutar el test y te muestra un resumen, con la explicación del bug lista para reportar.

```text
/test-healer tests/flujo-de-compra/compra-exitosa.spec.ts
/test-healer tests/flujo-de-compra/checkout-error.spec.ts el mensaje de error cambió de texto
```

### Specs (`specs/`)

Aquí viven los planes de pruebas en Markdown que genera el planner: escenarios, pasos y resultados esperados. Son la fuente que el generator convierte en tests. Puedes editarlos a mano antes de generar el código.

## 📋 4. Comandos

```bash
npm test                 # Ejecuta todos los tests (playwright test)
npm run test:ui          # Modo UI visual (playwright test --ui)
npm run test:headed      # Ejecuta los tests con el navegador visible
npm run test:debug       # Ejecuta los tests en modo debug con el Inspector
npm run report           # Abre el último reporte HTML (playwright show-report)
npm run codegen          # Graba acciones en el navegador y genera código
```

Para pasar argumentos extra a Playwright, ponlos después de `--`:

```bash
npm test -- tests/seed.spec.ts           # Un archivo específico
npm test -- --project=chromium           # Un solo navegador
npm test -- -g "login"                   # Filtrar tests por nombre
npm run codegen -- https://tu-app.com    # Abrir codegen directo en una URL
```

Los tests se ejecutan en Chromium, Firefox y WebKit, y guardan screenshots, video y trace de cada test en `test-results/`. El reporte HTML queda en `playwright-report/`.

## 📂 Estructura del proyecto

```text
.
├── .claude/agents/        # Agentes de Playwright: planner, generator, healer
├── .claude/commands/      # Comandos de Claude Code del proyecto (/test-seed, /test-planner, /test-generator, /test-healer)
├── .env.example           # Plantilla de variables de entorno (copiar a .env)
├── .mcp.json              # Servidor MCP playwright-test para Claude Code
├── playwright.config.ts   # Configuración de Playwright (navegadores, reporter, evidencias)
├── specs/                 # Planes de pruebas en Markdown (generados por el planner)
├── tests/                 # Tests de Playwright
│   └── seed.spec.ts       # Test seed: estado inicial para los agentes
└── package.json           # Dependencias y scripts npm
```
