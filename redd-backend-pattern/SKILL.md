---
name: redd-backend-pattern
description: Gera código backend seguindo o padrão arquitetural do REDD (AdonisJS v5). Use quando criar novas entidades, endpoints, CRUDs, ou refatorar código no redd-api. Aplica automaticamente as camadas Model, Repository, Service, Controller, Validator e Routes.
---

# REDD Backend Pattern

Skill para geração de código backend no projeto REDD seguindo a arquitetura em camadas.

## Estrutura de Camadas

```
redd-api/app/
├── Models/{Domain}/          # Entidades ORM
├── Repositories/{Domain}/    # Acesso a dados
├── Services/{Domain}/        # Lógica de negócio
├── Controllers/Http/{Domain}/ # Orquestração HTTP
├── Validator/{Domain}/       # Validação de requests
└── Enums/                    # Enumeradores
```

## Ordem de Criação

Ao criar uma nova entidade, siga esta ordem:

1. **Migration** → Estrutura do banco
2. **Model** → Entidade ORM
3. **Repository** → Acesso a dados
4. **Service** (se houver lógica complexa)
5. **Validator** → Validação de inputs
6. **Controller** → Endpoints HTTP
7. **Routes** → Registro de rotas

---

## 1. Migration

```typescript
// database/migrations/{timestamp}_{entity_plural}.ts
import BaseSchema from '@ioc:Adonis/Lucid/Schema';

export default class extends BaseSchema {
  protected tableName = 'entity_plural'; // snake_case, plural

  public async up() {
    this.schema.createTable(this.tableName, (table) => {
      table.increments('id');
      
      // Foreign keys com onDelete cascade
      table.integer('parent_id').unsigned().references('parents.id').onDelete('CASCADE');
      
      // Campos comuns
      table.string('name', 255).notNullable();
      table.text('description').nullable();
      table.boolean('is_active').defaultTo(true);
      
      // Índices para colunas de busca frequente
      table.index(['parent_id']);
      
      // Timestamps padrão
      table.timestamp('created_at', { useTz: true });
      table.timestamp('updated_at', { useTz: true });
      table.timestamp('deleted_at', { useTz: true }).nullable(); // Soft delete
    });
  }

  public async down() {
    this.schema.dropTable(this.tableName);
  }
}
```

---

## 2. Model

```typescript
// app/Models/{Domain}/Entity.ts
import { column, belongsTo, hasMany, BelongsTo, HasMany, beforeFind, beforeFetch } from '@ioc:Adonis/Lucid/Orm';
import { DateTime } from 'luxon';
import Model from 'App/Models/Model';
import EntityFilter from 'App/Models/Filters/{Domain}/EntityFilter';

export default class Entity extends Model {
  public static table = 'entity_plural';
  public static $filter = () => EntityFilter;

  @column({ isPrimary: true })
  public id: number;

  // FK: columnName em snake_case, serializeAs em camelCase
  @column({ columnName: 'parent_id', serializeAs: 'parentId' })
  public parentId: number;

  @column()
  public name: string;

  @column()
  public description: string | null;

  @column({ columnName: 'is_active', serializeAs: 'isActive' })
  public isActive: boolean;

  // Datas com timezone America/Sao_Paulo
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

  // Soft delete automático
  @beforeFind()
  @beforeFetch()
  public static ignoreDeletedBeforeFind(query) {
    query.whereNull(`${this.table}.deleted_at`);
  }

  // Relacionamentos
  @belongsTo(() => Parent, { foreignKey: 'parentId' })
  public parent: BelongsTo<typeof Parent>;

  @hasMany(() => Child, { foreignKey: 'entityId' })
  public children: HasMany<typeof Child>;
}
```

### Model Filter

