# React + Node.js Skills — Read/Write Guide

## React — Reading Code
Trace: root `<App>` → Router → Page components → shared hooks → context providers. State lives in the component closest to all consumers.

## React Modern Patterns
```tsx
// Custom hook — co-locate logic
function useUsers() {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError]   = useState<string|null>(null);

  useEffect(() => {
    let cancelled = false;
    fetch("/api/users")
      .then(r => r.json())
      .then(d => { if (!cancelled) setUsers(d); })
      .catch(e => { if (!cancelled) setError(e.message); })
      .finally(() => { if (!cancelled) setLoading(false); });
    return () => { cancelled = true; };
  }, []);

  return { users, loading, error };
}

// Component
export function UserList() {
  const { users, loading, error } = useUsers();
  if (loading) return <Spinner />;
  if (error)   return <Alert msg={error} />;
  return (
    <ul>
      {users.map(u => <UserCard key={u.id} user={u} />)}
    </ul>
  );
}
```

## React Context (global state)
```tsx
interface AuthCtx { user: User|null; login(u:User):void; logout():void; }
const AuthContext = createContext<AuthCtx|null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User|null>(null);
  return (
    <AuthContext.Provider value={{
      user,
      login:  u  => setUser(u),
      logout: () => setUser(null),
    }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be inside AuthProvider");
  return ctx;
}
```

## React Router v6
```tsx
import { createBrowserRouter, RouterProvider, Outlet, NavLink } from "react-router-dom";

const router = createBrowserRouter([
  { path: "/", element: <Layout />, children: [
    { index: true, element: <Home /> },
    { path: "users", element: <Users /> },
    { path: "users/:id", element: <UserDetail />,
      loader: ({ params }) => fetchUser(params.id!) },
    { path: "*", element: <NotFound /> },
  ]},
]);

function Layout() {
  return (
    <div>
      <nav>
        <NavLink to="/" className={({isActive})=>isActive?"active":""}>Home</NavLink>
        <NavLink to="/users">Users</NavLink>
      </nav>
      <Outlet />
    </div>
  );
}
```

## Node.js + Express API
```typescript
import express, { Request, Response, NextFunction } from "express";
import { z } from "zod";

const app = express();
app.use(express.json());

// Validation schema
const CreateUserSchema = z.object({
  name:  z.string().min(2).max(50),
  email: z.string().email(),
  age:   z.number().int().min(0).max(120).optional(),
});

// Async wrapper
const asyncHandler = (fn: Function) =>
  (req: Request, res: Response, next: NextFunction) =>
    Promise.resolve(fn(req, res, next)).catch(next);

// Routes
app.get("/api/users", asyncHandler(async (req: Request, res: Response) => {
  const users = await UserService.findAll();
  res.json({ data: users, total: users.length });
}));

app.post("/api/users", asyncHandler(async (req: Request, res: Response) => {
  const body = CreateUserSchema.parse(req.body);  // throws ZodError if invalid
  const user = await UserService.create(body);
  res.status(201).json(user);
}));

// Global error handler
app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => {
  if (err instanceof z.ZodError)
    return res.status(400).json({ error: "Validation", issues: err.issues });
  res.status(500).json({ error: err.message });
});

app.listen(3000, () => console.log("API running on :3000"));
```

## Node.js File I/O
```typescript
import fs from "node:fs/promises";
import path from "node:path";

// Read/write text
const content = await fs.readFile("data.txt", "utf8");
await fs.writeFile("out.txt", "Hello", "utf8");

// JSON
const data = JSON.parse(await fs.readFile("data.json", "utf8"));
await fs.writeFile("out.json", JSON.stringify(data, null, 2));

// Walk directory recursively
async function* walkDir(dir: string): AsyncGenerator<string> {
  for (const entry of await fs.readdir(dir, { withFileTypes: true })) {
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) yield* walkDir(full);
    else yield full;
  }
}

for await (const file of walkDir("./src")) {
  if (file.endsWith(".ts")) console.log(file);
}
```

## React + Vite Setup
```bash
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install
npm install react-router-dom zustand react-query zod axios
npm run dev
```

## State Management (Zustand)
```typescript
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface CartStore {
  items: CartItem[];
  add:    (item: CartItem) => void;
  remove: (id: string)     => void;
  clear:  ()               => void;
  total:  ()               => number;
}

export const useCart = create<CartStore>()(
  persist(
    (set, get) => ({
      items:  [],
      add:    item  => set(s => ({ items: [...s.items, item] })),
      remove: id    => set(s => ({ items: s.items.filter(i => i.id !== id) })),
      clear:  ()    => set({ items: [] }),
      total:  ()    => get().items.reduce((s, i) => s + i.price * i.qty, 0),
    }),
    { name: "cart-store" }
  )
);
```

## Testing React (Vitest + Testing Library)
```tsx
import { render, screen, fireEvent } from "@testing-library/react";
import { describe, it, expect, vi } from "vitest";
import { Counter } from "./Counter";

describe("Counter", () => {
  it("increments on click", () => {
    render(<Counter initial={0} />);
    const btn = screen.getByRole("button", { name: /increment/i });
    fireEvent.click(btn);
    expect(screen.getByText("1")).toBeInTheDocument();
  });

  it("calls onMax when limit reached", () => {
    const onMax = vi.fn();
    render(<Counter initial={9} max={10} onMax={onMax} />);
    fireEvent.click(screen.getByRole("button", { name: /increment/i }));
    expect(onMax).toHaveBeenCalledOnce();
  });
});
```
