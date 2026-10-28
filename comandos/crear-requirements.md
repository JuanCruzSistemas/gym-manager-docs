# Crear Requirements

Primer paso de **Spec-Driven Development (SDD)**. A partir del contexto de negocio de GymManager,
escribí un `requirements.md` con historias de usuario y criterios de aceptación en formato
**EARS**, trazables a casos de uso (CU) y reglas de negocio (RN) concretas.

La spec dice **qué y por qué**, nunca **cómo**. Prueba: si mañana cambia el framework, este
archivo no debería cambiar.

## Argumentos

- proyecto: `$1` — siempre `gym-manager`. Si viene otro valor, detenete y preguntá.
- feature: `$2` — slug en kebab-case de la feature (ver Metodología §2.7, por ejemplo
  `organizacion`, `membresias-metodos-pago`). Si no viene, preguntá cuál.

Si tu agente no reemplaza `$1`/`$2`, tomá los argumentos del mensaje del usuario.

---

## Instrucciones

### Fase 0: Ubicarse

1. `DOCS` = raíz de `gym-manager-docs/` (si estás en un repo de código, `../gym-manager-docs/`).
2. Todo lo que sigue se lee y se escribe dentro de `DOCS`.

### Fase 1: Cargar el contexto de negocio

1. Leer **completo** `DOCS/contexto/`:
   - `glosario.md` — términos oficiales (los únicos que se usan).
   - `dominio.md` — entidades, agregados, ciclos de vida.
   - `business-rules.md` — reglas de negocio numeradas (RN-xx) y notificaciones.
   - `permisos.md` — roles y matriz de permisos.
   - `casos-de-uso.md` — los CU con sus flujos principales y alternativos.
   - `alcance.md` — iteraciones y features.
   - `convenciones.md` — solo para no contradecir decisiones ya tomadas; no se vuelca nada
     técnico en la spec.
2. Si `contexto/` está vacío o falta algún archivo, avisá qué falta y preguntá si seguir igual.
   No completes lo que falta por tu cuenta.
3. Buscar en `alcance.md` (o en la Metodología §2.7) qué CU cubre la feature. Si la feature no
   existe ahí, detenete y preguntá.
4. Si `DOCS/specs/` ya tiene otras features, revisarlas para no pisar términos ni contradecir
   reglas de otra spec.

### Fase 2: Análisis de dominio (razonamiento previo, no va al archivo)

1. **Visión:** ¿qué problema resuelve la feature y con qué objetivos verificables?
2. **Actores:** ¿qué roles inician cada acción? (Administrador, Recepcionista, Profesor, Cliente,
   Solicitante, Sistema, Mercado Pago…). Respetar `permisos.md`.
3. **Lenguaje ubicuo:** ¿aparecen términos que no están en `glosario.md`? Proponelos en el
   reporte; **no** los agregues al glosario sin confirmación del equipo.
4. **Eventos de dominio:** hechos importantes, en pasado ("Cobro aprobado").
5. **Invariantes:** ¿qué tiene que ser siempre cierto? Citá la RN cuando exista.
6. **Casos de uso:** separá comandos (cambian estado) de consultas (solo leen).
7. **Escenarios:** al menos un caso feliz y uno por cada flujo alternativo del CU.

### Fase 3: Escribir requirements.md

Crear (o sobrescribir, si está en Borrador) `DOCS/specs/<feature>/requirements.md`:

```markdown
# Requirements: <Feature>

**Proyecto:** gym-manager
**Feature:** <feature>
**Iteración:** <n, según alcance.md>
**Creado:** <AAAA-MM-DD>
**Estado:** Borrador

## Introducción

<2 a 4 líneas: qué cubre la feature y su alcance. Qué queda afuera, si hace falta aclararlo.>

## Glosario

<Solo términos de esta feature que ayuden a leerla. Si ya están en contexto/glosario.md, se
citan, no se redefinen.>

## Requirements

### Requirement 1: <nombre>

**Historia de usuario:** Como <rol>, quiero <acción>, para <beneficio>.

#### Criterios de aceptación

1. WHEN <algo pasa> THEN THE SYSTEM SHALL <comportamiento>.
2. IF <condición de error> THEN THE SYSTEM SHALL <comportamiento>.
3. WHILE <estado> THE SYSTEM SHALL <comportamiento>.
4. THE SYSTEM SHALL <regla que vale siempre>.

<Repetir por cada requirement>

## Trazabilidad

| Requirement | CU | RN / RNF | Escenarios que cubre |
|---|---|---|---|
| 1 | C.U. 3 | RN-01 | Alta feliz, nombre duplicado |

## Preguntas abiertas

- [PENDIENTE] <pregunta concreta para el equipo>
```

Patrones EARS (palabras clave en inglés, contenido en castellano):

| Patrón | Forma | Para |
|---|---|---|
| Evento | `WHEN [algo pasa] THEN THE SYSTEM SHALL [...]` | Reacción a una acción |
| No deseado | `IF [condición de error] THEN THE SYSTEM SHALL [...]` | Errores, invariantes rotos |
| Estado | `WHILE [estado] THE SYSTEM SHALL [...]` | Comportamiento durante algo |
| Ubicuo | `THE SYSTEM SHALL [...]` | Reglas que valen siempre |

---

## Reglas de redacción

- **Solo negocio:** nada de tablas, endpoints, librerías ni nombres de clases.
- Los **flujos alternativos** del CU se vuelven escenarios y criterios `IF`.
- Lo transversal **no se repite, se cita** ("ver RN-12", "según la matriz de permisos").
- Lo que no se sabe se marca `[PENDIENTE]` con una pregunta concreta. No lo resuelvas solo.
- **Prohibido** en criterios: "rápido", "fácil", "etc.", "según corresponda", "adecuadamente".
- Cada criterio tiene que poder convertirse en un test sin interpretación.
- Una feature por spec. Si pasa de ~2 páginas, proponé partirla en dos.
- Los CRUD se agrupan en una sola spec.
- **Estados:** Borrador → En revisión → Aprobada. Este comando solo escribe **Borrador**; el
  equipo es quien la pasa a Aprobada. Si la spec ya está **Aprobada**, no la sobrescribas:
  avisá y preguntá.
- Todo dato del gimnasio vale **solo dentro de su gimnasio** (RNF 13); no hace falta repetirlo
  en cada criterio, pero tampoco escribir criterios que lo contradigan.

---

## Reporte

Al terminar, respondé con:

1. Ruta del archivo: `DOCS/specs/<feature>/requirements.md`
2. Requirements creados (uno por línea)
3. CU o flujos alternativos que **no** quedaron cubiertos, y por qué
4. Términos nuevos propuestos para el glosario
5. Preguntas abiertas (`[PENDIENTE]`) que tiene que responder el equipo
6. Siguiente paso: que el equipo la revise y la pase a **Aprobada**; después,
   `crear-design gym-manager <feature>`
