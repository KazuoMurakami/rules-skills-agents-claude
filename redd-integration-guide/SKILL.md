---
name: redd-integration-guide
description: Guia de integração entre Backend (AdonisJS) e Frontend (Next.js) do REDD. Use quando precisar entender como os dados fluem entre as camadas, como configurar novos endpoints, ou como consumir APIs corretamente.
---

# REDD Integration Guide

Guia completo de integração entre redd-api (Backend) e redd-client (Frontend).

## Fluxo de Dados Completo

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            FRONTEND                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────────┐  │
│  │   Page   │───▶│ Provider │───▶│useQuery/ │───▶│   API Resource   │  │
│  │Component │    │ (Context)│    │useMutation│   │  (HttpClient)    │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                        │
                              HTTP Request (Axios)
                              Authorization: Bearer {token}
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                             BACKEND                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────────┐  │
│  │  Routes  │───▶│Controller│───▶│Repository│───▶│  Model + Filter  │  │
│  │          │    │+Validator│    │          │    │                  │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                        │
                                   PostgreSQL
```

---

## 1. Criando Nova Feature (End-to-End)

### Exemplo: Módulo de "Tickets"

#### Backend (1/2)

```typescript
// 1. Migration
// database/migrations/TIMESTAMP_create_tickets.ts
public async up() {
  this.schema.createTable('tickets', (table) => {
    table.increments('id');
    table.integer('user_id').unsigned().references('users.id');
    table.integer('status_id').unsigned().references('ticket_statuses.id');
    table.string('title', 255).notNullable();
    table.text('description');
    table.integer('priority').defaultTo(1);
    table.timestamp('created_at', { useTz: true });
    table.timestamp('updated_at', { useTz: true });
    table.timestamp('deleted_at', { useTz: true });
    
    table.index(['user_id']);
    table.index(['status_id']);
  });
}

// 2. Model
// app/Models/Ticket/Ticket.ts
export default class Ticket extends Model {
  public static table = 'tickets';
  public static $filter = () => TicketFilter;

  @column({ isPrimary: true })
  public id: number;

  @column({ columnName: 'user_id', serializeAs: 'userId' })
  public userId: number;

  @column({ columnName: 'status_id', serializeAs: 'statusId' })
  public statusId: number;

  @column()
  public title: string;

  @column()
  public description: string | null;

  @column()
  public priority: number;

  @belongsTo(() => User, { foreignKey: 'userId' })
  public user: BelongsTo<typeof User>;

  @belongsTo(() => TicketStatus, { foreignKey: 'statusId' })
  public status: BelongsTo<typeof TicketStatus>;
}

// 3. Filter
// app/Models/Filters/Ticket/TicketFilter.ts
export default class TicketFilter extends Filter {
  public userId(value: number) {
    this.$query.where('userId', value);
  }

  public statusId(value: number | number[]) {
    const ids = Array.isArray(value) ? value : [value];
    this.$query.whereIn('statusId', ids);
  }

  public priority(value: number) {
    this.$query.where('priority', value);
  }

  public search(term: string) {
    this.$query.where((query) => {
      query.whereILike('title', `%${term}%`)
           .orWhereILike('description', `%${term}%`);
    });
  }

  public includes(value: string) {
    const includes = JSON.parse(value);
    for (const include of includes) {
      this.$query.preload(include.relation);
    }
  }

  public orderBy(value: string[]) {
    for (const order of value) {
      const [field, direction] = order.split(',');
      this.$query.orderBy(field, direction as 'asc' | 'desc');
    }
  }
}

// 4. Repository
// app/Repositories/Ticket/TicketRepository.ts
export default class TicketRepository extends Repository<Ticket> {
  protected model = Ticket;

  public async getByUser(userId: number): Promise<Ticket[]> {
    return this.model.query()
      .where('userId', userId)
      .preload('status')
      .orderBy('created_at', 'desc')
      .exec();
  }
}

