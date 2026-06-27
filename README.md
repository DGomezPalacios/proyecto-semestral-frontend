# Frontend — Proyecto Semestral DevOps

Interfaz de usuario para el sistema de Despachos y Ventas, construida con Vite + React y Tailwind CSS.

## Tecnologías

| Tecnología | Versión | Uso |
|---|---|---|
| React | 18.2 | Biblioteca UI |
| Vite | 5.2 | Bundler y dev server |
| Tailwind CSS | 3.4 | Estilos utilitarios |
| React Router DOM | 6.24 | Enrutamiento SPA |
| Axios | 1.6 | Cliente HTTP |
| React Hook Form | 7.52 | Manejo de formularios |
| SweetAlert2 | 11 | Alertas y modales |

## Iniciar en local (sin Docker)

```bash
# Instalar dependencias
npm install

# Levantar servidor de desarrollo
npm run dev
# Disponible en http://localhost:5173

# Compilar para producción
npm run build

# Previsualizar build de producción
npm run preview
```

## Iniciar con Docker

### Solo el frontend

```bash
# Construir imagen
docker build -t frontend-app .

# Ejecutar contenedor
docker run -p 80:80 frontend-app
# Disponible en http://localhost
```

### Stack completo con docker-compose

Ejecutar desde la raíz del proyecto (donde está `docker-compose.yml`):

```bash
# Levantar todos los servicios
docker-compose up --build

# Levantar en segundo plano
docker-compose up -d --build

# Ver logs
docker-compose logs -f frontend

# Detener todos los servicios
docker-compose down

# Detener y eliminar volúmenes
docker-compose down -v
```

## Estructura del proyecto

```
proyecto-semestral-frontend/
├── public/              # Archivos estáticos públicos
├── src/
│   ├── components/      # Componentes reutilizables
│   ├── pages/           # Vistas/páginas por ruta
│   ├── services/        # Llamadas a la API (Axios)
│   └── main.jsx         # Punto de entrada
├── Dockerfile           # Imagen multi-stage (node → nginx)
├── .dockerignore        # Exclusiones del contexto Docker
├── vite.config.js       # Configuración de Vite
├── tailwind.config.js   # Configuración de Tailwind
└── package.json         # Dependencias y scripts
```

## Puertos

| Entorno | URL |
|---|---|
| Desarrollo local | http://localhost:5173 |
| Producción (Docker) | http://localhost:80 |

## Best practices aplicadas

- **Multi-stage build**: imagen final basada en `nginx:alpine`, sin Node.js en runtime (imagen ~20 MB vs ~400 MB).
- **Usuario no-root**: el contenedor nginx corre con usuario sin privilegios de root.
- **`.dockerignore`**: excluye `node_modules`, `dist` y archivos de entorno para reducir el contexto de build.
- **Variables de entorno**: nunca comitear `.env.local` con credenciales reales.
- **npm ci**: usa `package-lock.json` para installs reproducibles y deterministas.

## ☁️ Despliegue en AWS

### Acceso al Frontend en producción

| URL | Descripción |
|---|---|
| http://3.83.173.99:80 | Frontend React en producción |

nginx sirve la aplicación en el puerto 80 estándar de la instancia EC2-web.

### Imagen Docker Hub

```bash
# Imagen publicada en Docker Hub
docker pull dgomezpalacios/proyecto-semestral-frontend:latest

# Ejecutar directamente desde Docker Hub
docker run -d -p 80:80 dgomezpalacios/proyecto-semestral-frontend:latest
```

En EC2-web el `docker-compose.yml` apunta a esta imagen, por lo que no se necesita build local:

```bash
docker-compose up -d   # descarga dgomezpalacios/proyecto-semestral-frontend:latest y levanta
```

### Conexión a EC2-web

```bash
# Permisos correctos antes de conectar (Linux/Mac)
chmod 400 devops-front.pem

# Conectar a EC2-web (única instancia con IP pública)
ssh -i devops-front.pem ec2-user@3.83.173.99

# Verificar que nginx y el contenedor frontend están activos
docker ps
docker-compose ps

# Ver logs del frontend
docker-compose logs -f frontend
```

### Arquitectura de las 3 instancias

