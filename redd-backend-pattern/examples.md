# Exemplos Reais do Projeto REDD

Exemplos extraídos do código existente para referência.

## Model Completo - Product

```typescript
import { column, belongsTo, hasMany, BelongsTo, HasMany, beforeFind, beforeFetch } from '@ioc:Adonis/Lucid/Orm';
import { DateTime } from 'luxon';
import Model from 'App/Models/Model';
import ProductFilter from 'App/Models/Filters/Product/ProductFilter';
import Supplier from 'App/Models/Supplier/Supplier';
import ProductVariation from 'App/Models/Product/ProductVariation';

export default class Product extends Model {
  public static table = 'products';
  public static $filter = () => ProductFilter;

  @column({ isPrimary: true })
  public id: number;

  @column({ columnName: 'supplier_id', serializeAs: 'supplierId' })
  public supplierId: number;

  @column()
  public name: string;

  @column()
  public sku: string;

  @column({ columnName: 'is_active', serializeAs: 'isActive' })
  public isActive: boolean;

  @column.dateTime({
    autoCreate: true,
    serialize: (value: DateTime | null) => value?.setZone('America/Sao_Paulo').toISO(),
  })
  public createdAt: DateTime;

  @column.dateTime({
    autoCreate: true,
    autoUpdate: true,
    serialize: (value: DateTime | null) => value?.setZone('America/Sao_Paulo').toISO(),
  })
  public updatedAt: DateTime;

  @column.dateTime({
    serialize: (value: DateTime | null) => value?.setZone('America/Sao_Paulo').toISO(),
  })
  public deletedAt: DateTime | null;

  @beforeFind()
  @beforeFetch()
  public static ignoreDeletedBeforeFind(query) {
    query.whereNull(`${this.table}.deleted_at`);
  }

  @belongsTo(() => Supplier, { foreignKey: 'supplierId' })
  public supplier: BelongsTo<typeof Supplier>;

  @hasMany(() => ProductVariation, { foreignKey: 'productId' })
  public variations: HasMany<typeof ProductVariation>;
}
```

## Repository - Exemplo com Métodos Customizados

```typescript
import Product from 'App/Models/Product/Product';
import Repository from 'App/Repositories/Repository';

export default class ProductRepository extends Repository<Product> {
  protected model = Product;

  public async setDiscontinuedProducts(importedProducts: string[], supplierId: number) {
    return await this.model
      .query()
      .where('supplierId', supplierId)
      .whereNotIn('sku', importedProducts)
      .update({ isActive: false });
  }

  public async findBySku(sku: string, supplierId: number): Promise<Product | null> {
    return await this.model
      .query()
      .where('sku', sku)
      .where('supplierId', supplierId)
      .first();
  }

  public async getWithVariations(id: number): Promise<Product> {
    return await this.model
      .query()
      .where('id', id)
      .preload('variations')
      .firstOrFail();
  }
}
```

## Controller - OrdersController

```typescript
import type { HttpContextContract } from '@ioc:Adonis/Core/HttpContext';
import Controller from 'App/Controllers/Controller';
import OrderRepository from 'App/Repositories/Order/OrderRepository';
import OrderValidator from 'App/Validator/Order/OrderValidator';

export default class OrdersController extends Controller {
  protected repository: OrderRepository;
  protected validator: OrderValidator;

  constructor() {
    super();
    this.repository = new OrderRepository();
    this.validator = new OrderValidator();
  }

  public async index(_ctx: HttpContextContract) {
    _ctx.params.includes = JSON.stringify([
      'customer',
      'seller',
      'items',
      'items.product',
    ]);
    return await super.index(_ctx);
  }

  public async show(_ctx: HttpContextContract) {
    _ctx.params.includes = JSON.stringify([
      'customer',
      'seller',
      'items',
      'items.product',
      'items.variation',
      'payments',
      'shipments',
    ]);
    return await super.show(_ctx);
  }

  public async store(_ctx: HttpContextContract) {
    _ctx.request.updateBody({
      ..._ctx.request.all(),
      creatorId: _ctx.auth.user?.id,
      status: 'pending',
    });
    return await super.store(_ctx);
  }

  public async approve(_ctx: HttpContextContract) {
    await this.validator.approveOrder(_ctx.request);
    const { id } = _ctx.params;
    
    const order = await this.repository.getOrFail(id);
    order.status = 'approved';
    order.approvedAt = DateTime.now();
    order.approvedBy = _ctx.auth.user?.id;
    await order.save();

    return _ctx.response.ok({ data: order });
  }
}
```

## Validator Completo

