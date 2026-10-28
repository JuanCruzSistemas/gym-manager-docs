# CLAUDE.md — gym-manager-docs

Las reglas del proyecto son las mismas para todos los agentes y están en `AGENTS.md`:

@AGENTS.md

## Específico de Claude Code

- Los comandos de SDD están disponibles como slash commands: `/crear-requirements`,
  `/crear-design`, `/crear-tasks`, `/implementar` y `/crear-plan`. Los archivos de
  `.claude/commands/` son atajos: la instrucción real está en `comandos/`. Si hay que cambiar un
  comando, se cambia en `comandos/`, nunca en el atajo.
- Si Claude Code se abre en la carpeta padre (`gym-manager/`) o en un repo de código, las rutas
  `DOCS` y `RAIZ` se resuelven igual (ver `comandos/README.md`).
