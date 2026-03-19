---
name: sql-optimization-patterns
description: Otimização de queries SQL e Lucid ORM no AdonisJS. Use quando debugar queries lentas, otimizar preloads, resolver N+1, criar índices ou analisar performance no redd-api.
---

# SQL Optimization Patterns - AdonisJS/Lucid

Otimização de queries para o projeto REDD usando Lucid ORM (AdonisJS v5).

## Quando Usar

- Debugar queries lentas
- Resolver problemas de N+1
- Otimizar preloads e eager loading
- Criar índices eficientes
- Analisar EXPLAIN de queries
- Melhorar paginação em tabelas grandes

---

## 1. Resolver N+1 no Lucid ORM

### Problema: N+1 Queries

```typescript
// ❌ RUIM: N+1 queries (1 + N consultas)
const orders = await Order.all();
for (const order of orders) {
  await order.load('customer'); // Query para cada order!
  await order.load('items');    // Mais queries!
}
```

### Solução: Preload (Eager Loading)

```typescript
// ✅ BOM: 3 queries apenas (orders, customers, items)
const orders = await Order.query()
  .preload('customer')
  .preload('items')
  .exec();

// ✅ Preload aninhado
const orders = await Order.query()
  .preload('customer')
  .preload('items', (query) => {
    query.preload('product');
  })
  .exec();

// ✅ Preload com filtro
const orders = await Order.query()
  .preload('items', (query) => {
    query.where('quantity', '>', 0).orderBy('created_at', 'desc');
  })
  .exec();
```

### Via Filter (QueryString)

```typescript
// No Filter do Model
public includes(value: string) {
  const includes = JSON.parse(value);
  for (const include of includes) {
    if (include.scope) {
      this.$query.preload(include.relation, (query) => {
        if (include.scope.limit) query.limit(include.scope.limit);
        if (include.scope.orderBy) query.orderBy(include.scope.orderBy);
      });
    } else {
      this.$query.preload(include.relation);
    }
  }
}
```

---

## 2. Otimizar Paginação

### Problema: OFFSET em Tabelas Grandes

```typescript
// ❌ RUIM: OFFSET alto é lento
const orders = await Order.query()
  .orderBy('created_at', 'desc')
  .paginate(1000, 25); // Offset de 25000 rows!
```

### Solução: Cursor-Based Pagination

```typescript
// ✅ BOM: Cursor-based (muito mais rápido)
const orders = await Order.query()
  .where('created_at', '<', lastSeenDate)
  .orderBy('created_at', 'desc')
  .limit(25);

// ✅ Com ID para desempate
const orders = await Order.query()
  .where((query) => {
    query
      .where('created_at', '<', lastSeenDate)
      .orWhere((q) => {
        q.where('created_at', '=', lastSeenDate)
          .where('id', '<', lastSeenId);
      });
  })
  .orderBy('created_at', 'desc')
  .orderBy('id', 'desc')
  .limit(25);
```

### Índice Necessário

```typescript
// Migration
table.index(['created_at', 'id'], 'idx_cursor_pagination');
```

---

## 3. Selecionar Apenas Colunas Necessárias

### Problema: SELECT *

```typescript
// ❌ RUIM: Traz todas as colunas
const users = await User.all();

// ❌ RUIM: Preload traz tudo do relacionamento
const orders = await Order.query().preload('customer').exec();
```

### Solução: Select Específico

```typescript
// ✅ BOM: Apenas colunas necessárias
const users = await User.query()
  .select('id', 'name', 'email')
  .exec();

// ✅ BOM: Preload com select
const orders = await Order.query()
  .select('id', 'customer_id', 'total', 'created_at')
  .preload('customer', (query) => {
    query.select('id', 'name', 'email');
  })
  .exec();
```

---

## 4. Otimizar Contagens

### Problema: COUNT(*) Lento

```typescript
// ❌ RUIM: Conta tudo
const total = await Order.query().count('* as total');
```

### Solução: withCount e Filtros

```typescript
// ✅ BOM: Count com filtro
const total = await Order.query()
  .where('status_id', StatusEnum.ACTIVE)
  .count('* as total');

// ✅ BOM: withCount em relacionamento
const users = await User.query()
  .withCount('orders')
  .withCount('orders', (query) => {
    query.where('status', 'completed').as('completedOrdersCount');
  })
  .exec();

// Acessa via $extras
users.forEach((user) => {
  console.log(user.$extras.orders_count);
  console.log(user.$extras.completedOrdersCount);
});
```

---

## 5. Índices Eficientes

### Migration com Índices

```typescript
public async up() {
  this.schema.createTable('orders', (table) => {
    table.increments('id');
    table.integer('customer_id').unsigned();
    table.integer('status_id').unsigned();
    table.timestamp('created_at');
    table.timestamp('delivery_date');
    
    // Foreign Keys
    table.foreign('customer_id').references('customers.id').onDelete('CASCADE');
    
    // Índices simples
    table.index(['customer_id']);
    table.index(['status_id']);
    table.index(['created_at']);
    
    // Índice composto (ordem importa!)
    table.index(['status_id', 'created_at'], 'idx_status_created');
    
    // Índice para busca de texto
    table.index(['customer_name'], 'idx_customer_name');
  });
}

// Índice parcial (PostgreSQL)
this.schema.raw(`
  CREATE INDEX idx_active_orders 
  ON orders (created_at) 
  WHERE deleted_at IS NULL AND status_id != 9
`);
```

### Quando Criar Índices

