# Frontend – React + Vite

A step-by-step guide to bootstrapping a React application with Vite, adding routing, state management, and connecting it to a backend API.

---

## Step 1 – Scaffold the project

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

Open [http://localhost:5173](http://localhost:5173) in your browser. You should see the default Vite + React page.

**What happened?**
- `npm create vite` used the official Vite scaffolding tool.
- `--template react-ts` selected the React + TypeScript template.
- `npm run dev` started the Vite development server with hot-module replacement (HMR).

---

## Step 2 – Understand the project structure

```
my-app/
├── index.html          # entry HTML; Vite injects the script here
├── vite.config.ts      # Vite configuration
├── tsconfig.json
├── src/
│   ├── main.tsx        # React root – mounts <App /> into #root
│   ├── App.tsx         # top-level component
│   └── assets/
└── public/             # static files served as-is
```

---

## Step 3 – Add React Router

```bash
npm install react-router-dom
```

Replace `src/App.tsx` with:

```tsx
import { BrowserRouter, Route, Routes } from 'react-router-dom';
import Home from './pages/Home';
import About from './pages/About';

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </BrowserRouter>
  );
}
```

Create `src/pages/Home.tsx`:

```tsx
export default function Home() {
  return <h1>Home</h1>;
}
```

Create `src/pages/About.tsx`:

```tsx
export default function About() {
  return <h1>About</h1>;
}
```

Navigate to `/` and `/about` to verify routing works.

---

## Step 4 – Manage global state with Zustand

```bash
npm install zustand
```

Create `src/store/counter.ts`:

```ts
import { create } from 'zustand';

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
}

export const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
  decrement: () => set((s) => ({ count: s.count - 1 })),
}));
```

Use the store in a component:

```tsx
import { useCounterStore } from '../store/counter';

export default function Counter() {
  const { count, increment, decrement } = useCounterStore();
  return (
    <div>
      <button onClick={decrement}>-</button>
      <span>{count}</span>
      <button onClick={increment}>+</button>
    </div>
  );
}
```

---

## Step 5 – Fetch data from a backend API

Create `src/hooks/usePosts.ts`:

```ts
import { useEffect, useState } from 'react';

export interface Post {
  id: number;
  title: string;
}

export function usePosts() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetch('http://localhost:3000/posts')
      .then((res) => {
        if (!res.ok) throw new Error('Network response was not ok');
        return res.json() as Promise<Post[]>;
      })
      .then(setPosts)
      .catch((err: Error) => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  return { posts, loading, error };
}
```

During development, avoid CORS issues by proxying API calls through Vite. Add to `vite.config.ts`:

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': 'http://localhost:3000',
    },
  },
});
```

Then prefix all fetch calls with `/api` instead of the full host.

---

## Step 6 – Build for production

```bash
npm run build
```

The output lands in `dist/`. You can preview it locally:

```bash
npm run preview
```

---

## Step 7 – Dockerise the frontend

Create a `Dockerfile` inside `my-app/`:

```dockerfile
# Build stage
FROM node:18-alpine AS build
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

Build and run:

```bash
docker build -t my-app .
docker run -p 8080:80 my-app
```

Open [http://localhost:8080](http://localhost:8080).

---

## Next steps

- Connect to the Express backend → [docs/backend/express](../backend/express/README.md)
- Deploy with infrastructure-as-code → [docs/infra](../infra/README.md)
