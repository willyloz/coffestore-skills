---
name: nuxt-directus-stack
description: >
  Expertise en el stack JavaScript moderno: Nuxt 4, Vue 3 Composition API,
  Directus como plataforma de datos, Node.js/Fastify para APIs, PostgreSQL 16,
  Docker y Nginx. Usar cuando se desarrollen componentes Vue, páginas Nuxt,
  queries Directus, endpoints Node, configuraciones Docker o Nginx.
  Trigger: Nuxt, Vue, composable, Pinia, SSR, Directus, colección, endpoint,
  Node, Fastify, Docker, Nginx, PostgreSQL, TypeScript, Vite, deploy.
---

# Stack JavaScript — Nuxt 4 + Directus + Node.js

## Contexto del stack

Stack de desarrollo JavaScript moderno para aplicaciones web full-stack:
- **Frontend**: Nuxt 4 + Vue 3 + TypeScript + Pinia + Nuxt UI v4
- **Plataforma de datos**: Directus v10/v11 + PostgreSQL 16
- **Backend**: Node.js con Fastify o server routes de Nuxt
- **Infraestructura**: Ubuntu Server + Docker + Nginx + Let's Encrypt

---

## Nuxt 4 — Patrones y convenciones

### Estructura de proyecto
```
app/
  components/      ← componentes auto-importados
  composables/     ← lógica reutilizable (useXxx)
  layouts/         ← plantillas de página
  middleware/      ← guards de navegación
  pages/           ← file-based routing
  plugins/         ← plugins del cliente/servidor
server/
  api/             ← server routes (endpoint.method.ts)
  middleware/      ← middleware del servidor
```

### Composables — patrón estándar
```typescript
// Siempre usar useState para estado compartido entre SSR y cliente
export const useAuth = () => {
  const user = useState<User | null>('auth:user', () => null)
  // ...
  return { user }
}
```

### Server routes — naming convention
```
server/api/
  users.get.ts       → GET  /api/users
  users.post.ts      → POST /api/users
  users/[id].get.ts  → GET  /api/users/:id
  users/[id].put.ts  → PUT  /api/users/:id
```

### Cookies en server routes
```typescript
// Leer cookie
const token = getCookie(event, 'auth:token')
// Escribir cookie
setCookie(event, 'auth:token', value, { httpOnly: true, secure: true })
```

### $fetch vs useFetch
- `useFetch` → en `<script setup>` de páginas (SSR-friendly, con caché)
- `$fetch` → en funciones, actions, event handlers (no SSR)
- `useAsyncData` → cuando necesitas control total sobre el fetching

---

## Vue 3 — Composition API

### Script setup estándar
```vue
<script setup lang="ts">
// Props con tipos
const props = defineProps<{ title: string; count?: number }>()
// Emits tipados
const emit = defineEmits<{ update: [value: string] }>()
// Computed
const doubled = computed(() => (props.count ?? 0) * 2)
</script>
```

### Patrones de reactividad
```typescript
// ref para primitivos, reactive para objetos
const count = ref(0)
const form  = reactive({ email: '', password: '' })
// toRefs para desestructurar reactive sin perder reactividad
const { email, password } = toRefs(form)
// watchEffect vs watch
watchEffect(() => console.log(count.value))     // auto-tracking
watch(count, (val, old) => console.log(val))    // explícito
```

---

## Directus — Queries y patrones

### SDK en Nuxt
```typescript
const config = useRuntimeConfig()
// $fetch directo (más simple para server routes)
const items = await $fetch(`${config.public.directusUrl}/items/orders`, {
  headers: { Authorization: `Bearer ${token}` },
  params: {
    'fields[]': ['id', 'status', 'total', 'customer_id.email'],
    'filter[status][_eq]': 'pending',
    'sort': '-date_created',
    'limit': 20,
  }
})
```

### Filtros comunes
```
Igual:          filter[campo][_eq]=valor
Diferente:      filter[campo][_neq]=valor
Mayor que:      filter[campo][_gt]=valor
En lista:       filter[campo][_in]=a,b,c
Contiene:       filter[campo][_contains]=texto
Es nulo:        filter[campo][_null]=true
Y lógico:       filter[_and][0][campo1][_eq]=x&filter[_and][1][campo2][_eq]=y
O lógico:       filter[_or][0][campo1][_eq]=x&filter[_or][1][campo2][_eq]=y
```

### Relaciones en fields
```
M2O:  fields[]=autor_id.nombre,autor_id.email
O2M:  fields[]=comentarios.id,comentarios.texto
M2M:  fields[]=tags.tag_id.nombre
```

### Aggregations
```
count:  aggregate[count]=id
sum:    aggregate[sum]=total
avg:    aggregate[avg]=precio
groupBy: groupBy[]=status
```

---

## PostgreSQL 16 — Queries avanzadas

### Window functions
```sql
SELECT
  nombre,
  total,
  RANK() OVER (ORDER BY total DESC) as ranking,
  SUM(total) OVER () as total_general
FROM orders;
```

### JSONB (para campos json de Directus)
```sql
-- Buscar en array JSONB
SELECT * FROM products WHERE tasting_notes @> '["chocolate"]'::jsonb;
-- Extraer valor
SELECT payload->>'result' FROM messages WHERE message_type = 'leader_response';
```

---

## Docker — Configuración para producción

### Dockerfile Nuxt (multi-stage)
```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.output ./
EXPOSE 3000
CMD ["node", "server/index.mjs"]
```

### docker-compose.yml básico
```yaml
services:
  nuxt:
    build: .
    restart: unless-stopped
    environment:
      - NUXT_PUBLIC_DIRECTUS_URL=https://tu-directus.com
    ports:
      - "3000:3000"
```

---

## Nginx — Reverse proxy para Nuxt

```nginx
server {
    listen 443 ssl;
    server_name tu-dominio.com;

    ssl_certificate     /etc/letsencrypt/live/tu-dominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tu-dominio.com/privkey.pem;

    location / {
        proxy_pass         http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection 'upgrade';
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## Errores comunes y soluciones

### Hidratación SSR
```
Error: Hydration mismatch
Causa: Valor diferente entre servidor y cliente (fechas, random, window)
Solución: Usar <ClientOnly> o inicializar en onMounted()
```

### Cookies con secure:true en HTTP local
```
Problema: Cookie no se guarda en desarrollo HTTP
Solución: secure: false en desarrollo, secure: true solo en producción
const token = useCookie('token', { secure: process.env.NODE_ENV === 'production' })
```

### Directus 403 en campos de relación
```
Problema: fields[]=relacion.campo devuelve 403
Causa: El rol no tiene acceso a la colección relacionada
Solución: Agregar permiso READ en la colección relacionada para ese rol
```
