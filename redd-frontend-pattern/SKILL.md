---
name: redd-frontend-pattern
description: Gera código frontend seguindo o padrão arquitetural do REDD (Next.js 13 + MUI + React Query). Use quando criar novas páginas, componentes, formulários, ou integrar com API no redd-client. Aplica automaticamente as camadas Page, Provider, Components, Hooks, Services e Types.
---

# REDD Frontend Pattern

Skill para geração de código frontend no projeto REDD seguindo a arquitetura em camadas.

## Estrutura de Camadas

```
redd-client/src/
├── pages/                    # Rotas Next.js
├── components/page/{Domain}/ # Componentes por domínio
│   ├── {Entity}.tsx          # Componente principal
│   ├── hooks/                # Providers e hooks
│   └── components/           # Sub-componentes
├── services/api/resources/   # API Resources
├── services/types/entities/  # Tipos TypeScript
└── enums/                    # Enumeradores
```

## Ordem de Criação

Ao criar uma nova página, siga esta ordem:

1. **Types** → Definição de tipos
2. **API Resource** → Chamadas HTTP
3. **Enums** (se necessário)
4. **Provider** → Estado e lógica
5. **Components** → UI components
6. **Page** → Componente de página
7. **Next.js Page** → Rota

---

## 1. Types

```typescript
// services/types/entities/{domain}/{entity}.d.ts
export interface EntityProps {
  id: number;
  name: string;
  description?: string;
  statusId: number;
  parentId: number;
  isActive: boolean;
  
  // Relacionamentos (opcionais - vêm com includes)
  status?: StatusProps;
  parent?: ParentProps;
  children?: ChildProps[];
  
  // Timestamps
  createdAt: string;
  updatedAt: string;
}

// Tipos para formulários
export interface EntityFormData {
  name: string;
  description?: string;
  statusId: number;
  parentId: number;
  isActive?: boolean;
}

// Tipos para filtros
export interface EntityFilters {
  search?: string;
  statusId?: number | number[];
  parentId?: number;
  isActive?: boolean;
}
```

---

## 2. API Resource

```typescript
// services/api/resources/{domain}/{entity}-resource.ts
import { HttpClient } from '../../http-client';
import { EntityProps } from '@/services/types/entities/{domain}/{entity}';

class EntityResource extends HttpClient<EntityProps> {
  constructor() {
    super('/entities');
  }

  // Métodos customizados
  async activate(id: number): Promise<EntityProps> {
    const response = await this.client.put(`${this.path}/${id}/activate`);
    return response.data;
  }

  async deactivate(id: number): Promise<EntityProps> {
    const response = await this.client.put(`${this.path}/${id}/deactivate`);
    return response.data;
  }

  async getByParent(parentId: number): Promise<EntityProps[]> {
    return this.getAll({
      listType: 'list',
      filters: { parentId }
    });
  }
}

export const entityResource = new EntityResource();
```

```typescript
// services/api/resources/{domain}/index.ts
export * from './{entity}-resource';
```

---

## 3. Enums (se necessário)

```typescript
// enums/{Entity}StatusEnum.ts
export enum EntityStatusEnum {
  PENDING = 1,
  ACTIVE = 2,
  INACTIVE = 3,
  CANCELLED = 4
}

export const EntityStatusLabels: Record<EntityStatusEnum, string> = {
  [EntityStatusEnum.PENDING]: 'Pendente',
  [EntityStatusEnum.ACTIVE]: 'Ativo',
  [EntityStatusEnum.INACTIVE]: 'Inativo',
  [EntityStatusEnum.CANCELLED]: 'Cancelado'
};
```

---

## 4. Provider (Context)

