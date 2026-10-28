# Arquitectura de Referencia: DDD + Hexagonal en NestJS

## Cómo usar este documento

Este es el documento de precedente arquitectónico para cualquier proyecto NestJS nuevo (o módulo nuevo dentro de uno existente) que vaya a seguir Domain-Driven Design + Arquitectura Hexagonal. Reemplaza cualquier README anterior de proyectos puntuales — es agnóstico a cualquier dominio de negocio específico.

Forma parte de un set de tres documentos:
- **`ARCHITECTURE.md`** (este archivo): reglas de las tres capas, checklists, niveles de complejidad.
- **`IMPLEMENTATION.md`**: guía concreta de infraestructura (Prisma).
- **`DECISIONS.md`**: el razonamiento detrás de cada regla — trade-offs, alternativas descartadas, y por qué. Consultar cuando una regla de acá no quede clara o alguien quiera cuestionarla.

Si sos un agente de terminal arrancando un proyecto nuevo: leé primero la sección 0 para decidir el nivel de complejidad, después la sección de checklist de la capa que estés tocando. No hace falta leer `DECISIONS.md` para aplicar las reglas — solo para entender por qué existen.

**Stack de referencia:** NestJS 11 (LTS) y Prisma como ORM (ver `IMPLEMENTATION.md` y `DECISIONS.md` §8). Los ejemplos de código de este set de documentos asumen esa versión de NestJS; si el proyecto corre sobre otra major, revisar que las firmas de los decoradores no hayan cambiado antes de copiar un snippet tal cual.

---

## 0. Niveles de complejidad — no todo proyecto necesita todo

Esta arquitectura completa (casos de uso, fachadas, Unit of Work, orquestadores) es una inversión que se paga con más código y más archivos. Antes de aplicar cualquier capa, evaluar en qué nivel está el proyecto o el módulo puntual.

### Nivel 1 — CRUD simple, sin reglas de negocio complejas

**Señales:** el módulo solo hace altas/bajas/consultas sin lógica de validación relevante más allá de tipos y formato; no hay invariantes de negocio que proteger; no se prevé que crezca.

**Qué aplicar:** un `Service` inyectable con el repositorio inyectado directo. Sin capa de casos de uso separada, sin fachada.

```typescript
@Injectable()
export class CategoriasService {
  constructor(@Inject(CATEGORIA_REPOSITORY_TOKEN) private readonly repo: ICategoriaRepository) {}

  async crear(dto: CrearCategoriaDto): Promise<Categoria> {
    return this.repo.save(Categoria.create(dto.nombre));
  }
}
```

**No aplicar acá:** Unit of Work, `processes/`, fachadas separadas por responsabilidad. Es sobre-ingeniería para este tamaño.

### Nivel 2 — Reglas de negocio no triviales, sin cruce entre módulos

**Señales:** hay invariantes de negocio reales (un precio no puede ser negativo, un stock no puede quedar en negativo), varias operaciones atómicas dentro del *mismo* agregado, o varios casos de uso que conviene agrupar bajo una fachada para no ensuciar el controller con múltiples inyecciones.

**Qué aplicar:** Casos de uso (`application/use-cases/`) + Fachada (`application/facades/`) como en la sección 2 de este documento. Unit of Work **solo si** una operación necesita atomicidad entre más de una escritura — si no, el caso de uso llama al repositorio directo, sin envolver nada.

**Sobre orquestación cross-módulo en este nivel:** si aparece una necesidad puntual y aislada de que un caso de uso toque dos módulos (por ejemplo, un módulo de Ventas necesita descontar Stock), y el proyecto es chico o esa necesidad no se va a repetir mucho, **es válido usar `forwardRef()`** entre los módulos de Nest en vez de armar la carpeta `processes/` completa. No es un anti-patrón en este nivel — es la solución proporcional. Ver `DECISIONS.md` para el detalle de este trade-off.

### Nivel 3 — Múltiples subdominios que se cruzan, transaccionalidad real entre agregados

**Señales:** varios módulos necesitan orquestarse entre sí de forma recurrente y en más de un lugar; hay operaciones que exigen atomicidad ACID entre agregados de distintos módulos; el número de casos de uso orquestadores cruzados ya generaría varios `forwardRef()` dispersos y difíciles de rastrear.

**Qué aplicar:** todo lo de Nivel 2, más:
- **Unit of Work** (ver `IMPLEMENTATION.md` para la implementación concreta).
- **Carpeta `processes/`** para casos de uso orquestadores cross-módulo, en vez de `forwardRef()` disperso.
- **Fachadas específicas por responsabilidad**, no una fachada gigante por módulo (ver sección 2.3).