// 5. Validator
// app/Validator/Ticket/TicketValidator.ts
export default class TicketValidator extends Validator implements ValidatorContract {
  public async create(request: RequestContract) {
    return await this.validate(
      schema.create({
        title: schema.string({}, [rules.maxLength(255)]),
        description: schema.string.optional(),
        statusId: schema.number([rules.exists({ table: 'ticket_statuses', column: 'id' })]),
        priority: schema.number.optional([rules.range(1, 5)])
      }),
      request
    );
  }

  public async update(request: RequestContract) {
    return await this.validate(
      schema.create({
        title: schema.string.optional({}, [rules.maxLength(255)]),
        description: schema.string.optional(),
        statusId: schema.number.optional([rules.exists({ table: 'ticket_statuses', column: 'id' })]),
        priority: schema.number.optional([rules.range(1, 5)])
      }),
      request
    );
  }
}

// 6. Controller
// app/Controllers/Http/Ticket/TicketsController.ts
export default class TicketsController extends Controller {
  protected repository: TicketRepository;
  protected validator: TicketValidator;

  constructor() {
    super();
    this.repository = new TicketRepository();
    this.validator = new TicketValidator();
  }

  public async index(ctx: HttpContextContract) {
    ctx.params.includes = JSON.stringify(['user', 'status']);
    return super.index(ctx);
  }

  public async store(ctx: HttpContextContract) {
    ctx.request.updateBody({
      ...ctx.request.all(),
      userId: ctx.auth.user?.id
    });
    return super.store(ctx);
  }
}

// 7. Routes
// start/routes/tickets.ts
Route.group(() => {
  Route.get('/', 'TicketsController.index');
  Route.post('/', 'TicketsController.store');
  Route.get('/:id', 'TicketsController.show');
  Route.put('/:id', 'TicketsController.update');
  Route.delete('/:id', 'TicketsController.destroy');
})
  .prefix('tickets')
  .middleware('auth')
  .namespace('App/Controllers/Http/Ticket');
```

#### Frontend (2/2)

```typescript
// 1. Types
// services/types/entities/ticket/ticket.d.ts
export interface TicketProps {
  id: number;
  userId: number;
  statusId: number;
  title: string;
  description?: string;
  priority: number;
  user?: UserProps;
  status?: TicketStatusProps;
  createdAt: string;
  updatedAt: string;
}

export interface TicketFormData {
  title: string;
  description?: string;
  statusId: number;
  priority?: number;
}

// 2. API Resource
// services/api/resources/ticket/ticket-resource.ts
import { HttpClient } from '../../http-client';
import { TicketProps } from '@/services/types/entities/ticket/ticket';

class TicketResource extends HttpClient<TicketProps> {
  constructor() {
    super('/tickets');
  }

  async getByUser(userId: number): Promise<TicketProps[]> {
    return this.getAll({
      listType: 'list',
      filters: { userId },
      includes: [{ relation: 'status' }]
    });
  }
}

export const ticketResource = new TicketResource();

// 3. Provider
// components/page/Tickets/hooks/ProviderTicket.tsx
import { createContext, useContext, useState, ReactNode } from 'react';
import { useQuery, useMutation, useQueryClient } from 'react-query';
import { ticketResource } from '@/services/api/resources/ticket';
import { TicketProps } from '@/services/types/entities/ticket/ticket';
import { Pagination } from '@/services/types/pagination';
import { useToast } from 'ui/hooks/toast';

const includes = [{ relation: 'user' }, { relation: 'status' }];

interface TicketContextData {
  query: UseQueryResult<Pagination<TicketProps>>;
  page: number;
  setPage: (page: number) => void;
  handleDelete: (id: number) => void;
}

const TicketContext = createContext({} as TicketContextData);

