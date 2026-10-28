# AGENTS.md — gym-manager-docs

Reglas para **cualquier agente de IA** que trabaje en GymManager. Claude Code las lee a través de
`CLAUDE.md`, que importa este archivo; el resto de los agentes leen este archivo directo. Son las
mismas reglas para todos (Metodología §2.5, D-29).

## Qué es este repo

La fuente de verdad del negocio y de las specs de **GymManager** (Seminario Integrador, UTN FRVM,
Grupo 22): contexto del dominio, specs por feature, contrato OpenAPI, diagramas, precedente de
arquitectura y comandos de Spec-Driven Development (SDD). No tiene código de la aplicación.

Los repos de código se clonan **al lado** de este, en la misma carpeta padre:

```
gym-manager/                  ← RAIZ
├── gym-manager-back/         API NestJS (TypeScript) + Prisma + PostgreSQL
├── gym-manager-desktop/      C# .NET WPF + SQLite (offline-first)
├── gym-manager-mobile/       Flutter (Android e iOS)
└── gym-manager-docs/         ← DOCS (este repo)
```

## Estructura

| Carpeta | Contenido | Quién la escribe |
|---|---|---|
| `documentos/` | Los cinco documentos del proyecto (.docx): Requerimientos, Casos de Uso 1–3, Diseño | El equipo |
| `contexto/` | Contexto del negocio en Markdown, generado desde `documentos/` (D-50) | El equipo (con ayuda de IA) |
| `specs/<feature>/` | `requirements.md`, `design.md`, `tasks.md` | Los comandos de SDD |
| `planes/` | Planes de cambios chicos o técnicos | `crear-plan` |
| `contratos/openapi.yaml` | Contrato REST compartido por back, desktop y mobile | Se actualiza con cada design |
| `arquitectura/` | Precedente DDD + hexagonal: `ARCHITECTURE.md`, `IMPLEMENTATION.md`, `DECISIONS.md` (D-49) | El equipo |
| `diagramas/` | `.drawio`: clases, transición de estados, componentes, despliegue y DER (D-06) | El equipo |
| `comandos/` | Instrucciones de los comandos, válidas para cualquier agente | El equipo |
| `.claude/commands/` | Atajos de Claude Code que apuntan a `comandos/` | El equipo |

Archivos de `contexto/`:

| Archivo | Sale de |
|---|---|
| `glosario.md` | Requerimientos §5 |
| `dominio.md` | Requerimientos §1 + Diseño §4 y §5 |
| `business-rules.md` | Requerimientos §2 (RN y notificaciones) |
| `permisos.md` | Requerimientos §6 y §7 |
| `convenciones.md` | Diseño §1, §2 y §4 + reglas de código |
| `casos-de-uso.md` | Requerimientos §4 + Casos de Uso partes 1 a 3 |
| `alcance.md` | Requerimientos §8 (iteraciones y features) |

Si `contexto/` y `documentos/` no coinciden, **manda el documento**: avisá la diferencia y no
la resuelvas por tu cuenta (D-50).

## Reglas

1. **Antes de cualquier tarea**, leer `contexto/` completo, la spec de la feature y, si la tarea
   toca código, el precedente de `arquitectura/`.
2. **No implementar nada** que no esté en una spec con `Estado: Aprobada` (o en un plan de
   `planes/` para cambios chicos o técnicos).
3. **Usar solo los términos del glosario.** En código: identificadores en español; carpetas y
   archivos propios del framework en inglés (`services/`, `dto/`, `entities/`…).
4. **Si falta información, preguntar o marcar `[PENDIENTE]`.** Nunca inventar reglas de negocio.
5. **Reportar contradicciones** entre specs, glosario, dominio, reglas y documentos en vez de
   resolverlas en silencio.
6. **Respetar la arquitectura** del Diseño §1 a §4 y del precedente de `arquitectura/`:
   - monolito modular, hexagonal, con DDD; REST con JSON; contrato en `contratos/openapi.yaml`;
   - toda consulta filtrada por gimnasio (RNF 13);
   - referencia directa dentro de un módulo y por identificador entre módulos, sin listas de
     identificadores (D-07);
   - orquestación entre módulos en `processes/`, nunca una fachada que llama a otra;
   - contraseñas y códigos siempre hasheados; nunca datos de tarjeta (RNF 5);
   - el historial médico no va en notificaciones, PDF ni logs (RNF 15).
7. **Una tarea de `tasks.md` por vez, con tests.** Commits atómicos y en español.
8. **Hablar con el equipo en español rioplatense**, directo y conciso.

## Flujo SDD

| # | Paso |
|---|---|
| 1 | Si la feature necesita reglas o términos nuevos, se agregan primero en `contexto/`. |
| 2 | `crear-requirements gym-manager <feature>` → historias y criterios EARS. El agente devuelve preguntas abiertas; el equipo responde. |
| 3 | El equipo revisa y cambia `Estado: Aprobada`. Nada se implementa sin esto. |
| 4 | `crear-design gym-manager <feature>` → módulos, componentes, contrato, errores, testing. El equipo revisa las decisiones técnicas. |
| 5 | `crear-tasks gym-manager <feature>` → checklist ejecutable con archivos reales. |
| 6 | `implementar specs/<feature>/tasks.md` → una tarea por vez, con tests derivados de los criterios. |
| 7 | PR en el repo de código; el equipo valida contra los criterios y mergea. El PR linkea la spec. |

Para cambios chicos o técnicos (bugfix, refactor): `crear-plan <pedido>` → `implementar
planes/<archivo>.md`.

## Cómo usar los comandos

Las instrucciones están en `comandos/` (ver `comandos/README.md`). Con cualquier agente:

```
Leé comandos/<comando>.md y seguilo al pie de la letra.
Argumentos: <argumentos>
```

El resultado tiene que quedar con el mismo formato y en la misma ruta que si lo hubiera generado
otro agente.

## Formato de los criterios: EARS

| Patrón | Forma |
|---|---|
| Evento | `WHEN [algo pasa] THEN THE SYSTEM SHALL [...]` |
| No deseado | `IF [condición de error] THEN THE SYSTEM SHALL [...]` |
| Estado | `WHILE [estado] THE SYSTEM SHALL [...]` |
| Ubicuo | `THE SYSTEM SHALL [...]` |

Palabras clave en inglés, contenido en castellano. Prohibido: "rápido", "fácil", "etc.",
"según corresponda". Cada criterio tiene que poder convertirse en un test.

## Git en este repo

- Commits atómicos y en español (`docs(contexto): agrega RN-29`, `spec(organizacion): aprobada`).
- Los cambios de `contexto/` y el paso de una spec a **Aprobada** se hacen por PR, revisado por
  el equipo.
- No se suben secretos, tokens ni datos reales de personas.