**Regla de decisión rápida entre Nivel 2 y Nivel 3, sobre `processes/` vs `forwardRef`:** si ya tenés o preveés **más de dos o tres** casos de uso orquestadores cruzando los mismos módulos, migrar a `processes/`. Si es una necesidad puntual y aislada, `forwardRef()` es más simple y no hace falta pagar el costo de la carpeta compartida todavía.

> **Importante:** el nivel se evalúa **por módulo o por necesidad puntual**, no necesariamente por proyecto entero. Un proyecto grande puede tener módulos en Nivel 1 (un catálogo de categorías simple) conviviendo con módulos en Nivel 3 (ventas que cruzan stock y facturación). No hay que subir de nivel todo el proyecto solo porque una parte lo necesita.

---

## 1. Capa de Dominio

### 1.1. Regla de dependencia

**Prohibido** importar `@nestjs/*`, `typeorm`, `@prisma/client`, o cualquier librería de infraestructura/framework dentro de `domain/`. Si un archivo de dominio necesita importar algo así, la lógica está mal ubicada.

### 1.2. Entities

Dos variantes, según la entidad componga o no sus propios Value Objects o entidades hijas. En ambas, los parámetros de `create()`/`reconstitute()` se tipan como interfaces en `domain/inputs/` (sección 1.2.3) — nunca parámetros sueltos.

#### 1.2.1. Entidad simple — sin Value Objects ni entidades hijas propias

Constructor privado, `create()`/`reconstitute()` en la propia clase.

```typescript
// domain/inputs/create-proveedor.interface.ts
export interface CreateProveedorInput {
  nombre: string;
}

// domain/inputs/reconstitute-proveedor.interface.ts
export interface ReconstituteProveedorInput {
  id: string;
  nombre: string;
}

// domain/entities/proveedor.entity.ts
import { CreateProveedorInput } from '../inputs/create-proveedor.interface';
import { ReconstituteProveedorInput } from '../inputs/reconstitute-proveedor.interface';

export class Proveedor {
  private constructor(
    private readonly id: string | null,
    private nombre: string,
  ) {}

  // Fábrica para un objeto NUEVO — valida invariantes de creación
  static create(data: CreateProveedorInput): Proveedor {
    if (!data.nombre?.trim()) throw new NombreProveedorInvalidoException();
    return new Proveedor(null, data.nombre);
  }

  // Fábrica para REHIDRATAR desde persistencia — la usa SOLO el mapper
  static reconstitute(data: ReconstituteProveedorInput): Proveedor {
    return new Proveedor(data.id, data.nombre);
  }

  getId(): string | null { return this.id; }
  getNombre(): string { return this.nombre; }
}
```

#### 1.2.2. Entidad compuesta — Factory aparte en `domain/factories/`

Cuando la entidad instancia uno o más Value Objects propios, o arma entidades hijas (un agregado, como una `Venta` con sus `DetalleVenta`), esa construcción no queda en la entidad: va en una clase `XFactory` en `domain/factories/`. La entidad pasa a tener **constructor público** y sin `create()`/`reconstitute()` propios — es un contenedor de comportamiento, no de construcción.

```typescript
// domain/inputs/create-producto.interface.ts
export interface CreateProductoInput {
  nombre: string;
  precio: number;
  proveedorId: string;
}

// domain/inputs/reconstitute-producto.interface.ts
export interface ReconstituteProductoInput {
  id: string;
  nombre: string;
  precio: number;
  proveedorId: string;
}

// domain/entities/producto.entity.ts
import { ProductoPrice } from '../value-objects/producto-price.vo';

export class Producto {
  constructor(
    private readonly id: string | null,
    private nombre: string,
    private precio: ProductoPrice,
    private readonly proveedorId: string,
  ) {}

  actualizarPrecio(nuevoPrecio: number): void {
    this.precio = ProductoPrice.create(nuevoPrecio);
  }

  getId(): string | null { return this.id; }
  getNombre(): string { return this.nombre; }
  getPrecio(): number { return this.precio.getValue(); }
  getProveedorId(): string { return this.proveedorId; }
}

// domain/factories/productos.factory.ts
import { Producto } from '../entities/producto.entity';
import { ProductoPrice } from '../value-objects/producto-price.vo';
import { CreateProductoInput } from '../inputs/create-producto.interface';
import { ReconstituteProductoInput } from '../inputs/reconstitute-producto.interface';

export class ProductosFactory {
  static create(data: CreateProductoInput): Producto {
    const precio = ProductoPrice.create(data.precio);
    return new Producto(null, data.nombre, precio, data.proveedorId);
  }

  // SIEMPRE reconstituye los VOs validando de nuevo — la usa SOLO el mapper
  static reconstitute(data: ReconstituteProductoInput): Producto {
    const precio = ProductoPrice.create(data.precio);
    return new Producto(data.id, data.nombre, precio, data.proveedorId);
  }
}
```

