---
name: react-performance-patterns
description: Otimização de performance no React/Next.js 13. Use quando resolver problemas de re-renders desnecessários, otimizar useEffect, melhorar performance de listas, ou debugar problemas de renderização no redd-client.
---

# React Performance Patterns - Next.js 13

Padrões de otimização para evitar re-renders desnecessários e melhorar performance no projeto REDD.

## Quando Usar

- Componentes re-renderizando sem necessidade
- useEffect executando múltiplas vezes
- Listas grandes com performance ruim
- Formulários lentos
- Memory leaks
- Prop drilling causando re-renders

---

## 1. Evitar Re-renders Desnecessários

### Problema: Re-render em Cascata

```tsx
// ❌ RUIM: Todos os filhos re-renderizam quando parent muda
function Parent() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');

  return (
    <div>
      <Counter count={count} setCount={setCount} />
      <NameInput name={name} setName={setName} />
      <ExpensiveComponent /> {/* Re-renderiza mesmo sem props */}
    </div>
  );
}
```

### Solução: React.memo

```tsx
// ✅ BOM: Componente só re-renderiza se props mudarem
const ExpensiveComponent = React.memo(function ExpensiveComponent() {
  // Computação pesada aqui
  return <div>Expensive</div>;
});

// ✅ BOM: Com comparação customizada
const ListItem = React.memo(
  function ListItem({ item, onSelect }: Props) {
    return <div onClick={() => onSelect(item.id)}>{item.name}</div>;
  },
  (prevProps, nextProps) => {
    // Retorna true se NÃO deve re-renderizar
    return prevProps.item.id === nextProps.item.id;
  }
);
```

---

## 2. Callbacks Estáveis com useCallback

### Problema: Callback Recriado

```tsx
// ❌ RUIM: handleClick é recriado a cada render
function Parent() {
  const [items, setItems] = useState([]);

  const handleClick = (id: number) => {
    setItems(items.filter((item) => item.id !== id));
  };

  return <List items={items} onItemClick={handleClick} />;
}

// List (memo) re-renderiza porque handleClick é nova referência!
const List = React.memo(function List({ items, onItemClick }) {
  return items.map((item) => (
    <Item key={item.id} item={item} onClick={onItemClick} />
  ));
});
```

### Solução: useCallback

```tsx
// ✅ BOM: Callback estável
function Parent() {
  const [items, setItems] = useState([]);

  const handleClick = useCallback((id: number) => {
    setItems((prev) => prev.filter((item) => item.id !== id));
  }, []); // Array vazio = nunca recria

  return <List items={items} onItemClick={handleClick} />;
}

// ✅ BOM: Com dependências quando necessário
const handleSearch = useCallback((term: string) => {
  fetchResults(term, filters);
}, [filters]); // Recria apenas quando filters muda
```

---

## 3. Valores Memoizados com useMemo

### Problema: Cálculo Repetido

```tsx
// ❌ RUIM: Filtra a cada render
function OrderList({ orders, statusFilter }) {
  // Executa a cada render, mesmo se orders/statusFilter não mudou
  const filteredOrders = orders.filter((o) => o.status === statusFilter);

  return <DataGrid rows={filteredOrders} />;
}
```

### Solução: useMemo

```tsx
// ✅ BOM: Memoiza resultado
function OrderList({ orders, statusFilter }) {
  const filteredOrders = useMemo(
    () => orders.filter((o) => o.status === statusFilter),
    [orders, statusFilter] // Recalcula apenas quando mudam
  );

  return <DataGrid rows={filteredOrders} />;
}

// ✅ BOM: Para objetos de configuração
function DataGridWrapper() {
  const columns = useMemo(() => [
    { field: 'id', headerName: 'ID' },
    { field: 'name', headerName: 'Nome' },
  ], []); // Nunca recria

  return <DataGrid columns={columns} />;
}
```

---

## 4. useEffect Otimizado

### Problema: useEffect Executando Demais

```tsx
// ❌ RUIM: Executa em TODA renderização
useEffect(() => {
  fetchData();
}); // Sem array de dependências!

// ❌ RUIM: Dependência de objeto/array recriado
useEffect(() => {
  applyFilters(filters);
}, [{ status: 'active' }]); // Objeto novo a cada render!

// ❌ RUIM: Faz fetch dentro de useEffect sem cleanup
useEffect(() => {
  fetch(`/api/users/${userId}`).then(setUser);
}, [userId]); // Race condition se userId mudar rápido
```

### Solução: Dependências Corretas

```tsx
// ✅ BOM: Array de dependências correto
useEffect(() => {
  fetchData();
}, []); // Executa apenas no mount

// ✅ BOM: Dependências primitivas
const statusFilter = 'active';
useEffect(() => {
  applyFilters({ status: statusFilter });
}, [statusFilter]);

// ✅ BOM: Com cleanup e abort controller
useEffect(() => {
  const controller = new AbortController();
  
  fetch(`/api/users/${userId}`, { signal: controller.signal })
    .then((res) => res.json())
    .then(setUser)
    .catch((err) => {
      if (err.name !== 'AbortError') throw err;
    });

  return () => controller.abort();
}, [userId]);
```

