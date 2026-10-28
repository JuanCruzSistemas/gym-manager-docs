# Implementar

Ejecutá un `tasks.md` (generado por `crear-tasks`) o un plan (generado por `crear-plan`): una
tarea por vez, con tests, validando cada paso y dejando el estado actualizado.

## Argumentos

- ruta: `$ARGUMENTS` — ruta al archivo. Ejemplos:
  - `specs/organizacion/tasks.md` (desde `gym-manager-docs/`)
  - `../gym-manager-docs/specs/organizacion/tasks.md` (desde un repo de código)
  - `planes/2026-11-03-corregir-validacion-dni.md`

Si tu agente no reemplaza `$ARGUMENTS`, tomá la ruta del mensaje del usuario. Si no hay ruta,
preguntá.

---

## Instrucciones

### Fase 1: Entender el plan

1. `DOCS` = raíz de `gym-manager-docs/`. `RAIZ` = su carpeta padre, donde están los repos de
   código.
2. **Leer el archivo completo.** No hojearlo.
3. Si es un `tasks.md` de una feature, leer también `requirements.md` y `design.md` de la misma
   carpeta, y verificar que `requirements.md` esté **Aprobada**. Si no lo está, detenete.
4. Leer `DOCS/CLAUDE.md` (o `AGENTS.md`), `DOCS/contexto/convenciones.md` y el precedente de
   `DOCS/arquitectura/` antes de tocar código.
5. Verificar bloqueos: preguntas abiertas, `[PENDIENTE]` sin resolver, decisiones del design
   marcadas "a aprobar". Si hay alguno que afecte la tarea, detenete y preguntá.
6. El **Estado** tiene que ser "Borrador" o "En curso". Si es "Implementado", avisá.

### Fase 2: Ejecutar, una tarea por vez

Para cada `### Paso N` todavía sin tildar, en orden:

1. Ir al **repo** que indica la tarea (`RAIZ/<repo>/`).
2. Leer los archivos afectados antes de modificarlos.
3. Escribir primero o junto con el código los **tests** que indica la tarea, derivados de los
   criterios EARS.
4. Hacer los cambios. Archivos completos, nunca borradores ni `TODO` sin aviso.
5. Correr los tests y el lint del repo. Si fallan, corregir antes de seguir.
6. Tildar `- [x] Completado` en la tarea y cambiar el Estado del plan a **En curso**.
7. Hacer **un commit atómico en español** en ese repo, que cite la feature y el paso
   (ej. `feat(organizacion): alta de servicio por sucursal — paso 3`). No hagas push salvo que
   el equipo lo pida.
8. Mostrar un resumen corto de la tarea y **esperar confirmación** antes de pasar a la
   siguiente, salvo que el usuario haya pedido ejecutar todas de corrido.

Si una tarea no se puede hacer como está escrita:

- Si la intención es clara, adaptate y anotá la desviación.
- Si no lo es, o la adaptación cambia una regla de negocio o una decisión de diseño, detenete y
  preguntá. Nunca resuelvas una contradicción en silencio.

### Fase 3: Validar

1. Recorrer la **Lista de Validación** y marcar cada ítem.
2. Verificar los **Criterios de Éxito**.
3. Si la feature cambió el contrato, verificar que `DOCS/contratos/openapi.yaml` esté al día.
4. Verificar que se respetaron las convenciones de nombres y la arquitectura.

### Fase 4: Cerrar

Cuando todas las tareas estén hechas, cambiar `**Estado:**` a **Implementado** y agregar al final:

```markdown
---

## Notas de Implementación

**Implementado:** <AAAA-MM-DD>

### Resumen

<Qué se hizo>

### Commits

- `<repo>` <hash corto> — <mensaje>

### Desviaciones del Plan

<Cambios respecto del plan, o "Ninguna">

### Problemas Encontrados

<Problemas y cómo se resolvieron, o "Ninguno">
```

Los cambios en `DOCS` (estado del plan) se commitean en `gym-manager-docs`.

---

## Reporte

```
## Implementación

### Resumen
- <qué se hizo>

### Archivos
**Creados:** ...
**Modificados:** ...

### Tests
- <suite>: <resultado>

### Validación
- [x] <ítem>

### Desviaciones
<Ninguna, o la lista>

### Siguiente paso
<PR en <repo>, linkeando specs/<feature>/>
```
