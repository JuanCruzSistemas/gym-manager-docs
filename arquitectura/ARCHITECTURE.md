# Arquitectura de Referencia: DDD + Hexagonal en NestJS

## Cómo usar este documento

Este es el documento de precedente arquitectónico para cualquier proyecto NestJS nuevo (o módulo nuevo dentro de uno existente) que vaya a seguir Domain-Driven Design + Arquitectura Hexagonal. Reemplaza cualquier README anterior de proyectos puntuales — es agnóstico a cualquier dominio de negocio específico.

Forma parte de un set de tres documentos:
- **`ARCHITECTURE.md`** (este archivo): reglas de las tres capas, checklists, niveles de complejidad.
- **`IMPLEMENTATION.md`**: guía concreta de infraestructura/ORM (TypeORM y Prisma en paralelo).
- **`DECISIONS.md`**: el razonamiento detrás de cada regla — trade-offs, alternativas descartadas, y por qué. Consultar cuando una regla de acá no quede clara o alguien quiera cuestionarla.

Si sos un agente de terminal arrancando un proyecto nuevo: leé primero la sección 0 para decidir el nivel de complejidad, después la sección de checklist de la capa que estés tocando. No hace falta leer `DECISIONS.md` para aplicar las reglas — solo para entender por qué existen.

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
- **Unit of Work** (ver `IMPLEMENTATION.md` para las dos variantes posibles).
- **Carpeta `processes/`** para casos de uso orquestadores cross-módulo, en vez de `forwardRef()` disperso.
- **Fachadas específicas por responsabilidad**, no una fachada gigante por módulo (ver sección 2.3).

**Regla de decisión rápida entre Nivel 2 y Nivel 3, sobre `processes/` vs `forwardRef`:** si ya tenés o preveés **más de dos o tres** casos de uso orquestadores cruzando los mismos módulos, migrar a `processes/`. Si es una necesidad puntual y aislada, `forwardRef()` es más simple y no hace falta pagar el costo de la carpeta compartida todavía.

> **Importante:** el nivel se evalúa **por módulo o por necesidad puntual**, no necesariamente por proyecto entero. Un proyecto grande puede tener módulos en Nivel 1 (un catálogo de categorías simple) conviviendo con módulos en Nivel 3 (ventas que cruzan stock y facturación). No hay que subir de nivel todo el proyecto solo porque una parte lo necesita.

---

## 1. Capa de Dominio

### 1.1. Regla de dependencia

**Prohibido** importar `@nestjs/*`, `typeorm`, `@prisma/client`, o cualquier librería de infraestructura/framework dentro de `domain/`. Si un archivo de dominio necesita importar algo así, la lógica está mal ubicada.

### 1.2. Entities

```typescript
// domain/entities/producto.entity.ts
import { ProductoPrice } from '../value-objects/producto-price.vo';

export class Producto {
  private constructor(
    private readonly id: string | null,
    private nombre: string,
    private precio: ProductoPrice,
  ) {}

  // Fábrica para un objeto NUEVO — valida invariantes de creación
  static create(nombre: string, precio: number): Producto {
    return new Producto(null, nombre, ProductoPrice.create(precio));
  }

  // Fábrica para REHIDRATAR desde persistencia — la usa SOLO el mapper
  static reconstitute(id: string, nombre: string, precio: number): Producto {
    return new Producto(id, nombre, ProductoPrice.create(precio));
  }

  actualizarPrecio(nuevoPrecio: number): void {
    this.precio = ProductoPrice.create(nuevoPrecio);
  }

  getId(): string | null { return this.id; }
  getNombre(): string { return this.nombre; }
  getPrecio(): number { return this.precio.getValue(); }
}
```

**Regla:** `create()` valida las condiciones de un objeto que nace ahora. `reconstitute()` reconstruye un objeto que ya existía y ya pasó esa validación en su momento — **nunca** lo usa código de negocio, solo el mapper de infraestructura. Confundir los dos (usar `create()` para rehidratar) es un error de diseño, no un detalle de estilo.

### 1.3. Value Objects

Usar Value Objects cuando un dato primitivo tiene una regla de validación propia que se repetiría en varios lugares si no se encapsula.

