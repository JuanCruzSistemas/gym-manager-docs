# gym-manager-docs

Fuente de verdad del negocio y de las specs de **GymManager**, el sistema de gestión de
gimnasios del Seminario Integrador (UTN FRVM, Grupo 22).

Acá viven el contexto del dominio, las specs por feature, el contrato OpenAPI, los diagramas, el
precedente de arquitectura y los comandos de Spec-Driven Development (SDD). El código está en
los otros tres repos de la organización **JuanCruzSistemas**:

| Repo | Contenido |
|---|---|
| [gym-manager-back](https://github.com/JuanCruzSistemas/gym-manager-back) | API NestJS |
| [gym-manager-desktop](https://github.com/JuanCruzSistemas/gym-manager-desktop) | Aplicación C# / WPF |
| [gym-manager-mobile](https://github.com/JuanCruzSistemas/gym-manager-mobile) | Aplicación Flutter |
| gym-manager-docs | Este repo |

Tablero: GitHub Projects **"GymManager Project"**.

## Puesta en marcha

Los cuatro repos van **uno al lado del otro** en la misma carpeta, igual en todas las máquinas,
para que los comandos y las rutas relativas funcionen:

```bash
mkdir gym-manager && cd gym-manager
git clone https://github.com/JuanCruzSistemas/gym-manager-docs.git
git clone https://github.com/JuanCruzSistemas/gym-manager-back.git
git clone https://github.com/JuanCruzSistemas/gym-manager-desktop.git
git clone https://github.com/JuanCruzSistemas/gym-manager-mobile.git
```

```
gym-manager/
├── gym-manager-back/
├── gym-manager-desktop/
├── gym-manager-mobile/
└── gym-manager-docs/
```

Cada repo de código tiene un `CLAUDE.md` y un `AGENTS.md` cortos que mandan a leer
`../gym-manager-docs/AGENTS.md`, así cualquier integrante arranca con las mismas reglas, use el
agente que use.

## Estructura

```
gym-manager-docs/
├── AGENTS.md              reglas para cualquier agente de IA
├── CLAUDE.md              importa AGENTS.md (Claude Code)
├── comandos/              instrucciones de los comandos de SDD, para cualquier agente
├── .claude/commands/      atajos de Claude Code que apuntan a comandos/
├── documentos/            los cinco documentos del proyecto (.docx)
├── contexto/              contexto del negocio en Markdown, generado desde documentos/
├── specs/<feature>/       requirements.md, design.md, tasks.md
├── planes/                planes de cambios chicos o técnicos
├── contratos/openapi.yaml contrato REST
├── arquitectura/          precedente DDD + hexagonal
└── diagramas/             .drawio: clases, estados, componentes, despliegue y DER
```

El detalle de cada carpeta y las reglas de trabajo están en [AGENTS.md](AGENTS.md).

## Flujo de trabajo (SDD)

1. Si la feature necesita reglas o términos nuevos, se agregan en `contexto/`.
2. `crear-requirements gym-manager <feature>` → `specs/<feature>/requirements.md`.
3. El equipo lo revisa y lo pasa a **Estado: Aprobada**.
4. `crear-design gym-manager <feature>` → `design.md`. El equipo revisa las decisiones.
5. `crear-tasks gym-manager <feature>` → `tasks.md`.
6. `implementar specs/<feature>/tasks.md` → una tarea por vez, con tests.
7. PR en el repo de código, linkeando la spec.

Cambios chicos o técnicos: `crear-plan <pedido>` → `implementar planes/<archivo>.md`.

### Cómo correr un comando

**Claude Code** (abierto en este repo o en uno de código):

```
/crear-requirements gym-manager organizacion
```

**Cualquier otro agente:**

```
Leé comandos/crear-requirements.md y seguilo al pie de la letra.
Argumentos: gym-manager organizacion
```

Desde un repo de código, la ruta es `../gym-manager-docs/comandos/…`. Lista completa en
[comandos/README.md](comandos/README.md).

## Features por iteración

| Iteración | Feature (slug) |
|---|---|
| 0 | `fundacion` |
| 1 (MVP) | `cuenta-usuarios-personal`, `organizacion`, `membresias-metodos-pago`, `solicitud-aceptacion-cobro`, `asistencias-rachas` |
| 2 | `entrenamiento` |
| 3 | `clases-inscripciones`, `notificaciones-push`, `cobro-recurrente` |
| 4 | `sync-offline` |

Los CU de cada feature están en `contexto/alcance.md` y en la Metodología §2.7.