**Regla de decisión:** ¿entidad simple o Factory aparte? Si `create()`/`reconstitute()` solo necesitan validar tipos primitivos y asignarlos, la entidad simple alcanza. En el momento en que aparece un Value Object propio o una entidad hija que ensamblar (sumar un total, derivar un subtotal, construir una lista de hijos), esa lógica de ensamblado no encaja bien como método estático de la propia clase — ahí se separa en una Factory. No es una regla de "siempre" o "nunca": conviven las dos variantes en el mismo proyecto, cada entidad con la que le corresponde por su propia complejidad.

**Regla, en ambas variantes:** `create()` valida las condiciones de un objeto que nace ahora. `reconstitute()` reconstruye un objeto que ya existía y ya pasó esa validación en su momento — **nunca** lo usa código de negocio, solo el mapper de infraestructura. Confundir los dos (usar `create()` para rehidratar) es un error de diseño, no un detalle de estilo.

**El costo de la entidad compuesta:** el constructor público ya no impide construir la entidad sin pasar por la Factory. Se acepta ese costo porque el Value Object sigue validando su propia invariante igual (`ProductoPrice.create()` revienta si el precio es inválido) — lo único que se pierde es la garantía de que el *ensamblado* (qué Factory, qué reglas de composición) se haga siempre igual, no la validación de cada dato en sí.

#### 1.2.3. `domain/inputs/` — parámetros de `create()`/`reconstitute()` como interface

Tanto en la variante simple como en la compuesta, los parámetros de `create()`/`reconstitute()` nunca son argumentos sueltos — se tipan como una interface en `domain/inputs/` (`CreateXInput`, `ReconstituteXInput`). Esto permite que el input de un agregado anide el de sus entidades hijas (por ejemplo, `CreateVentaInput` puede traer un array de `CreateDetalleInput`), y deja el método de creación con una sola firma legible en vez de una lista larga de parámetros posicionales.

### 1.3. Value Objects

Usar Value Objects cuando un dato primitivo tiene una regla de validación propia que se repetiría en varios lugares si no se encapsula. Los instancia `create()` (si la entidad es simple) o la Factory (si la entidad es compuesta) — nunca el caso de uso ni el repositorio.

```typescript
export class ProductoPrice {
  private constructor(private readonly value: number) {}

  static create(value: number): ProductoPrice {
    if (value < 0) throw new PrecioInvalidoException();
    return new ProductoPrice(value);
  }

  getValue(): number { return this.value; }
}
```

**Cuándo NO hace falta un Value Object:** si el dato no tiene ninguna regla de validación más allá del tipo (`string`, `number`), no se justifica envolverlo. No todo campo necesita ser un VO.

### 1.4. Domain Exceptions

Toda excepción de negocio —de dominio o de aplicación, sección 2.2.1— hereda, directa o indirectamente, de una única base compartida en `common/`:

```typescript
// common/exceptions/app-error-code.enum.ts
export enum AppErrorCode {
  INVALID_INPUT = 'INVALID_INPUT',
  CONFLICT = 'CONFLICT',
  NOT_FOUND = 'NOT_FOUND',
}

// common/exceptions/app-exception.ts
export class AppException extends Error {
  constructor(
    message: string,
    public readonly code: AppErrorCode,
  ) {
    super(message);
  }
}
```

Las excepciones de **dominio** (invariantes de una entidad o Value Object) extienden `DomainException`, que no agrega nada a `AppException` — solo marca de qué capa vienen, por carpeta y por import:

```typescript
// common/domain/exceptions/domain-exception.ts
import { AppException } from '../../exceptions/app-exception';
import { AppErrorCode } from '../../exceptions/app-error-code.enum';

export class DomainException extends AppException {
  constructor(message: string, code: AppErrorCode) {
    super(message, code);
  }
}

// domain/exceptions/precio-invalido.exception.ts
import { DomainException } from '../../../../common/domain/exceptions/domain-exception';
import { AppErrorCode } from '../../../../common/exceptions/app-error-code.enum';

export class PrecioInvalidoException extends DomainException {
  constructor() {
    super('Precio no válido, debe ser mayor a 0', AppErrorCode.INVALID_INPUT);
  }
}
```

**Por qué un `code` en vez de que el filtro distinga por `instanceof`:** cada excepción declara su propio `AppErrorCode` en el constructor, una sola vez, en el mismo archivo donde se define. El filtro de infraestructura (sección 3.2) mapea `code` → status HTTP con una tabla fija — agregar una excepción nueva nunca implica tocar el filtro.

### 1.5. Checklist — Capa de Dominio

