# Frontend Frameworks Skills — Flutter, MERN, Vue, CSS, TypeScript

## Dart / Flutter

### Flutter App Structure
```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

void main() => runApp(
  MultiProvider(
    providers: [ChangeNotifierProvider(create: (_) => AppState())],
    child: const MyApp(),
  ),
);

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  @override
  Widget build(BuildContext context) => MaterialApp(
    title: 'My App',
    theme: ThemeData(colorSchemeSeed: Colors.blue, useMaterial3: true),
    darkTheme: ThemeData.dark(useMaterial3: true),
    home: const HomePage(),
  );
}
```

### Flutter Widgets
```dart
// Stateful widget with animation
class PulseButton extends StatefulWidget {
  final String label; final VoidCallback onTap;
  const PulseButton({super.key, required this.label, required this.onTap});
  @override State<PulseButton> createState() => _PulseButtonState();
}

class _PulseButtonState extends State<PulseButton>
    with SingleTickerProviderStateMixin {
  late AnimationController _ctrl;
  late Animation<double>   _scale;

  @override void initState() {
    super.initState();
    _ctrl  = AnimationController(vsync: this, duration: const Duration(milliseconds: 150));
    _scale = Tween(begin: 1.0, end: 0.95).animate(CurvedAnimation(parent: _ctrl, curve: Curves.easeInOut));
  }

  @override void dispose() { _ctrl.dispose(); super.dispose(); }

  @override Widget build(BuildContext context) => GestureDetector(
    onTapDown:   (_) => _ctrl.forward(),
    onTapUp:     (_) { _ctrl.reverse(); widget.onTap(); },
    onTapCancel: ()  => _ctrl.reverse(),
    child: ScaleTransition(scale: _scale,
      child: Container(
        padding: const EdgeInsets.symmetric(vertical: 16, horizontal: 32),
        decoration: BoxDecoration(
          gradient: const LinearGradient(colors: [Color(0xFF00D4FF), Color(0xFF0088FF)]),
          borderRadius: BorderRadius.circular(14),
          boxShadow: [BoxShadow(color: const Color(0xFF00D4FF).withOpacity(0.3), blurRadius: 20)],
        ),
        child: Text(widget.label, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.w800, color: Colors.white)),
      ),
    ),
  );
}
```

### Flutter HTTP & State
```dart
import 'package:dio/dio.dart';
import 'package:flutter/foundation.dart';

class ApiService {
  static final _dio = Dio(BaseOptions(
    baseUrl: 'https://api.example.com',
    connectTimeout: const Duration(seconds: 10),
    headers: {'Content-Type': 'application/json'},
  ));

  static Future<List<User>> getUsers() async {
    final r = await _dio.get('/users');
    return (r.data as List).map((j) => User.fromJson(j)).toList();
  }
}

class UserState extends ChangeNotifier {
  List<User> users = []; bool loading = false; String? error;

  Future<void> fetchUsers() async {
    loading = true; notifyListeners();
    try {
      users = await ApiService.getUsers();
      error = null;
    } catch (e) { error = e.toString(); }
    finally { loading = false; notifyListeners(); }
  }
}
```

---

## MERN Stack (MongoDB + Express + React + Node)

### Express + MongoDB API
```javascript
// server.js
import express from "express";
import mongoose from "mongoose";
import cors from "cors";

const app = express();
app.use(cors({ origin: process.env.CLIENT_URL }));
app.use(express.json());

await mongoose.connect(process.env.MONGODB_URI);

// Schema
const userSchema = new mongoose.Schema({
  name:      { type: String, required: true, trim: true },
  email:     { type: String, required: true, unique: true, lowercase: true },
  password:  { type: String, select: false },
  role:      { type: String, enum: ["user","admin"], default: "user" },
  createdAt: { type: Date,   default: Date.now },
});
const User = mongoose.model("User", userSchema);

// Routes
app.get("/api/users", async (req, res) => {
  const { page=1, limit=20, search="" } = req.query;
  const query = search ? { name: { $regex: search, $options: "i" } } : {};
  const [users, total] = await Promise.all([
    User.find(query).limit(+limit).skip((+page-1)*+limit).select("-password"),
    User.countDocuments(query),
  ]);
  res.json({ data: users, meta: { total, page: +page, pages: Math.ceil(total/+limit) } });
});

app.listen(5000, () => console.log("API on :5000"));
```

---

## Vue 3 + Vite

### Vue 3 Composition API
```vue
<!-- components/UserList.vue -->
<script setup lang="ts">
import { ref, computed, onMounted } from "vue";
import { useUsersStore } from "@/stores/users";

const store  = useUsersStore();
const search = ref("");
const filtered = computed(() =>
  store.users.filter(u => u.name.toLowerCase().includes(search.value.toLowerCase()))
);

onMounted(() => store.fetchUsers());
</script>

<template>
  <div class="container">
    <input v-model="search" placeholder="Search..." class="search" />
    <div v-if="store.loading" class="spinner" />
    <div v-else-if="store.error" class="error">{{ store.error }}</div>
    <ul v-else>
      <li v-for="user in filtered" :key="user.id" class="user-card">
        <span>{{ user.name }}</span>
        <span class="badge">{{ user.role }}</span>
      </li>
    </ul>
  </div>
</template>
```