```typescript
import { RequestContract } from '@ioc:Adonis/Core/Request';
import { rules, schema } from '@ioc:Adonis/Core/Validator';
import Validator from 'App/Validator/Validator';
import { ValidatorContract } from 'App/Contracts/ValidatorContract';

export default class OrderValidator extends Validator implements ValidatorContract {
  public async id(request: RequestContract) {
    const getSchema = schema.create({
      params: schema.object().members({
        id: schema.number([
          rules.unsigned(),
          rules.exists({ table: 'orders', column: 'id' }),
        ]),
      }),
    });
    return await super.validate(getSchema, request);
  }

  public async index(request: RequestContract) {
    const getSchema = schema.create({
      customerId: schema.number.optional([rules.unsigned()]),
      sellerId: schema.number.optional([rules.unsigned()]),
      status: schema.enum.optional(['pending', 'approved', 'shipped', 'delivered', 'cancelled']),
      startDate: schema.date.optional({ format: 'yyyy-MM-dd' }),
      endDate: schema.date.optional({ format: 'yyyy-MM-dd' }),
      page: schema.number.optional([rules.unsigned()]),
      limit: schema.number.optional([rules.unsigned(), rules.range(1, 100)]),
    });
    return await super.validate(getSchema, request);
  }

  public async create(request: RequestContract) {
    const postSchema = schema.create({
      customerId: schema.number([
        rules.unsigned(),
        rules.exists({ table: 'customers', column: 'id' }),
      ]),
      sellerId: schema.number.optional([
        rules.unsigned(),
        rules.exists({ table: 'users', column: 'id' }),
      ]),
      items: schema.array([rules.minLength(1)]).members(
        schema.object().members({
          productId: schema.number([
            rules.unsigned(),
            rules.exists({ table: 'products', column: 'id' }),
          ]),
          variationId: schema.number.optional([
            rules.unsigned(),
            rules.exists({ table: 'product_variations', column: 'id' }),
          ]),
          quantity: schema.number([rules.unsigned(), rules.range(1, 9999)]),
          unitPrice: schema.number([rules.unsigned()]),
        })
      ),
      notes: schema.string.optional({ trim: true }),
    });
    return await super.validate(postSchema, request);
  }

  public async approveOrder(request: RequestContract) {
    const postSchema = schema.create({
      params: schema.object().members({
        id: schema.number([
          rules.unsigned(),
          rules.exists({ table: 'orders', column: 'id' }),
        ]),
      }),
      approvalNotes: schema.string.optional({ trim: true }),
    });
    return await super.validate(postSchema, request);
  }
}
```

## Filter - ProductFilter

```typescript
import { ModelQueryBuilderContract } from '@ioc:Adonis/Lucid/Orm';
import Filter from 'App/Models/Filters/Filter';
import Product from 'App/Models/Product/Product';

export default class ProductFilter extends Filter {
  public $query: ModelQueryBuilderContract<typeof Product, Product>;

  public supplierId(supplierId: number | number[]) {
    const ids = Array.isArray(supplierId) ? supplierId : [supplierId];
    this.$query.whereIn('supplierId', ids);
  }

  public categoryId(categoryId: number | number[]) {
    const categories = Array.isArray(categoryId) ? categoryId : [categoryId];
    this.$query.whereHas('subcategories', (query) => {
      query.whereIn('categoryId', categories);
    });
  }

  public isActive(isActive: boolean) {
    this.$query.where('isActive', isActive);
  }

  public search(term: string) {
    this.$query.where((query) => {
      query
        .whereILike('name', `%${term}%`)
        .orWhereILike('sku', `%${term}%`)
        .orWhereILike('description', `%${term}%`);
    });
  }

  public priceRange(min: number, max: number) {
    this.$query.whereBetween('basePrice', [min, max]);
  }
}
```

## Service com Transação

```typescript
import Database from '@ioc:Adonis/Lucid/Database';
import OrderRepository from 'App/Repositories/Order/OrderRepository';
import OrderItemRepository from 'App/Repositories/Order/OrderItemRepository';
import InventoryRepository from 'App/Repositories/Inventory/InventoryRepository';

export default class OrderService {
  private orderRepository: OrderRepository;
  private orderItemRepository: OrderItemRepository;
  private inventoryRepository: InventoryRepository;

  constructor() {
    this.orderRepository = new OrderRepository();
    this.orderItemRepository = new OrderItemRepository();
    this.inventoryRepository = new InventoryRepository();
  }

  public async createOrder(data: CreateOrderDTO): Promise<Order> {
    return await Database.transaction(async (trx) => {
      // Criar pedido
      const order = await this.orderRepository.create({
        customerId: data.customerId,
        sellerId: data.sellerId,
        status: 'pending',
      }, trx);

      // Criar itens e reservar estoque
      for (const item of data.items) {
        await this.orderItemRepository.create({
          orderId: order.id,
          productId: item.productId,
          variationId: item.variationId,
          quantity: item.quantity,
          unitPrice: item.unitPrice,
        }, trx);

        // Reservar estoque
        await this.inventoryRepository.reserve({
          productId: item.productId,
          variationId: item.variationId,
          quantity: item.quantity,
          referenceType: 'order',
          referenceId: order.id,
        }, trx);
      }

      return order;
    });
  }

  public async cancelOrder(orderId: number): Promise<void> {
    return await Database.transaction(async (trx) => {
      const order = await this.orderRepository.getOrFail(orderId);
      
      // Liberar reservas de estoque
      await this.inventoryRepository.releaseByReference('order', orderId, trx);
      
      // Cancelar pedido
      order.status = 'cancelled';
      order.cancelledAt = DateTime.now();
      await order.useTransaction(trx).save();
    });
  }
}
```

## Routes Completas

```typescript
// start/routes/orders.ts
import Route from '@ioc:Adonis/Core/Route';

Route.group(() => {
  // CRUD padrão
  Route.get('/', 'OrdersController.index');
  Route.post('/', 'OrdersController.store');
  Route.get('/:id', 'OrdersController.show');
  Route.put('/:id', 'OrdersController.update');
  Route.delete('/:id', 'OrdersController.destroy');

  // Ações de status
  Route.post('/:id/approve', 'OrdersController.approve');
  Route.post('/:id/ship', 'OrdersController.ship');
  Route.post('/:id/deliver', 'OrdersController.deliver');
  Route.post('/:id/cancel', 'OrdersController.cancel');

  // Rotas aninhadas
  Route.get('/:id/items', 'OrderItemsController.index');
  Route.post('/:id/items', 'OrderItemsController.store');
  Route.get('/:id/payments', 'OrderPaymentsController.index');
})
  .prefix('orders')
  .middleware('auth')
  .namespace('App/Controllers/Http/Orders');
```