```typescript
// components/page/{Entity}/hooks/Provider{Entity}.tsx
import {
  createContext,
  useContext,
  ReactNode,
  useState,
  useCallback
} from 'react';
import { useQuery, useMutation, useQueryClient, UseQueryResult } from 'react-query';
import { GridSortModel } from '@mui/x-data-grid';
import { entityResource } from '@/services/api/resources/{domain}';
import { EntityProps, EntityFilters } from '@/services/types/entities/{domain}/{entity}';
import { Pagination } from '@/services/types/pagination';
import { useToast } from 'ui/hooks/toast';

// Includes padrão para listagem
const includes = [
  { relation: 'status' },
  { relation: 'parent' }
];

// Interface do contexto
interface EntityContextData {
  // Query principal
  query: UseQueryResult<Pagination<EntityProps>, unknown>;
  
  // Paginação
  page: number;
  setPage: (page: number) => void;
  perPage: number;
  setPerPage: (perPage: number) => void;
  
  // Ordenação
  sortModel: GridSortModel;
  setSortModel: (model: GridSortModel) => void;
  
  // Filtros
  filters: EntityFilters;
  setFilters: (filters: EntityFilters) => void;
  
  // Ações
  handleDelete: (id: number) => void;
  handleRefresh: () => void;
  
  // Loading states
  deleteLoading: boolean;
}

const EntityContext = createContext({} as EntityContextData);

export function ProviderEntity({ children }: { children: ReactNode }) {
  const { addToast } = useToast();
  const queryClient = useQueryClient();
  
  // Estado de paginação
  const [page, setPage] = useState(1);
  const [perPage, setPerPage] = useState(25);
  
  // Estado de ordenação
  const [sortModel, setSortModel] = useState<GridSortModel>([
    { field: 'createdAt', sort: 'desc' }
  ]);
  
  // Estado de filtros
  const [filters, setFilters] = useState<EntityFilters>({});

  // Query principal
  const query = useQuery<Pagination<EntityProps>>(
    ['entities', { page, perPage, sortModel, filters }],
    () => entityResource.paginate({
      page,
      perPage,
      includes,
      filters: {
        search: filters.search || undefined,
        statusId: filters.statusId || undefined,
        parentId: filters.parentId || undefined
      },
      orderBy: sortModel.map((m) => `${m.field},${m.sort}`)
    }),
    {
      refetchOnWindowFocus: false,
      keepPreviousData: true
    }
  );

  // Mutation de delete
  const deleteMutation = useMutation(
    (id: number) => entityResource.delete(id),
    {
      onSuccess: () => {
        queryClient.invalidateQueries(['entities']);
        addToast({ type: 'success', description: 'Registro removido com sucesso' });
      },
      onError: () => {
        addToast({ type: 'error', description: 'Erro ao remover registro' });
      }
    }
  );

  // Handlers
  const handleDelete = useCallback((id: number) => {
    deleteMutation.mutate(id);
  }, [deleteMutation]);

  const handleRefresh = useCallback(() => {
    query.refetch();
  }, [query]);

  return (
    <EntityContext.Provider
      value={{
        query,
        page,
        setPage,
        perPage,
        setPerPage,
        sortModel,
        setSortModel,
        filters,
        setFilters,
        handleDelete,
        handleRefresh,
        deleteLoading: deleteMutation.isLoading
      }}
    >
      {children}
    </EntityContext.Provider>
  );
}

export function useEntity() {
  const context = useContext(EntityContext);
  if (!context) {
    throw new Error('useEntity must be used within ProviderEntity');
  }
  return context;
}
```

---

## 5. Components

### 5.1 DataGrid

```typescript
// components/page/{Entity}/components/{Entity}DataGrid/{Entity}DataGrid.tsx
import { DataGrid, GridColDef, GridRenderCellParams } from '@mui/x-data-grid';
import { useEntity } from '../../hooks/Provider{Entity}';
import { EntityProps } from '@/services/types/entities/{domain}/{entity}';
import { Actions } from './components/Actions';
import { formatDate } from 'utils/date';

const columns: GridColDef[] = [
  { field: 'id', headerName: 'ID', width: 70 },
  { field: 'name', headerName: 'Nome', flex: 1, minWidth: 200 },
  {
    field: 'status',
    headerName: 'Status',
    width: 150,
    valueGetter: (params) => params.row.status?.name
  },
  {
    field: 'parent',
    headerName: 'Pai',
    width: 150,
    valueGetter: (params) => params.row.parent?.name
  },
  {
    field: 'createdAt',
    headerName: 'Criado em',
    width: 150,
    valueFormatter: (params) => formatDate(params.value)
  },
  {
    field: 'actions',
    headerName: 'Ações',
    width: 120,
    sortable: false,
    filterable: false,
    renderCell: (params: GridRenderCellParams<EntityProps>) => (
      <Actions entity={params.row} />
    )
  }
];

export function EntityDataGrid() {
  const { query, page, setPage, perPage, setPerPage, sortModel, setSortModel } = useEntity();

  return (
    <DataGrid
      rows={query.data?.data || []}
      columns={columns}
      loading={query.isLoading}
      // Paginação server-side
      pagination
      paginationMode="server"
      page={page - 1}
      pageSize={perPage}
      rowCount={query.data?.total || 0}
      onPageChange={(newPage) => setPage(newPage + 1)}
      onPageSizeChange={setPerPage}
      rowsPerPageOptions={[10, 25, 50, 100]}
      // Ordenação server-side
      sortingMode="server"
      sortModel={sortModel}
      onSortModelChange={setSortModel}
      // Config
      autoHeight
      disableSelectionOnClick
      getRowId={(row) => row.id}
    />
  );
}
```

