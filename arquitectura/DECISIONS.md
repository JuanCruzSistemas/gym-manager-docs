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

## 3. Unit of Work: contexto ambiental (`AsyncLocalStorage`) en vez de un método por repositorio o uno genérico por token

**Decisión:** `IUnitOfWork` se reduce a `runInTransaction<T>(work: () => Promise<T>): Promise<T>` — sin ningún método para obtener repositorios. Un `Gestor` (`GestorPrismaClient`) usa `AsyncLocalStorage` de Node para publicar el cliente/manager activo de la transacción, y cada repositorio lo lee solo, vía un getter (`gestor.actual`), sin que nadie se lo pase.

**Las dos formas que se descartaron, y por qué parecían razonables igual:** antes de encontrar este patrón, el precedente documentaba dos variantes para que el caso de uso obtuviera repositorios "transaccionales" desde el UoW:

| | Variante por-método | Variante genérica por token |
|---|---|---|
| Forma | `uow.getProductoRepository()` — un método por repo | `uow.getRepository<T>(token)` — un método genérico |
| Tipado | Explícito, sin casteos | Requiere especificar `T` a mano, sin chequeo de compilación entre token y tipo |
| Crecimiento de la interfaz | Un método nuevo por cada módulo que se sume | La interfaz no cambia nunca, pero hay que mantener un `Map` de factories |

Ambas resuelven el mismo problema de fondo: que el repositorio use el cliente de la transacción (`tx`) y no el cliente global, cuando hay una transacción en curso. Para lograrlo, las dos obligan a que **el UoW sea quien construye el repositorio** — pasándole el cliente correcto por constructor — en vez de que el repositorio se registre como provider normal de Nest con `useClass`.

**Por qué se reemplazaron por `AsyncLocalStorage`:** el problema no es "cómo le paso el cliente correcto al repositorio" — es que, al resolverlo pasándolo explícitamente, el repositorio deja de ser un provider común: ya no se puede inyectar directo en un caso de uso sin pasar por el UoW, y el UoW termina conociendo (o un `Map` de factories) la lista completa de repositorios del sistema. `AsyncLocalStorage` saca esa necesidad: publica el cliente activo en un contexto ambiental que cualquier código que corra dentro de la misma cadena de `await` puede leer sin que se lo pasen. El repositorio vuelve a ser un provider normal — se inyecta directo donde haga falta, con o sin transacción activa — y el UoW se reduce a lo que realmente es: un mecanismo para abrir y cerrar la transacción.

**El costo real de esta variante — no es gratis:** `AsyncLocalStorage` es contexto implícito. Si una porción de código dentro de `runInTransaction` no se `await`ea (fire-and-forget, un `setTimeout`, una promesa que sigue corriendo después de que el bloque ya resolvió), ese código corre fuera de la cadena de contexto y `gestor.actual` le devuelve el cliente global en vez del de la transacción — silenciosamente, sin ningún error en el momento. Es un costo de disciplina (ver `IMPLEMENTATION.md` §6) a cambio de eliminar una capa de indirección explícita. Se acepta ese trade-off porque el caso de "un `await` olvidado dentro de una transacción" ya es, en la práctica, un bug grave por sí mismo (la operación corre fuera de la transacción de todos modos, sea cual sea el mecanismo de UoW) — `AsyncLocalStorage` no inventa un riesgo nuevo, solo cambia cómo se manifiesta.

**Cuándo reconsiderar:** si en algún momento se necesitara propagar más que un cliente de transacción por el contexto ambiental (por ejemplo, metadata de auditoría, tenant, usuario autenticado) y el patrón se empieza a sentir sobrecargado para una sola responsabilidad, vale la pena revisar si sigue siendo la pieza correcta o si conviene separar responsabilidades.

---

## 4. `processes/` vs `forwardRef()` para orquestación cross-módulo

**El problema:** una fachada de un módulo nunca debe llamar directo a la fachada de otro módulo — genera acoplamiento circular (A depende de B, B depende de A).

**`forwardRef()`:** mecanismo de NestJS para resolver dependencias circulares entre módulos, permitiendo que un caso de uso de un módulo inyecte directamente un caso de uso de otro. Es la solución **más simple** — no requiere reorganizar carpetas ni crear un módulo neutral.

