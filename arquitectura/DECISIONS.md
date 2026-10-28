# Registro de Decisiones y Alternativas

> Este documento explica el **por qué** detrás de las reglas de `ARCHITECTURE.md` e `IMPLEMENTATION.md`. No hace falta leerlo para aplicar las reglas — es para cuando alguien pregunta "¿por qué así y no de otra forma?", o para reevaluar una decisión si el contexto cambió.

---

## 1. Fachada de Aplicación vs CQRS

**Decisión:** Fachada de Aplicación como default de este precedente.

**El problema que ambas resuelven:** cuando un módulo tiene varios casos de uso (crear, actualizar, borrar, ajustar...), inyectarlos todos uno por uno en el controller ensucia el constructor.

**CQRS** (`@nestjs/cqrs`, `CommandBus`/`QueryBus`): el controller despacha un comando plano y Nest busca en memoria qué handler lo resuelve (`@CommandHandler`). Ventajas: muy desacoplado, facilita meter eventos de dominio, es el estándar en DDD maduro a gran escala. Desventaja: agrega una capa de resolución implícita ("magia") y curva de aprendizaje — un desarrollador nuevo no puede simplemente seguir un `import` para encontrar qué código ejecuta un comando, tiene que saber que existe el mecanismo de resolución de handlers.

**Fachada de Aplicación:** un servicio inyectable que **no tiene lógica de negocio**, solo delega al caso de uso correspondiente. Es el clásico `service.ts`, pero con un cambio de rol: deja de ser "el que sabe todo" para ser una mesa de entradas tonta.

**Por qué se eligió Fachada:** simplicidad, código 100% nativo de NestJS (sin dependencia adicional), curva de aprendizaje baja. Para el tamaño de proyecto típico de este precedente (sistemas de gestión internos, no plataformas con miles de eventos de dominio concurrentes), el costo de CQRS no se justifica.

**Cuándo reconsiderar CQRS:** si el proyecto crece a un punto donde los eventos de dominio (ej.: "cuando se crea un pedido, notificar a tres sistemas distintos de forma desacoplada") se vuelven un patrón recurrente, o el equipo ya tiene experiencia sólida con el patrón y el costo de la curva de aprendizaje ya está pagado.

**El riesgo de la Fachada a medida que el sistema crece:** si un módulo termina con lógica de alta, stock, precios y proveedores todo en una sola fachada, ese archivo termina con decenas de imports y un árbol de dependencias gigante. La mitigación (fachadas específicas por responsabilidad, documentada en `ARCHITECTURE.md` §2.3) es lo que mantiene viable esta elección a largo plazo sin migrar a CQRS.

---

## 2. Unit of Work: cuándo se justifica

**Regla:** Unit of Work solo cuando existe genuina transaccionalidad cross-agregado — fabricar casos de UoW para "parecer prolijo" es un anti-patrón.

**Por qué es un anti-patrón fabricarlo sin necesidad:** agrega una capa de indirección (interfaz `IUnitOfWork`, implementación concreta, wiring en el módulo) que no aporta nada si ningún caso de uso combina más de una escritura atómica. El costo de mantenimiento (un archivo más, una capa más para razonar) no tiene contrapartida.

**La forma acoplada que el UoW evita**, si se manejara la transacción directo en el caso de uso:

```typescript
// ❌ Acoplado: el caso de uso conoce el ORM directamente
async execute(dto: CrearProductoDto): Promise<void> {
  await this.dataSource.transaction(async (manager) => {
    // ... lógica mezclada con detalles de TypeORM
  });
}
```

Dos problemas: (1) el caso de uso deja de ser agnóstico a la infraestructura — cambiar de ORM implica tocar lógica de negocio, no solo infraestructura; (2) es difícil de testear sin una base real — para unit-testear el caso de uso habría que mockear objetos de bajo nivel del ORM (`DataSource`, `QueryRunner`, `EntityManager`) en vez de una interfaz simple.

---

## 3. Variante A (por-método) vs Variante B (genérica por token) de Unit of Work

| | Variante A | Variante B |
|---|---|---|
| Tipado | Explícito, sin casteos | Requiere especificar `T` a mano, sin chequeo de compilación entre token y tipo |
| Crecimiento de la interfaz | Un método nuevo por cada módulo que se sume | La interfaz no cambia nunca |
| Descubribilidad | Alta — autocompletado lista los métodos disponibles | Baja — hay que conocer los tokens de antemano |
| Mejor para | Proyectos con pocos módulos participando de transacciones cruzadas | Proyectos grandes donde la lista de métodos ya es incómoda |

**Cuándo migrar de A a B:** cuando `IUnitOfWork` ya acumula muchos métodos `getXRepository()` y se vuelve tedioso mantenerlos, o cuando se prevé que casi todos los módulos del sistema van a necesitar participar de transacciones cruzadas en algún momento.

---

## 4. `processes/` vs `forwardRef()` para orquestación cross-módulo

**El problema:** una fachada de un módulo nunca debe llamar directo a la fachada de otro módulo — genera acoplamiento circular (A depende de B, B depende de A).

**`forwardRef()`:** mecanismo de NestJS para resolver dependencias circulares entre módulos, permitiendo que un caso de uso de un módulo inyecte directamente un caso de uso de otro. Es la solución **más simple** — no requiere reorganizar carpetas ni crear un módulo neutral.

