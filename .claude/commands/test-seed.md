---
description: Analiza una página (con login opcional) y genera tests/seed.spec.ts
argument-hint: <url> [usuario] [contraseña]
allowed-tools: Read, Write, Edit, Bash(npx playwright test:*), mcp__playwright-test__browser_navigate, mcp__playwright-test__browser_snapshot, mcp__playwright-test__browser_type, mcp__playwright-test__browser_fill_form, mcp__playwright-test__browser_click, mcp__playwright-test__browser_press_key, mcp__playwright-test__browser_wait_for, mcp__playwright-test__browser_generate_locator, mcp__playwright-test__browser_close
---

Genera el archivo `tests/seed.spec.ts` para esta aplicación.

## Parámetros

- URL: `$1`
- Usuario: `$2`
- Contraseña: `$3`

Si no se recibió URL, detente y pide al usuario que ejecute `/test-seed <url> [usuario] [contraseña]`.
Si se recibe usuario pero no contraseña (o al revés), detente y pide ambos.

## Pasos

1. **Variables de entorno.** Actualiza `.env` en la raíz (créalo desde `.env.example` si no existe):
   - `BASE_URL` = el origen de la URL (esquema + host + puerto, sin ruta).
   - Si hay credenciales: `APP_USER` y `APP_PASSWORD`.
   - Conserva el resto de variables existentes. Nunca escribas las credenciales en otro archivo que no sea `.env`.

2. **Analizar la página.** Con el navegador del MCP `playwright-test`:
   - Navega a la URL y toma un snapshot.
   - Identifica el propósito de la página y sus elementos principales.
   - Si hay credenciales: localiza los campos de usuario, contraseña y el botón de envío. Haz el login con las credenciales y confirma con un snapshot que entró (cambio de URL, menú de usuario, mensaje de bienvenida, etc.). Anota qué elemento demuestra que el login fue exitoso.
   - Si no hay credenciales: identifica un elemento estable que confirme que la página cargó.
   - Obtén los locators con `browser_generate_locator`. Prioriza `getByRole`, `getByLabel`, `getByPlaceholder` y `getByTestId`; evita selectores CSS frágiles o por posición.
   - Cierra el navegador al terminar.

3. **Escribir `tests/seed.spec.ts`.** Sobrescribe el archivo con esta estructura:

   ```ts
   import { test, expect } from '@playwright/test';

   test.describe('Test group', () => {
     test('seed', async ({ page }) => {
       await page.goto('<ruta relativa a BASE_URL, ej. "/" o "/login">');

       // Solo si hay credenciales:
       await <locator usuario>.fill(process.env.APP_USER!);
       await <locator contraseña>.fill(process.env.APP_PASSWORD!);
       await <locator botón>.click();

       await expect(<locator que confirma el estado inicial>).toBeVisible();
     });
   });
   ```

   - Usa rutas relativas en `page.goto` (la base viene de `BASE_URL`).
   - Las credenciales siempre vía `process.env`, nunca en texto plano.
   - Mantén `test.describe('Test group')` y el nombre `seed`: los agentes de Playwright lo buscan así.

4. **Verificar.** Ejecuta `npx playwright test tests/seed.spec.ts --project=chromium`. Si falla, revisa los locators y vuelve a intentarlo (máximo 3 intentos). Si sigue fallando, muestra el error.

5. **Resumen.** Reporta al usuario:
   - Qué página se analizó y qué estado deja el seed (ej. "dashboard con sesión iniciada").
   - Las variables que se escribieron en `.env` (sin mostrar la contraseña).
   - El resultado de la ejecución del seed.
   - Siguiente paso sugerido: usar el agente `playwright-test-planner` para generar el plan en `specs/`.