```typescript
export class ProductoPrice {
  private constructor(private readonly value: number) {}

  static create(value: number): ProductoPrice {
    if (value < 0) throw new InvalidProductoPriceException(value);
    return new ProductoPrice(value);
  }

  getValue(): number { return this.value; }
}
```

**Cuándo NO hace falta un Value Object:** si el dato no tiene ninguna regla de validación más allá del tipo (`string`, `number`), no se justifica envolverlo. No todo campo necesita ser un VO.

### 1.4. Domain Exceptions

```typescript
export class DomainException extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'DomainException';
  }
}

export class InvalidProductoPriceException extends DomainException {
  constructor(precio: number) {
    super(`El precio (${precio}) no puede ser negativo.`);
  }
}
```

Todas las excepciones de negocio heredan de una base común (`DomainException`) para que un único filtro de infraestructura (sección 3) las pueda capturar a todas.

### 1.5. Checklist — Capa de Dominio

- [ ] La entidad tiene `create()` y `reconstitute()` separados.
- [ ] Ningún archivo de `domain/` importa una librería de framework/infraestructura.
- [ ] Las reglas de negocio (invariantes) están en la entidad o en un Value Object, no en el repositorio ni en el caso de uso.
- [ ] Las excepciones de negocio heredan de una base común.

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
    const producto = Producto.create(dto.nombre, dto.precio);
    return this.repo.save(producto);
  }
}
```

**Regla de mapeo de respuesta:** la forma en la que se devuelve el dato al controller (qué campos, qué shape) se decide en el caso de uso, no en el controller. El controller queda limpio, solo delega y devuelve lo que el caso de uso ya armó.

**Cuándo NO hace falta esta capa:** en Nivel 1 (sección 0), un `Service` simple sin caso de uso separado es suficiente. No fragmentar en `UseCase` + `Facade` una operación CRUD trivial solo por seguir la plantilla al pie de la letra.

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

```typescript
// processes/venta-lifecycle/procesar-venta.use-case.ts
// vive en un módulo NEUTRAL, fuera de Ventas y de Stock — ninguno de los dos
// depende del otro directamente
@Injectable()
export class ProcesarVentaUseCase {
  constructor(
    private readonly crearFactura: CrearFacturaUseCase,
    private readonly descontarStock: DescontarStockUseCase,
  ) {}

  async execute(dto: VentaDto) {
    await this.crearFactura.execute(dto);
    await this.descontarStock.execute(dto.productoId, dto.cantidad);
  }
}
```

La diferencia entre ambos snippets es mínima a propósito — **la migración de uno a otro es barata** cuando llega el momento (mover el archivo a `processes/`, sacar el `forwardRef`, importar el módulo neutral desde ambos lados). No hace falta anticiparse a Nivel 3 desde el día uno "por las dudas".

### 2.5. Unit of Work — cuándo aplicarlo

**Regla de decisión:** Unit of Work se justifica únicamente cuando una operación combina **más de una escritura que deben ser atómicas** (todas o ninguna). Si un módulo o caso de uso no tiene ese escenario, el caso de uso llama al repositorio inyectado directo — sin ninguna capa de UoW encima.

La implementación concreta (dos variantes posibles, con TypeORM y con Prisma) está en `IMPLEMENTATION.md` — no se repite acá para no duplicar contenido que puede desactualizarse en un solo lugar.

### 2.6. Checklist — Capa de Aplicación

- [ ] Se evaluó el nivel de complejidad (sección 0) antes de decidir si hace falta caso de uso separado, fachada, o UoW.
- [ ] Ningún caso de uso importa una librería de ORM directamente (salvo que el proyecto haya decidido explícitamente saltear el patrón Repository — no es el default de este precedente).
- [ ] Si hay orquestación cross-módulo: se usó `forwardRef()` (puntual) o `processes/` (recurrente), nunca una fachada llamando a otra fachada de otro módulo.
- [ ] El shape de la respuesta se arma en el caso de uso, no en el controller.

---

## 3. Capa de Infraestructura

Esta capa depende de Aplicación y Dominio, nunca al revés — implementa los contratos (interfaces) definidos en las capas internas.

### 3.1. Controllers

```typescript
@Controller('productos')
@UseFilters(DomainExceptionFilter)
export class ProductoController {
  constructor(private readonly facade: ProductoFacade) {}