### 5.2 Filters

```typescript
// components/page/{Entity}/components/{Entity}Filters/{Entity}Filters.tsx
import { useState } from 'react';
import Box from '@mui/material/Box';
import TextField from '@mui/material/TextField';
import Button from '@mui/material/Button';
import AddIcon from '@mui/icons-material/Add';
import FilterListIcon from '@mui/icons-material/FilterList';
import { useEntity } from '../../hooks/Provider{Entity}';
import useDebounce from 'ui/hooks/queries/useDebounce';

interface EntityFiltersProps {
  onAdd: () => void;
}

export function EntityFilters({ onAdd }: EntityFiltersProps) {
  const { filters, setFilters } = useEntity();
  const [search, setSearch] = useState(filters.search || '');
  
  // Debounce para busca
  const debouncedSearch = useDebounce(search, 500);
  
  // Atualiza filtros quando debounce muda
  useEffect(() => {
    setFilters({ ...filters, search: debouncedSearch });
  }, [debouncedSearch]);

  return (
    <Box display="flex" gap={2} alignItems="center" mb={2}>
      <TextField
        size="small"
        placeholder="Buscar..."
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        sx={{ minWidth: 250 }}
      />
      
      <Button
        variant="outlined"
        startIcon={<FilterListIcon />}
        onClick={() => {/* Abrir modal de filtros avançados */}}
      >
        Filtros
      </Button>

      <Box flexGrow={1} />

      <Button
        variant="contained"
        startIcon={<AddIcon />}
        onClick={onAdd}
      >
        Adicionar
      </Button>
    </Box>
  );
}
```

### 5.3 Form com Schema

```typescript
// components/page/{Entity}/components/Form{Entity}/schema.ts
import { z } from 'zod';

export const entitySchema = z.object({
  name: z
    .string()
    .min(1, 'Nome é obrigatório')
    .max(255, 'Máximo 255 caracteres'),
  description: z
    .string()
    .max(1000, 'Máximo 1000 caracteres')
    .optional()
    .nullable(),
  statusId: z
    .number({ required_error: 'Status é obrigatório' }),
  parentId: z
    .number({ required_error: 'Pai é obrigatório' }),
  isActive: z
    .boolean()
    .optional()
    .default(true)
});

export type EntityFormData = z.infer<typeof entitySchema>;

// Schema para edição (todos opcionais exceto id)
export const entityUpdateSchema = entitySchema.partial().extend({
  id: z.number()
});

export type EntityUpdateFormData = z.infer<typeof entityUpdateSchema>;
```