**Carpeta `processes/`:** un caso de uso orquestador vive en un módulo neutral (ni A ni B), y **ambos** módulos lo importan sin depender el uno del otro. Elimina la necesidad de `forwardRef()` porque no hay dependencia circular — hay un tercer punto que depende de los dos, y ninguno de los dos depende de él a su vez desde la perspectiva de acoplamiento (no hay ciclo).

**Por qué no usar `processes/` desde el día uno en todos los casos:** para una necesidad puntual y aislada (un solo cruce entre dos módulos, que no se prevé que se repita), montar la estructura de carpeta compartida es más costo que beneficio. `forwardRef()` resuelve el mismo problema técnico con menos fricción organizativa.

**Por qué migrar a `processes/` cuando la orquestación se vuelve recurrente:** con múltiples casos de uso orquestadores dispersos usando `forwardRef()` en distintos puntos del código, se vuelve difícil rastrear **dónde** vive la lógica de cruce entre módulos — queda esparcida. Centralizar en `processes/` da un único lugar reconocible para toda la lógica de orquestación cross-módulo.

**Señal concreta para decidir el cambio:** más de dos o tres casos de uso orquestadores cruzando los mismos módulos, o la sensación de que "hay que acordarse dónde están todos los `forwardRef()` para no romper algo".

### 4.2. Por qué el caso de uso de `processes/` inyecta repositorios, no los casos de uso del otro módulo

**El error que parece la migración obvia desde `forwardRef()`:** al mover un caso de uso orquestador de Nivel 2 a `processes/`, lo más directo parece ser sacarle el `forwardRef()` y dejar las mismas dependencias — el caso de uso de `processes/` sigue inyectando `CrearFacturaUseCase` de Ventas y `DescontarStockUseCase` de Stock, ahora sin `forwardRef` porque "ya no hay ciclo, está en un módulo neutral". Eso es falso en la mayoría de los casos reales: para inyectar esos casos de uso, el módulo de `processes/` tiene que importar `VentasModule` y `StockModule` completos (son los que los declaran como providers). Y si el controller de Ventas necesita exponer la operación orquestada — lo habitual, porque el endpoint de "crear venta" es justamente el que dispara el descuento de stock — `VentasModule` necesita importar de vuelta el módulo de `processes/` para inyectar su fachada en el controller. El ciclo que `processes/` existía para eliminar reaparece, solo que ahora cruza tres módulos en vez de dos.

**La resolución adoptada (confirmada contra `proyecto-venta-base`, que la implementa en sus tres `processes/` reales):** el caso de uso de `processes/` no inyecta los casos de uso ni la fachada de los módulos de dominio — inyecta **los repositorios** de cada uno, a través de un submódulo hoja de persistencia por módulo (`<modulo>-persistence.module.ts`) que solo cablea el token del repositorio y no depende de nada más. El módulo de `processes/` importa esos submódulos hoja, nunca los módulos de dominio completos. Esto implica que el caso de uso de `processes/` **reimplementa** la porción de lógica que necesita (usando las entidades y sus métodos de negocio directo), en vez de reutilizar el caso de uso ya escrito del otro módulo.

**Por qué se acepta esa duplicación de código y no se prioriza "no repetir lógica":** la alternativa — reusar el caso de uso del otro módulo — requiere importar el módulo completo que lo declara, y ese módulo completo es exactamente el que necesita, casi siempre, importar de vuelta al módulo de `processes/` para su propio controller. No hay forma de tener ambas cosas (reutilizar el caso de uso ajeno y mantener el grafo de módulos sin ciclos) en el caso general, así que entre las dos, se prioriza la ausencia de ciclo: es un error de arranque de la aplicación (`Nest can't resolve dependencies...`), no un defecto sutil que aparece después. En la práctica la "duplicación" suele ser menor de lo que parece — el caso de uso de `processes/` normalmente necesita una porción más chica de la lógica de cada módulo (por ejemplo, solo "descontar stock" y no todo lo que sabe hacer `StockModule`), y las piezas que sí son genuinamente compartidas (excepciones de dominio/aplicación, mappers de respuesta) se reutilizan igual — se importan como clases de TypeScript directo desde el archivo del otro módulo, sin pasar por `@Module({ imports: [...] })`, así que no agregan una arista al grafo de módulos y no generan ciclo.

