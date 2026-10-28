# Guía de Implementación: Capa de Infraestructura (ORM)

> Parte del set de precedente arquitectónico junto a `ARCHITECTURE.md` (reglas de las tres capas) y `DECISIONS.md` (razonamiento/trade-offs). Este documento asume que ya se leyó la sección 0 de `ARCHITECTURE.md` (niveles de complejidad) — en particular, si el módulo no necesita Unit of Work, saltear directo a las secciones 3.1–3.3 / 4.1–4.3 de este documento e ignorar las secciones de UoW.

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
        │   ├── value-objects/
        │   ├── repository/
        │   └── exceptions/
        ├── application/
        │   ├── facades/
        │   ├── use-cases/
        │   ├── dtos/
        │   └── unit-of-work/          # solo si el módulo lo necesita — ver ARCHITECTURE.md §2.5
        ├── infrastructure/            # ← ESTA GUÍA CUBRE SOLO ESTA CARPETA
        │   ├── controllers/
        │   └── persistence/
        │       ├── typeorm/           # si el proyecto eligió TypeORM
        │       └── prisma/            # si el proyecto eligió Prisma
        └── productos.module.ts
```

Un proyecto usa **un solo ORM** — las dos subcarpetas conviven acá solo para comparar.

---

## 2. Punto de partida (dominio y aplicación, ya definidos antes de tocar infraestructura)

```typescript
// domain/entities/proveedor.entity.ts
export class Proveedor {
  private constructor(
    private readonly id: string | null,
    private nombre: string,
  ) {}

  static create(nombre: string): Proveedor {
    if (!nombre?.trim()) throw new Error('El nombre del proveedor es requerido');
    return new Proveedor(null, nombre);
  }

  static reconstitute(id: string, nombre: string): Proveedor {
    return new Proveedor(id, nombre);
  }

  getId(): string | null { return this.id; }
  getNombre(): string { return this.nombre; }
}
```

```typescript
// domain/entities/producto.entity.ts
import { ProductoPrice } from '../value-objects/producto-price.vo';

export class Producto {
  private constructor(
    private readonly id: string | null,
    private nombre: string,
    private precio: ProductoPrice,
    private readonly proveedorId: string,
  ) {}

  static create(nombre: string, precio: number, proveedorId: string): Producto {
    return new Producto(null, nombre, ProductoPrice.create(precio), proveedorId);
  }

  static reconstitute(id: string, nombre: string, precio: number, proveedorId: string): Producto {
    return new Producto(id, nombre, ProductoPrice.create(precio), proveedorId);
  }

  actualizarPrecio(nuevoPrecio: number): void {
    this.precio = ProductoPrice.create(nuevoPrecio);
  }