### Quando NÃO Usar useEffect

```tsx
// ❌ RUIM: useEffect para derivar estado
const [items, setItems] = useState([]);
const [filteredItems, setFilteredItems] = useState([]);

useEffect(() => {
  setFilteredItems(items.filter((i) => i.active));
}, [items]); // Re-render extra!

// ✅ BOM: Derivar durante render
const [items, setItems] = useState([]);
const filteredItems = useMemo(
  () => items.filter((i) => i.active),
  [items]
);

// ❌ RUIM: useEffect para evento
useEffect(() => {
  if (submitted) {
    sendAnalytics('form_submitted');
  }
}, [submitted]);

// ✅ BOM: No handler do evento
const handleSubmit = () => {
  submitForm();
  sendAnalytics('form_submitted');
};
```

---

## 5. Estado Local vs Global

### Problema: Estado no Lugar Errado

```tsx
// ❌ RUIM: Estado de modal no contexto global
// Causa re-render de TODOS os consumers
const AppContext = createContext({
  modalOpen: false,
  setModalOpen: () => {},
  user: null,
  orders: []
});

function Header() {
  const { modalOpen } = useContext(AppContext); // Re-render quando orders muda!
}
```

### Solução: Separar Contextos

```tsx
// ✅ BOM: Contextos separados por responsabilidade
const ModalContext = createContext({ open: false, setOpen: () => {} });
const UserContext = createContext({ user: null });
const OrdersContext = createContext({ orders: [] });

function Header() {
  const { open } = useContext(ModalContext); // Só re-render quando modal muda
}

// ✅ BOM: Estado local quando possível
function OrderList() {
  // Filtro é estado LOCAL, não precisa estar no contexto
  const [filter, setFilter] = useState('');
  const { orders } = useContext(OrdersContext);
  
  const filtered = useMemo(
    () => orders.filter((o) => o.name.includes(filter)),
    [orders, filter]
  );
}
```

---

## 6. Listas Otimizadas

### Problema: Lista Grande Re-renderizando

```tsx
// ❌ RUIM: Todos os items re-renderizam
function List({ items, onSelect }) {
  return items.map((item) => (
    <div key={item.id} onClick={() => onSelect(item.id)}>
      {item.name}
    </div>
  ));
}
```

### Solução: Virtualização e Memo

```tsx
// ✅ BOM: Item memoizado
const ListItem = React.memo(function ListItem({ item, onSelect }: Props) {
  const handleClick = useCallback(() => {
    onSelect(item.id);
  }, [item.id, onSelect]);

  return <div onClick={handleClick}>{item.name}</div>;
});

function List({ items, onSelect }) {
  const handleSelect = useCallback((id: number) => {
    onSelect(id);
  }, [onSelect]);

  return items.map((item) => (
    <ListItem key={item.id} item={item} onSelect={handleSelect} />
  ));
}

// ✅ MELHOR: Virtualização para listas grandes
import { FixedSizeList } from 'react-window';

function VirtualList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>{items[index].name}</div>
  );

  return (
    <FixedSizeList
      height={400}
      itemCount={items.length}
      itemSize={35}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

---

## 7. Formulários Performáticos

### Problema: Re-render a Cada Tecla

```tsx
// ❌ RUIM: Form inteiro re-renderiza a cada tecla
function Form() {
  const [formData, setFormData] = useState({ name: '', email: '', phone: '' });

  return (
    <form>
      <input
        value={formData.name}
        onChange={(e) => setFormData({ ...formData, name: e.target.value })}
      />
      <input
        value={formData.email}
        onChange={(e) => setFormData({ ...formData, email: e.target.value })}
      />
      <ExpensiveComponent /> {/* Re-renderiza a cada tecla! */}
    </form>
  );
}
```

### Solução: React Hook Form

```tsx
// ✅ BOM: React Hook Form (não causa re-render)
import { useForm, Controller } from 'react-hook-form';

