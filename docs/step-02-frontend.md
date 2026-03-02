# Step 2 – Frontend (React + Vite)

Build a React application with two routes, server-state management with React Query, and a full-featured data table with TanStack Table.

---

## 2.1 – Scaffold the project

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
```

Start the dev server:

```bash
npm run dev
```

Open [http://localhost:5173](http://localhost:5173) — you should see the default Vite + React page.

---

## 2.2 – Understand the project structure

```
my-app/
├── index.html          # entry HTML – Vite injects the script here
├── vite.config.ts      # Vite configuration
├── tsconfig.json
├── src/
│   ├── main.tsx        # React root – mounts <App /> into #root
│   ├── App.tsx         # top-level component
│   └── assets/
└── public/             # static files served as-is
```

---

## 2.3 – Install dependencies

```bash
npm install react-router-dom @tanstack/react-query @tanstack/react-table
```

| Library | Purpose |
|---------|---------|
| `react-router-dom` | Client-side routing |
| `@tanstack/react-query` | Server-state management (fetching, caching, syncing) |
| `@tanstack/react-table` | Headless, sortable, filterable data table |

---

## 2.4 – Set up React Router

Create two pages:

- **Orders** (`/`) — displays a table of orders fetched from the Express API
- **Events** (`/events`) — displays processed events fetched from the NestJS API

Create `src/pages/OrdersPage.tsx`:

```tsx
export default function OrdersPage() {
  return <h1>Orders</h1>;
}
```

Create `src/pages/EventsPage.tsx`:

```tsx
export default function EventsPage() {
  return <h1>Events</h1>;
}
```

Replace `src/App.tsx`:

```tsx
import { BrowserRouter, Link, Route, Routes } from 'react-router-dom';
import OrdersPage from './pages/OrdersPage';
import EventsPage from './pages/EventsPage';

export default function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Orders</Link> | <Link to="/events">Events</Link>
      </nav>
      <Routes>
        <Route path="/" element={<OrdersPage />} />
        <Route path="/events" element={<EventsPage />} />
      </Routes>
    </BrowserRouter>
  );
}
```

---

## 2.5 – Set up React Query

Wrap the app with `QueryClientProvider` in `src/main.tsx`:

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import App from './App';

const queryClient = new QueryClient();

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </React.StrictMode>,
);
```

Create `src/hooks/useOrders.ts`:

```ts
import { useQuery } from '@tanstack/react-query';

export interface Order {
  id: number;
  customerName: string;
  product: string;
  quantity: number;
  status: string;
  createdAt: string;
}

async function fetchOrders(): Promise<Order[]> {
  const res = await fetch('/api/orders');
  if (!res.ok) throw new Error('Failed to fetch orders');
  return res.json();
}

export function useOrders() {
  return useQuery({ queryKey: ['orders'], queryFn: fetchOrders });
}
```

Create `src/hooks/useEvents.ts`:

```ts
import { useQuery } from '@tanstack/react-query';

export interface ProcessedEvent {
  id: string;
  type: string;
  orderId: number;
  processedAt: string;
}

async function fetchEvents(): Promise<ProcessedEvent[]> {
  const res = await fetch('/api/events');
  if (!res.ok) throw new Error('Failed to fetch events');
  return res.json();
}

export function useEvents() {
  return useQuery({ queryKey: ['events'], queryFn: fetchEvents });
}
```

---

## 2.6 – Build the Orders table with TanStack Table

Update `src/pages/OrdersPage.tsx`:

```tsx
import {
  createColumnHelper,
  flexRender,
  getCoreRowModel,
  getSortedRowModel,
  SortingState,
  useReactTable,
} from '@tanstack/react-table';
import { useState } from 'react';
import { Order, useOrders } from '../hooks/useOrders';

const columnHelper = createColumnHelper<Order>();

const columns = [
  columnHelper.accessor('id', { header: 'ID' }),
  columnHelper.accessor('customerName', { header: 'Customer' }),
  columnHelper.accessor('product', { header: 'Product' }),
  columnHelper.accessor('quantity', { header: 'Qty' }),
  columnHelper.accessor('status', { header: 'Status' }),
  columnHelper.accessor('createdAt', {
    header: 'Created',
    cell: (info) => new Date(info.getValue()).toLocaleString(),
  }),
];

export default function OrdersPage() {
  const { data: orders = [], isLoading, error } = useOrders();
  const [sorting, setSorting] = useState<SortingState>([]);

  const table = useReactTable({
    data: orders,
    columns,
    state: { sorting },
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });

  if (isLoading) return <p>Loading orders…</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <>
      <h1>Orders</h1>
      <table>
        <thead>
          {table.getHeaderGroups().map((hg) => (
            <tr key={hg.id}>
              {hg.headers.map((header) => (
                <th
                  key={header.id}
                  onClick={header.column.getToggleSortingHandler()}
                  style={{ cursor: 'pointer' }}
                >
                  {flexRender(header.column.columnDef.header, header.getContext())}
                  {{ asc: ' ↑', desc: ' ↓' }[header.column.getIsSorted() as string] ?? ''}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {table.getRowModel().rows.map((row) => (
            <tr key={row.id}>
              {row.getVisibleCells().map((cell) => (
                <td key={cell.id}>
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </>
  );
}
```

---

## 2.7 – Build the Events page

Update `src/pages/EventsPage.tsx`:

```tsx
import { useEvents } from '../hooks/useEvents';

export default function EventsPage() {
  const { data: events = [], isLoading, error } = useEvents();

  if (isLoading) return <p>Loading events…</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <>
      <h1>Processed Events</h1>
      <table>
        <thead>
          <tr>
            <th>ID</th>
            <th>Type</th>
            <th>Order ID</th>
            <th>Processed At</th>
          </tr>
        </thead>
        <tbody>
          {events.map((evt) => (
            <tr key={evt.id}>
              <td>{evt.id}</td>
              <td>{evt.type}</td>
              <td>{evt.orderId}</td>
              <td>{new Date(evt.processedAt).toLocaleString()}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </>
  );
}
```

---

## 2.8 – Configure the Vite dev proxy

To avoid CORS issues during development, proxy API calls through Vite. Update `vite.config.ts`:

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api/orders': 'http://localhost:3000',
      '/api/events': 'http://localhost:3001',
    },
  },
});
```

---

## 2.9 – Build for production

```bash
npm run build
```

Output goes to `dist/`. Preview it locally:

```bash
npm run preview
```

---

## 2.10 – Dockerise the frontend

Create `my-app/Dockerfile`:

```dockerfile
# Build stage
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Serve stage
FROM nginx:stable-alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Build and test locally:

```bash
docker build -t my-app .
docker run -p 8080:80 my-app
```

Open [http://localhost:8080](http://localhost:8080).

---

## Next step

➡️ [Step 3 – Express API](step-03-express-api.md)
