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

## Seguridad

- No exponer variables de entorno con prefijo `VITE_` que contengan secretos; son visibles en el bundle final.
- Mantener dependencias actualizadas: `npm audit` periódicamente.
- Usar HTTPS en producción con un reverse proxy (nginx, Traefik) frente al contenedor.

## DevOps

El `Dockerfile` sigue el patrón multi-stage:

1. **Stage `builder`** (`node:18-alpine`): instala dependencias y compila el proyecto.
2. **Stage runtime** (`nginx:alpine`): sirve los archivos estáticos del `dist/` generado.

Este enfoque garantiza que la imagen de producción no contiene el código fuente, las herramientas de build ni `node_modules`.
