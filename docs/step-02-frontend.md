# Step 2 – Frontend (React + Vite)

Build a React application with two routes, server-state management with React Query, and a full-featured data table with TanStack Table.

> **Repository:** work inside your `my-app-frontend` repo for this step.

---

## 2.1 – Scaffold the project

```bash
npm create vite@latest . -- --template react-ts
npm install
npm run dev
```

Open [http://localhost:5173](http://localhost:5173) — you should see the default Vite + React page.

---

## 2.2 – Understand the project structure

```
my-app-frontend/
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

You need two pages:
- **Orders** (`/`) — displays a table of orders from the Express API
- **Events** (`/events`) — displays processed events from the NestJS API

> **🤖 Ask your AI assistant:**
> ```
> Create two empty page components: OrdersPage.tsx and EventsPage.tsx.
> Then update App.tsx to use BrowserRouter with two routes: / for OrdersPage
> and /events for EventsPage. Add a simple nav with links between them.
> ```

---

## 2.5 – Set up React Query

Wrap the app with `QueryClientProvider` so all child components can fetch data.

> **🤖 Ask your AI assistant:**
> ```
> Wrap the app in main.tsx with QueryClientProvider from @tanstack/react-query.
> ```

Then create two custom hooks — one per API:

> **🤖 Ask your AI assistant:**
> ```
> Create src/hooks/useOrders.ts — a useQuery hook that fetches from /api/orders
> and returns a typed Order array (id, customerName, product, quantity, status, createdAt).
> ```

> **🤖 Ask your AI assistant:**
> ```
> Create src/hooks/useEvents.ts — a useQuery hook that fetches from /api/events
> and returns a typed ProcessedEvent array (id, type, orderId, processedAt).
> ```

---

## 2.6 – Build the Orders table

> **🤖 Ask your AI assistant:**
> ```
> Update OrdersPage.tsx to use the useOrders hook and render the data
> in a TanStack Table with sortable columns: id, customerName, product,
> quantity, status, createdAt.
> ```

---

## 2.7 – Build the Events page

> **🤖 Ask your AI assistant:**
> ```
> Update EventsPage.tsx to use the useEvents hook and render a simple
> HTML table with columns: id, type, orderId, processedAt.
> ```

---

## 2.8 – Configure the Vite dev proxy

To avoid CORS issues during local development, proxy API calls through Vite.

> **🤖 Ask your AI assistant:**
> ```
> Update vite.config.ts to proxy /api/orders to http://localhost:3000
> and /api/events to http://localhost:3001.
> ```

---

## 2.9 – Build for production

```bash
npm run build
```

Output goes to `dist/`. Preview locally:

```bash
npm run preview
```

---

## 2.10 – Dockerise the frontend

> **🤖 Ask your AI assistant:**
> ```
> Create a multi-stage Dockerfile for the React + Vite app.
> Stage 1: build with node:22-alpine (npm ci, npm run build).
> Stage 2: serve the dist/ folder with nginx:stable-alpine on port 80.
> ```

Build and test locally:

```bash
docker build -t my-app-frontend .
docker run -p 8080:80 my-app-frontend
```

Open [http://localhost:8080](http://localhost:8080).

---

## 2.11 – Push to GitHub

```bash
git add .
git commit -m "feat: initial frontend scaffold"
git remote add origin https://github.com/<YOUR_ORG>/my-app-frontend.git
git push -u origin main
```

---

## Next step

➡️ [Step 3 – Express API](step-03-express-api.md)