export function ProviderTicket({ children }: { children: ReactNode }) {
  const { addToast } = useToast();
  const queryClient = useQueryClient();
  const [page, setPage] = useState(1);

  const query = useQuery<Pagination<TicketProps>>(
    ['tickets', { page }],
    () => ticketResource.paginate({ page, perPage: 25, includes }),
    { refetchOnWindowFocus: false }
  );

  const deleteMutation = useMutation(
    (id: number) => ticketResource.delete(id),
    {
      onSuccess: () => {
        queryClient.invalidateQueries(['tickets']);
        addToast({ type: 'success', description: 'Ticket removido' });
      }
    }
  );

  return (
    <TicketContext.Provider value={{ query, page, setPage, handleDelete: deleteMutation.mutate }}>
      {children}
    </TicketContext.Provider>
  );
}

export const useTicket = () => useContext(TicketContext);

// 4. Schema (Zod)
// components/page/Tickets/components/FormTicket/schema.ts
import { z } from 'zod';

export const ticketSchema = z.object({
  title: z.string().min(1, 'Título é obrigatório').max(255),
  description: z.string().optional(),
  statusId: z.number({ required_error: 'Status é obrigatório' }),
  priority: z.number().min(1).max(5).optional().default(1)
});

export type TicketFormData = z.infer<typeof ticketSchema>;

// 5. Page Component
// components/page/Tickets/Tickets.tsx
import { useState } from 'react';
import { Page } from '@/components/shared/Page/Page';
import { ProviderTicket } from './hooks/ProviderTicket';
import { TicketDataGrid } from './components/TicketDataGrid';
import { TicketFilters } from './components/TicketFilters';
import { ModalTicket } from './components/FormTicket/ModalTicket';

export function Tickets() {
  const [modalOpen, setModalOpen] = useState(false);

  return (
    <Page>
      <ProviderTicket>
        <TicketFilters onAdd={() => setModalOpen(true)} />
        <TicketDataGrid />
        <ModalTicket open={modalOpen} onClose={() => setModalOpen(false)} />
      </ProviderTicket>
    </Page>
  );
}
```

---

## 2. Mapeamento de Dados

### Nomenclatura

| Backend (DB/Model) | Frontend (TypeScript) | JSON Response |
|-------------------|----------------------|---------------|
| `user_id` | `userId` | `userId` |
| `status_id` | `statusId` | `statusId` |
| `created_at` | `createdAt` | `createdAt` |
| `is_active` | `isActive` | `isActive` |

### Como Funciona

```typescript
// Backend Model
@column({ columnName: 'status_id', serializeAs: 'statusId' })
public statusId: number;

// JSON enviado: { "statusId": 1 }

// Frontend Type
interface TicketProps {
  statusId: number; // Recebe camelCase
}
```

---

## 3. Filtros e Includes

### Frontend Request

```typescript
// No Provider
const query = useQuery(
  ['tickets', { page, filters }],
  () => ticketResource.paginate({
    page,
    perPage: 25,
    filters: {
      search: 'termo',
      statusId: [1, 2],
      priority: 3
    },
    includes: [
      { relation: 'user' },
      { relation: 'status' },
      { relation: 'comments', scope: { limit: 5 } }
    ],
    orderBy: ['createdAt,desc', 'priority,asc']
  })
);
```

### URL Gerada

```
GET /tickets?page=1&perPage=25&search=termo&statusId[]=1&statusId[]=2&priority=3&includes=[{"relation":"user"},{"relation":"status"},{"relation":"comments","scope":{"limit":5}}]&orderBy[]=createdAt,desc&orderBy[]=priority,asc
```

### Backend Filter Processing

```typescript
// No Filter do Model
public search(term: string) {
  this.$query.whereILike('title', `%${term}%`);
}

public statusId(ids: number | number[]) {
  const arr = Array.isArray(ids) ? ids : [ids];
  this.$query.whereIn('statusId', arr);
}

public includes(value: string) {
  const includes = JSON.parse(value);
  for (const include of includes) {
    if (include.scope?.limit) {
      this.$query.preload(include.relation, (q) => q.limit(include.scope.limit));
    } else {
      this.$query.preload(include.relation);
    }
  }
}

