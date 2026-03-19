# Exemplos Reais do Frontend REDD

Exemplos extraídos do código existente para referência.

## Provider Completo - Orders

```typescript
import {
  createContext,
  useContext,
  ReactNode,
  useState,
  useCallback,
  useEffect
} from 'react';
import { useQuery, UseQueryResult } from 'react-query';
import { GridSortModel } from '@mui/x-data-grid';
import { orderResource } from '@/services/api/resources/order';
import { OrderProps } from '@/services/types/entities/order/order';
import { Pagination } from '@/services/types/pagination';
import { useAuth } from 'auth/context/AuthContext';
import { RoleEnum } from 'enums/role';
import { isAny, isOnly } from 'utils/authorization';
import useDebounce from 'ui/hooks/queries/useDebounce';
import { ArrayParam, StringParam, useQueryParams, withDefault } from 'use-query-params';

const includes = [
  { relation: 'status' },
  { relation: 'seller' },
  { relation: 'company' },
  { relation: 'purchases' }
];

// Filtros baseados em role
const getFilters = (user: User) => {
  if (isAny(user, [RoleEnum.ADMIN, RoleEnum.SUPER_ADMIN])) {
    return {};
  }

  const filters: { sellerId?: number; companyId?: number[] } = {};

  if (isOnly(user, [RoleEnum.VENDAS])) {
    filters.sellerId = user.id;
  }

  if (user?.companies?.length) {
    filters.companyId = user.companies.map((c) => c.id);
  }

  return filters;
};

interface OrderContextData {
  query: UseQueryResult<Pagination<OrderProps>, unknown>;
  page: number;
  setPage: (page: number) => void;
  perPage: number;
  search: string;
  handleSearch: (search: string) => void;
  sortModel: GridSortModel;
  handleSortModelChange: (model: GridSortModel) => void;
}

const OrderContext = createContext({} as OrderContextData);

export function ProviderOrderDataGrid({ children }: { children: ReactNode }) {
  const { user } = useAuth();
  
  const [page, setPage] = useState(1);
  const [perPage, setPerPage] = useState(100);
  const [search, setSearch] = useState('');
  const [sortModel, setSortModel] = useState<GridSortModel>([
    { field: 'deliveryDate', sort: 'asc' }
  ]);

  // URL Query Params para persistir filtros
  const [queryParams, setQueryParams] = useQueryParams({
    companyId: withDefault(ArrayParam, []),
    sellerId: withDefault(ArrayParam, []),
    status: withDefault(ArrayParam, []),
    search: withDefault(StringParam, '')
  });

  const debouncedSearch = useDebounce(search, 1000);

  useEffect(() => {
    setSearch(queryParams.search);
  }, []);

  const handleSearch = (value: string) => {
    setSearch(value);
    setQueryParams((prev) => ({ ...prev, search: value }));
  };

  const query = useQuery<Pagination<OrderProps>>(
    ['orders', { page, perPage, sortModel, search: debouncedSearch, ...queryParams }],
    () => {
      const baseFilters = getFilters(user as User);
      
      return orderResource.paginate({
        page,
        perPage,
        includes,
        withCount: ['deliveries'],
        orderBy: sortModel.map((m) => `${m.field},${m.sort}`),
        filters: {
          search: debouncedSearch || undefined,
          companyId: baseFilters.companyId,
          sellerId: baseFilters.sellerId
        }
      });
    },
    {
      refetchInterval: 10000, // Auto refresh a cada 10s
      enabled: !!user
    }
  );

  const handleSortModelChange = (newModel: GridSortModel) => {
    setSortModel(newModel.map((m) => {
      if (m.field === 'status') return { ...m, field: 'statusId' };
      return m;
    }));
  };

  return (
    <OrderContext.Provider
      value={{
        query,
        page,
        setPage,
        perPage,
        search,
        handleSearch,
        sortModel,
        handleSortModelChange
      }}
    >
      {children}
    </OrderContext.Provider>
  );
}

export function useOrderDataGrid() {
  return useContext(OrderContext);
}
```

## API Resource - HttpClient Base

