---
description: Orquesta el agente playwright-test-generator para generar los tests a partir de un plan en specs/
argument-hint: [specs/plan.md] [escenarios a generar, ej. "1.1, 1.3" o "todos"]
allowed-tools: Agent, SendMessage, Read, Glob, Bash(npx playwright test:*)
---

Genera los tests de Playwright a partir de un plan de pruebas con el agente `playwright-test-generator`.

## Solicitud del usuario

$ARGUMENTS

## Pasos

1. **Elegir el plan.**
   - Si la solicitud indica un archivo de `specs/`, úsalo.
   - Si no, lista los `.md` de `specs/` (ignora `specs/README.md`):
     - Si hay uno solo, úsalo.
     - Si hay varios, detente y pregunta cuál usar.
     - Si no hay ninguno, detente y sugiere ejecutar primero `/test-planner <descripción del flujo>`.

2. **Leer el plan y extraer los escenarios.** Para cada escenario obtén:
   - Suite: el título del grupo de nivel superior, sin numeración (ej. `### 1. Flujo de compra` → `Flujo de compra`).
   - Nombre: el título del escenario, sin numeración (ej. `#### 1.1 Compra exitosa` → `Compra exitosa`).
   - Seed: el indicado en el plan (`**Seed:**`); si no hay, `tests/seed.spec.ts`.
   - Cuerpo: precondiciones, pasos y resultados esperados, copiados tal cual del plan.
   - Archivo destino: `tests/<suite-en-kebab-case>/<nombre-en-kebab-case>.spec.ts`.

   Si la solicitud limita los escenarios (por número o nombre), quédate solo con esos. Si un escenario ya
   tiene archivo en `tests/`, no lo regeneres salvo que la solicitud lo pida; anótalo como omitido.

   Verifica que el seed exista y no esté vacío; si lo está, detente y sugiere `/test-seed`.

3. **Delegar al generator, un escenario a la vez.** Lanza el agente `playwright-test-generator`
   (Agent con `subagent_type: "playwright-test-generator"`) **en secuencia, nunca en paralelo**: todos
   comparten el mismo navegador del MCP `playwright-test`. Usa este prompt por escenario:

   ```
   <test-suite>{suite}</test-suite>
   <test-name>{nombre}</test-name>
   <test-file>{archivo destino}</test-file>
   <seed-file>{seed}</seed-file>
   <body>{cuerpo del escenario}</body>
   ```

   Agrega al prompt: leer las credenciales con `process.env` (nunca en texto plano) y usar rutas
   relativas en `page.goto` (la base viene de `BASE_URL`).

4. **Ejecutar los tests generados.** Corre
   `npx playwright test <archivos generados> --project=chromium --reporter=list` y registra cuáles pasan y
   cuáles fallan. No intentes corregir los que fallan aquí.

5. **Resumen.** Reporta al usuario:
   - Plan usado.
   - Tabla con: escenario, archivo generado y resultado (✅ pasa / ❌ falla / ⏭️ omitido).
   - Escenarios que el agente no pudo generar y por qué.
   - Siguiente paso sugerido: si hay fallos, usar el agente `playwright-test-healer` para corregirlos; si todo
     pasa, correr la suite completa con `npm test` y revisar el reporte con `npm run report`.