```
                        Internet
                           │
                           ▼
             ┌───────────────────────────────┐
             │  EC2-web    3.83.173.99:80    │  ← IP pública
             │  Privada:   10.0.12.224       │
             │  nginx + React (Docker Hub)   │
             └──────────────┬────────────────┘
                            │ VPC proyecto-semestral-vpc
                            │     10.0.0.0/16
                            ▼
             ┌───────────────────────────────┐
             │  EC2-app   10.0.131.198       │
             │  Despachos  :8081             │
             │  Ventas     :8082             │
             └──────────────┬────────────────┘
                            │ VPC interna
                            ▼
             ┌───────────────────────────────┐
             │  EC2-datos  10.0.145.181      │
             │  MySQL 8.0  :3306             │
             └───────────────────────────────┘
```

### Flujo de conexión web → app → datos

1. **Usuario** accede a `http://3.83.173.99:80` desde su navegador.
2. **EC2-web** (nginx) sirve los archivos estáticos de React y enruta las llamadas API hacia EC2-app.
3. **EC2-app** recibe las peticiones en los puertos `8081` (Despachos) y `8082` (Ventas) via IP privada `10.0.131.198`.
4. **EC2-datos** responde las consultas SQL en el puerto `3306` via IP privada `10.0.145.181`.

Todo el tráfico entre EC2-app y EC2-datos circula dentro de la VPC (`10.0.0.0/16`), sin exposición a internet.

### Endpoints disponibles desde el navegador

| Recurso | URL |
|---|---|
| Frontend | http://3.83.173.99:80 |
| API Despachos | http://3.83.173.99:8081 |
| API Ventas | http://3.83.173.99:8082 |
| Swagger Despachos | http://3.83.173.99:8081/swagger-ui.html |
| Swagger Ventas | http://3.83.173.99:8082/swagger-ui.html |

## Seguridad

- No exponer variables de entorno con prefijo `VITE_` que contengan secretos; son visibles en el bundle final.
- Mantener dependencias actualizadas: `npm audit` periódicamente.
- Usar HTTPS en producción con un reverse proxy (nginx, Traefik) frente al contenedor.

## DevOps

El `Dockerfile` sigue el patrón multi-stage:

1. **Stage `builder`** (`node:18-alpine`): instala dependencias y compila el proyecto.
2. **Stage runtime** (`nginx:alpine`): sirve los archivos estáticos del `dist/` generado.

Este enfoque garantiza que la imagen de producción no contiene el código fuente, las herramientas de build ni `node_modules`.

## ☁️ EP3 — Despliegue en AWS ECS Fargate

### Arquitectura EP3
El frontend migró de EC2 con Docker a AWS ECS Fargate con orquestación serverless.

- **Orquestador:** AWS ECS Fargate
- **Registry:** Amazon ECR (proyecto-semestral-frontend)
- **Load Balancer:** Application Load Balancer (ALB)
- **URL pública:** http://proyecto-semestral-alb-619727221.us-east-1.elb.amazonaws.com

### Recursos AWS

| Recurso | Nombre |
|---------|--------|
| Clúster ECS | proyecto-semestral-cluster |
| Servicio ECS | frontend-service |
| Task Definition | frontend-task:1 |
| Repositorio ECR | proyecto-semestral-frontend |
| Target Group | frontend-tg |
| Región | us-east-1 |

### Pipeline CI/CD (EP3)
El pipeline se dispara automáticamente en cada push a la rama deploy:
1. Build — construye la imagen Docker
2. Push — sube la imagen a Amazon ECR con tag :latest y :<commit-sha>
3. Deploy — ejecuta aws ecs update-service --force-new-deployment en ECS

#### Secrets requeridos en GitHub
| Secret | Descripción |
|--------|-------------|
| AWS_ACCESS_KEY_ID | Credencial AWS |
| AWS_SECRET_ACCESS_KEY | Credencial AWS |
| AWS_SESSION_TOKEN | Token de sesión AWS Academy |

### Autoscaling ECS
| Parámetro | Valor |
|-----------|-------|
| Tipo | Target Tracking Scaling |
| Métrica | ECSServiceAverageCPUUtilization |
| Umbral | 50% CPU |
| Mínimo tasks | 1 |
| Máximo tasks | 3 |
| Cooldown | 60 segundos |

### Logs
Los logs del contenedor se envían automáticamente a CloudWatch Logs en el grupo /ecs/frontend.

## 👥 Equipo
- Daniela Gómez Palacios
- Berta Soto Jerez

**Curso:** ISY1101 — Introducción a Herramientas DevOps  
**Profesor:** Álvaro Mellado Pimentel  
**Instituto:** DuocUC — 2025