**Señal de que esto se volvió un problema real:** si la lógica reimplementada en varios `processes/` empieza a divergir de la del módulo de dominio original (una regla se actualiza en un lado y no en el otro), es momento de extraer esa porción compartida a un servicio de dominio o de aplicación propio, inyectable desde ambos lados sin arrastrar el módulo completo — no de volver a inyectar el caso de uso ajeno.

---

## 4.1. Por qué un puerto de notificaciones (o cualquier puerto sin clases de dominio) no necesita CQRS

**El escenario que puede confundirse con esto:** una acción de negocio dispara un efecto secundario (notificar por email o push) desde varios módulos distintos — Cobros, Suscripciones, Usuarios, Clases, cada uno con su propio evento. A primera vista parece el caso de libro para domain events + un bus de eventos (`@nestjs/cqrs` u otro), porque es fan-out desde varios orígenes hacia un mismo destino.

**Por qué no hace falta igual:** CQRS con `CommandBus`/`EventBus` resuelve el problema de **desacoplar quién dispara un efecto de quién lo ejecuta**, normalmente porque hay reintentos, un registro de qué se entregó, o varios handlers independientes reaccionando al mismo evento sin que el emisor los liste. Si el envío es fire-and-forget (sin reintentos, sin bandeja de pendientes, el resultado queda solo en el log) y cada evento tiene un único destino claro, ese desacople no aporta nada — es la misma complejidad que fabricar un Unit of Work sin escrituras atómicas que proteger (sección 2).

**La resolución simple:** el módulo de notificaciones es un puerto de infraestructura sin clases de dominio (ver `ARCHITECTURE.md` §2.4.1) — cualquier caso de uso que necesite notificar lo inyecta directo (`IEmailSender`, `IPushNotifier`) y lo llama al final de su `execute()`, igual que llama al repositorio. No hay módulos de dominio enfrentados, así que no aplica ni `forwardRef()` ni `processes/`, y no hace falta la infraestructura de CQRS para evitar un acoplamiento que no existe.

**Cuándo sí reconsiderar:** si aparecen reintentos con estado propio, una bandeja de notificaciones (leídas/no leídas), o un mismo evento con múltiples handlers que no deberían conocerse entre sí — ahí el puerto dejó de ser "sin lógica propia" y el análisis de la sección 1 (Fachada vs CQRS) vuelve a aplicar.

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

## 6.1. Factory aparte para entidades compuestas, y `domain/inputs/` en vez de parámetros sueltos

**El problema:** `create()`/`reconstitute()` en la propia entidad funciona bien cuando la entidad no tiene nada más que construir que sus propios campos primitivos (caso `Proveedor`). Pero cuando la entidad compone Value Objects propios o entidades hijas (caso `Producto`, que instancia `ProductoPrice`; o `Venta`, que instancia una lista de `DetalleVenta`), ese mismo método termina mezclando dos responsabilidades distintas: la construcción/validación de las piezas que la componen, y la construcción de la entidad en sí.

**La regla adoptada:**
- **Entidad simple** (sin VOs ni entidades hijas propias): `create()`/`reconstitute()` siguen viviendo en la propia clase — constructor privado, como siempre.
- **Entidad compuesta** (instancia VOs o entidades hijas): la entidad pasa a tener **constructor público** y pierde `create()`/`reconstitute()`; esa responsabilidad se muda a una clase `XFactory` separada en `domain/factories/`, que es quien instancia los VOs/hijos y después llama al constructor de la entidad.

**Por qué no forzar siempre Factory (ni siempre entidad-level):** una Factory separada para una entidad simple es indirección sin beneficio — no hay nada que componer, así que separar "quién construye" de "qué se construye" no gana nada y agrega un archivo más para seguir. A la inversa, meter la composición de VOs/hijos dentro de un `create()` en la propia entidad compuesta funciona hasta que esa composición crece (más de un VO, validaciones cruzadas entre hijos) y el método se vuelve difícil de leer mezclado con el resto de la clase.