- [ ] Se decidió si la entidad es simple (`create()`/`reconstitute()` propios) o compuesta (Factory aparte en `domain/factories/`) según si instancia Value Objects o arma entidades hijas (sección 1.2.2).
- [ ] Los parámetros de `create()`/`reconstitute()` están tipados como interface en `domain/inputs/`, no como parámetros sueltos.
- [ ] Ningún archivo de `domain/` importa una librería de framework/infraestructura.
- [ ] Las reglas de negocio (invariantes) están en la entidad o en un Value Object, no en el repositorio ni en el caso de uso.
- [ ] Las excepciones de dominio extienden `DomainException` y declaran su `AppErrorCode`.

---

## 2. Capa de Aplicación

### 2.1. DTOs

```typescript
export class CrearProductoDto {
  @IsUUID(4)
  readonly id: string;

  @IsString()
  @IsNotEmpty()
  readonly nombre: string;

  @IsNumber()
  readonly precio: number;
}
```

Los DTOs viven en `application/dtos/` y validan la **forma sintáctica** del dato (tipos, formato) con `class-validator`. La validación **semántica** (reglas de negocio) va en el dominio (Value Objects), no acá — son dos barreras distintas, ver `DECISIONS.md`.

### 2.2. Use Cases

Un caso de uso es una clase con **un solo método público** (`execute`), que orquesta una operación atómica de negocio.

```typescript
@Injectable()
export class CrearProductoUseCase {
  constructor(@Inject(PRODUCTO_REPOSITORY_TOKEN) private readonly repo: IProductoRepository) {}

  async execute(dto: CrearProductoDto): Promise<Producto> {
    // ProductosFactory porque Producto es una entidad compuesta (sección 1.2.2) —
    // si fuera simple, acá iría Proveedor.create(dto) directo.
    const producto = ProductosFactory.create(dto);
    return this.repo.save(producto);
  }
}
```

**Regla de mapeo de respuesta:** la forma en la que se devuelve el dato al controller (qué campos, qué shape) se decide en el caso de uso, no en el controller. El controller queda limpio, solo delega y devuelve lo que el caso de uso ya armó.

**Cuándo NO hace falta esta capa:** en Nivel 1 (sección 0), un `Service` simple sin caso de uso separado es suficiente. No fragmentar en `UseCase` + `Facade` una operación CRUD trivial solo por seguir la plantilla al pie de la letra.

### 2.2.1. Application Exceptions

No toda excepción de negocio es una invariante que proteja una entidad o un Value Object. "No existe la categoría con ese id", "ya existe un producto con ese nombre", "el producto está inactivo" son reglas que se verifican consultando el repositorio desde el caso de uso (o un servicio de aplicación auxiliar) — no algo que `Producto.create()` pueda validar por sí solo. Esas van en `application/exceptions/`, extendiendo `ApplicationException` en vez de `DomainException`:

```typescript
// common/application/exceptions/application-exception.ts
import { AppException } from '../../exceptions/app-exception';
import { AppErrorCode } from '../../exceptions/app-error-code.enum';

export class ApplicationException extends AppException {
  constructor(message: string, code: AppErrorCode) {
    super(message, code);
  }
}

// application/exceptions/producto-no-encontrado.exception.ts
import { ApplicationException } from '../../../../common/application/exceptions/application-exception';
import { AppErrorCode } from '../../../../common/exceptions/app-error-code.enum';

export class ProductoNoEncontradoException extends ApplicationException {
  constructor() {
    super('Producto no encontrado', AppErrorCode.NOT_FOUND);
  }
}
```

`DomainException` y `ApplicationException` son hermanas, ambas extienden `AppException` sin agregarle nada — la distinción es de carpeta e import, no de comportamiento. El filtro de la sección 3.2 captura `AppException` en general, así que no importa cuál de las dos se lance: el mapeo a HTTP es el mismo mecanismo para las dos.

**Dónde va la validación de existencia/duplicados que se repite entre casos de uso:** cuando dos o más casos de uso del mismo módulo necesitan la misma verificación (por ejemplo, `existsByNombre` antes de crear y antes de actualizar), conviene extraerla a un servicio de aplicación propio (`application/services/`, p. ej. `ProductoExistenceValidator`) en vez de repetir el `if` en cada caso de uso.

### 2.3. Facade vs CQRS vs Servicio directo

Tres formas válidas de exponer casos de uso al controller. Elegir según nivel de complejidad (sección 0):

| Opción | Cuándo usarla | Costo |
|---|---|---|
| **Servicio directo** (Nivel 1) | CRUD simple, sin casos de uso separados | Ninguno — es lo más simple |
| **Fachada de Aplicación** (Nivel 2/3, elegida por default en este precedente) | Varios casos de uso por módulo, se quiere mantener el controller limpio sin usar CQRS | Un archivo `*.facade.ts` por responsabilidad |
| **CQRS** (`@nestjs/cqrs`) | Sistemas grandes donde ya se necesita separación físicia comando/query, eventos de dominio, o el equipo ya tiene experiencia con el patrón | Curva de aprendizaje y "magia" de resolución implícita de handlers |

