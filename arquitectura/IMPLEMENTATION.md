# Guía de Implementación: Capa de Infraestructura (Prisma)

> Parte del set de precedente arquitectónico junto a `ARCHITECTURE.md` (reglas de las tres capas) y `DECISIONS.md` (razonamiento/trade-offs). Este documento asume que ya se leyó la sección 0 de `ARCHITECTURE.md` (niveles de complejidad) — en particular, si el módulo no necesita Unit of Work, saltear directo a las secciones 3.1–3.3 de este documento e ignorar las secciones de UoW.
>
> El ORM de este precedente es **Prisma** (ver `DECISIONS.md` §8). No se documenta una alternativa en paralelo.

## Dominio de ejemplo usado en este documento

`Producto` y `Proveedor`, con relación 1 a N (un proveedor provee muchos productos) — mismo dominio de referencia que el resto de la documentación del proyecto.

```
Proveedor (1) ──────< (N) Producto
  id                      id
  nombre                  nombre
                          precio
                          proveedorId (FK)
```

---

## 1. Estructura de carpetas

```text
src/
└── modules/
    └── productos/
        ├── domain/
        │   ├── entities/
        │   ├── factories/              # solo entidades compuestas — ver ARCHITECTURE.md §1.2.2
        │   ├── inputs/                 # CreateXInput / ReconstituteXInput — ver ARCHITECTURE.md §1.2.3
        │   ├── value-objects/
        │   ├── repository/
        │   └── exceptions/
        ├── application/
        │   ├── facades/
        │   ├── use-cases/
        │   ├── dtos/
        │   └── exceptions/              # ApplicationException — ver ARCHITECTURE.md §2.2.1
        ├── infrastructure/              # ← ESTA GUÍA CUBRE SOLO ESTA CARPETA
        │   ├── controllers/
        │   └── persistence/
        │       ├── prisma/
        │       └── productos-persistence.module.ts   # submódulo hoja — ver nota abajo
        └── productos.module.ts
```

> **El `productos-persistence.module.ts` es su propio módulo de Nest, separado de `productos.module.ts`.** Solo cablea `{ provide: PRODUCTO_REPOSITORY_TOKEN, useClass: ProductoPrismaRepository }` y exporta el token — sin casos de uso, sin fachada, sin controller. `productos.module.ts` lo importa para armar sus propios casos de uso, **y cualquier módulo de `processes/` que necesite el repositorio de Productos lo importa directo a él**, nunca a `productos.module.ts`. Esta separación es la que evita el ciclo de imports cuando el módulo participa de orquestación cross-módulo — el razonamiento completo está en `ARCHITECTURE.md` §2.4. Si el módulo nunca va a participar de `processes/`, esta separación es opcional: cablear el repositorio directo en `productos.module.ts` alcanza (ver sección 3.5 más abajo para ambos casos).

---

## 2. Punto de partida (dominio y aplicación, ya definidos antes de tocar infraestructura)

`Proveedor` es una entidad simple (ARCHITECTURE.md §1.2.1): sin Value Objects propios, `create()`/`reconstitute()` en la propia clase.

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
import { NombreProveedorInvalidoException } from '../exceptions/nombre-proveedor-invalido.exception';

export class Proveedor {
  private constructor(
    private readonly id: string | null,
    private nombre: string,
  ) {}

  static create(data: CreateProveedorInput): Proveedor {
    if (!data.nombre?.trim()) throw new NombreProveedorInvalidoException();
    return new Proveedor(null, data.nombre);
  }

  static reconstitute(data: ReconstituteProveedorInput): Proveedor {
    return new Proveedor(data.id, data.nombre);
  }

  getId(): string | null { return this.id; }
  getNombre(): string { return this.nombre; }
}
```

`Producto` es una entidad compuesta (ARCHITECTURE.md §1.2.2): instancia un Value Object propio (`ProductoPrice`), así que `create()`/`reconstitute()` no viven en la clase sino en `ProductosFactory`.

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

```typescript
// domain/repository/proveedor.repository.interface.ts
import { Proveedor } from '../entities/proveedor.entity';

export interface IProveedorRepository {
  save(proveedor: Proveedor): Promise<Proveedor>;
  findById(id: string): Promise<Proveedor | null>;
}

export const PROVEEDOR_REPOSITORY_TOKEN = Symbol('IProveedorRepository');
```

```typescript
// domain/repository/producto.repository.interface.ts
import { Producto } from '../entities/producto.entity';

export interface IProductoRepository {
  save(producto: Producto): Promise<Producto>;
  findById(id: string): Promise<Producto | null>;
  findByProveedorId(proveedorId: string): Promise<Producto[]>;
}