**El costo real de la entidad compuesta con Factory:** dos archivos en vez de uno por entidad (`producto.entity.ts` + `productos.factory.ts`), y hay que acordarse de no llamar `new Producto(...)` directo desde fuera de la Factory o el mapper — el constructor público ya no impide construir la entidad en un estado inválido por fuera del único camino pensado para eso. Se acepta porque, a cambio, cada Factory concentra en un solo lugar toda la lógica de composición de una entidad, en vez de esparcirla entre un `create()` sobrecargado y validaciones repetidas en cada punto que construye la entidad.

**Por qué `domain/inputs/` (interfaces `CreateXInput`/`ReconstituteXInput`) en vez de parámetros sueltos:** tanto el `create()`/`reconstitute()` de una entidad simple como los de una Factory reciben, en los casos reales, varios campos — y esa lista crece con el tiempo. Parámetros posicionales sueltos (`create(nombre, precio, proveedorId)`) se vuelven frágiles: agregar un campo nuevo, o reordenar, rompe silenciosamente cualquier llamada existente que no use nombres (TypeScript no avisa si dos parámetros son el mismo tipo primitivo y se invierten por error). Una interfaz nombra cada campo explícitamente, permite anidar (el input de un agregado padre puede incluir el input de sus hijos: `CreateVentaInput` anida `CreateDetalleInput[]`), y dado que ya existe `ReconstituteXInput` como forma natural para la reconstrucción desde el mapper, se adopta la misma convención también para `create()` por consistencia.

---

## 7. Response mapping en el caso de uso, no en el controller

**Decisión:** el shape de lo que se devuelve al cliente HTTP se decide en el caso de uso.

**Por qué:** mantiene el controller como una capa completamente delgada — su única responsabilidad es recibir el request, delegarlo, y devolver lo que le llega. Si el mapping de respuesta viviera en el controller, cambiar el shape de salida de una operación implicaría tocar infraestructura en vez de aplicación, y duplicaría lógica si el mismo caso de uso se invoca desde dos controllers distintos (HTTP y, por ejemplo, un consumer de eventos).

---

## 8. ORM: Prisma

**Decisión:** Prisma, sin alternativa documentada en paralelo. `IMPLEMENTATION.md` cubre solo este camino.

**Por qué Prisma y no TypeORM, en el contexto de este precedente:** se prioriza velocidad de desarrollo y tipado automático de queries de punta a punta con TypeScript, y se evita mantener clases de entidad ORM a mano — el trade-off que más pesa para los sistemas de gestión interna que son el caso de uso típico de este precedente (sección 0). El control fino con QueryBuilder y la convivencia con SQL crudo extenso, donde TypeORM es más fuerte, no son necesidades recurrentes en ese perfil de proyecto.

**Riesgo a tener en cuenta con Prisma:** la herramienta introduce cambios de arquitectura relevantes entre versiones majors (por ejemplo, la introducción de driver adapters obligatorios y el archivo `prisma.config.ts` en versiones recientes). Antes de fijar una versión en un proyecto nuevo, confirmar cuál es la última **estable** (no una Release Candidate) — el tag `latest` de npm no siempre apunta a la versión más madura disponible.

**Si en algún momento un proyecto concreto necesita TypeORM** (control fino de queries, legacy con SQL crudo extenso), ese proyecto documenta su propia desviación de este precedente — no se reintroduce la comparación acá para no mantener dos caminos de infraestructura sin que haya una necesidad real pesando los dos.

---

## 9. Testing: unitario con mocks vs integración con base real

**Decisión:** casos de uso se testean unitariamente con el repositorio mockeado; los repositorios concretos se testean de integración contra una base real.

**Por qué no mockear todo:** un test unitario de caso de uso con repo mockeado verifica la **orquestación y las reglas de negocio** — no verifica que el SQL/query generado por el repositorio concreto sea correcto. Para eso hace falta un test de integración real contra Postgres (vía Docker), sin mockear el ORM — es la única forma de detectar, por ejemplo, un `include`/`join` mal armado o una constraint de base violada que el mock no puede simular con fidelidad.

**Costo de esta separación:** dos suites de test distintas por módulo (unitaria + integración), con setup distinto (mocks vs Docker + reset de base entre tests). Se acepta este costo porque cada suite cubre una preocupación que la otra no puede cubrir igual de bien.