### Pinia Store
```typescript
// stores/users.ts
import { defineStore } from "pinia";
import { ref, computed } from "vue";
import axios from "axios";

export const useUsersStore = defineStore("users", () => {
  const users   = ref<User[]>([]);
  const loading = ref(false);
  const error   = ref<string|null>(null);
  const total   = computed(() => users.value.length);

  async function fetchUsers() {
    loading.value = true; error.value = null;
    try {
      const { data } = await axios.get("/api/users");
      users.value = data.data;
    } catch(e: any) { error.value = e.message; }
    finally { loading.value = false; }
  }

  return { users, loading, error, total, fetchUsers };
});
```

---

## CSS Design Skills

### Design System Variables
```css
:root {
  /* Colors */
  --color-primary:   #00D4FF;
  --color-bg:        #0A0E1A;
  --color-card:      #141B2D;
  --color-text:      #E8F4FF;
  --color-muted:     #8899BB;

  /* Typography */
  --font-sans:  "Cairo", system-ui, sans-serif;
  --font-mono:  "JetBrains Mono", "Courier New", monospace;
  --text-xs:    0.75rem; --text-sm: 0.875rem;
  --text-base:  1rem;    --text-lg: 1.125rem;
  --text-xl:    1.25rem; --text-2xl: 1.5rem;

  /* Spacing */
  --space-1: 4px; --space-2: 8px;  --space-3: 12px;
  --space-4: 16px; --space-6: 24px; --space-8: 32px;

  /* Border radius */
  --radius-sm: 8px; --radius-md: 14px;
  --radius-lg: 20px; --radius-xl: 28px;

  /* Shadows */
  --shadow-sm:   0 2px 8px rgba(0,0,0,0.15);
  --shadow-md:   0 8px 32px rgba(0,0,0,0.4);
  --shadow-cyan: 0 0 30px rgba(0,212,255,0.2);

  /* Transitions */
  --transition: 0.3s cubic-bezier(0.4,0,0.2,1);
  --bounce:     0.4s cubic-bezier(0.34,1.56,0.64,1);
}

/* RTL utilities */
[dir="rtl"] .icon-before { margin-inline-start: 0; margin-inline-end: 8px; }
.text-start { text-align: start; }  /* respects dir */
.text-end   { text-align: end; }
```

### Glass Morphism Cards
```css
.glass-card {
  background:    rgba(20, 27, 45, 0.7);
  backdrop-filter: blur(20px) saturate(1.8);
  -webkit-backdrop-filter: blur(20px) saturate(1.8);
  border:        1px solid rgba(255,255,255,0.08);
  border-radius: var(--radius-lg);
  box-shadow:    var(--shadow-md), inset 0 1px 0 rgba(255,255,255,0.05);
}

/* Gradient text */
.gradient-text {
  background: linear-gradient(135deg, #fff, var(--color-primary));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

/* Neon glow button */
.btn-neon {
  background: transparent;
  border: 2px solid var(--color-primary);
  color: var(--color-primary);
  text-shadow: 0 0 8px currentColor;
  box-shadow: 0 0 15px rgba(0,212,255,0.3), inset 0 0 15px rgba(0,212,255,0.05);
  transition: var(--transition);
}
.btn-neon:hover {
  background: rgba(0,212,255,0.1);
  box-shadow: 0 0 30px rgba(0,212,255,0.6), inset 0 0 20px rgba(0,212,255,0.1);
}
```

---

## TypeScript Advanced Patterns
```typescript
// Generic API response type
type ApiResponse<T> =
  | { ok: true;  data: T;      meta?: Record<string,unknown> }
  | { ok: false; error: string; code: number };

// Utility types
type Nullable<T>  = T | null;
type Optional<T>  = T | undefined;
type DeepPartial<T> = { [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K] };
type ReadonlyDeep<T> = { readonly [K in keyof T]: T[K] extends object ? ReadonlyDeep<T[K]> : T[K] };

// Discriminated union with exhaustiveness
type State =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User[] }
  | { status: "error";   error: string };

function render(state: State): string {
  switch (state.status) {
    case "idle":    return "Ready";
    case "loading": return "Loading...";
    case "success": return `${state.data.length} users`;
    case "error":   return `Error: ${state.error}`;
    default: return state satisfies never;  // exhaustiveness check
  }
}

// Zod schema + type inference
import { z } from "zod";
const UserSchema = z.object({
  id:    z.number().int().positive(),
  name:  z.string().min(2).max(50),
  email: z.string().email(),
  role:  z.enum(["user","admin","moderator"]),
  tags:  z.array(z.string()).default([]),
});
type User = z.infer<typeof UserSchema>;  // auto-generate type from schema
```