export const PRODUCTO_REPOSITORY_TOKEN = Symbol('IProductoRepository');
```

### Contrato de Unit of Work — contexto ambiental con `AsyncLocalStorage`

Aplicar solo si el módulo lo necesita (ver `ARCHITECTURE.md` §2.5). El contrato es deliberadamente mínimo — no expone forma de obtener repositorios, porque los repositorios no se piden al UoW: son providers comunes que detectan solos si hay una transacción activa.

```typescript
// common/application/unit-of-work/unit-of-work.interface.ts
export const UNIT_OF_WORK_TOKEN = Symbol('IUnitOfWork');

export interface IUnitOfWork {
  runInTransaction<T>(work: () => Promise<T>): Promise<T>;
}
```

**Por qué no un método por repositorio (o uno genérico por token):** esas dos formas existen para resolver el mismo problema — que el repositorio use el cliente/manager de la transacción en vez del global — pasándoselo explícitamente desde el UoW. `AsyncLocalStorage` (módulo nativo de Node) resuelve lo mismo sin pasar nada: publica el cliente activo en un contexto ambiental que cualquier código que corra dentro de esa cadena de `await` puede leer, sin que nadie se lo pase por parámetro. El repositorio deja de necesitar que lo construya el UoW — es un provider más, igual que cualquier otro, y funciona igual esté o no dentro de una transacción.

---

## 3. Prisma

### 3.1. Schema declarativo

```prisma
// prisma/schema.prisma
model Proveedor {
  id        String     @id @default(uuid())
  nombre    String
  productos Producto[]

  @@map("proveedores")
}

model Producto {
  id          String    @id @default(uuid())
  nombre      String
  precio      Decimal   @db.Decimal(10, 2)
  proveedorId String    @map("proveedor_id")
  proveedor   Proveedor @relation(fields: [proveedorId], references: [id])

  @@map("productos")
}
```

Después de editar el schema: `npx prisma migrate dev --name init` — genera la migración SQL, la aplica, y regenera el cliente tipado automáticamente. No se escribe una clase de entidad ORM a mano.

> **Nota de versión:** a partir de Prisma 7, `PrismaClient` requiere un **driver adapter** explícito (no trae motor de queries embebido por default) y la URL de conexión vive en `prisma.config.ts`. Confirmar la versión del proyecto antes de asumir que estos detalles no cambiaron — Prisma modifica esto con cierta frecuencia entre majors.

### 3.2. Mappers

```typescript
// infrastructure/persistence/prisma/mappers/producto.mapper.ts
import { Producto } from '../../../../domain/entities/producto.entity';
import type { Producto as ProductoPrisma } from '../../../../generated/prisma/client';

export class ProductoMapper {
  static toPersistence(producto: Producto): {
    nombre: string;
    precio: number;
    proveedorId: string;
  } {
    return {
      nombre: producto.getNombre(),
      precio: producto.getPrecio(),
      proveedorId: producto.getProveedorId(),
    };
  }

  static toDomain(row: ProductoPrisma): Producto {
    return ProductosFactory.reconstitute({
      id: row.id,
      nombre: row.nombre,
      precio: Number(row.precio),
      proveedorId: row.proveedorId,
    });
  }
}
```

Prisma no tiene clases de entidad ORM, pero el mapper **igual hace falta**: el objeto que devuelve Prisma es una estructura plana de datos, no la entidad de dominio con su lógica de negocio. Para `Producto` (entidad compuesta), el mapper llama a `ProductosFactory.reconstitute()`, no a la entidad directamente — es la Factory la que sabe reconstruir el Value Object `ProductoPrice`. Para una entidad simple como `Proveedor`, el mapper llamaría a `Proveedor.reconstitute(...)` en su propia clase, igual que antes.

### 3.3. El `Gestor`: quién publica el cliente activo

Antes del repositorio concreto hace falta la pieza que hace posible el contrato de UoW de la sección 2: un wrapper sobre `PrismaClient` que, durante una transacción, publica el `Prisma.TransactionClient` activo en un contexto ambiental (`AsyncLocalStorage`). Cualquier repositorio puede leerlo con un simple getter, sin que nadie se lo pase por parámetro.

```typescript
// common/infrastructure/prisma-uow/gestor-prisma-client.ts
import { AsyncLocalStorage } from 'node:async_hooks';
import { Injectable } from '@nestjs/common';
import { PrismaClient, Prisma } from '../../../generated/prisma/client';

const transactionStore = new AsyncLocalStorage<Prisma.TransactionClient>();

@Injectable()
export class GestorPrismaClient {
  constructor(private readonly prisma: PrismaClient) {}

  // El cliente activo: el de la transacción en curso, o el global si no hay ninguna
  get actual(): PrismaClient | Prisma.TransactionClient {
    return transactionStore.getStore() ?? this.prisma;
  }

