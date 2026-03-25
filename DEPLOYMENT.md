# Despliegue de Nutribiotics

## Stack Tecnologico

Nutribiotics es una aplicacion monorepo compuesta por dos proyectos independientes:

### Frontend

- **React 18** con **TypeScript**
- **Vite** como bundler
- **Tailwind CSS** para estilos
- **Zustand** para manejo de estado
- **TanStack Query** para fetching de datos
- **React Router** para navegacion

Al compilar (`npm run build`), Vite genera una carpeta `dist/` con archivos estaticos (HTML, CSS, JS). Estos archivos no necesitan un servidor Node.js para correr — se pueden servir desde cualquier hosting de archivos estaticos.

### Backend

- **NestJS 11** (framework Node.js) con **TypeScript**
- **Mongoose** como ODM para MongoDB
- **Passport.js** con estrategia JWT para autenticacion
- **BullMQ** para colas de trabajo en segundo plano (requiere Redis)
- **Integraciones de IA**: OpenAI, Google Generative AI, Perplexity (opcional)
- **Stagehand** para web scraping con automatizacion de navegador

El backend es un servidor Node.js que debe estar corriendo permanentemente. Expone una API REST y un panel de monitoreo de colas en `/queues`.

### Servicios Externos Requeridos

| Servicio | Proposito | Obligatorio |
| --- | --- | --- |
| **MongoDB** | Base de datos principal | Si |
| **Redis** | Cola de trabajos (BullMQ) | Si |
| **OpenAI API** | Scraping inteligente y recomendaciones | Si |
| **Google AI API** | Modelos Gemini para procesamiento | Si |
| **Perplexity API** | Busquedas enriquecidas | No |
| **Browserbase** | Navegador en la nube para scraping | No |

---

## Infraestructura del Frontend

El frontend compilado es un conjunto de archivos estaticos. Solo necesitas un servicio que sirva estos archivos y soporte SPA (redirigir todas las rutas a `index.html`).

### Opciones de Proveedores

| Proveedor | Tier Gratuito | Ideal para |
| --- | --- | --- |
| **Vercel** | Si | Despliegue rapido, preview por PR |
| **Netlify** | Si | Similar a Vercel, buena integracion con Git |
| **Cloudflare Pages** | Si | CDN global, muy rapido |
| **AWS S3 + CloudFront** | Pago | Control total, escalabilidad empresarial |
| **Firebase Hosting** | Si (limitado) | Si ya usas el ecosistema Google |

### Variables de Entorno del Frontend

```env
VITE_API_BASE_URL=https://api.tu-dominio.com
VITE_WITH_BACKEND=true
```

> `VITE_API_BASE_URL` debe apuntar a la URL publica donde esta corriendo el backend.

### Ejemplo: Despliegue en Vercel