  getId(): string | null { return this.id; }
  getNombre(): string { return this.nombre; }
  getPrecio(): number { return this.precio.getValue(); }
  getProveedorId(): string { return this.proveedorId; }
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

### Contrato de Unit of Work — dos variantes posibles

Aplicar solo si el módulo lo necesita (ver `ARCHITECTURE.md` §2.5). El equipo elige **una** variante para todo el proyecto, no se mezclan.

**Variante A — un método explícito por repositorio:**

```typescript
export interface IUnitOfWork {
  getProveedorRepository(): IProveedorRepository;
  getProductoRepository(): IProductoRepository;
  runInTransaction<T>(work: () => Promise<T>): Promise<T>;
}

export const UNIT_OF_WORK_TOKEN = Symbol('IUnitOfWork');
```

- ✅ Tipado explícito, sin casteos, autocompletado inmediato.
- ❌ Crece un método por cada módulo nuevo que participe de transacciones cross-módulo.

**Variante B — un método genérico por token:**

```typescript
export interface IUnitOfWork {
  getRepository<T>(token: symbol): T;
  runInTransaction<T>(work: () => Promise<T>): Promise<T>;
}

export const UNIT_OF_WORK_TOKEN = Symbol('IUnitOfWork');
```

- ✅ La interfaz nunca crece, escala mejor con muchos módulos.
- ❌ Sin tipado automático — el `T` se especifica a mano en cada llamada, y un token mal asociado no se detecta en compilación, solo en runtime.

**Recomendación:** Variante A para proyectos con pocos módulos participando de transacciones cruzadas; Variante B cuando esa lista de métodos ya es incómoda de mantener. Ver `DECISIONS.md` para el detalle completo.

---

## 3. Camino A: TypeORM

### 3.1. Entidades ORM

```typescript
// infrastructure/persistence/typeorm/entities/proveedor.orm-entity.ts
import { Entity, Column, PrimaryGeneratedColumn, OneToMany } from 'typeorm';
import { ProductoOrmEntity } from './producto.orm-entity';

@Entity('proveedores')
export class ProveedorOrmEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 255 })
  nombre: string;

  @OneToMany(() => ProductoOrmEntity, (producto) => producto.proveedor)
  productos: ProductoOrmEntity[];
}
```

```typescript
// infrastructure/persistence/typeorm/entities/producto.orm-entity.ts
import { Entity, Column, PrimaryGeneratedColumn, ManyToOne, JoinColumn } from 'typeorm';
import { ProveedorOrmEntity } from './proveedor.orm-entity';

@Entity('productos')
export class ProductoOrmEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 255 })
  nombre: string;

  @Column({ type: 'numeric', precision: 10, scale: 2 })
  precio: number;

  @Column({ name: 'proveedor_id' })
  proveedorId: string;

  @ManyToOne(() => ProveedorOrmEntity, (proveedor) => proveedor.productos)
  @JoinColumn({ name: 'proveedor_id' })
  proveedor: ProveedorOrmEntity;
}
```

### 3.2. Mappers

```typescript
// infrastructure/persistence/typeorm/mappers/producto.mapper.ts
import { Producto } from '../../../../domain/entities/producto.entity';
import { ProductoOrmEntity } from '../entities/producto.orm-entity';

export class ProductoMapper {
  static toOrm(producto: Producto): ProductoOrmEntity {
    const orm = new ProductoOrmEntity();
    if (producto.getId() !== null) orm.id = producto.getId()!;
    orm.nombre = producto.getNombre();
    orm.precio = producto.getPrecio();
    orm.proveedorId = producto.getProveedorId();
    return orm;
  }

  static toDomain(orm: ProductoOrmEntity): Producto {
    // SIEMPRE reconstitute() al leer de infraestructura — nunca create()
    return Producto.reconstitute(orm.id, orm.nombre, Number(orm.precio), orm.proveedorId);
  }
}
```

*(`ProveedorMapper` es análogo, se omite por brevedad.)*

### 3.3. Repositorio concreto

```typescript
// infrastructure/persistence/typeorm/repositories/producto.typeorm.repository.ts
import { EntityManager } from 'typeorm';
import { IProductoRepository } from '../../../../domain/repository/producto.repository.interface';
import { Producto } from '../../../../domain/entities/producto.entity';
import { ProductoOrmEntity } from '../entities/producto.orm-entity';
import { ProductoMapper } from '../mappers/producto.mapper';

export class ProductoTypeOrmRepository implements IProductoRepository {
  // Recibe el manager global O el de una transacción activa — no le importa cuál
  constructor(private readonly entityManager: EntityManager) {}

  async save(producto: Producto): Promise<Producto> {
    const orm = ProductoMapper.toOrm(producto);
    const saved = await this.entityManager.save(ProductoOrmEntity, orm);
    return ProductoMapper.toDomain(saved);
  }

  async findById(id: string): Promise<Producto | null> {
    const orm = await this.entityManager.findOne(ProductoOrmEntity, { where: { id } });
    return orm ? ProductoMapper.toDomain(orm) : null;
  }

  async findByProveedorId(proveedorId: string): Promise<Producto[]> {
    const orms = await this.entityManager.find(ProductoOrmEntity, { where: { proveedorId } });
    return orms.map(ProductoMapper.toDomain);
  }
}
```

### 3.4. Unit of Work con TypeORM (ambas variantes)

**Variante A:**

```typescript
@Injectable()
export class TypeOrmUnitOfWork implements IUnitOfWork {
  private transactionManager: EntityManager | null = null;

  constructor(private readonly dataSource: DataSource) {}

  private get manager(): EntityManager {
    return this.transactionManager ?? this.dataSource.manager;
  }

  getProveedorRepository(): IProveedorRepository {
    return new ProveedorTypeOrmRepository(this.manager);
  }

  getProductoRepository(): IProductoRepository {
    return new ProductoTypeOrmRepository(this.manager);
  }

  async runInTransaction<T>(work: () => Promise<T>): Promise<T> {
    if (this.transactionManager) return work();

    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();
    this.transactionManager = queryRunner.manager;

    try {
      const result = await work();
      await queryRunner.commitTransaction();
      return result;
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await queryRunner.release();
      this.transactionManager = null;
    }
  }
}
```

**Variante B** (mismo `runInTransaction`, cambia solo la resolución de repos):

```typescript
type RepoFactory = (manager: EntityManager) => unknown;

@Injectable()
export class TypeOrmUnitOfWork implements IUnitOfWork {
  private transactionManager: EntityManager | null = null;

  private readonly factories = new Map<symbol, RepoFactory>([
    [PROVEEDOR_REPOSITORY_TOKEN, (m) => new ProveedorTypeOrmRepository(m)],
    [PRODUCTO_REPOSITORY_TOKEN, (m) => new ProductoTypeOrmRepository(m)],
  ]);

  constructor(private readonly dataSource: DataSource) {}

  private get manager(): EntityManager {
    return this.transactionManager ?? this.dataSource.manager;
  }

  getRepository<T>(token: symbol): T {
    const factory = this.factories.get(token);
    if (!factory) throw new Error(`Sin repositorio registrado para ${String(token)}`);
    return factory(this.manager) as T;
  }

  async runInTransaction<T>(work: () => Promise<T>): Promise<T> {
    if (this.transactionManager) return work();

    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();
    this.transactionManager = queryRunner.manager;

    try {
      const result = await work();
      await queryRunner.commitTransaction();
      return result;
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await queryRunner.release();
      this.transactionManager = null;
    }
  }
}
```

### 3.5. Cableado del módulo (TypeORM)

```typescript
@Module({
  imports: [TypeOrmModule.forFeature([ProveedorOrmEntity, ProductoOrmEntity])],
  providers: [
    CrearProductoUseCase,
    { provide: UNIT_OF_WORK_TOKEN, useClass: TypeOrmUnitOfWork },
  ],
})
export class ProductosModule {}
```

---

## 4. Camino B: Prisma

### 4.1. Schema declarativo

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

### 4.2. Mappers

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
    return Producto.reconstitute(row.id, row.nombre, Number(row.precio), row.proveedorId);
  }
}
```

Prisma no tiene clases de entidad ORM, pero el mapper **igual hace falta**: el objeto que devuelve Prisma es una estructura plana de datos, no la entidad de dominio con su lógica de negocio.

### 4.3. Repositorio concreto

```typescript
// infrastructure/persistence/prisma/repositories/producto.prisma.repository.ts
import type { PrismaClient, Prisma } from '../../../../generated/prisma/client';
import { IProductoRepository } from '../../../../domain/repository/producto.repository.interface';
import { Producto } from '../../../../domain/entities/producto.entity';
import { ProductoMapper } from '../mappers/producto.mapper';