**Carpeta `processes/`:** un caso de uso orquestador vive en un módulo neutral (ni A ni B), y **ambos** módulos lo importan sin depender el uno del otro. Elimina la necesidad de `forwardRef()` porque no hay dependencia circular — hay un tercer punto que depende de los dos, y ninguno de los dos depende de él a su vez desde la perspectiva de acoplamiento (no hay ciclo).

**Por qué no usar `processes/` desde el día uno en todos los casos:** para una necesidad puntual y aislada (un solo cruce entre dos módulos, que no se prevé que se repita), montar la estructura de carpeta compartida es más costo que beneficio. `forwardRef()` resuelve el mismo problema técnico con menos fricción organizativa.

**Por qué migrar a `processes/` cuando la orquestación se vuelve recurrente:** con múltiples casos de uso orquestadores dispersos usando `forwardRef()` en distintos puntos del código, se vuelve difícil rastrear **dónde** vive la lógica de cruce entre módulos — queda esparcida. Centralizar en `processes/` da un único lugar reconocible para toda la lógica de orquestación cross-módulo.

**Señal concreta para decidir el cambio:** más de dos o tres casos de uso orquestadores cruzando los mismos módulos, o la sensación de que "hay que acordarse dónde están todos los `forwardRef()` para no romper algo".

---

## 5. Validación en dos barreras (DTO + Value Object)

**Decisión:** validación sintáctica en `class-validator` sobre el DTO (capa de infraestructura/aplicación), validación semántica en el Value Object (capa de dominio).

**Por qué no alcanza con una sola:**
- El DTO con `class-validator` asegura que las peticiones externas (HTTP, un job, un evento) tengan el **formato** correcto — campos requeridos, tipos correctos. No sabe nada de reglas de negocio.
- El Value Object valida en tiempo de ejecución que ninguna operación interna del software viole las reglas de negocio esenciales (ej.: un precio nunca negativo), **sin importar por dónde entre el dato** — HTTP, un job interno, un test, un seed de datos. Si solo se validara en el DTO, cualquier código que construya la entidad sin pasar por ese DTO específico (un test, un script de migración de datos, otro caso de uso) podría crear un objeto en un estado inválido.

---

## 6. `create()` vs `reconstitute()` en las entidades

**El problema que resuelve:** un mapper que usa `Repository.create()` o el constructor de fábrica de "objeto nuevo" al leer datos de la base está re-ejecutando validaciones de "objeto recién nacido" sobre datos que ya pasaron esa validación en su momento — y potencialmente fallando o comportándose de forma inesperada si la regla de validación cambió desde que ese registro se creó originalmente.

**La distinción:** `create()` es para un objeto que nace ahora — valida invariantes de creación. `reconstitute()` es para reconstruir en memoria un objeto que ya existía — la usa exclusivamente el mapper de infraestructura, nunca código de negocio ni casos de uso.

---

## 7. Response mapping en el caso de uso, no en el controller

**Decisión:** el shape de lo que se devuelve al cliente HTTP se decide en el caso de uso.

**Por qué:** mantiene el controller como una capa completamente delgada — su única responsabilidad es recibir el request, delegarlo, y devolver lo que le llega. Si el mapping de respuesta viviera en el controller, cambiar el shape de salida de una operación implicaría tocar infraestructura en vez de aplicación, y duplicaría lógica si el mismo caso de uso se invoca desde dos controllers distintos (HTTP y, por ejemplo, un consumer de eventos).

---

## 8. TypeORM vs Prisma — criterios de elección

No hay una respuesta única; ambos son válidos según el contexto:

| Criterio | Favorece TypeORM | Favorece Prisma |
|---|---|---|
| Equipo ya tiene experiencia con uno de los dos | — | — |
| Se necesita control fino sobre queries complejas con QueryBuilder | ✅ | |
| Se prioriza velocidad de desarrollo y tipado automático de queries | | ✅ |
| El proyecto necesita convivir con SQL crudo extenso o legacy | ✅ | |
| Se quiere evitar mantener clases de entidad ORM a mano | | ✅ |

**Riesgo a tener en cuenta con Prisma:** la herramienta introduce cambios de arquitectura relevantes entre versiones majors (por ejemplo, la introducción de driver adapters obligatorios y el archivo `prisma.config.ts` en versiones recientes). Antes de fijar una versión en un proyecto nuevo, confirmar cuál es la última **estable** (no una Release Candidate) — el tag `latest` de npm no siempre apunta a la versión más madura disponible.

---

## 9. Testing: unitario con mocks vs integración con base real

**Decisión:** casos de uso se testean unitariamente con el repositorio mockeado; los repositorios concretos se testean de integración contra una base real.

**Por qué no mockear todo:** un test unitario de caso de uso con repo mockeado verifica la **orquestación y las reglas de negocio** — no verifica que el SQL/query generado por el repositorio concreto sea correcto. Para eso hace falta un test de integración real contra Postgres (vía Docker), sin mockear el ORM — es la única forma de detectar, por ejemplo, un `include`/`join` mal armado o una constraint de base violada que el mock no puede simular con fidelidad.

**Costo de esta separación:** dos suites de test distintas por módulo (unitaria + integración), con setup distinto (mocks vs Docker + reset de base entre tests). Se acepta este costo porque cada suite cubre una preocupación que la otra no puede cubrir igual de bien.