  @Post()
  async crear(@Body() dto: CrearProductoDto) {
    return this.facade.crear(dto);
  }
}
```

El controller solo inyecta la fachada (o el servicio, en Nivel 1) y delega — no arma lógica de negocio ni decide el shape de la respuesta.

### 3.2. Exception Filters

```typescript
@Catch(DomainException)
export class DomainExceptionFilter implements ExceptionFilter {
  catch(exception: DomainException, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse();
    const status = this.resolveStatus(exception);
    response.status(status).json({
      statusCode: status,
      error: exception.name,
      message: exception.message,
      timestamp: new Date().toISOString(),
    });
  }

  private resolveStatus(exception: DomainException): number {
    if (exception instanceof ProductoNotFoundException) return HttpStatus.NOT_FOUND;
    if (exception instanceof InvalidProductoPriceException) return HttpStatus.UNPROCESSABLE_ENTITY;
    return HttpStatus.BAD_REQUEST;
  }
}
```

Un único filtro captura toda la jerarquía de `DomainException` y mapea cada subtipo a su código HTTP correspondiente — el dominio nunca sabe que existe HTTP.

### 3.3. Persistencia (ORM, mappers, repositorios, Unit of Work)

Ver `IMPLEMENTATION.md` para la guía completa con TypeORM y Prisma en paralelo.

### 3.4. Checklist — Capa de Infraestructura

- [ ] El controller no contiene lógica de negocio ni arma el shape de la respuesta.
- [ ] Hay un Exception Filter que traduce excepciones de dominio a códigos HTTP, sin que el dominio conozca HTTP.
- [ ] La implementación de persistencia sigue `IMPLEMENTATION.md`.

---

## 4. Testing

- **Casos de uso / servicios:** test unitario, con el repositorio **mockeado** (nunca contra una base real) — se testea la orquestación y las reglas de negocio en aislamiento.
- **Repositorios concretos (TypeORM/Prisma):** test de integración contra una base real (Docker), sin mockear el ORM — acá sí importa que el SQL generado sea correcto.
- **Endpoints completos:** test e2e con `supertest`, contra la app levantada con su `ValidationPipe` y filtros reales.

---

## 5. Gotchas de TypeScript acumulados en este precedente

- **`satisfies` no aplica a declaraciones de clase.** Solo funciona sobre expresiones (objetos literales, arrays, etc.) — no lo intentes sobre `class X implements Y { }`.
- **Estado transitorio en servicios singleton es un riesgo de concurrencia.** Un provider de Nest es singleton por default — guardar estado por-usuario/por-request como propiedad de instancia (`this.currentUser = ...`) corrompe datos bajo carga concurrente. El estado transitorio va en el request/response, no en la instancia del servicio.
- **La complejidad debe ganarse su lugar.** Si una acción es simple, el código tiene que reflejar esa simplicidad — no agregar capas, abstracciones o archivos "porque la plantilla los tiene", ver sección 0.

---

## 6. Checklist maestro para iniciar un módulo nuevo

1. Definir el nivel de complejidad del módulo (sección 0).
2. Escribir la(s) entidad(es) de dominio con `create()`/`reconstitute()`, Value Objects si corresponde, y excepciones propias.
3. Escribir la interfaz del repositorio + su token de inyección.
4. Decidir: ¿Servicio directo, o Casos de Uso + Fachada? (sección 2.3)
5. Si hay operaciones que cruzan escrituras atómicas: ¿hace falta Unit of Work? (sección 2.5, detalle en `IMPLEMENTATION.md`)
6. Si hay orquestación cross-módulo: ¿`forwardRef()` puntual o `processes/` recurrente? (sección 2.4)
7. Implementar infraestructura (entidad ORM/schema, mapper, repositorio concreto) siguiendo `IMPLEMENTATION.md`.
8. Controller + DTOs + Exception Filter.
9. Tests: unitarios de casos de uso (repo mockeado), integración de repositorio (base real), e2e de los endpoints.
