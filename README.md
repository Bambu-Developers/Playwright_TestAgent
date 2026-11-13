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
```

El servidor MCP `playwright-test` ya está configurado en `.mcp.json`. Claude Code lo levanta solo (`npx playwright run-test-mcp-server`) al abrir el proyecto. Si te pide aprobarlo, acéptalo.

### Adaptar la plantilla a un nuevo proyecto

1. Cambia `name` y `description` en `package.json`.
2. Configura el seed (`tests/seed.spec.ts`) con la URL y el estado inicial de tu app (ver abajo).
3. Opcional: define `baseURL` en `playwright.config.ts` (`use.baseURL`) para usar rutas relativas como `page.goto('/')`.
4. Ajusta los navegadores en `projects` de `playwright.config.ts` según lo que necesites cubrir.

## 🚀 3. Uso

El flujo es: **seed → planner → specs → generator → tests → healer**.

### Seed (`tests/seed.spec.ts`)

El seed es el test base que los agentes ejecutan para dejar la página en su estado inicial antes de explorar o generar código. Todo lo que pongas ahí (navegación, login, fixtures) lo heredan los tests generados.

Ejemplo genérico (reemplaza la URL, los selectores y las credenciales por los de tu app):

```ts
import { test, expect } from '@playwright/test';

test.describe('Test group', () => {
  test('seed', async ({ page }) => {
    await page.goto('https://tu-app.com');

    // Opcional: login u otro estado inicial
    // await page.getByLabel('Usuario').fill(process.env.APP_USER!);
    // await page.getByLabel('Contraseña').fill(process.env.APP_PASSWORD!);
    // await page.getByRole('button', { name: 'Iniciar sesión' }).click();
  });
});
```

> Usa variables de entorno para las credenciales en lugar de escribirlas en el código.

### Agents (`.claude/agents/`)

Desde Claude Code, pídele al agente lo que necesitas en lenguaje natural:

| Agente | Para qué sirve | Ejemplo de prompt |
|---|---|---|
| `playwright-test-planner` | Explora la app a partir del seed y escribe un plan de pruebas en `specs/` | *"Usa el planner para crear un plan de pruebas del módulo de login"* |
| `playwright-test-generator` | Toma un escenario del plan y genera el `.spec.ts` en `tests/` | *"Genera los tests del plan `specs/login.md`"* |
| `playwright-test-healer` | Ejecuta los tests que fallan, los depura y corrige locators o esperas | *"Usa el healer para arreglar los tests que fallan"* |

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
├── .mcp.json              # Servidor MCP playwright-test para Claude Code
├── playwright.config.ts   # Configuración de Playwright (navegadores, reporter, evidencias)
├── specs/                 # Planes de pruebas en Markdown (generados por el planner)
├── tests/                 # Tests de Playwright
│   └── seed.spec.ts       # Test seed: estado inicial para los agentes
└── package.json           # Dependencias y scripts npm
```
