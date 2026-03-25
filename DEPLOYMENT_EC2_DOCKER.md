# Despliegue en EC2 con Docker Compose

Esta guía describe cómo levantar **Nutribiotics** en una instancia **Amazon EC2** usando el archivo `docker-compose.yml` del repositorio. El stack incluye:

| Servicio | Rol |
| --- | --- |
| **gateway** | Nginx: puerto público (HTTP), enruta `/api/*` al backend y el resto al frontend. |
| **frontend** | Build estático de Vite servido con Nginx (solo red interna de Compose). |
| **backend** | API NestJS (solo red interna). |
| **redis** | Colas BullMQ (solo red interna; datos en volumen `redis-data`). |

**MongoDB** no va en este Compose: debe ser una instancia externa (p. ej. **MongoDB Atlas**), igual que en [DEPLOYMENT.md](DEPLOYMENT.md).

Para reglas de security group y tamaño de instancia, puedes cruzar con [RED_Y_EC2.md](RED_Y_EC2.md). Con este Compose, en **entrada pública** suele bastar el puerto del **gateway** (por defecto **80**); el backend **3000** y Redis **6379** no deben exponerse a Internet.

---

## 1. Requisitos previos

- Cuenta AWS y permisos para crear EC2 y security groups.
- Un cluster **MongoDB Atlas** (o compatible) y su `MONGODB_URI`.
- Claves de API que el backend exija (`OPENAI_API_KEY`, `GOOGLE_API_KEY`, etc.); revisa la sección de variables en [DEPLOYMENT.md](DEPLOYMENT.md).
- (Opcional) Dominio apuntando a la IP pública o el DNS de la instancia, si más adelante añades HTTPS.

---

## 2. Crear la instancia EC2

1. En la consola EC2, crea una instancia (por ejemplo **Ubuntu Server 22.04 LTS** o **Amazon Linux 2023**).
2. Tipo sugerido para pruebas: **t3.small** o superior; con carga ligera **t3.micro** puede ser justa. Más detalle en [RED_Y_EC2.md](RED_Y_EC2.md).
3. Asocia un volumen raíz acorde (p. ej. 20–30 GiB); el volumen Docker para Redis crece con el uso.
4. Asigna una **Elastic IP** si necesitas una IP fija para DNS o firewall en Atlas.

---

## 3. Security group (firewall)

**Entrada (inbound)**

| Puerto | Origen | Uso |
| --- | --- | --- |
| **22** | Tu IP / bastion / SSM | Solo administración SSH (evita `0.0.0.0/0`). |
| **80** | `0.0.0.0/0` o tus redes | HTTP al **gateway** (app + API bajo `/api`). |

Con este diseño **no** abras **3000** ni **6379** al público: el backend y Redis no publican puertos en el host.

**Salida (outbound)**

Permite al menos **HTTPS (443)** hacia Internet para MongoDB Atlas, APIs de IA y pulls de imágenes Docker.

**MongoDB Atlas:** en *Network Access*, permite la IP pública de la instancia (o la Elastic IP), o la regla que uses en tu entorno.

---

## 4. Instalar Docker Engine y Docker Compose

Instala **Docker Engine** y el plugin **Compose v2** siguiendo la documentación oficial para tu distribución:

