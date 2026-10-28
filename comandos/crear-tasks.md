# Crear Tasks

Tercer paso de **Spec-Driven Development (SDD)**. A partir del `design.md` de una feature,
escribí el `tasks.md`: una lista ordenada de tareas concretas, con los archivos reales que toca
cada una, en el formato que ejecuta `implementar`.

## Argumentos

- proyecto: `$1` — siempre `gym-manager`. Si viene otro valor, detenete y preguntá.
- feature: `$2` — slug de la feature; tiene que existir `specs/<feature>/design.md`.

Si tu agente no reemplaza `$1`/`$2`, tomá los argumentos del mensaje del usuario.

---

## Instrucciones

### Fase 0: Ubicarse

1. `DOCS` = raíz de `gym-manager-docs/`. `RAIZ` = su carpeta padre (`gym-manager/`).

### Fase 1: Investigación

1. Leer completos `DOCS/specs/<feature>/requirements.md` y `design.md`.
   - Si falta alguno, detenete y sugerí el comando que corresponde.
   - Si `requirements.md` no está **Aprobada**, detenete.
2. Explorar los repos de código en `RAIZ/` para identificar los **archivos reales** que toca cada
   tarea. Si un archivo todavía no existe, la ruta tiene que seguir la estructura del precedente
   de `DOCS/arquitectura/` y la del repo. No inventes rutas sin verificarlas.

### Fase 2: Escribir tasks.md

Crear `DOCS/specs/<feature>/tasks.md` con **exactamente** esta estructura (la misma que genera
`crear-plan`, para que `implementar` la ejecute sin cambios):

```markdown
# Plan: <Feature>

**Creado:** <AAAA-MM-DD>
**Estado:** Borrador
**Pedido:** Implementación de <feature>, según specs/<feature>/requirements.md y design.md

---

## Descripción General

### Qué Logra Este Plan

<2 o 3 oraciones>

### Por Qué Importa

<Referencia a requirements.md>

---

## Tareas Paso a Paso

### Paso 1: <Título>

- [ ] Completado

<Descripción>

**Repo:** gym-manager-back

**Acciones:**

- <Acción específica>

**Archivos afectados:**

- `<ruta real dentro del repo>`

**Tests:**

- `<archivo de test>` — <criterio(s) EARS que verifica>

**Requirement(s) que cubre:** <N>

---

<Repetir por cada tarea, en orden de dependencia>

---

## Lista de Validación

- [ ] <Verificación derivada de un escenario de requirements.md>
- [ ] Tests pasan
- [ ] contratos/openapi.yaml actualizado, si la feature cambia el contrato
- [ ] specs/<feature>/tasks.md con el estado real

---

## Criterios de Éxito

1. <Derivado de los objetivos verificables de requirements.md>

---

## Notas

<Contexto adicional>
```

Cada tarea:

- Se completa y se revisa de una sola vez (un commit atómico).
- Dice en qué **repo** se hace.
- Indica qué requirement cubre y qué tests la verifican.
- Lista archivos reales, no genéricos.

---

## Reglas

- **Ejecutable:** cualquier integrante, con cualquier agente, puede tomar una tarea y completarla
  sin releer todo el design.
- **Orden:** respetar dependencias (dominio → aplicación → infraestructura → presentación;
  migración antes que repositorio; back antes que los clientes que lo consumen).
- **Trazabilidad completa:** todo requirement queda cubierto por al menos una tarea.
- **Compatibilidad:** no renombrar secciones; `implementar` depende de ellas.

---

## Reporte

1. Ruta del archivo: `DOCS/specs/<feature>/tasks.md`
2. Cantidad de tareas y una línea por tarea
3. Requirements sin tarea, si quedó alguno
4. Cómo ejecutarlo: `implementar specs/<feature>/tasks.md` (desde un repo de código:
   `../gym-manager-docs/specs/<feature>/tasks.md`)
