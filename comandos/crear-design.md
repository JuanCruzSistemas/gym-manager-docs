# Crear Design

Segundo paso de **Spec-Driven Development (SDD)**. A partir de un `requirements.md` **Aprobado**,
escribí el `design.md` técnico de la feature: módulos y capas, componentes, modelo, contrato,
errores y estrategia de testing.

## Argumentos

- proyecto: `$1` — siempre `gym-manager`. Si viene otro valor, detenete y preguntá.
- feature: `$2` — slug de la feature; tiene que existir `specs/<feature>/requirements.md`.

Si tu agente no reemplaza `$1`/`$2`, tomá los argumentos del mensaje del usuario.

---

## Instrucciones

### Fase 0: Ubicarse

1. `DOCS` = raíz de `gym-manager-docs/`. `RAIZ` = su carpeta padre (`gym-manager/`), donde están
   `gym-manager-back/`, `gym-manager-desktop/` y `gym-manager-mobile/`.

### Fase 1: Investigación

1. Leer `DOCS/specs/<feature>/requirements.md` completo.
   - Si no existe, detenete y sugerí `crear-requirements gym-manager <feature>`.
   - Si su **Estado** no es **Aprobada**, detenete: no se diseña sobre una spec sin aprobar.
2. Leer `DOCS/contexto/` completo, con foco en `convenciones.md` (stack, reglas de código,
   decisiones) y `dominio.md` (entidades, agregados, referencias).
3. Leer el precedente de arquitectura en `DOCS/arquitectura/`:
   - `ARCHITECTURE.md` — capas, niveles de complejidad por módulo, fachadas, `processes/`.
   - `IMPLEMENTATION.md` — cómo se escribe cada pieza (casos de uso, `create()` /
     `reconstitute()`, value objects, mappers, unidad de trabajo).
   - `DECISIONS.md` — por qué se eligió cada patrón.
   Decidí el **nivel de complejidad** de cada módulo que toca la feature según ese precedente.
4. Leer `DOCS/contratos/openapi.yaml`: esquemas de seguridad, componentes comunes y endpoints ya
   definidos, para reusarlos y no duplicarlos.
5. Explorar el código existente en `RAIZ/gym-manager-back/` (y en desktop o mobile si la feature
   los toca) para encajar con lo que ya está construido: módulos, nombres, patrones.
6. Revisar los `design.md` de otras features en `DOCS/specs/` para mantener la consistencia.

### Fase 2: Escribir design.md

Crear `DOCS/specs/<feature>/design.md`:

```markdown
# Design: <Feature>

**Proyecto:** gym-manager
**Feature:** <feature>
**Creado:** <AAAA-MM-DD>
**Estado:** Borrador
**Basado en:** specs/<feature>/requirements.md

## Resumen

<2 o 3 oraciones con la solución técnica>

## Repos y módulos afectados

| Repo | Módulo / capa | Nivel de complejidad | Qué cambia |
|---|---|---|---|
| gym-manager-back | Organizacion | <según el precedente> | <...> |

## Componentes

| Componente | Capa | Responsabilidad | Requirement(s) |
|---|---|---|---|
| <nombre en español> | dominio / aplicación / infraestructura / presentación | <qué hace> | 1, 2 |

## Modelo

<Entidades, value objects y relaciones que toca la feature, respetando dominio.md y el modelo de
datos (DER). Referencia directa dentro de un módulo, por identificador entre módulos.>

## Flujo

<Cómo viaja un comando o consulta típico por las capas. Si cruza módulos, por processes/.>

## Contrato (OpenAPI)

<Endpoints nuevos o modificados: método, ruta, roles que lo usan, request, respuestas y errores.
Todo lo que se agregue acá se vuelca después en contratos/openapi.yaml.>

## Manejo de errores

| Criterio IF | Error de dominio | Respuesta HTTP |
|---|---|---|
| Req 1, crit. 2 | <...> | 409 |

## Estrategia de testing

| Escenario de requirements.md | Tipo de test | Qué verifica |
|---|---|---|
| <escenario> | unitario / integración / e2e | <...> |

## Decisiones de diseño

1. **<Decisión>**: <justificación>. Alternativas: <...>.

## Trazabilidad

| Requirement | Componente(s) / decisión |
|---|---|
| 1 | <...> |
```

---

## Reglas

- **Cobertura:** todo requirement aparece en la tabla de trazabilidad.
- **Arquitectura fija:** monolito modular hexagonal con DDD, REST con JSON, contrato en OpenAPI,
  toda consulta filtrada por gimnasio (RNF 13), referencia directa dentro de un módulo y por
  identificador entre módulos, orquestación entre módulos en `processes/` (nunca una fachada que
  llama a otra). No se cambia sin decisión explícita del equipo.
- **Seguridad:** contraseñas y códigos siempre hasheados; nunca datos de tarjeta; el historial
  médico no va en notificaciones, PDF ni logs (RNF 15).
- **Nombres:** identificadores en español con los términos del glosario; carpetas propias del
  framework en inglés.
- **Tecnología nueva:** si hace falta una librería o patrón que no está en `convenciones.md`,
  marcalo como decisión **a aprobar**, no lo des por hecho.
- **Sin ambigüedad:** cada componente tiene una responsabilidad clara y no solapada.

---

## Reporte

1. Ruta del archivo: `DOCS/specs/<feature>/design.md`
2. Componentes y decisiones principales (uno por línea)
3. Decisiones que el equipo tiene que aprobar antes de seguir
4. Cambios que hay que volcar en `contratos/openapi.yaml`
5. Siguiente paso: que el equipo revise las decisiones; después,
   `crear-tasks gym-manager <feature>`
