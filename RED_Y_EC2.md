# Red y tamaño de instancia EC2 (Nutribiotics)

Documento corto: **qué puertos abrir** en despliegue y **orientación de tipo de instancia EC2** para el backend Node (NestJS). El frontend compilado suele ir en hosting estático (Vercel, S3, etc.); aquí se asume un **EC2 con API + proxy inverso** (p. ej. Nginx), que es el escenario típico descrito en [DEPLOYMENT.md](DEPLOYMENT.md).

---

## 1. Puertos a abrir (Security Group / firewall)

### Entrada (inbound) — hacia internet o tu red

| Puerto | Protocolo | ¿Abrir al público? | Uso |
| ------ | --------- | ------------------- | --- |
| **443** | TCP | **Sí** (tráfico de usuarios y clientes API) | HTTPS: Nginx (o ALB) termina TLS y hace proxy al backend. |
| **80** | TCP | Opcional | Redirección HTTP → HTTPS; si solo usas 443, puedes cerrar 80. |
| **22** | TCP | **No** a `0.0.0.0/0` | SSH solo desde tu IP, un bastion o AWS Systems Manager Session Manager (preferible sin abrir 22 al mundo). |

### No exponer a internet

| Puerto | Motivo |
| ------ | ------ |
| **3000** (u otro `PORT` del backend) | Debe escuchar en **localhost** (`127.0.0.1`) o en la VPC; Nginx/ALB hace proxy. No lo abras en el security group público. |
| **27017** (MongoDB) | Si MongoDB estuviera en la misma máquina, solo red privada. Con **MongoDB Atlas**, no aplica en EC2. |
| **6379** (Redis) | Igual: solo interno. Con **Redis gestionado** (Upstash, ElastiCache, etc.), el backend sale por **salida** hacia internet o VPC. |

### Salida (outbound)

| Destino típico | Puerto | Uso |
| -------------- | ------ | --- |
| MongoDB Atlas / gestionado | **443** (TLS) | Conexión a cluster en la nube. |
| Redis Upstash / Redis Cloud / similar | **443** o el que indique el proveedor | Muchos servicios serverless usan TLS sobre 443. |
| APIs Google Generative AI, etc. | **443** | Llamadas HTTPS salientes. |

Regla práctica: permitir **salida TCP 443** (y lo que exijan tus proveedores). Evita `0.0.0.0/0` en **entrada** salvo 80/443 para la aplicación.

---

## 2. Tamaño recomendado de instancia EC2

El backend es **Node.js (NestJS)** con **Mongoose**, **BullMQ** (workers en el mismo proceso) y llamadas **HTTP salientes** a IA; la CPU en EC2 no entrena modelos, pero sí serializa JSON, colas y concurrencia moderada.

### Recomendación de partida (producción pequeña / mediana)

| Tipo | vCPU | RAM | Comentario |
| ---- | ---- | --- | ---------- |
| **t3.small** | 2 | 2 GiB | Buen equilibrio coste/rendimiento: Nest + Node heap, algo de cola concurrente y picos breves. |
| **t3.medium** | 2 | 4 GiB | Si hay **muchos jobs en paralelo**, muchos usuarios simultáneos o quieres margen para logs/monitoreo sin presionar memoria. |

### Entornos muy ligeros o pruebas

| Tipo | RAM | Riesgo |
| ---- | --- | ------ |
| **t3.micro** | 1 GiB | Posible **presión de memoria** con Node + dependencias; solo para demos o tráfico muy bajo con concurrencia limitada. |

### Notas

- Familia **t3** es **burstable**: adecuada si la carga no es CPU al 100% todo el tiempo; vigila créditos en CloudWatch.
- Si la cola y el API compiten mucho, valorar **separar worker en otra instancia** o usar **ECS/Fargate** más adelante; no es obligatorio al inicio.
- **Almacenamiento:** 20–30 GiB gp3 suele bastar para SO + despliegue + logs rotados.

---

## 3. Resumen

- **Abrir al público:** principalmente **443** (y opcionalmente **80**). **SSH** restringido, no el backend en **3000**.
- **EC2 razonable para empezar:** **t3.small**; subir a **t3.medium** si ves uso alto de RAM o colas concurrentes.

Para despliegue paso a paso (Nginx, PM2, variables de entorno), sigue [DEPLOYMENT.md](DEPLOYMENT.md).