public orderBy(values: string[]) {
  for (const value of values) {
    const [field, dir] = value.split(',');
    this.$query.orderBy(field, dir as 'asc' | 'desc');
  }
}
```

---

## 4. Autenticação

### Frontend: Armazenamento e Envio

```typescript
// Auth Service armazena tokens após login
const { token, refreshToken } = await authService.login(email, password);

// Interceptor adiciona token em todas requests
// services/api/config/setupApiInterceptor.ts
client.interceptors.request.use((config) => {
  const token = getToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Refresh automático quando token expira
// auth/services/token.ts
export function setupRefreshToken({ client }) {
  client.interceptors.response.use(
    (response) => response,
    async (error) => {
      if (error.response?.status === 401 && !error.config._retry) {
        error.config._retry = true;
        const newToken = await refreshToken();
        error.config.headers.Authorization = `Bearer ${newToken}`;
        return client(error.config);
      }
      return Promise.reject(error);
    }
  );
  return client;
}
```

### Backend: Middleware de Auth

```typescript
// start/kernel.ts
Server.middleware.registerNamed({
  auth: () => import('App/Middleware/Auth')
});

// Routes
Route.group(() => {
  Route.get('/tickets', 'TicketsController.index');
})
  .middleware('auth'); // Requer autenticação

// Controller acessa usuário
public async store(ctx: HttpContextContract) {
  const userId = ctx.auth.user?.id; // Usuário autenticado
}
```

---

## 5. Tratamento de Erros

### Backend: Exceções

```typescript
// Lançar erro
throw new HttpException('Ticket não encontrado', 404);
throw new HttpException('Não autorizado', 403);
throw new ValidationException([{ field: 'title', message: 'Título é obrigatório' }]);
```

### Frontend: Tratamento

```typescript
// No useMutation
const mutation = useMutation(
  (data: TicketFormData) => ticketResource.create(data),
  {
    onSuccess: () => {
      addToast({ type: 'success', description: 'Ticket criado' });
    },
    onError: (error: any) => {
      // Erro de validação (422)
      if (error.response?.status === 422) {
        const errors = error.response.data.errors;
        errors.forEach((err) => {
          setError(err.field, { message: err.message });
        });
        return;
      }
      
      // Erro genérico
      const message = error.response?.data?.error || 'Erro ao criar ticket';
      addToast({ type: 'error', description: message });
    }
  }
);
```

---

## 6. Paginação

### Backend Response

```json
{
  "data": [...],
  "total": 150,
  "perPage": 25,
  "currentPage": 1,
  "lastPage": 6,
  "firstPage": 1,
  "firstPageUrl": "/?page=1",
  "lastPageUrl": "/?page=6",
  "nextPageUrl": "/?page=2",
  "previousPageUrl": null
}
```

### Frontend Type

```typescript
// services/types/pagination.d.ts
export interface Pagination<T> {
  data: T[];
  total: number;
  perPage: number;
  currentPage: number;
  lastPage: number;
  firstPage: number;
}
```

### Frontend Usage

```typescript
const { data } = useQuery<Pagination<TicketProps>>(['tickets', { page }], ...);

// No DataGrid
<DataGrid
  rows={data?.data || []}
  rowCount={data?.total || 0}
  page={page - 1} // MUI é 0-indexed
  pageSize={data?.perPage || 25}
  onPageChange={(newPage) => setPage(newPage + 1)}
  paginationMode="server"
/>
```

---

## Checklist de Integração

### Backend

```
[ ] Migration com índices e FKs
[ ] Model com Filter e relacionamentos
[ ] Repository estendendo base
[ ] Validator para create/update
[ ] Controller com includes padrão
[ ] Routes com middleware auth
```

### Frontend

```
[ ] Types em services/types/entities/
[ ] API Resource em services/api/resources/
[ ] Provider com useQuery
[ ] Schema Zod para forms
[ ] Components com tipagem
[ ] Error handling no mutation
```

### Integração

```
[ ] Nomenclatura camelCase consistente
[ ] Includes configurados corretamente
[ ] Filtros funcionando
[ ] Paginação server-side
[ ] Auth token enviado
[ ] Erros tratados
```