export class ProductoPrismaRepository implements IProductoRepository {
  // Recibe el cliente global O el de una transacción activa (tx) — no le importa cuál
  constructor(private readonly client: PrismaClient | Prisma.TransactionClient) {}

  async save(producto: Producto): Promise<Producto> {
    const data = ProductoMapper.toPersistence(producto);
    const row = producto.getId() === null
      ? await this.client.producto.create({ data })
      : await this.client.producto.update({ where: { id: producto.getId()! }, data });
    return ProductoMapper.toDomain(row);
  }

  async findById(id: string): Promise<Producto | null> {
    const row = await this.client.producto.findUnique({ where: { id } });
    return row ? ProductoMapper.toDomain(row) : null;
  }

  async findByProveedorId(proveedorId: string): Promise<Producto[]> {
    const rows = await this.client.producto.findMany({ where: { proveedorId } });
    return rows.map(ProductoMapper.toDomain);
  }
}
```

### 4.4. Unit of Work con Prisma (ambas variantes)

**Variante A:**

```typescript
@Injectable()
export class PrismaUnitOfWork implements IUnitOfWork {
  private tx: Prisma.TransactionClient | null = null;

  constructor(private readonly prisma: PrismaClient) {}

  private get client(): PrismaClient | Prisma.TransactionClient {
    return this.tx ?? this.prisma;
  }

  getProveedorRepository(): IProveedorRepository {
    return new ProveedorPrismaRepository(this.client);
  }

  getProductoRepository(): IProductoRepository {
    return new ProductoPrismaRepository(this.client);
  }

  async runInTransaction<T>(work: () => Promise<T>): Promise<T> {
    if (this.tx) return work();

    return this.prisma.$transaction(async (tx) => {
      this.tx = tx;
      try {
        return await work();
      } finally {
        this.tx = null;
      }
    });
  }
}
```

**Variante B:**

```typescript
type RepoFactory = (client: PrismaClient | Prisma.TransactionClient) => unknown;

@Injectable()
export class PrismaUnitOfWork implements IUnitOfWork {
  private tx: Prisma.TransactionClient | null = null;

  private readonly factories = new Map<symbol, RepoFactory>([
    [PROVEEDOR_REPOSITORY_TOKEN, (c) => new ProveedorPrismaRepository(c)],
    [PRODUCTO_REPOSITORY_TOKEN, (c) => new ProductoPrismaRepository(c)],
  ]);