**Default de este precedente: Fachada de Aplicación.** Ver `DECISIONS.md` para el razonamiento completo. CQRS es una opción válida, no descartada — se documenta la elección para no tener que re-derivarla en cada proyecto, no para prohibir la alternativa.

**Regla al crecer con Fachada:** si una única fachada por módulo empieza a acumular demasiadas responsabilidades (alta de productos, ajustes de stock, precios, todo junto), dividir en **fachadas específicas por responsabilidad** dentro del mismo módulo:

```
application/facades/
  ├── producto-maintenance.facade.ts   → alta/baja/modificación
  └── producto-stock.facade.ts          → ajustes de inventario
```

### 2.4. Orquestación cross-módulo: `processes/` vs `forwardRef()`

Cuando una acción de negocio cruza dos subdominios (ej.: una venta descuenta stock), **la fachada de un módulo nunca llama directo a la fachada de otro módulo** — eso genera acoplamiento circular. La orquestación va en un caso de uso dedicado.

**Nivel 2 (necesidad puntual, aislada):**

```typescript
// dentro del módulo Ventas, usando forwardRef hacia Stock
@Injectable()
export class ProcesarVentaUseCase {
  constructor(
    private readonly crearFactura: CrearFacturaUseCase,       // propio de Ventas
    @Inject(forwardRef(() => DescontarStockUseCase))
    private readonly descontarStock: DescontarStockUseCase,   // de Stock, vía forwardRef
  ) {}

  async execute(dto: VentaDto) {
    await this.crearFactura.execute(dto);
    await this.descontarStock.execute(dto.productoId, dto.cantidad);
  }
}
```

**Nivel 3 (orquestación recurrente entre varios módulos):**

A diferencia del `forwardRef()` de Nivel 2, el caso de uso de `processes/` **no inyecta los casos de uso de los módulos de dominio** (`CrearFacturaUseCase`, `DescontarStockUseCase`) — inyecta **los repositorios** de cada módulo directo, y reimplementa ahí la porción de lógica que necesita. La razón de este cambio no es de estilo: si inyectara los casos de uso de Ventas y de Stock, el módulo de `processes/` tendría que importar `VentasModule` y `StockModule` completos (con sus fachadas y controllers) — y si el controller de alguno de esos módulos necesita exponer la operación orquestada (lo normal: el endpoint `POST /ventas` tiene que terminar llamando a `ProcesarVentaUseCase`), ese módulo de dominio va a necesitar importar de vuelta el módulo de `processes/`. Ahí está el ciclo: `processes/` → `VentasModule` → `processes/`.

```typescript
// modules/ventas/infrastructure/persistence/ventas-persistence.module.ts
// módulo HOJA: solo cablea el token del repositorio, sin casos de uso, sin fachada, sin controller
@Module({
  providers: [{ provide: VENTA_REPOSITORY_TOKEN, useClass: VentaPrismaRepository }],
  exports: [VENTA_REPOSITORY_TOKEN],
})
export class VentasPersistenceModule {}

// modules/stock/infrastructure/persistence/stock-persistence.module.ts
@Module({
  providers: [{ provide: STOCK_REPOSITORY_TOKEN, useClass: StockPrismaRepository }],
  exports: [STOCK_REPOSITORY_TOKEN],
})
export class StockPersistenceModule {}
```

```typescript
// processes/venta-lifecycle/procesar-venta.use-case.ts
// vive en un módulo NEUTRAL, fuera de Ventas y de Stock
@Injectable()
export class ProcesarVentaUseCase {
  constructor(
    @Inject(UNIT_OF_WORK_TOKEN) private readonly uow: IUnitOfWork,
    @Inject(VENTA_REPOSITORY_TOKEN) private readonly ventaRepo: IVentaRepository,
    @Inject(STOCK_REPOSITORY_TOKEN) private readonly stockRepo: IStockRepository,
  ) {}

  async execute(dto: VentaDto) {
    await this.uow.runInTransaction(async () => {
      const stock = await this.stockRepo.findByProductoId(dto.productoId);
      if (!stock || !stock.hayDisponible(dto.cantidad)) {
        throw new StockInsuficienteException(dto.productoId);
      }

      const venta = VentasFactory.create(dto);
      await this.ventaRepo.save(venta);

      stock.descontar(dto.cantidad); // lógica de negocio en la ENTIDAD Stock
      await this.stockRepo.save(stock);
    });
  }
}

// processes/venta-lifecycle/venta-lifecycle.module.ts
@Module({
  imports: [
    VentasPersistenceModule,  // módulo HOJA de Ventas — no VentasModule completo
    StockPersistenceModule,   // módulo HOJA de Stock — no StockModule completo
  ],
  providers: [ProcesarVentaUseCase, ProcesarVentaFacade, { provide: PROCESAR_VENTA_PORT, useClass: ProcesarVentaFacade }],
  exports: [PROCESAR_VENTA_PORT],
})
export class VentaLifecycleModule {}
```

