---
description: Orquesta el agente playwright-test-healer para arreglar un test que falla; si es un bug de la app, lo marca como skipped
argument-hint: <tests/archivo.spec.ts> [contexto adicional]
allowed-tools: Agent, SendMessage, Read, Glob, Bash(npx playwright test:*), Bash(git diff:*)
---

Arregla el test que falla con el agente `playwright-test-healer`.

## Solicitud del usuario

$ARGUMENTS

## Pasos

1. **Validar el archivo.**
   - Si la solicitud no indica un archivo, detente y pide ejecutar `/test-healer tests/<archivo>.spec.ts`.
   - Si el archivo no existe, busca coincidencias en `tests/**/*.spec.ts` y sugiérelas.

2. **Confirmar la falla.** Ejecuta `npx playwright test <archivo> --project=chromium --reporter=list`.
   - Si todo pasa, detente e informa que no hay nada que arreglar.
   - Si falla, guarda el error (mensaje, línea y aserción) para el resumen y para el agente.
   - Lee el archivo y conserva su contenido original para compararlo después.

3. **Delegar al healer.** Lanza el agente `playwright-test-healer` (Agent con
   `subagent_type: "playwright-test-healer"`) con un prompt que incluya:
   - El archivo a arreglar (solo ese archivo; no toques otros tests) y el error del paso 2.
   - El contexto adicional de la solicitud, si lo hay.
   - Estas reglas, que **sustituyen** sus reglas por defecto sobre aserciones y `test.fixme()`:

     > Antes de editar, clasifica la causa de la falla:
     >
     > - **Problema del test** (selector desactualizado, timing/sincronización, datos dinámicos, paso mal
     >   escrito): corrige el test hasta que pase.
     > - **Bug funcional de la app** (la app no cumple el resultado esperado del plan: un mensaje de error
     >   que no aparece, un total mal calculado, un botón que no hace nada, etc.): **no modifiques la
     >   aserción ni el valor esperado**. Cambia `test(` por `test.skip(` en ese test y agrega justo
     >   antes un comentario con este formato:
     >
     >   ```ts
     >   // BUG: <qué esperaba el test> — <qué hace la app en realidad>.
     >   // Detectado por /test-healer el <fecha>. Quitar test.skip cuando se corrija.
     >   ```
     >
     > Si no estás seguro, trátalo como bug funcional: es preferible dejar un test skipped bien explicado
     > a ocultar un bug cambiando la aserción.
     >
     > Al terminar, responde con: clasificación (problema del test / bug funcional), causa raíz y los
     > cambios que hiciste.

4. **Verificar.**
   - Revisa el diff con `git diff <archivo>` (si el archivo no está en git, compáralo con el contenido que
     leíste antes de delegar). Si el agente clasificó la falla como bug funcional pero
     modificó alguna línea de `expect(...)` o un valor esperado, pídele con SendMessage que revierta ese
     cambio y aplique solo el `test.skip` con su comentario.
   - Vuelve a ejecutar `npx playwright test <archivo> --project=chromium --reporter=list`. El resultado
     esperado es que pase o que quede como skipped; si sigue fallando, retoma el agente con SendMessage
     (máximo 1 intento más).

5. **Resumen.** Reporta al usuario:
   - Archivo y error original.
   - Clasificación: **problema del test** o **bug funcional de la app**.
   - Causa raíz en una o dos frases.
   - Cambios aplicados (resumen del diff).
   - Resultado final: ✅ pasa / ⏭️ skipped por bug / ❌ sigue fallando.
   - Si es un bug: explicación de por qué se considera bug de la app (esperado vs. real), lista para
     copiar en un reporte de bug.