```typescript
// components/page/{Entity}/components/Form{Entity}/Form{Entity}.tsx
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { useMutation, useQueryClient } from 'react-query';
import Box from '@mui/material/Box';
import TextField from '@mui/material/TextField';
import Button from '@mui/material/Button';
import CircularProgress from '@mui/material/CircularProgress';
import { entityResource } from '@/services/api/resources/{domain}';
import { useToast } from 'ui/hooks/toast';
import { entitySchema, EntityFormData } from './schema';
import { StatusSelect } from './components/StatusSelect';
import { ParentSelect } from './components/ParentSelect';

interface FormEntityProps {
  initialData?: Partial<EntityFormData>;
  entityId?: number;
  onSuccess?: () => void;
}

export function FormEntity({ initialData, entityId, onSuccess }: FormEntityProps) {
  const { addToast } = useToast();
  const queryClient = useQueryClient();
  const isEditing = !!entityId;

  const {
    control,
    handleSubmit,
    reset,
    formState: { errors, isDirty }
  } = useForm<EntityFormData>({
    resolver: zodResolver(entitySchema),
    defaultValues: {
      name: '',
      description: '',
      statusId: undefined,
      parentId: undefined,
      isActive: true,
      ...initialData
    }
  });

  const mutation = useMutation(
    (data: EntityFormData) =>
      isEditing
        ? entityResource.update(entityId, data)
        : entityResource.create(data),
    {
      onSuccess: () => {
        queryClient.invalidateQueries(['entities']);
        addToast({
          type: 'success',
          description: isEditing
            ? 'Registro atualizado com sucesso'
            : 'Registro criado com sucesso'
        });
        if (!isEditing) reset();
        onSuccess?.();
      },
      onError: (error: any) => {
        const message = error?.response?.data?.error || 'Erro ao salvar';
        addToast({ type: 'error', description: message });
      }
    }
  );

  const onSubmit = handleSubmit((data) => {
    mutation.mutate(data);
  });

  return (
    <form onSubmit={onSubmit}>
      <Box display="flex" flexDirection="column" gap={2}>
        <Controller
          name="name"
          control={control}
          render={({ field }) => (
            <TextField
              {...field}
              label="Nome"
              error={!!errors.name}
              helperText={errors.name?.message}
              fullWidth
              required
            />
          )}
        />

        <Controller
          name="description"
          control={control}
          render={({ field }) => (
            <TextField
              {...field}
              label="Descrição"
              multiline
              rows={3}
              error={!!errors.description}
              helperText={errors.description?.message}
              fullWidth
            />
          )}
        />

        <Controller
          name="statusId"
          control={control}
          render={({ field }) => (
            <StatusSelect
              value={field.value}
              onChange={field.onChange}
              error={errors.statusId?.message}
            />
          )}
        />

        <Controller
          name="parentId"
          control={control}
          render={({ field }) => (
            <ParentSelect
              value={field.value}
              onChange={field.onChange}
              error={errors.parentId?.message}
            />
          )}
        />

        <Box display="flex" gap={2} justifyContent="flex-end" mt={2}>
          <Button
            type="submit"
            variant="contained"
            disabled={mutation.isLoading || !isDirty}
            startIcon={mutation.isLoading && <CircularProgress size={16} />}
          >
            {isEditing ? 'Salvar' : 'Criar'}
          </Button>
        </Box>
      </Box>
    </form>
  );
}
```

### 5.4 Modal Form

```typescript
// components/page/{Entity}/components/Form{Entity}/Modal{Entity}.tsx
import Dialog from '@mui/material/Dialog';
import DialogTitle from '@mui/material/DialogTitle';
import DialogContent from '@mui/material/DialogContent';
import IconButton from '@mui/material/IconButton';
import CloseIcon from '@mui/icons-material/Close';
import { FormEntity } from './Form{Entity}';
import { EntityFormData } from './schema';

interface ModalEntityProps {
  open: boolean;
  onClose: () => void;
  initialData?: Partial<EntityFormData>;
  entityId?: number;
}

export function ModalEntity({ open, onClose, initialData, entityId }: ModalEntityProps) {
  const title = entityId ? 'Editar Registro' : 'Novo Registro';

  return (
    <Dialog open={open} onClose={onClose} maxWidth="sm" fullWidth>
      <DialogTitle>
        {title}
        <IconButton
          onClick={onClose}
          sx={{ position: 'absolute', right: 8, top: 8 }}
        >
          <CloseIcon />
        </IconButton>
      </DialogTitle>
      <DialogContent dividers>
        <FormEntity
          initialData={initialData}
          entityId={entityId}
          onSuccess={onClose}
        />
      </DialogContent>
    </Dialog>
  );
}
```

---

## 6. Page Component

```typescript
// components/page/{Entity}/{Entity}.tsx
import { useState } from 'react';
import Box from '@mui/material/Box';
import { Page } from '@/components/shared/Page/Page';
import { ProviderEntity } from './hooks/Provider{Entity}';
import { EntityDataGrid } from './components/{Entity}DataGrid/{Entity}DataGrid';
import { EntityFilters } from './components/{Entity}Filters/{Entity}Filters';
import { ModalEntity } from './components/Form{Entity}/Modal{Entity}';

export function Entity() {
  const [modalOpen, setModalOpen] = useState(false);
  const [editingId, setEditingId] = useState<number | undefined>();

  const handleAdd = () => {
    setEditingId(undefined);
    setModalOpen(true);
  };

  const handleEdit = (id: number) => {
    setEditingId(id);
    setModalOpen(true);
  };

  const handleCloseModal = () => {
    setModalOpen(false);
    setEditingId(undefined);
  };

  return (
    <Page>
      <ProviderEntity>
        <EntityFilters onAdd={handleAdd} />
        <Box mt={2}>
          <EntityDataGrid onEdit={handleEdit} />
        </Box>
        <ModalEntity
          open={modalOpen}
          onClose={handleCloseModal}
          entityId={editingId}
        />
      </ProviderEntity>
    </Page>
  );
}
```

