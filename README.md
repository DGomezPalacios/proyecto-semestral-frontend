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

La aplicación está disponible públicamente a través de la instancia EC2-web:

| URL | Descripción |
|---|---|
| http://3.83.173.99 | Frontend React en producción |

No se requiere puerto adicional; nginx sirve en el puerto 80 estándar.

### Conexión a EC2-web

```bash
# Conectar a EC2-web (instancia con IP pública)
ssh -i devops-front.pem ec2-user@3.83.173.99

# Verificar que nginx y el contenedor frontend están activos
docker ps
docker-compose ps

# Ver logs del frontend
docker-compose logs -f frontend
```

> Permisos del archivo de clave antes de conectar:
> ```bash
> chmod 400 devops-front.pem
> ```

### Arquitectura de las 3 instancias

```
                        Internet
                           │
                           ▼
             ┌─────────────────────────┐
             │  EC2-web                │  IP Pública:  3.83.173.99
             │  IP Privada: 10.0.12.224│
             │  nginx + React (Docker) │
             └────────────┬────────────┘
                          │ VPC interna
                          ▼
             ┌─────────────────────────┐
             │  EC2-app                │  IP Privada: 10.0.131.198
             │  Spring Boot Despachos  │  Puerto 8081
             │  Spring Boot Ventas     │  Puerto 8082
             └────────────┬────────────┘
                          │ VPC interna
                          ▼
             ┌─────────────────────────┐
             │  EC2-datos              │  IP Privada: 10.0.145.181
             │  MySQL 8.0              │  Puerto 3306
             └─────────────────────────┘

             VPC: proyecto-semestral-vpc
```

### Flujo de conexión web → app → datos

1. **Usuario** accede a `http://3.83.173.99` desde su navegador.
2. **EC2-web** (nginx) sirve los archivos estáticos de React y actúa como reverse proxy para las llamadas a la API.
3. **EC2-app** recibe las peticiones API en los puertos `8081` (Despachos) y `8082` (Ventas) via IP privada `10.0.131.198`.
4. **EC2-datos** responde las consultas SQL en el puerto `3306` via IP privada `10.0.145.181`.

Todo el tráfico entre EC2-app y EC2-datos circula dentro de la VPC, sin exposición a internet.

### Endpoints disponibles desde el navegador

| Recurso | URL |
|---|---|
| Frontend | http://3.83.173.99 |
| API Despachos | http://3.83.173.99/api-despachos |
| API Ventas | http://3.83.173.99/api-ventas |
| Swagger Despachos | http://3.83.173.99/api-despachos/swagger-ui.html |
| Swagger Ventas | http://3.83.173.99/api-ventas/swagger-ui.html |

## Seguridad

- No exponer variables de entorno con prefijo `VITE_` que contengan secretos; son visibles en el bundle final.
- Mantener dependencias actualizadas: `npm audit` periódicamente.
- Usar HTTPS en producción con un reverse proxy (nginx, Traefik) frente al contenedor.

## DevOps

El `Dockerfile` sigue el patrón multi-stage:

1. **Stage `builder`** (`node:18-alpine`): instala dependencias y compila el proyecto.
2. **Stage runtime** (`nginx:alpine`): sirve los archivos estáticos del `dist/` generado.

Este enfoque garantiza que la imagen de producción no contiene el código fuente, las herramientas de build ni `node_modules`.
