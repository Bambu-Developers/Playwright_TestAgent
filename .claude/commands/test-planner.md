---
description: Orquesta el agente playwright-test-planner para crear un plan de pruebas en specs/
argument-hint: <descripción del plan: flujo, alcance, número de escenarios, seed>
allowed-tools: Agent, SendMessage, Read, Glob
---

Crea un plan de pruebas con el agente `playwright-test-planner`.

## Solicitud del usuario

$ARGUMENTS

## Pasos

1. **Validar la solicitud.** Si la solicitud está vacía, detente y pide al usuario que ejecute
   `/test-planner <descripción del flujo a probar>`.

2. **Validar el seed.**
   - Usa el seed que mencione la solicitud; si no menciona ninguno, usa `tests/seed.spec.ts`.
   - Lee el archivo. Si no existe o el test `seed` está vacío (sin `page.goto`), detente y sugiere
     ejecutar primero `/test-seed <url> [usuario] [contraseña]`.

3. **Revisar planes existentes.** Lista los archivos de `specs/` para no duplicar un plan que ya existe.
   Si hay uno que cubre el mismo flujo, menciónalo en el prompt del agente para que lo tome en cuenta.

4. **Delegar al planner.** Lanza el agente `playwright-test-planner` (Agent con
   `subagent_type: "playwright-test-planner"`) con un prompt que incluya:
   - La solicitud del usuario tal cual, sin recortarla ni reinterpretarla.
   - La ruta del seed validado.
   - Las restricciones explícitas de la solicitud (número de escenarios, casos de error, flujos incluidos o
     excluidos), destacadas para que el agente las respete.
   - Que guarde el plan en `specs/` como Markdown, con un nombre en kebab-case que describa el flujo
     (ej. `specs/flujo-de-compra.md`).
   - Que cada escenario tenga: título, precondiciones, pasos numerados y resultados esperados.

5. **Verificar el resultado.** Cuando el agente termine, lee el plan generado y comprueba que cumple
   las restricciones de la solicitud (ej. número de escenarios, al menos un caso de error). Si no las
   cumple, indica qué falta y retoma el agente con SendMessage para que lo corrija (máximo 1 intento).

6. **Resumen.** Reporta al usuario:
   - Ruta del plan generado.
   - Lista de escenarios (un renglón por escenario).
   - Cualquier restricción que no se haya podido cumplir.
   - Siguiente paso sugerido: generar los tests con el agente `playwright-test-generator` a partir del plan.