```typescript
// components/page/{Entity}/index.ts
export * from './{Entity}';
```

---

## 7. Next.js Page

```typescript
// pages/{entities}.tsx
import { Entity } from '@/components/page/{Entity}';
import { BasicLayout } from '_layouts/BasicLayout/BasicLayout';

export default function EntityPage() {
  return (
    <BasicLayout title="Entidades">
      <Entity />
    </BasicLayout>
  );
}
```

---

## Convenções Importantes

### Estrutura de Pastas

```
components/page/{Entity}/
├── {Entity}.tsx                    # Componente principal
├── index.ts                        # Re-export
├── hooks/
│   └── Provider{Entity}.tsx        # Context provider
└── components/
    ├── {Entity}DataGrid/
    │   ├── {Entity}DataGrid.tsx
    │   ├── columns.tsx             # (opcional) colunas separadas
    │   └── components/
    │       └── Actions.tsx
    ├── {Entity}Filters/
    │   └── {Entity}Filters.tsx
    └── Form{Entity}/
        ├── Form{Entity}.tsx
        ├── Modal{Entity}.tsx
        ├── schema.ts
        └── components/
            ├── StatusSelect.tsx
            └── ParentSelect.tsx
```

### Regras de Tamanho

- **Máximo 200-300 linhas** por arquivo
- Extrair lógica para hooks customizados
- Extrair colunas complexas para arquivos separados
- Extrair campos de select para componentes

### Nomenclatura

| Elemento | Padrão | Exemplo |
|----------|--------|---------|
| Page Component | PascalCase singular | `Order.tsx` |
| Provider | Provider + Nome | `ProviderOrder.tsx` |
| Hook | use + Nome | `useOrder()` |
| DataGrid | Nome + DataGrid | `OrderDataGrid.tsx` |
| Form | Form + Nome | `FormOrder.tsx` |
| Modal | Modal + Nome | `ModalOrder.tsx` |
| Schema | `schema.ts` | `schema.ts` |
| Resource | nome-resource.ts | `order-resource.ts` |

### React Query Keys

```typescript
// Padrão de keys
['entities']                           // Lista
['entities', { page, filters }]        // Lista com params
['entity', id]                         // Item único
['entity', id, 'related']              // Relacionados
```

---

## Checklist de Nova Página

```
[ ] Types criados em services/types/entities/
[ ] API Resource criado em services/api/resources/
[ ] Enum criado (se necessário)
[ ] Provider com query e mutations
[ ] DataGrid com colunas tipadas
[ ] Filters com debounce
[ ] Form com schema Zod
[ ] Modal de criação/edição
[ ] Page component com composição
[ ] Next.js page com layout
```

---

## Integração Backend ↔ Frontend

### Mapeamento de Campos

| Backend (snake_case) | Frontend (camelCase) |
|---------------------|---------------------|
| `status_id` | `statusId` |
| `parent_id` | `parentId` |
| `created_at` | `createdAt` |
| `is_active` | `isActive` |

### Includes (Eager Loading)

```typescript
// Frontend
const includes = [
  { relation: 'status' },
  { relation: 'parent' },
  { relation: 'children', scope: { limit: 10 } }
];

// Vira query string
?includes=[{"relation":"status"},{"relation":"parent"}]

// Backend processa via Filter
public includes(value: string) {
  const includes = JSON.parse(value);
  for (const include of includes) {
    this.$query.preload(include.relation);
  }
}
```

### Filtros

```typescript
// Frontend
const filters = {
  search: 'termo',
  statusId: [1, 2],
  parentId: 5
};

// Vira query string
?search=termo&statusId[]=1&statusId[]=2&parentId=5

// Backend processa via Filter
public search(term: string) { ... }
public statusId(ids: number[]) { ... }
public parentId(id: number) { ... }
```