- Ubuntu: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- Amazon Linux 2022/2023: [Install Docker Engine](https://docs.docker.com/engine/install/) (elige la variante que corresponda).

Comprueba:

```bash
docker --version
docker compose version
```

Añade tu usuario al grupo `docker` si no quieres usar `sudo` en cada comando (requiere cerrar sesión y volver a entrar):

```bash
sudo usermod -aG docker "$USER"
```

---

## 5. Obtener el código en el servidor

```bash
sudo apt-get update && sudo apt-get install -y git   # Ubuntu; en AL2023 usa dnf/yum según corresponda
git clone <URL-de-tu-repositorio> nutribiotics
cd nutribiotics
```

Si despliegas por **SSH desde tu máquina**, también puedes usar `scp`/`rsync` para subir una copia del proyecto.

---

## 6. Variables de entorno

En la raíz del repo (donde está `docker-compose.yml`):

```bash
cp compose.env.example .env
nano .env   # o el editor que prefieras
```

Completa como mínimo:

- `MONGODB_URI`
- `JWT_SECRET`, `JWT_REFRESH_SECRET` (valores largos y aleatorios)
- `SEED_USER_EMAIL`, `SEED_USER_PASSWORD`
- Claves de IA y las opcionales que uses

**Importante:** no definas `REDIS_URL` en `.env` para este despliegue: `docker-compose.yml` fija `redis://redis:6379` para el contenedor del backend.

Para el frontend compilado, el valor por defecto en Compose es **`VITE_API_BASE_URL=/api`**: el navegador llama a la misma máquina y el **gateway** enruta `/api` al backend. Solo necesitas cambiarlo si montas la API en otro origen (dominio distinto, sin gateway, etc.); si lo cambias, hay que **reconstruir** la imagen del frontend (ver sección 9).

---

## 7. Arrancar el stack

Primera vez (construye imágenes y levanta contenedores):

```bash
docker compose up -d --build
```

Solo arrancar (sin rebuild):

```bash
docker compose up -d
```

Ver estado y logs:

```bash
docker compose ps
docker compose logs -f
```

---

## 8. Comprobar que funciona

Sustituye `TU_IP_O_DNS` por la IP pública o el DNS de la instancia.

| URL | Qué deberías ver |
| --- | --- |
| `http://TU_IP_O_DNS/` | SPA (redirige al dashboard tras login si aplica). |
| `http://TU_IP_O_DNS/api/` | Respuesta del backend en la ruta raíz de la API (p. ej. mensaje del `AppController`). |
| `http://TU_IP_O_DNS/queues` | Panel Bull Board (si está habilitado en el backend). |

Inicia sesión con el usuario definido en `SEED_USER_EMAIL` / `SEED_USER_PASSWORD`.

Si la app carga pero las peticiones fallan, revisa logs del backend (`docker compose logs backend`) y que Atlas permita la IP de salida de la instancia.

---

## 9. Actualizar después de un `git pull`

Cuando cambie código de **backend** o **frontend**:

```bash
cd nutribiotics
git pull
docker compose up -d --build
```

Si solo cambian variables en `.env` (sin tocar código del backend en imagen), suele bastar:

```bash
docker compose up -d --force-recreate backend
```

Si cambias `VITE_*` usadas en build, reconstruye el frontend:

```bash
docker compose up -d --build frontend gateway
```

El **gateway** monta `gateway/nginx.conf` por volumen: si solo editas ese archivo, recarga Nginx sin rebuild:

```bash
docker compose exec gateway nginx -s reload
```

---

## 10. Puerto del gateway

Por defecto el gateway escucha en el **80** del host. Para usar otro puerto (p. ej. **8080**), en `.env`:

```env
GATEWAY_PUBLISH_PORT=8080
```

Luego `docker compose up -d`.

---

## 11. HTTPS (TLS)

Este Compose solo expone **HTTP** en el puerto configurado. Para producción se recomienda HTTPS, por ejemplo:

- **Application Load Balancer (ALB)** en AWS con certificado ACM, apuntando al puerto 80 de la instancia; o
- Un contenedor o servicio adicional (Caddy, Nginx con Let’s Encrypt) delante del gateway; o
- Terminación TLS en la misma máquina con certificados montados en un Nginx “edge”.

La configuración exacta depende de tu dominio y de si usas ALB u otra capa.

---

## 12. Copias de seguridad y datos

- **Redis:** los datos de cola/persistencia BullMQ están en el volumen nombrado `redis-data`. Para backups serios, documenta `docker volume inspect nutribiotics_redis-data` y tu política de snapshots del disco o exportación.
- **MongoDB:** sigue siendo la fuente de verdad en Atlas (o el servicio que uses); configura backups allí.

---

## 13. Problemas frecuentes

| Síntoma | Qué revisar |
| --- | --- |
| No carga la web | Security group **80** (o `GATEWAY_PUBLISH_PORT`), `docker compose ps`, logs del `gateway`. |
| Error de conexión a MongoDB | `MONGODB_URI`, red en Atlas, IP saliente de la instancia. |
| API 502 desde el navegador | Logs de `backend` y `gateway`; que el backend haya terminado de arrancar (`depends_on` no espera “healthy” del backend). |
| Frontend en blanco o API incorrecta | Que `VITE_API_BASE_URL` coincida con el enrutamiento (`/api` con este gateway). Rebuild del frontend si la cambiaste. |

---

## 14. Archivos de referencia en el repo

| Archivo | Uso |
| --- | --- |
| `docker-compose.yml` | Definición de servicios y redes. |
| `compose.env.example` | Plantilla para copiar a `.env`. |
| `gateway/nginx.conf` | Rutas `/api/` → backend, `/queues` → Bull Board, resto → frontend. |
| `backend/Dockerfile`, `frontend/Dockerfile` | Imágenes de API y UI. |

Para variables detalladas del backend y del frontend en otros entornos, sigue [DEPLOYMENT.md](DEPLOYMENT.md).