```typescript
// app/Models/Filters/{Domain}/EntityFilter.ts
import { ModelQueryBuilderContract } from '@ioc:Adonis/Lucid/Orm';
import Filter from 'App/Models/Filters/Filter';
import Entity from 'App/Models/{Domain}/Entity';

export default class EntityFilter extends Filter {
  public $query: ModelQueryBuilderContract<typeof Entity, Entity>;

  public parentId(parentId: number | number[]) {
    const ids = Array.isArray(parentId) ? parentId : [parentId];
    this.$query.whereIn('parentId', ids);
  }

  public isActive(isActive: boolean) {
    this.$query.where('isActive', isActive);
  }

  public search(term: string) {
    this.$query.where((query) => {
      query.whereILike('name', `%${term}%`).orWhereILike('description', `%${term}%`);
    });
  }
}
```

---

## 3. Repository

```typescript
// app/Repositories/{Domain}/EntityRepository.ts
import Entity from 'App/Models/{Domain}/Entity';
import Repository from 'App/Repositories/Repository';

export default class EntityRepository extends Repository<Entity> {
  protected model = Entity;

  // Métodos customizados quando necessário
  public async findByParent(parentId: number): Promise<Entity[]> {
    return await this.model.query().where('parentId', parentId).exec();
  }

  public async softDelete(id: number): Promise<void> {
    await this.model.query().where('id', id).update({ deletedAt: new Date() });
  }
}
```

---

## 4. Service (quando há lógica complexa)

```typescript
// app/Services/{Domain}/EntityService.ts
import Database from '@ioc:Adonis/Lucid/Database';
import EntityRepository from 'App/Repositories/{Domain}/EntityRepository';
import RelatedRepository from 'App/Repositories/{Domain}/RelatedRepository';

export default class EntityService {
  private entityRepository: EntityRepository;
  private relatedRepository: RelatedRepository;

  constructor() {
    this.entityRepository = new EntityRepository();
    this.relatedRepository = new RelatedRepository();
  }

  public async createWithRelations(data: CreateEntityDTO): Promise<Entity> {
    return await Database.transaction(async (trx) => {
      const entity = await this.entityRepository.create(data.entity, trx);

      for (const related of data.relations) {
        await this.relatedRepository.create({ ...related, entityId: entity.id }, trx);
      }

      return entity;
    });
  }

  public async processBusinessLogic(entityId: number): Promise<ProcessResult> {
    const entity = await this.entityRepository.getOrFail(entityId);
    
    // Lógica de negócio aqui
    
    return result;
  }
}
```

---

## 5. Validator

```typescript
// app/Validator/{Domain}/EntityValidator.ts
import { RequestContract } from '@ioc:Adonis/Core/Request';
import { rules, schema } from '@ioc:Adonis/Core/Validator';
import Validator from 'App/Validator/Validator';
import { ValidatorContract } from 'App/Contracts/ValidatorContract';

export default class EntityValidator extends Validator implements ValidatorContract {
  public async id(request: RequestContract) {
    const getSchema = schema.create({
      params: schema.object().members({
        id: schema.number([rules.unsigned(), rules.exists({ table: 'entities', column: 'id' })]),
      }),
    });

    return await super.validate(getSchema, request);
  }

  public async index(request: RequestContract) {
    const getSchema = schema.create({
      parentId: schema.number.optional([rules.unsigned()]),
      isActive: schema.boolean.optional(),
      page: schema.number.optional([rules.unsigned()]),
      limit: schema.number.optional([rules.unsigned(), rules.range(1, 100)]),
    });

    return await super.validate(getSchema, request);
  }

  public async create(request: RequestContract) {
    const postSchema = schema.create({
      parentId: schema.number([
        rules.unsigned(),
        rules.exists({ table: 'parents', column: 'id' }),
      ]),
      name: schema.string({ trim: true }, [rules.minLength(2), rules.maxLength(255)]),
      description: schema.string.optional({ trim: true }),
      isActive: schema.boolean.optional(),
    });

    return await super.validate(postSchema, request);
  }

  public async update(request: RequestContract) {
    const putSchema = schema.create({
      parentId: schema.number.optional([
        rules.unsigned(),
        rules.exists({ table: 'parents', column: 'id' }),
      ]),
      name: schema.string.optional({ trim: true }, [rules.minLength(2), rules.maxLength(255)]),
      description: schema.string.optional({ trim: true }),
      isActive: schema.boolean.optional(),
    });

    return await super.validate(putSchema, request);
  }
}
```