```typescript
// services/api/http-client.ts
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import { API_BASE_URL } from 'utils/constants';
import { setupApiInterceptor } from './config/setupApiInterceptor';
import { setupRefreshToken } from 'auth/services/token';
import { IncludeContract } from '../contracts/include-contract';
import { Entity } from './entities/entity';
import { Pagination } from './entities/pagination';

export abstract class HttpClient<T extends Entity> {
  protected client: AxiosInstance;

  constructor(
    protected readonly path: string,
    protected readonly basePath?: string
  ) {
    const client = axios.create({ baseURL: API_BASE_URL });
    setupApiInterceptor({ client });
    this.client = setupRefreshToken({ client });
  }

  async getAll(params?: {
    listType?: 'list';
    filters?: Record<string, any>;
    includes?: IncludeContract[];
    orderBy?: string[];
  }): Promise<T[]> {
    const response = await this.client.get(this.path, {
      params: {
        listType: params?.listType || 'list',
        ...params?.filters,
        includes: params?.includes ? JSON.stringify(params.includes) : undefined,
        orderBy: params?.orderBy
      }
    });
    return response.data;
  }

  async paginate({
    page,
    perPage = 25,
    filters,
    includes,
    withCount,
    orderBy
  }: {
    page: number;
    perPage?: number;
    filters?: Record<string, any>;
    includes?: IncludeContract[];
    withCount?: string[];
    orderBy?: string[];
  }): Promise<Pagination<T>> {
    const response = await this.client.get(this.path, {
      params: {
        page,
        perPage,
        orderBy,
        ...filters,
        includes: includes ? JSON.stringify(includes) : undefined,
        withCount
      }
    });
    return response.data;
  }

  async getById(id: number | string, params?: { includes?: IncludeContract[] }): Promise<T> {
    const response = await this.client.get(`${this.path}/${id}`, {
      params: {
        includes: params?.includes ? JSON.stringify(params.includes) : undefined
      }
    });
    return response.data;
  }

  async create(data: Record<string, any>, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.post(this.path, data, config);
    return response.data;
  }

  async update(id: number | string, data: Record<string, any>): Promise<T> {
    const response = await this.client.put(`${this.path}/${id}`, data);
    return response.data;
  }

  async delete(id: number | string): Promise<void> {
    await this.client.delete(`${this.path}/${id}`);
  }
}
```

## Form Completo com Zod

```typescript
// schema.ts
import { z } from 'zod';

export const orderSchema = z.object({
  customerName: z
    .string()
    .min(1, 'Nome do cliente é obrigatório')
    .max(255),
  sellerId: z
    .number({ required_error: 'Vendedor é obrigatório' }),
  companyId: z
    .number({ required_error: 'Empresa é obrigatória' }),
  deliveryDate: z
    .string()
    .min(1, 'Data de entrega é obrigatória'),
  purchases: z
    .array(z.object({
      supplierId: z.number({ required_error: 'Fornecedor é obrigatório' }),
      description: z.string().min(1, 'Descrição é obrigatória'),
      quantity: z.number().min(1, 'Quantidade mínima é 1'),
      color: z.string().optional()
    }))
    .min(1, 'Adicione pelo menos uma compra')
});

export type OrderFormData = z.infer<typeof orderSchema>;
```

```typescript
// FormOrder.tsx
import { useForm, Controller, useFieldArray } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { useMutation, useQuery, useQueryClient } from 'react-query';
import Box from '@mui/material/Box';
import TextField from '@mui/material/TextField';
import Button from '@mui/material/Button';
import IconButton from '@mui/material/IconButton';
import AddIcon from '@mui/icons-material/Add';
import DeleteIcon from '@mui/icons-material/Delete';
import { orderResource } from '@/services/api/resources/order';
import { useToast } from 'ui/hooks/toast';
import { orderSchema, OrderFormData } from './schema';
import { SellerSelect } from './components/SellerSelect';
import { CompanySelect } from './components/CompanySelect';
import { SupplierSelect } from './components/SupplierSelect';

export function FormOrder({ onSuccess }: { onSuccess?: () => void }) {
  const { addToast } = useToast();
  const queryClient = useQueryClient();

  const {
    control,
    handleSubmit,
    reset,
    formState: { errors }
  } = useForm<OrderFormData>({
    resolver: zodResolver(orderSchema),
    defaultValues: {
      customerName: '',
      sellerId: undefined,
      companyId: undefined,
      deliveryDate: '',
      purchases: [{ supplierId: undefined, description: '', quantity: 1, color: '' }]
    }
  });

  // useFieldArray para array dinâmico
  const { fields, append, remove } = useFieldArray({
    control,
    name: 'purchases'
  });

  const mutation = useMutation(
    (data: OrderFormData) => orderResource.create(data),
    {
      onSuccess: () => {
        queryClient.invalidateQueries(['orders']);
        addToast({ type: 'success', description: 'Pedido criado com sucesso' });
        reset();
        onSuccess?.();
      },
      onError: (error: any) => {
        const message = error?.response?.data?.error || 'Erro ao criar pedido';
        addToast({ type: 'error', description: message });
      }
    }
  );

  return (
    <form onSubmit={handleSubmit((data) => mutation.mutate(data))}>
      <Box display="flex" flexDirection="column" gap={2}>
        <Controller
          name="customerName"
          control={control}
          render={({ field }) => (
            <TextField
              {...field}
              label="Nome do Cliente"
              error={!!errors.customerName}
              helperText={errors.customerName?.message}
              fullWidth
            />
          )}
        />

        <Box display="flex" gap={2}>
          <Controller
            name="sellerId"
            control={control}
            render={({ field }) => (
              <SellerSelect
                value={field.value}
                onChange={field.onChange}
                error={errors.sellerId?.message}
              />
            )}
          />

          <Controller
            name="companyId"
            control={control}
            render={({ field }) => (
              <CompanySelect
                value={field.value}
                onChange={field.onChange}
                error={errors.companyId?.message}
              />
            )}
          />
        </Box>

        <Controller
          name="deliveryDate"
          control={control}
          render={({ field }) => (
            <TextField
              {...field}
              type="date"
              label="Data de Entrega"
              InputLabelProps={{ shrink: true }}
              error={!!errors.deliveryDate}
              helperText={errors.deliveryDate?.message}
              fullWidth
            />
          )}
        />

        {/* Array de Compras */}
        <Box>
          <Typography variant="subtitle1" gutterBottom>
            Compras
          </Typography>
          
          {fields.map((field, index) => (
            <Box key={field.id} display="flex" gap={2} mb={2}>
              <Controller
                name={`purchases.${index}.supplierId`}
                control={control}
                render={({ field }) => (
                  <SupplierSelect
                    value={field.value}
                    onChange={field.onChange}
                    error={errors.purchases?.[index]?.supplierId?.message}
                  />
                )}
              />

              <Controller
                name={`purchases.${index}.description`}
                control={control}
                render={({ field }) => (
                  <TextField
                    {...field}
                    label="Descrição"
                    error={!!errors.purchases?.[index]?.description}
                    fullWidth
                  />
                )}
              />

              <Controller
                name={`purchases.${index}.quantity`}
                control={control}
                render={({ field }) => (
                  <TextField
                    {...field}
                    type="number"
                    label="Qtd"
                    sx={{ width: 100 }}
                    onChange={(e) => field.onChange(Number(e.target.value))}
                  />
                )}
              />

              <IconButton
                color="error"
                onClick={() => remove(index)}
                disabled={fields.length === 1}
              >
                <DeleteIcon />
              </IconButton>
            </Box>
          ))}

          <Button
            variant="outlined"
            startIcon={<AddIcon />}
            onClick={() => append({ supplierId: undefined, description: '', quantity: 1, color: '' })}
          >
            Adicionar Compra
          </Button>
        </Box>

        <Button
          type="submit"
          variant="contained"
          disabled={mutation.isLoading}
          fullWidth
        >
          {mutation.isLoading ? 'Salvando...' : 'Criar Pedido'}
        </Button>
      </Box>
    </form>
  );
}
```