  constructor(private readonly prisma: PrismaClient) {}

  private get client(): PrismaClient | Prisma.TransactionClient {
    return this.tx ?? this.prisma;
  }

  getRepository<T>(token: symbol): T {
    const factory = this.factories.get(token);
    if (!factory) throw new Error(`Sin repositorio registrado para ${String(token)}`);
    return factory(this.client) as T;
  }

  async runInTransaction<T>(work: () => Promise<T>): Promise<T> {
    if (this.tx) return work();

    return this.prisma.$transaction(async (tx) => {
      this.tx = tx;
      try {
        return await work();
      } finally {
        this.tx = null;
      }
    });
  }
}
```

### 4.5. Cableado del módulo (Prisma)

```typescript
export const PRISMA_CLIENT_TOKEN = Symbol('PrismaClient');

export const PrismaClientProvider = {
  provide: PRISMA_CLIENT_TOKEN,
  useFactory: () => {
    const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL! });
    return new PrismaClient({ adapter });
  },
};

@Module({
  providers: [
    PrismaClientProvider,
    CrearProductoUseCase,
    {
      provide: UNIT_OF_WORK_TOKEN,
      useFactory: (prisma) => new PrismaUnitOfWork(prisma),
      inject: [PRISMA_CLIENT_TOKEN],
    },
  ],
})
export class ProductosModule {}
```

> En un proyecto real, `PrismaClientProvider` se define una sola vez en un módulo compartido (`@Global()`), no repetido por módulo.

---

## 5. Caso de uso orquestador (idéntico sin importar el ORM elegido)

```typescript
@Injectable()
export class ActualizarPrecioProductoUseCase {
  constructor(@Inject(UNIT_OF_WORK_TOKEN) private readonly uow: IUnitOfWork) {}

  async execute(productoId: string, nuevoPrecio: number): Promise<void> {
    await this.uow.runInTransaction(async () => {
      const repo = this.uow.getProductoRepository(); // o getRepository<...>(TOKEN) en Variante B

      const producto = await repo.findById(productoId);
      if (!producto) throw new Error(`Producto ${productoId} no encontrado`);

      producto.actualizarPrecio(nuevoPrecio); // lógica de negocio en la ENTIDAD
      await repo.save(producto);
    });
  }
}
```

Este archivo no importa `typeorm` ni `@prisma/client` en ningún lado — es la prueba de que cambiar de ORM no toca la capa de aplicación.

---

## 6. Checklist para agregar un módulo nuevo (infraestructura)

1. Escribir la entidad de dominio con `create()`/`reconstitute()` (ya cubierto en `ARCHITECTURE.md` §1).
2. Escribir la interfaz del repositorio + su token.
3. **TypeORM:** crear la entidad ORM decorada. **Prisma:** agregar el `model` en `schema.prisma` y correr `migrate dev`.
4. Escribir el mapper (`toOrm`/`toPersistence` + `toDomain`, siempre con `reconstitute()`).
5. Escribir el repositorio concreto.
6. Si el módulo participa de transacciones cross-módulo: registrar el repo en el Unit of Work (método nuevo en Variante A, entrada nueva en el `Map` en Variante B).
7. Si el módulo no necesita UoW: inyectar el repositorio directo en el caso de uso.
8. Cablear en el `*.module.ts`.

---

## 7. Tabla comparativa

| Pieza | TypeORM | Prisma |
|---|---|---|
| Modelo físico | Clase con decoradores | Declarativo en `schema.prisma`, se genera solo |
| ¿Se escribe una clase de entidad ORM? | Sí | No |
| Objeto que maneja queries/transacciones | `EntityManager` | `PrismaClient` \| `Prisma.TransactionClient` |
| Apertura de transacción | `QueryRunner` + `startTransaction()` | `prisma.$transaction(async (tx) => {...})` |
| ¿El repo distingue si está en tx? | No — recibe el manager que sea | No — recibe el client que sea |
| Migraciones | Aparte del modelo | Automáticas al correr `migrate dev` |

---

## 8. Errores comunes a evitar

- Usar `create()` en el mapper al leer de la base — siempre `reconstitute()`.
- Poner lógica de negocio en el repositorio en vez de en la entidad o el caso de uso.
- Usar el cliente/manager global dentro de un callback de transacción en vez del que provee la transacción (`tx`) — esa operación queda fuera del rollback/commit sin que nadie lo note hasta que falla en producción.
- Importar `typeorm` o `@prisma/client` fuera de `infrastructure/`.
- Armar Unit of Work para un módulo que nunca combina escrituras atómicas — ver `ARCHITECTURE.md` §0 y §2.5.