```typescript
// modules/ventas/ventas.module.ts
@Module({
  imports: [
    VentasPersistenceModule, // el propio módulo hoja, para los casos de uso que ya tenía Ventas (listar, buscar)
    VentaLifecycleModule,    // el módulo de processes/ — importado en UN SOLO SENTIDO
  ],
  controllers: [VentasController], // inyecta PROCESAR_VENTA_PORT además de su propia fachada
  // ...
})
export class VentasModule {}
```

**Por qué esto no genera ciclo:** `VentaLifecycleModule` solo depende de los módulos hoja de persistencia (`VentasPersistenceModule`, `StockPersistenceModule`), que no dependen de nada — no exportan casos de uso ni fachadas, solo el token del repositorio. `VentasModule` depende de `VentaLifecycleModule`, pero `VentaLifecycleModule` nunca depende de `VentasModule` de vuelta. El grafo de imports queda en una sola dirección (un DAG), nunca un ciclo.

Si el caso de uso orquestador necesita reutilizar algo del módulo de dominio que no sea un provider de Nest — una excepción, un mapper de respuesta, una entidad — se importa directo como clase de TypeScript (`import { CategoriaNoEncontradaException } from '../../modules/categorias/application/exceptions/...'`). Eso no agrega una arista al grafo de módulos: solo `@Module({ imports: [...] })` puede generar un ciclo, no un `import` de TypeScript a un archivo que no está decorado como módulo ni registrado como provider en otro lado.

La diferencia con el snippet de Nivel 2 es más que cosmética: no es solo "sacar el `forwardRef`, mover el archivo" — también cambia **qué se inyecta** (repositorios de cada dominio en vez de los casos de uso del otro módulo). Migrar de Nivel 2 a Nivel 3 implica reescribir esa porción de lógica usando los repositorios directo, no solo mover el archivo de carpeta.

### 2.4.1. Esto no aplica a puertos de infraestructura sin clases de dominio

`forwardRef()`/`processes/` resuelven un problema específico: dos módulos de **dominio** que se necesitan mutuamente y podrían generar una dependencia circular. No es el mismo problema que un caso de uso llamando a un puerto de infraestructura sin lógica de negocio propia (envío de emails, notificaciones push, un gateway de pagos) — eso es una dependencia normal en un solo sentido, sin riesgo de ciclo, igual que inyectar un repositorio.

**Señal para distinguirlos:** si el módulo del otro lado no tiene entidades ni invariantes de dominio propias — solo implementa un puerto (`IEmailSender`, `IPushNotifier`) — cualquier caso de uso que lo necesite lo inyecta directo, sin pasar por `forwardRef()` ni por `processes/`. Esto es frecuente cuando una acción de negocio dispara una notificación: el caso de uso que la origina (aprobar un cobro, cancelar una membresía) llama al puerto al final de su `execute()`, igual que llamaría al repositorio.

**Cuándo esto sí se convierte en el problema de la sección 2.4:** si el "módulo de notificaciones" empieza a acumular lógica propia (reintentos con estado, una bandeja de notificaciones leídas/no leídas, reglas de cuándo reenviar) deja de ser un puerto sin clases de dominio y pasa a ser un módulo de dominio más — ahí sí aplican las reglas de orquestación cross-módulo de esta sección.

### 2.5. Unit of Work — cuándo aplicarlo

**Regla de decisión:** Unit of Work se justifica únicamente cuando una operación combina **más de una escritura que deben ser atómicas** (todas o ninguna). Si un módulo o caso de uso no tiene ese escenario, el caso de uso llama al repositorio inyectado directo — sin ninguna capa de UoW encima.

El caso de uso que sí necesita UoW lo usa como un wrapper transaccional alrededor de su lógica, sin pedirle los repositorios — esos se inyectan como siempre, directo en el constructor:

```typescript
@Injectable()
export class ActualizarPrecioProductoUseCase {
  constructor(
    @Inject(UNIT_OF_WORK_TOKEN) private readonly uow: IUnitOfWork,
    @Inject(PRODUCTO_REPOSITORY_TOKEN) private readonly repo: IProductoRepository,
  ) {}

  async execute(productoId: string, nuevoPrecio: number): Promise<void> {
    await this.uow.runInTransaction(async () => {
      const producto = await this.repo.findById(productoId);
      if (!producto) throw new ProductoNoEncontradoException();

      producto.actualizarPrecio(nuevoPrecio);
      await this.repo.save(producto);
    });
  }
}
```

