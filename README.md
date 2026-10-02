## Arquitectura

    navegador ───> reservas-frontend ───> reservas-api ───────> reservas-db
                   React + Vite            Node 20 + Express    MySQL 8.4
                   nginx :8080             :3000                :3306
                   (host 3000)             (host 3001, solo     (sin puerto
                                            depuración)          publicado)

| Capa | Imagen | Puerto interno | Puerto publicado |
| --- | --- | --- | --- |
| `reservas-frontend` | React + Vite servido por nginx | 8080 | 3000 |
| `reservas-api` | Node 20 + Express | 3000 | 3001 (solo depuración) |
| `reservas-db` | MySQL 8.4 | 3306 | — |

El frontend es el único punto de entrada: nadie le habla a la base directamente, y a la API le habla el frontend.

La variable `NGINX_ENVSUBST_FILTER=API_` restringe el mecanismo de sustitución de plantillas de Nginx (`envsubst`) para que procese 
únicamente las variables de entorno que comiencen con el prefijo `API_` (como `API_HOST` y `API_PORT`). Sin este filtro, `envsubst` 
reemplaza cualquier patrón `$variable` en la configuración, consumiendo y dejando vacías variables internas esenciales de Nginx como
`$uri`, `$host` o `$proxy_add_x_forwarded_for`, lo que genera una configuración rota e inválida en silencio.

## Punto 8 del tp2: medición y comparación de imágenes

| Imagen | Imagen Base | Tamaño | Reducción vs Ingenua | Justificación técnica de la decisión |
| :--- | :--- | :--- | :--- | :--- |
| `reservas-api:ingenua` | `node:20` | 1.59GB | 0% (Línea base) | Se utilizó una imagen completa basada en Debian que incluye compiladores C/C++, Python y herramientas de compilación innecesarias en ejecución. Copia todo el contexto sin filtrar. |
| `reservas-api:v1` | `node:20-alpine` | 205  MB | ~87% | Multi-stage build: las herramientas de instalación se aíslan en la etapa `deps` (`npm ci --omit=dev`), se descartan cachés y se usa Alpine Linux como base mínima final junto con `.dockerignore`. |
| `reservas-frontend:v1` | `nginxinc/nginx-unprivileged:1.27-alpine` | 73.9 MB | ~96% | Multi-stage build completo: todo el runtime de Node.js, dependencias y código fuente de React/Vite se descartan tras compilar. La imagen final solo contiene Nginx ligero y los archivos estáticos de `dist/`. |