| Situação | Índice |
|----------|--------|
| WHERE em coluna | Índice simples |
| WHERE em múltiplas colunas | Índice composto |
| ORDER BY | Índice na coluna de ordenação |
| JOIN/FK | Índice na FK |
| LIKE 'termo%' | Índice B-tree |
| LIKE '%termo%' | Índice GIN (full-text) |

---

## 6. Subqueries Eficientes

### Problema: Subquery Correlacionada

```typescript
// ❌ RUIM: Executa subquery para cada linha
const users = await Database.rawQuery(`
  SELECT u.*, 
    (SELECT COUNT(*) FROM orders WHERE user_id = u.id) as order_count
  FROM users u
`);
```

### Solução: JOIN ou Window Function

```typescript
// ✅ BOM: JOIN com GROUP BY
const users = await User.query()
  .select('users.*')
  .select(Database.raw('COUNT(orders.id) as order_count'))
  .leftJoin('orders', 'users.id', 'orders.user_id')
  .groupBy('users.id')
  .exec();

// ✅ BOM: withCount do Lucid
const users = await User.query()
  .withCount('orders')
  .exec();
```

---

## 7. Batch Operations

### Insert em Lote

```typescript
// ❌ RUIM: Insert individual
for (const item of items) {
  await OrderItem.create(item);
}

// ✅ BOM: Insert em lote
await OrderItem.createMany(items);

// ✅ BOM: Com chunk para lotes grandes
const chunkSize = 1000;
for (let i = 0; i < items.length; i += chunkSize) {
  const chunk = items.slice(i, i + chunkSize);
  await OrderItem.createMany(chunk);
}
```

### Update em Lote

```typescript
// ❌ RUIM: Update individual
for (const id of ids) {
  await Order.query().where('id', id).update({ status: 'processed' });
}

// ✅ BOM: Update em lote
await Order.query()
  .whereIn('id', ids)
  .update({ status: 'processed' });
```

---

## 8. Transações Otimizadas

```typescript
import Database from '@ioc:Adonis/Lucid/Database';

// ✅ Transação com rollback automático em erro
const result = await Database.transaction(async (trx) => {
  const order = await Order.create({ ... }, { client: trx });
  
  // Insert em lote na transação
  await OrderItem.createMany(
    items.map((item) => ({ ...item, orderId: order.id })),
    { client: trx }
  );
  
  // Update relacionado
  await Product.query({ client: trx })
    .whereIn('id', items.map((i) => i.productId))
    .decrement('stock', 1);
  
  return order;
});
```

---

## 9. Query Debugging

### Habilitar Query Log

```typescript
// Em desenvolvimento
import Database from '@ioc:Adonis/Lucid/Database';

Database.on('query', (query) => {
  console.log(query.sql);
  console.log(query.bindings);
});

// Log apenas queries lentas (> 100ms)
Database.on('query', (query) => {
  if (query.duration > 100) {
    console.warn('Slow query:', query.sql, query.duration);
  }
});
```

### Obter SQL da Query

```typescript
// Ver SQL sem executar
const query = Order.query()
  .where('status_id', 1)
  .preload('customer');

console.log(query.toSQL());
// ou
console.log(query.toQuery());
```

### EXPLAIN no PostgreSQL

```typescript
// Via raw query
const explain = await Database.rawQuery(`
  EXPLAIN ANALYZE
  SELECT * FROM orders 
  WHERE status_id = 1 
  AND created_at > NOW() - INTERVAL '30 days'
`);

console.log(explain.rows);
```

---

## 10. Cache de Queries

### Com Redis

```typescript
import Redis from '@ioc:Adonis/Addons/Redis';

class OrderRepository extends Repository<Order> {
  public async getActiveOrders(): Promise<Order[]> {
    const cacheKey = 'orders:active';
    
    // Tentar cache
    const cached = await Redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // Buscar no banco
    const orders = await this.model.query()
      .where('status_id', StatusEnum.ACTIVE)
      .preload('customer')
      .exec();
    
    // Salvar no cache (5 minutos)
    await Redis.setex(cacheKey, 300, JSON.stringify(orders));
    
    return orders;
  }
  
  // Invalidar cache ao modificar
  public async updateStatus(id: number, statusId: number): Promise<Order> {
    const order = await super.update(id, { statusId });
    await Redis.del('orders:active');
    return order;
  }
}
```

---

## 11. Padrões Anti-Pattern

### ❌ Evitar

```typescript
// N+1
for (const order of orders) {
  await order.load('customer');
}

// SELECT * desnecessário
await Order.all();

// Count sem filtro em tabela grande
await Order.query().count('*');

// LIKE com % no início (não usa índice)
.whereILike('name', '%termo%')

// Offset alto
.paginate(10000, 25)

// Muitos preloads não utilizados
.preload('a').preload('b').preload('c').preload('d')
```

### ✅ Preferir

```typescript
// Preload antecipado
await Order.query().preload('customer').exec();

// Select específico
await Order.query().select('id', 'total').exec();

// Count com filtro
await Order.query().where('status', 1).count('*');

// LIKE apenas no final (usa índice)
.whereILike('name', 'termo%')

// Cursor pagination
.where('id', '<', lastId).limit(25)

// Preload apenas o necessário
.preload('customer')
```

---

## Checklist de Otimização

```
[ ] Preloads ao invés de loads individuais
[ ] Select apenas colunas necessárias
[ ] Índices em colunas de WHERE e JOIN
[ ] Índices compostos para filtros frequentes
[ ] Paginação com cursor para tabelas grandes
[ ] Batch para operações em massa
[ ] Cache para queries repetitivas
[ ] EXPLAIN em queries complexas
[ ] Logs de queries lentas habilitados
```