El repositorio, por dentro, sabe automáticamente si hay una transacción activa y la usa — el caso de uso no se entera del mecanismo. Cómo logra eso el repositorio (contexto ambiental con `AsyncLocalStorage`) está en `IMPLEMENTATION.md` — no se repite acá para no duplicar contenido que puede desactualizarse en un solo lugar.

### 2.6. Checklist — Capa de Aplicación

- [ ] Se evaluó el nivel de complejidad (sección 0) antes de decidir si hace falta caso de uso separado, fachada, o UoW.
- [ ] Ningún caso de uso importa una librería de ORM directamente (salvo que el proyecto haya decidido explícitamente saltear el patrón Repository — no es el default de este precedente).
- [ ] Si hay orquestación cross-módulo: se usó `forwardRef()` (puntual) o `processes/` (recurrente), nunca una fachada llamando a otra fachada de otro módulo — salvo que el otro lado sea un puerto de infraestructura sin clases de dominio (sección 2.4.1).
- [ ] Si se usó `processes/`: el caso de uso orquestador inyecta los **repositorios** de cada módulo (vía su submódulo hoja de persistencia), nunca los casos de uso o la fachada de esos módulos — eso reintroduciría el ciclo que `processes/` existe para evitar.
- [ ] El shape de la respuesta se arma en el caso de uso, no en el controller.
- [ ] Las excepciones de negocio que no son invariantes de dominio extienden `ApplicationException` (sección 2.2.1), no se lanza un `Error` genérico ni se reusa `DomainException` por comodidad.

---

## 3. Capa de Infraestructura

Esta capa depende de Aplicación y Dominio, nunca al revés — implementa los contratos (interfaces) definidos en las capas internas.

### 3.1. Controllers

```typescript
@Controller('productos')
export class ProductoController {
  constructor(private readonly facade: ProductoFacade) {}

  @Post()
  async crear(@Body() dto: CrearProductoDto) {
    return this.facade.crear(dto);
  }
}
```

El controller inyecta la fachada (o el servicio, en Nivel 1) **directo, como clase concreta** — no hace falta una interface/puerto de aplicación entre el controller y la fachada (eso agregaría una capa de indirección sin otro consumidor que la necesite: a diferencia del puerto de persistencia, acá no hay una implementación alternativa que intercambiar). El controller solo delega, no arma lógica de negocio ni decide el shape de la respuesta. El `AppExceptionFilter` de la sección 3.2 se registra una sola vez, global (ver `main.ts`), no por controller.

### 3.2. Exception Filters

Un único filtro captura toda la jerarquía de `AppException` (domain + application, secciones 1.4 y 2.2.1) y resuelve el status HTTP con una tabla fija por `AppErrorCode` — no hay `instanceof` por subtipo de excepción:

```typescript
// common/exceptions/app-error-code.enum.ts  (repetido de la sección 1.4, vive en común)
export enum AppErrorCode {
  INVALID_INPUT = 'INVALID_INPUT',
  CONFLICT = 'CONFLICT',
  NOT_FOUND = 'NOT_FOUND',
}

// common/filters/status-by-code.ts
import { AppErrorCode } from '../exceptions/app-error-code.enum';

export const STATUS_BY_CODE: Record<AppErrorCode, number> = {
  [AppErrorCode.NOT_FOUND]: 404,
  [AppErrorCode.CONFLICT]: 409,
  [AppErrorCode.INVALID_INPUT]: 400,
};

// common/filters/app-exception.filter.ts
import { ArgumentsHost, Catch, ExceptionFilter } from '@nestjs/common';
import type { Response } from 'express';
import { AppException } from '../exceptions/app-exception';
import { STATUS_BY_CODE } from './status-by-code';

@Catch(AppException)
export class AppExceptionFilter implements ExceptionFilter {
  catch(exception: AppException, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse<Response>();
    const status = STATUS_BY_CODE[exception.code] ?? 500;

    response.status(status).json({
      message: exception.message,
      statusCode: status,
    });
  }
}
```

**Por qué no un `resolveStatus()` con `instanceof` por excepción:** ese enfoque obliga a tocar el filtro cada vez que se agrega una excepción nueva, y el mapeo a HTTP queda lejos de donde se define la excepción. Acá el `AppErrorCode` se declara una sola vez, en el constructor de la excepción misma (secciones 1.4 y 2.2.1) — el filtro es una tabla de 3 entradas que no crece con el número de excepciones, solo con el número de *categorías* de error (que son pocas y estables: entrada inválida, conflicto, no encontrado). Agregar `AppErrorCode.FORBIDDEN` o similar si hace falta un cuarto status es la única razón para tocar `status-by-code.ts`.