  get inActualTransaction(): boolean {
    return transactionStore.getStore() !== undefined;
  }

  async runInTransaction<T>(work: () => Promise<T>): Promise<T> {
    if (this.inActualTransaction) return work(); // ya hay una tx activa, no anidar otra

    return this.prisma.$transaction((tx) => transactionStore.run(tx, work));
  }
}
```

```typescript
// common/application/unit-of-work/prisma-unit-of-work.ts
import { Injectable } from '@nestjs/common';
import { IUnitOfWork } from './unit-of-work.interface';
import { GestorPrismaClient } from '../../infrastructure/prisma-uow/gestor-prisma-client';

@Injectable()
export class PrismaUnitOfWork implements IUnitOfWork {
  constructor(private readonly gestor: GestorPrismaClient) {}

  runInTransaction<T>(work: () => Promise<T>): Promise<T> {
    return this.gestor.runInTransaction(work);
  }
}
```

`PrismaUnitOfWork` es una fachada delgada sobre `GestorPrismaClient` — existe solo para que el caso de uso dependa de `IUnitOfWork` (un contrato de aplicación) y no directamente de una pieza de infraestructura.

### 3.4. Repositorio concreto

```typescript
// infrastructure/persistence/prisma/repositories/producto.prisma.repository.ts
import { Injectable } from '@nestjs/common';
import { IProductoRepository } from '../../../../domain/repository/producto.repository.interface';
import { Producto } from '../../../../domain/entities/producto.entity';
import { ProductoMapper } from '../mappers/producto.mapper';
import { GestorPrismaClient } from '../../../../../../common/infrastructure/prisma-uow/gestor-prisma-client';

@Injectable()
export class ProductoPrismaRepository implements IProductoRepository {
  // No recibe el cliente/manager por parámetro — lo lee del Gestor en cada llamada
  constructor(private readonly gestor: GestorPrismaClient) {}

  async save(producto: Producto): Promise<Producto> {
    const data = ProductoMapper.toPersistence(producto);
    const row = producto.getId() === null
      ? await this.gestor.actual.producto.create({ data })
      : await this.gestor.actual.producto.update({ where: { id: producto.getId()! }, data });
    return ProductoMapper.toDomain(row);
  }

  async findById(id: string): Promise<Producto | null> {
    const row = await this.gestor.actual.producto.findUnique({ where: { id } });
    return row ? ProductoMapper.toDomain(row) : null;
  }

  async findByProveedorId(proveedorId: string): Promise<Producto[]> {
    const rows = await this.gestor.actual.producto.findMany({ where: { proveedorId } });
    return rows.map(ProductoMapper.toDomain);
  }
}
```

El repositorio es un provider normal de Nest — `@Injectable()`, sin estado propio, sin saber si está dentro de una transacción. `this.gestor.actual` resuelve eso solo.

### 3.5. Cableado del módulo (Prisma)

El `PrismaService` (extiende `PrismaClient`), el `GestorPrismaClient` y el token de Unit of Work se registran **una sola vez**, en un módulo global compartido por todo el proyecto — no por módulo de dominio.

```typescript
// common/infrastructure/prisma-uow/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '../../../generated/prisma/client';
import { PrismaPg } from '@prisma/adapter-pg';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  constructor() {
    super({ adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL! }) });
  }

  async onModuleInit() { await this.$connect(); }
  async onModuleDestroy() { await this.$disconnect(); }
}
```

```typescript
// common/infrastructure/prisma-uow/prisma-uow.module.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { GestorPrismaClient } from './gestor-prisma-client';
import { PrismaUnitOfWork } from '../../application/unit-of-work/prisma-unit-of-work';
import { UNIT_OF_WORK_TOKEN } from '../../application/unit-of-work/unit-of-work.interface';