function Form() {
  const { control, handleSubmit } = useForm({
    defaultValues: { name: '', email: '', phone: '' }
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Controller
        name="name"
        control={control}
        render={({ field }) => <input {...field} />}
      />
      <Controller
        name="email"
        control={control}
        render={({ field }) => <input {...field} />}
      />
      <ExpensiveComponent /> {/* NÃO re-renderiza! */}
    </form>
  );
}

// ✅ BOM: Componentes isolados para inputs complexos
const NameField = React.memo(function NameField() {
  const { control } = useFormContext();
  
  return (
    <Controller
      name="name"
      control={control}
      render={({ field }) => <TextField {...field} />}
    />
  );
});
```

---

## 8. React Query Otimizado

### Problema: Queries Redundantes

```tsx
// ❌ RUIM: Fetch duplicado
function ComponentA() {
  const { data } = useQuery(['users'], fetchUsers);
}

function ComponentB() {
  const { data } = useQuery(['users'], fetchUsers); // Fetch duplicado?
}
```

### Solução: Configuração Correta

```tsx
// ✅ BOM: Stale time evita refetch desnecessário
const { data } = useQuery(['users'], fetchUsers, {
  staleTime: 5 * 60 * 1000, // 5 minutos
  cacheTime: 10 * 60 * 1000, // 10 minutos
  refetchOnWindowFocus: false,
  refetchOnMount: false
});

// ✅ BOM: Select para transformar dados
const { data: activeUsers } = useQuery(
  ['users'],
  fetchUsers,
  {
    select: (users) => users.filter((u) => u.active),
    // Componente só re-renderiza se activeUsers mudar
  }
);

// ✅ BOM: Placeholder data
const { data } = useQuery(
  ['user', userId],
  () => fetchUser(userId),
  {
    placeholderData: () => {
      // Usa dados do cache de lista enquanto carrega
      return queryClient
        .getQueryData(['users'])
        ?.find((u) => u.id === userId);
    }
  }
);
```

---

## 9. Context Performance

### Problema: Context Causando Re-renders

```tsx
// ❌ RUIM: Objeto novo a cada render
function Provider({ children }) {
  const [state, setState] = useState({});
  
  return (
    <Context.Provider value={{ state, setState }}>
      {children}
    </Context.Provider>
  );
}
```

### Solução: Memoizar Value

```tsx
// ✅ BOM: Value memoizado
function Provider({ children }) {
  const [state, setState] = useState({});
  
  const value = useMemo(
    () => ({ state, setState }),
    [state] // setState é estável
  );

  return (
    <Context.Provider value={value}>
      {children}
    </Context.Provider>
  );
}

// ✅ MELHOR: Separar state e dispatch
const StateContext = createContext(null);
const DispatchContext = createContext(null);

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <DispatchContext.Provider value={dispatch}>
      <StateContext.Provider value={state}>
        {children}
      </StateContext.Provider>
    </DispatchContext.Provider>
  );
}

// Componente que só dispara ações não re-renderiza quando state muda
function AddButton() {
  const dispatch = useContext(DispatchContext);
  return <button onClick={() => dispatch({ type: 'ADD' })}>Add</button>;
}
```

---

## 10. Debug de Performance

### React DevTools Profiler

```tsx
// Adicionar Profiler para medir
import { Profiler } from 'react';

function onRenderCallback(
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) {
  console.log({
    id,
    phase,
    actualDuration, // Tempo real de render
    baseDuration    // Tempo sem memoização
  });
}

<Profiler id="OrderList" onRender={onRenderCallback}>
  <OrderList />
</Profiler>
```

### why-did-you-render

```typescript
// Em desenvolvimento
import React from 'react';

if (process.env.NODE_ENV === 'development') {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true
  });
}

// Marcar componente específico
OrderList.whyDidYouRender = true;
```

---

## Checklist de Performance

```
[ ] Componentes pesados com React.memo
[ ] Callbacks estáveis com useCallback
[ ] Cálculos pesados com useMemo
[ ] useEffect com dependências corretas
[ ] Cleanup em useEffects com async
[ ] Estado no nível mais baixo possível
[ ] Contextos separados por responsabilidade
[ ] Listas grandes virtualizadas
[ ] Formulários com React Hook Form
[ ] React Query com staleTime configurado
[ ] Profiler para identificar gargalos
```

---

## Padrões Anti-Pattern

### ❌ Evitar

```tsx
// Objeto/Array inline em props
<Component style={{ margin: 10 }} />
<List items={items.filter(i => i.active)} />

// useEffect sem dependências
useEffect(() => { ... })

// useEffect para derivar estado
useEffect(() => { setFiltered(items.filter(...)) }, [items])

// Callback inline em loops
items.map(item => <Item onClick={() => handleClick(item.id)} />)

// Estado global para tudo
<AppContext.Provider value={{ ...tudo }} />
```

### ✅ Preferir

```tsx
// Memoizar estilo
const style = useMemo(() => ({ margin: 10 }), [])
<Component style={style} />

// useMemo para filtro
const filtered = useMemo(() => items.filter(i => i.active), [items])
<List items={filtered} />

// useEffect com deps
useEffect(() => { ... }, [dep1, dep2])

// Derivar durante render
const filtered = useMemo(() => items.filter(...), [items])

// useCallback para handlers
const handleClick = useCallback((id) => { ... }, [])
items.map(item => <Item onClick={handleClick} itemId={item.id} />)

// Contextos separados
<UserContext.Provider /><OrderContext.Provider />
```
