# Crear Plan

Plan de implementación para **cambios chicos o técnicos** que no ameritan SDD: un bugfix, un
refactor, una mejora de tooling, un ajuste de configuración. Si el pedido agrega o cambia reglas
de negocio, no va por acá: va por `crear-requirements`.

**Estás creando un PLAN, no implementando.** Investigá, pensá y escribí el plan completo; lo
ejecuta después `implementar`.

## Argumentos

- pedido: `$ARGUMENTS` — qué hay que planificar.

Si tu agente no reemplaza `$ARGUMENTS`, tomá el pedido del mensaje del usuario.

---

## Instrucciones

### Fase 0: ¿Corresponde un plan?

Si el pedido introduce comportamiento nuevo visible para algún rol, cambia una regla de negocio o
un permiso, detenete y sugerí el flujo SDD (`crear-requirements gym-manager <feature>`).

### Fase 1: Investigación

1. `DOCS` = raíz de `gym-manager-docs/`. `RAIZ` = su carpeta padre, con los repos de código.
2. Leer `DOCS/CLAUDE.md` (o `AGENTS.md`), `DOCS/contexto/convenciones.md` y el precedente de
   `DOCS/arquitectura/`.
3. Explorar el repo y los archivos que toca el cambio. Entender qué depende de ellos.
4. Si existe una spec relacionada en `DOCS/specs/`, leerla para no contradecirla.

### Fase 2: Escribir el plan

Crear `DOCS/planes/AAAA-MM-DD-<nombre-en-kebab-case>.md` con la fecha de hoy y **esta
estructura**, la misma que usa `tasks.md`:

```markdown
# Plan: <título>

**Creado:** <AAAA-MM-DD>
**Estado:** Borrador
**Pedido:** <una línea>

---

## Descripción General

### Qué Logra Este Plan

<2 o 3 oraciones>

### Por Qué Importa

<Qué problema resuelve>

---

## Estado Actual

<Archivos y comportamiento actual relevantes; qué está mal o qué falta>

---

## Decisiones

1. **<Decisión>**: <justificación>. Alternativas: <...>.

### Preguntas abiertas

- [PENDIENTE] <si hay>

---

## Tareas Paso a Paso

### Paso 1: <Título>

- [ ] Completado

<Descripción>

**Repo:** <gym-manager-back | gym-manager-desktop | gym-manager-mobile | gym-manager-docs>

**Acciones:**

- <Acción específica>

**Archivos afectados:**

- `<ruta real>`

**Tests:**

- `<archivo de test>` — <qué verifica>

---

<Repetir por cada tarea>

---

## Lista de Validación

- [ ] <Verificación>
- [ ] Tests pasan

---

## Criterios de Éxito

1. <Criterio medible>

---

## Notas

<Contexto adicional>
```

---

## Reglas

- Completo: ninguna sección con texto de marcador.
- Ejecutable: `implementar` tiene que poder seguirlo sin preguntar.
- Respeta las convenciones y la arquitectura de `DOCS/arquitectura/`.

---

## Reporte

1. Resumen del plan
2. Preguntas abiertas
3. Ruta: `DOCS/planes/AAAA-MM-DD-<nombre>.md`
4. Siguiente paso: `implementar planes/AAAA-MM-DD-<nombre>.md`