El dominio nunca sabe que existe HTTP: `AppErrorCode` vive en `common/exceptions/`, no en `common/domain/` — es un concepto compartido entre dominio, aplicación e infraestructura, no exclusivo del dominio.

### 3.3. Persistencia (ORM, mappers, repositorios, Unit of Work)

Ver `IMPLEMENTATION.md` para la guía completa con Prisma.

### 3.4. Checklist — Capa de Infraestructura

- [ ] El controller no contiene lógica de negocio ni arma el shape de la respuesta, e inyecta la fachada como clase concreta (sin puerto de aplicación intermedio).
- [ ] Hay un único `AppExceptionFilter`, registrado global, que mapea `AppErrorCode` → status HTTP con una tabla fija, sin lógica por excepción.
- [ ] La implementación de persistencia sigue `IMPLEMENTATION.md`.

---

## 4. Testing

- **Casos de uso / servicios:** test unitario, con el repositorio **mockeado** (nunca contra una base real) — se testea la orquestación y las reglas de negocio en aislamiento.
- **Repositorios concretos (Prisma):** test de integración contra una base real (Docker), sin mockear el ORM — acá sí importa que el SQL generado sea correcto.
- **Endpoints completos:** test e2e con `supertest`, contra la app levantada con su `ValidationPipe` y filtros reales.

---

## 5. Gotchas de TypeScript acumulados en este precedente

- **`satisfies` no aplica a declaraciones de clase.** Solo funciona sobre expresiones (objetos literales, arrays, etc.) — no lo intentes sobre `class X implements Y { }`.
- **Estado transitorio en servicios singleton es un riesgo de concurrencia.** Un provider de Nest es singleton por default — guardar estado por-usuario/por-request como propiedad de instancia (`this.currentUser = ...`) corrompe datos bajo carga concurrente. El estado transitorio va en el request/response, no en la instancia del servicio.
- **La complejidad debe ganarse su lugar.** Si una acción es simple, el código tiene que reflejar esa simplicidad — no agregar capas, abstracciones o archivos "porque la plantilla los tiene", ver sección 0.
- **El contexto de `AsyncLocalStorage` del Unit of Work (ver `IMPLEMENTATION.md`) se puede perder.** Si dentro de una transacción el código dispara trabajo sin esperarlo (`setTimeout`, un evento fire-and-forget, una promesa no awaiteada) o usa alguna librería que no propaga el contexto de ejecución de Node, ese código ya no ve la transacción activa y termina leyendo o escribiendo fuera de ella sin ningún error visible. Dentro de un `runInTransaction()`, todo el trabajo relevante se `await`ea en la misma cadena de llamadas.

---

## 6. Checklist maestro para iniciar un módulo nuevo

1. Definir el nivel de complejidad del módulo (sección 0).
2. Escribir la(s) entidad(es) de dominio: ¿simple (`create()`/`reconstitute()` propios) o compuesta (Factory en `domain/factories/`)? (sección 1.2.2), con sus inputs en `domain/inputs/`, Value Objects si corresponde, y excepciones propias (`DomainException`, sección 1.4).
3. Escribir la interfaz del repositorio + su token de inyección. Si el módulo va a participar de `processes/` (ahora o previsiblemente pronto), cablear ese token en su propio submódulo hoja de persistencia (`<modulo>-persistence.module.ts`), separado del `<modulo>.module.ts` principal — ver sección 2.4.
4. Decidir: ¿Servicio directo, o Casos de Uso + Fachada? (sección 2.3) Las excepciones de orquestación (no encontrado, duplicados) van como `ApplicationException` (sección 2.2.1).
5. Si hay operaciones que cruzan escrituras atómicas: ¿hace falta Unit of Work? (sección 2.5, detalle en `IMPLEMENTATION.md`)
6. Si hay orquestación cross-módulo: ¿`forwardRef()` puntual o `processes/` recurrente? (sección 2.4; sección 2.4.1 si el otro lado es un puerto sin clases de dominio). Con `processes/`, el módulo nuevo importa su submódulo hoja de persistencia desde el módulo de `processes/` — nunca el módulo principal del otro dominio.
7. Implementar infraestructura (schema, mapper, repositorio concreto) siguiendo `IMPLEMENTATION.md`.
8. Controller (inyectando la fachada directo, sin puerto intermedio) + DTOs. El `AppExceptionFilter` ya está registrado global, no hace falta nada por módulo.
9. Tests: unitarios de casos de uso (repo mockeado), integración de repositorio (base real), e2e de los endpoints.