## Select Reutilizável com React Query

```typescript
// components/SellerSelect.tsx
import { useQuery } from 'react-query';
import Autocomplete from '@mui/material/Autocomplete';
import TextField from '@mui/material/TextField';
import { getUsers } from '@/services/api/user';
import { User } from '@/services/types/entities/user/user';
import { RoleEnum } from 'enums/role';

interface SellerSelectProps {
  value?: number;
  onChange: (value: number | undefined) => void;
  error?: string;
}

export function SellerSelect({ value, onChange, error }: SellerSelectProps) {
  const { data: sellers = [], isLoading } = useQuery<User[]>(
    ['sellers-list'],
    () => getUsers({
      params: {
        listType: 'list',
        is: RoleEnum.VENDAS,
        orderBy: ['name,asc']
      }
    }),
    { refetchOnWindowFocus: false }
  );

  const selectedSeller = sellers.find((s) => s.id === value) || null;

  return (
    <Autocomplete
      options={sellers}
      value={selectedSeller}
      onChange={(_, newValue) => onChange(newValue?.id)}
      getOptionLabel={(option) => option.fullName || `${option.firstName} ${option.lastName}`}
      loading={isLoading}
      renderInput={(params) => (
        <TextField
          {...params}
          label="Vendedor"
          error={!!error}
          helperText={error}
          fullWidth
        />
      )}
      fullWidth
    />
  );
}
```

## Page Component Completo

```typescript
// components/page/Orders/Orders.tsx
import { useEffect, useState } from 'react';
import Box from '@mui/material/Box';
import { usePanel } from '_layouts/BasicLayout/hooks/usePanel';
import { Page } from '@/components/shared/Page/Page';
import { ProviderOrderDataGrid } from './components/OrdersDataGrid/hooks/ProviderOrderDataGrid';
import { OrdersDataGrid } from './components/OrdersDataGrid/OrdersDataGrid';
import { ModalOrderFilters } from './components/OrderFilters/ModalOrderFilters';
import { ProviderNewOrder } from './components/FormNewOrder/hooks/ProviderNewOrder';
import { ModalFormNewOrder } from './components/FormNewOrder/ModalFormNewOrder';
import { ApprovalOrder } from './components/ApproveOrder/ApprovalOrder';

export function Orders() {
  const [openModalNew, setOpenModalNew] = useState(false);
  const { setHeaderRightContent } = usePanel();

  // Adiciona botão no header
  useEffect(() => {
    setHeaderRightContent(<ApprovalOrder />);
    return () => setHeaderRightContent(undefined);
  }, []);

  return (
    <Page>
      <ProviderOrderDataGrid>
        <ModalOrderFilters handleAddOrder={() => setOpenModalNew(true)} />
        <Box mx={-2} mt={-1}>
          <OrdersDataGrid />
        </Box>
      </ProviderOrderDataGrid>

      <ProviderNewOrder handleCloseModal={() => setOpenModalNew(false)}>
        {openModalNew && (
          <ModalFormNewOrder
            openModalNew={openModalNew}
            handleCloseModal={() => setOpenModalNew(false)}
          />
        )}
      </ProviderNewOrder>
    </Page>
  );
}
```