1. Conecta tu repositorio de GitHub en [vercel.com](https://vercel.com)
2. Configura el proyecto:
   - **Root Directory**: `frontend`
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`
3. Agrega las variables de entorno en el panel de Vercel
4. Cada push a `main` desplegara automaticamente

### Ejemplo: Despliegue en Netlify

1. Conecta tu repositorio en [netlify.com](https://netlify.com)
2. Configura:
   - **Base Directory**: `frontend`
   - **Build Command**: `npm run build`
   - **Publish Directory**: `frontend/dist`
3. Agrega un archivo `frontend/netlify.toml`:
   ```toml
   [[redirects]]
     from = "/*"
     to = "/index.html"
     status = 200
   ```
4. Agrega las variables de entorno en el panel

---

## Infraestructura del Backend

El backend es un servidor Node.js que necesita estar corriendo 24/7. Requiere conectividad a MongoDB y Redis.

### Opciones de Proveedores

| Proveedor | Tier Gratuito | Ideal para |
| --- | --- | --- |
| **Railway** | Si (limitado) | Facil de configurar, soporte para MongoDB y Redis integrado |
| **Render** | Si (limitado) | Buena opcion para APIs Node.js |
| **Fly.io** | Si (limitado) | Despliegue global, buen rendimiento |
| **DigitalOcean App Platform** | Pago (~$5/mes) | Simple, predecible |
| **AWS EC2 / Lightsail** | Pago | Control total del servidor |
| **VPS generico (Hetzner, Contabo)** | Pago (~$4/mes) | Maximo control, menor costo |

### Variables de Entorno del Backend

```env
# Base de datos
MONGODB_URI=mongodb+srv://usuario:password@cluster.mongodb.net/nutribiotics

# Autenticacion
JWT_SECRET=secreto-seguro-generado-aleatoriamente
JWT_REFRESH_SECRET=otro-secreto-seguro-diferente
JWT_EXPIRATION=15m
JWT_REFRESH_EXPIRATION=7d

# Usuario semilla (se crea al primer inicio)
SEED_USER_EMAIL=admin@tu-dominio.com
SEED_USER_PASSWORD=contraseña-segura

# Redis
REDIS_URL=redis://default:password@host:6379

# Claves de IA
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=AIza...

# Opcionales
PERPLEXITY_API_KEY=pplx-...
BROWSERBASE_API_KEY=...
BROWSERBASE_PROJECT_ID=...

# Puerto
PORT=3000
```

### Ejemplo: Despliegue en Railway

1. Crea un proyecto en [railway.app](https://railway.app)
2. Agrega los servicios:
   - **Backend**: Conecta el repositorio y configura el root directory como `backend`
   - **MongoDB**: Agrega un plugin de MongoDB desde el marketplace
   - **Redis**: Agrega un plugin de Redis desde el marketplace
3. Railway genera automaticamente las URLs de conexion para MongoDB y Redis
4. Configura las variables de entorno en el panel
5. **Build Command**: `npm run build`
6. **Start Command**: `npm run start:prod`

### Ejemplo: Despliegue en un VPS (DigitalOcean, Hetzner, etc.)

1. Instala Node.js 20+, MongoDB y Redis en el servidor
2. Clona el repositorio y configura las variables de entorno
3. Compila e inicia con PM2:
   ```bash
   cd backend
   npm install
   npm run build
   pm2 start dist/main.js --name nutribiotics-api
   pm2 save && pm2 startup
   ```
4. Configura Nginx como proxy reverso:
   ```nginx
   server {
       listen 80;
       server_name api.tu-dominio.com;

       location / {
           proxy_pass http://localhost:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```

---

## Proveedores de Base de Datos (MongoDB)

Si no quieres administrar MongoDB por tu cuenta, puedes usar un servicio gestionado:

| Proveedor | Tier Gratuito | Notas |
| --- | --- | --- |
| **MongoDB Atlas** | Si (512 MB) | La opcion mas popular, facil de configurar |
| **Railway** | Si (limitado) | Integrado si ya usas Railway para el backend |
| **AWS DocumentDB** | Pago | Compatible con MongoDB, para cargas empresariales |

---

## Proveedores de Redis

| Proveedor | Tier Gratuito | Notas |
| --- | --- | --- |
| **Upstash** | Si (10,000 comandos/dia) | Redis serverless, ideal para empezar |
| **Railway** | Si (limitado) | Integrado si ya usas Railway |
| **Redis Cloud** | Si (30 MB) | Servicio oficial de Redis |
| **AWS ElastiCache** | Pago | Para produccion a escala |

---

## Arquitectura Recomendada

```
                    ┌─────────────────────┐
                    │     Usuarios         │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │   CDN / Hosting      │
                    │  (Vercel, Netlify)   │
                    │   Frontend React     │
                    └─────────┬───────────┘
                              │ HTTPS
                    ┌─────────▼───────────┐
                    │   Servidor Node.js   │
                    │  (Railway, VPS)      │
                    │   Backend NestJS     │
                    └───┬─────────────┬───┘
                        │             │
              ┌─────────▼──┐   ┌──────▼────────┐
              │  MongoDB    │   │    Redis       │
              │  (Atlas)    │   │  (Upstash)     │
              └─────────────┘   └───────────────┘
```

---

## Resumen Rapido

| Componente | Que necesita | Ejemplo economico | Ejemplo robusto |
| --- | --- | --- | --- |
| **Frontend** | Hosting estatico con soporte SPA | Vercel (gratis) | AWS S3 + CloudFront |
| **Backend** | Servidor Node.js 24/7 | Railway (gratis) | VPS + PM2 + Nginx |
| **MongoDB** | Instancia de base de datos | Atlas (gratis 512MB) | Atlas dedicado o self-hosted |
| **Redis** | Instancia de Redis | Upstash (gratis) | Redis Cloud o self-hosted |