---

## 6. Controller

```typescript
// app/Controllers/Http/{Domain}/EntitiesController.ts
import type { HttpContextContract } from '@ioc:Adonis/Core/HttpContext';
import Controller from 'App/Controllers/Controller';
import EntityRepository from 'App/Repositories/{Domain}/EntityRepository';
import EntityValidator from 'App/Validator/{Domain}/EntityValidator';

export default class EntitiesController extends Controller {
  protected repository: EntityRepository;
  protected validator: EntityValidator;

  constructor() {
    super();
    this.repository = new EntityRepository();
    this.validator = new EntityValidator();
  }

  // Sobrescrever para configurar includes/preloads
  public async index(_ctx: HttpContextContract) {
    _ctx.params.includes = JSON.stringify(['parent', 'children']);
    return await super.index(_ctx);
  }

  public async show(_ctx: HttpContextContract) {
    _ctx.params.includes = JSON.stringify(['parent', 'children']);
    return await super.show(_ctx);
  }

  // Sobrescrever para adicionar dados automáticos
  public async store(_ctx: HttpContextContract) {
    _ctx.request.updateBody({
      ..._ctx.request.all(),
      creatorId: _ctx.auth.user?.id,
    });
    return await super.store(_ctx);
  }

  // Métodos customizados
  public async customAction(_ctx: HttpContextContract) {
    const { id } = _ctx.params;
    const entity = await this.repository.getOrFail(id);
    
    // Lógica customizada
    
    return _ctx.response.ok({ data: entity });
  }
}
```

---

## 7. Routes

```typescript
// start/routes/{domain}.ts
import Route from '@ioc:Adonis/Core/Route';

Route.group(() => {
  // CRUD padrão
  Route.get('/', 'EntitiesController.index');
  Route.post('/', 'EntitiesController.store');
  Route.get('/:id', 'EntitiesController.show');
  Route.put('/:id', 'EntitiesController.update');
  Route.delete('/:id', 'EntitiesController.destroy');

  // Rotas customizadas
  Route.post('/:id/custom-action', 'EntitiesController.customAction');
})
  .prefix('entities')
  .middleware('auth')
  .namespace('App/Controllers/Http/{Domain}');
```

---

## Convenções Importantes

### Nomenclatura

| Elemento | Padrão | Exemplo |
|----------|--------|---------|
| Tabela (migration) | snake_case, plural | `order_items` |
| Model | PascalCase, singular | `OrderItem.ts` |
| Repository | Model + Repository | `OrderItemRepository.ts` |
| Service | Domínio + Service | `OrderService.ts` |
| Controller | Model plural + Controller | `OrderItemsController.ts` |
| Validator | Model + Validator | `OrderItemValidator.ts` |
| Filter | Model + Filter | `OrderItemFilter.ts` |

### Serialização

- Colunas do banco: `snake_case`
- JSON de resposta: `camelCase` (usar `serializeAs`)
- Datas: sempre com timezone `America/Sao_Paulo`

### Soft Delete

- Todos os models devem ter `deleted_at`
- Usar hooks `@beforeFind()` e `@beforeFetch()` para filtrar automaticamente

### Validação

- Sempre validar `params`, `body` e `query`
- Usar `rules.exists()` para validar FKs
- Usar `rules.unsigned()` para IDs

### Transações

- Operações que afetam múltiplas tabelas devem usar `Database.transaction()`
- Passar `trx` para os repositories dentro da transação

---

## Checklist de Nova Entidade

```
[ ] Migration criada com soft delete e índices
[ ] Model com Filter, relacionamentos e hooks
[ ] Repository estendendo Repository<T>
[ ] Service (se houver lógica de negócio)
[ ] Validator com schemas por operação
[ ] Controller estendendo Controller base
[ ] Rotas registradas com middleware auth
[ ] Teste unitário para lógica de negócio
```
