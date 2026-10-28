# Comandos

Instrucciones de trabajo para **cualquier agente de IA**. Cada archivo es un prompt completo: el
agente lo lee y lo sigue al pie de la letra.

| Comando | Argumentos | Qué hace |
|---|---|---|
| [crear-requirements](crear-requirements.md) | `gym-manager <feature>` | Paso 1 de SDD: escribe `specs/<feature>/requirements.md` (historias + criterios EARS). |
| [crear-design](crear-design.md) | `gym-manager <feature>` | Paso 2 de SDD: escribe `specs/<feature>/design.md`. |
| [crear-tasks](crear-tasks.md) | `gym-manager <feature>` | Paso 3 de SDD: escribe `specs/<feature>/tasks.md`. |
| [implementar](implementar.md) | `<ruta a tasks.md o a un plan>` | Ejecuta las tareas una por una, con tests, y actualiza el estado. |
| [crear-plan](crear-plan.md) | `<pedido>` | Cambios chicos o técnicos sin SDD (bugfix, refactor): escribe `planes/AAAA-MM-DD-<nombre>.md`. |

## Cómo se usan

**Claude Code:** como slash command (`/crear-requirements gym-manager organizacion`). Los archivos
de `.claude/commands/` son atajos que solo apuntan a esta carpeta.

**Cualquier otro agente:** pedile que lea el archivo y lo ejecute con los argumentos. Por ejemplo:

```
Leé comandos/crear-requirements.md y seguilo al pie de la letra.
Argumentos: gym-manager organizacion
```

Desde un repo de código (back, desktop, mobile) la ruta es `../gym-manager-docs/comandos/…`.

## Convenciones comunes a todos los comandos

- **`DOCS`** = la raíz de este repo (`gym-manager-docs/`). Si el agente está parado en otro repo,
  es `../gym-manager-docs/`.
- **`RAIZ`** = la carpeta padre de `DOCS` (`gym-manager/`), donde están los cuatro repos clonados
  uno al lado del otro.
- **Argumentos:** `$1`, `$2` y `$ARGUMENTS` son los argumentos que pasa el usuario. El primero de
  los comandos de SDD es siempre el proyecto (`gym-manager`); se conserva para que los comandos
  sean compatibles con otros workspaces.
- **Salida:** todo resultado se escribe en `DOCS`, con el mismo formato y en la misma ruta, sin
  importar qué agente lo genere.
- Si falta información, se pregunta o se marca `[PENDIENTE]`. Nunca se inventan reglas de negocio.