@Global()
@Module({
  providers: [
    PrismaService,
    GestorPrismaClient,
    { provide: UNIT_OF_WORK_TOKEN, useClass: PrismaUnitOfWork },
  ],
  exports: [UNIT_OF_WORK_TOKEN, GestorPrismaClient, PrismaService],
})
export class PrismaUowModule {}
```

Con el módulo marcado `@Global()`, cualquier módulo de dominio (`ProductosModule`, etc.) solo necesita declarar sus propios providers — el repositorio concreto, inyectando `GestorPrismaClient` — sin repetir el cableado de Prisma ni del Unit of Work:

```typescript
@Module({
  providers: [
    CrearProductoUseCase,
    ActualizarPrecioProductoUseCase,
    { provide: PRODUCTO_REPOSITORY_TOKEN, useClass: ProductoPrismaRepository },
  ],
})
export class ProductosModule {}
```

**Si el módulo participa (o va a participar) de `processes/`**, el cableado del repositorio se separa en su propio submódulo hoja, y `ProductosModule` lo importa en vez de declarar el provider directo:

```typescript
// infrastructure/persistence/prisma/productos-persistence.module.ts
@Module({
  providers: [{ provide: PRODUCTO_REPOSITORY_TOKEN, useClass: ProductoPrismaRepository }],
  exports: [PRODUCTO_REPOSITORY_TOKEN],
})
export class ProductosPersistenceModule {}
```

```typescript
@Module({
  imports: [
    ProductosPersistenceModule,
    AjustarStockModule, // módulo de processes/ — importado en un solo sentido, ver ARCHITECTURE.md §2.4
  ],
  providers: [CrearProductoUseCase, ActualizarPrecioProductoUseCase],
  controllers: [ProductosController], // puede inyectar también el puerto que expone AjustarStockModule
})
export class ProductosModule {}
```

El módulo de `processes/` correspondiente importa `ProductosPersistenceModule` directo — nunca `ProductosModule` — para no crear el ciclo. Ver `ARCHITECTURE.md` §2.4 para el razonamiento completo y un ejemplo con dos módulos cruzándose.

---

## 4. Caso de uso orquestador

```typescript
@Injectable()
export class ActualizarPrecioProductoUseCase {
  constructor(
    @Inject(UNIT_OF_WORK_TOKEN) private readonly uow: IUnitOfWork,
    @Inject(PRODUCTO_REPOSITORY_TOKEN) private readonly productoRepository: IProductoRepository,
  ) {}

  async execute(productoId: string, nuevoPrecio: number): Promise<void> {
    await this.uow.runInTransaction(async () => {
      const producto = await this.productoRepository.findById(productoId);
      if (!producto) throw new ProductoNoEncontradoException(productoId);

      producto.actualizarPrecio(nuevoPrecio); // lógica de negocio en la ENTIDAD
      await this.productoRepository.save(producto);
    });
  }
}
```

El repositorio se inyecta **directo**, igual que en un caso de uso sin transacciones — no se obtiene a través del `uow`. El UoW solo envuelve el bloque en una transacción; dentro de ella, `ProductoPrismaRepository` detecta sola (vía `GestorPrismaClient`) que hay una transacción activa y usa ese cliente en lugar del global. Este archivo no importa `@prisma/client` en ningún lado — es la prueba de que la capa de aplicación sigue siendo agnóstica a los detalles de infraestructura.

---

## 5. Checklist para agregar un módulo nuevo (infraestructura)

1. Escribir la entidad de dominio — simple (`create()`/`reconstitute()` en la clase) o compuesta (Factory en `domain/factories/`) — con sus `domain/inputs/` (ya cubierto en `ARCHITECTURE.md` §1).
2. Escribir la interfaz del repositorio + su token (`IXRepository`/`X_REPOSITORY_TOKEN`).
3. Agregar el `model` en `schema.prisma` y correr `migrate dev`.
4. Escribir el mapper (`toPersistence` + `toDomain`, siempre con `reconstitute()` — vía la Factory si la entidad es compuesta).
5. Escribir el repositorio concreto, inyectando `GestorPrismaClient` y usando `this.gestor.actual` en cada acceso a la base.
6. Si el módulo necesita Unit of Work: no hay nada que registrar — el repositorio ya funciona dentro o fuera de una transacción por sí solo. Solo hace falta envolver el caso de uso con `uow.runInTransaction(...)`.
7. Cablear el repositorio en el `*.module.ts` del módulo (el `PrismaUowModule` global ya expone `GestorPrismaClient` y `UNIT_OF_WORK_TOKEN`).

---

## 6. Errores comunes a evitar

- Usar `create()` (o la Factory `create()`) en el mapper al leer de la base — siempre `reconstitute()`.
- Poner lógica de negocio en el repositorio en vez de en la entidad o el caso de uso.
- Inyectar `PrismaClient`/`PrismaService` directo en un repositorio en vez de `GestorPrismaClient` — eso vuelve a acoplar el repositorio al cliente global y pierde la detección automática de transacción.
- Hacer trabajo asincrónico sin `await` (fire-and-forget, `setTimeout`, un callback que se dispara y sigue corriendo después de que `runInTransaction` ya resolvió) dentro de una transacción: ese código corre fuera de la cadena de `await` que `AsyncLocalStorage` propaga, así que `gestor.actual` le devuelve el cliente global en vez del `tx` — la operación queda fuera del commit/rollback sin que nadie lo note hasta que falla en producción.
- Importar `@prisma/client` fuera de `infrastructure/` (o de `common/infrastructure/` para las piezas compartidas).
- Armar Unit of Work para un módulo que nunca combina escrituras atómicas — ver `ARCHITECTURE.md` §0 y §2.5.