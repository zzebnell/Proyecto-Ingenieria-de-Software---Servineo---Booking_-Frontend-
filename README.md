# Guía de Configuración Frontend - Consumo de APIs

## 📋 Resumen Ejecutivo

Este documento explica cómo está configurado el frontend de Next.js para consumir las APIs del backend, incluyendo la configuración del cliente HTTP, variables de entorno y proxy.

## 🏗️ Arquitectura del Proyecto

```
servineo_frontend/          # Frontend (Next.js)
├── src/
│   ├── lib/
│   │   └── api.ts         # Cliente HTTP para consumir APIs
│   └── app/               # Páginas y componentes
├── next.config.ts         # Configuración de Next.js
└── .env.local            # Variables de entorno (crear manualmente)

servineo_backend/          # Backend (API separada)
└── APIs REST en puerto 8000
```

## 🔧 Configuración del Cliente API (`src/lib/api.ts`)

### ¿Por qué esta configuración?

**1. Separación de responsabilidades:**
- El frontend NO genera APIs, solo las consume
- El backend maneja toda la lógica de negocio y base de datos
- Comunicación vía HTTP entre ambos

**2. Clase ApiClient moderna:**
```typescript
class ApiClient {
  // Manejo centralizado de requests
  // Timeouts configurables
  // Interceptores de errores
  // Tipos TypeScript
}
```

**3. Métodos HTTP completos:**
- `GET` - Obtener datos
- `POST` - Crear recursos
- `PUT` - Actualizar recursos completos
- `PATCH` - Actualizar recursos parciales
- `DELETE` - Eliminar recursos
- `uploadFile` - Subir archivos

### Características principales:

**✅ Manejo de errores robusto:**
```typescript
interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
  message?: string;
}
```

**✅ Timeouts configurables:**
- Por defecto: 10 segundos
- Evita requests colgados

**✅ Headers automáticos:**
- `Content-Type: application/json`
- Headers personalizables

**✅ Upload de archivos:**
- FormData automático
- Datos adicionales opcionales

## 🌐 Configuración de Next.js (`next.config.ts`)

### ¿Por qué esta configuración?

**1. Proxy de API:**
```typescript
async rewrites() {
  return [
    {
      source: '/api/:path*',
      destination: `${process.env.NEXT_PUBLIC_API_URL}/api/:path*`,
    },
  ];
}
```

**¿Qué significa "Proxy hacia tu API backend"?**

- **Sin proxy:** Frontend hace requests directos a `http://localhost:8000/api/users`
- **Con proxy:** Frontend hace requests a `/api/users` y Next.js los redirige automáticamente

**Ventajas del proxy:**
- ✅ Evita problemas de CORS
- ✅ URLs más limpias en el código
- ✅ Configuración centralizada
- ✅ Fácil cambio de entorno (dev/prod)

**Ejemplo práctico:**
```typescript
// Sin proxy (problemático):
const response = await fetch('http://localhost:8000/api/users');

// Con proxy (recomendado):
const response = await api.get('/api/users');
```

**2. Configuración de imágenes:**
```typescript
images: {
  domains: ['localhost'],
  remotePatterns: [
    {
      protocol: 'http',
      hostname: 'localhost',
      port: '8000',
      pathname: '/**',
    },
  ],
}
```

**¿Por qué?** Para usar `next/image` con imágenes del backend.

**3. Headers de seguridad:**
```typescript
async headers() {
  return [
    {
      source: '/(.*)',
      headers: [
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'X-Content-Type-Options', value: 'nosniff' },
      ],
    },
  ];
}
```

## 🔐 Variables de Entorno

### Archivo `.env.local` (crear manualmente):

```env
# Puerto del frontend (Next.js)
PORT=3000

# URL del backend (API)
NEXT_PUBLIC_API_URL=http://localhost:8000

# Entorno
NODE_ENV=development
```

### ¿Por qué `NEXT_PUBLIC_`?

- Variables con este prefijo son accesibles en el navegador
- Necesarias para el cliente API
- Variables sin prefijo solo en servidor

## 🚀 Cómo usar en el código

### 1. Importar el cliente:
```typescript
import { api, ApiResponse } from '@/lib/api';
```

### 2. Definir tipos (recomendado):
```typescript
interface User {
  id: number;
  name: string;
  email: string;
}
```

### 3. Hacer requests:
```typescript
// GET - Obtener usuarios
const getUsers = async (): Promise<User[]> => {
  const response: ApiResponse<User[]> = await api.get('/api/users');
  
  if (response.success) {
    return response.data!;
  } else {
    throw new Error(response.error);
  }
};

// POST - Crear usuario
const createUser = async (userData: Omit<User, 'id'>): Promise<User> => {
  const response: ApiResponse<User> = await api.post('/api/users', userData);
  
  if (response.success) {
    return response.data!;
  } else {
    throw new Error(response.error);
  }
};

// Upload de archivo
const uploadAvatar = async (file: File, userId: number): Promise<string> => {
  const response: ApiResponse<{ url: string }> = await api.uploadFile(
    '/api/upload/avatar',
    file,
    { userId }
  );
  
  if (response.success) {
    return response.data!.url;
  } else {
    throw new Error(response.error);
  }
};
```

### 4. Usar en componentes:
```typescript
'use client';

import { useState, useEffect } from 'react';
import { api, ApiResponse } from '@/lib/api';

interface User {
  id: number;
  name: string;
  email: string;
}

export default function UsersList() {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchUsers = async () => {
      try {
        const response: ApiResponse<User[]> = await api.get('/api/users');
        
        if (response.success) {
          setUsers(response.data!);
        } else {
          setError(response.error || 'Error al cargar usuarios');
        }
      } catch (err) {
        setError('Error de conexión');
      } finally {
        setLoading(false);
      }
    };

    fetchUsers();
  }, []);

  if (loading) return <div>Cargando...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      <h1>Usuarios</h1>
      <ul>
        {users.map(user => (
          <li key={user.id}>
            {user.name} - {user.email}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## 🔄 Flujo de comunicación

```
1. Usuario interactúa con componente React
2. Componente llama a función del cliente API
3. Cliente API hace request HTTP al backend
4. Backend procesa y responde
5. Cliente API formatea respuesta
6. Componente actualiza UI
```

## 🛠️ Comandos de desarrollo

```bash
# Instalar dependencias
npm install

# Ejecutar frontend (puerto 3000)
npm run dev

# Ejecutar backend (puerto 8000) - en otro terminal
# cd ../servineo_backend
# npm run dev
```

## 🐛 Troubleshooting

### Error de CORS:
- ✅ Usar proxy en `next.config.ts`
- ✅ Verificar `NEXT_PUBLIC_API_URL`

### Error de conexión:
- ✅ Verificar que backend esté ejecutándose en puerto 8000
- ✅ Verificar variables de entorno

### Error de tipos TypeScript:
- ✅ Definir interfaces para respuestas de API
- ✅ Usar `ApiResponse<T>` para tipado

## 📚 Recursos adicionales

- [Next.js API Routes](https://nextjs.org/docs/api-routes/introduction)
- [Next.js Environment Variables](https://nextjs.org/docs/basic-features/environment-variables)
- [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [TypeScript Interfaces](https://www.typescriptlang.org/docs/handbook/interfaces.html)
