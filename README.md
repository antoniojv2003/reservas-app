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


## Punto 2 del tp3: creación del contenedor de la base de datos y justificación de la ausencia del parámetro -p

Para crear este contenedor se usó el siguiente comando:
`docker run -d --name reservas-db --network reservas-net -v reservas-db-data:/var/lib/mysql -e MYSQL_PASSWORD=reservas_pass -e MYSQL_DATABASE=reservasdb -e MYSQL_USER=reservas_user -e MYSQL_ROOT_PASSWORD=reservas_root_pass mysql:8.4`

La base de datos constituye la capa más interna y crítica de la arquitectura. Al prescindir del flag -p, sus puertos de escucha no se enlazan ni se exponen en la 
interfaz del anfitrión (host), evitando accesos no autorizados desde redes externas. La única vía de comunicación permitida es interna a través de la red bridge 
personalizada reservas-net, alcanzable exclusivamente por contenedores autorizados conectados a dicha red (como reservas-api).

## Punto 4 del tp3: explicación de por qué el proceso termina en lugar de arrancar en estado degradado
Nginx resuelve el upstream de proxy_pass una única vez, en tiempo de inicialización estricta al arrancar. Al omitir --network reservas-net, el contenedor opera en la red bridge
predeterminada sin servidor DNS interno; al no poder resolver el nombre reservas-api, el proceso termina de forma inmediata con código 1 en lugar de arrancar en un estado degradado.


## Punto 5 del tp3: solicitud se dirige a /api/reservas sobre el mismo origen (localhost:3000 o 127.0.0.1:3000) y no al backend directamente
Al realizar la reserva desde la interfaz web, la solicitud HTTP (POST /api/reservas) se envía de manera relativa al mismo origen ([http://127.0.0.1:3000/api/reservas](http://127.0.0.1:3000/api/reservas)) 
desde el cual fue descargada la SPA. Dado que la petición no cruza fronteras de protocolo, host ni puerto, el navegador la considera una solicitud del mismo origen (Same-Origin).
En consecuencia, no aplica la política restrictiva de CORS ni se dispara una petición preliminar de comprobación previa (preflight request con método OPTIONS), fluyendo 
la petición directamente a través del reverse proxy de Nginx hacia el contenedor reservas-api.


## Punto 6 del tp3: consulta a la api directamente y mediante el nginx usando /api/
Las pruebas validaron que Nginx funciona como un reverse proxy transparente, entregando la misma respuesta tanto al consultar la API de forma directa (localhost:3001/salas) como 
a través del frontend (localhost:3000/api/salas). Esto se debe a la barra final en proxy_pass http://${API_HOST}:${API_PORT}/;, que recorta el prefijo /api y envía la ruta limpia 
/salas a Express, evitando un error 404. A su vez, esta configuración centraliza la entrega de archivos estáticos y el consumo del backend bajo un mismo origen (Same-Origin), 
prescindiendo de negociaciones de CORS y solicitudes OPTIONS.


## Punto 7 del tp3: persistencia de los datos
La prueba demostró que el ciclo de vida de los datos es independiente del ciclo de vida del contenedor: al destruir y recrear reservas-db, la información persistió intacta gracias al 
uso del volumen nombrado reservas-db-data, permitiendo que la base retome su estado previo sin pérdida de registros.


## Consigna 8: Orden de arranque y dependencias del sistema

### Orden de inicialización requerido
El orden estricto de arranque manual para el correcto funcionamiento del sistema es:
red → volumen → base → API → frontend


### Diagnóstico de fallas al invertir pasos

* **Invertir Red -  Contenedores:**  
  Si se intenta ejecutar cualquier contenedor especificando `--network reservas-net` antes de haber creado la red bridge, el daemon de Docker aborta el comando inmediatamente con 
  el error: `network reservas-net not found`.

* **Invertir Volumen -  Base de Datos:**  
  Aunque Docker crea volúmenes nombrados de forma automática si no existen previamente al hacer `-v reservas-db-data:/var/lib/mysql`, omitir su creación explícita o desacoplarlo 
  rompe el control de infraestructura como código y la persistencia predeterminada. Si el contenedor se inicializa sin el montaje, los datos residen en la capa volátil de 
  escritura del contenedor y se destruyen permanentemente al removerlo.

* **Invertir Base de Datos - API:**  
  Si `reservas-api` arranca antes de que `reservas-db` esté lista y escuchando en el puerto 3306, el proceso de Node/Express falla de inmediato al ejecutar las migraciones iniciales 
  de Sequelize o la siembra de datos (`SEED_DEMO=true`). El contenedor finaliza con error de conexión `ECONNREFUSED reservas-db:3306` o fallo de resolución DNS interno.

* **Invertir API - Frontend:**  
  Si `reservas-frontend` se inicia antes que `reservas-api`, el proceso maestro de Nginx falla fatalmente durante su inicialización estricta. Al no encontrar el hostname 
  `reservas-api` en el servidor DNS embebido de Docker, arroja `[emerg] host not found in upstream "reservas-api"` y el contenedor pasa a estado `Exited (1)`.


### Limitación del despliegue actual
Actualmente, el orden de arranque, las dependencias de red, el montaje de volúmenes y las variables de entorno residen exclusivamente en la memoria y criterio del operador 
que levanta la aplicación, y no en un archivo declarativo ejecutable.  
Esta limitación se resuelve la semana siguiente mediante la incorporación de **Docker Compose**, permitiendo orquestar todo el stack de forma declarativa a través de 
directivas como `depends_on`, `networks` y `volumes` dentro de un único archivo `docker-compose.yml`.
