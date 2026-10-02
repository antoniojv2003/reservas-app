# ==========================================
# ETAPA 1: build (Compilación de React con Vite)
# ==========================================
FROM node:20-alpine AS build

WORKDIR /app

# Copia de manifiestos y aprovechamiento del caché
COPY package.json package-lock.json ./

# Instalación reproducible completa (incluye devDependencies necesarias para compilar Vite)
RUN npm ci

# Copia del código fuente y build de producción
COPY . .
RUN npm run build


# ==========================================
# ETAPA 2: runtime (Nginx sin privilegios)
# ==========================================
FROM nginxinc/nginx-unprivileged:1.27-alpine

# Requisito consigna 6: variables de runtime y filtro envsubst
ENV API_HOST=reservas-api \
    API_PORT=3000 \
    NGINX_ENVSUBST_FILTER=API_

# Copia únicamente el build estático generado a la raíz servida por Nginx
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
