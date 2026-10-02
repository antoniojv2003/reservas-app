#****ETAPA dist*****
# imagen base. Nota: se puede usar la de alpine también, para que no haya tanta carga en la red
FROM node:20 AS deps

# directorio de trabajo
WORKDIR /app-reservas-api

# traigo package.json y package-lock.json
COPY package.json package-lock.json ./

# instalar las dependencias
RUN npm ci --omit=dev


#****ETAPA final****
# imagen base (base mínima --> uso alpine ya que es más liviana que slim)
FROM node:20-alpine

# variable de entorno solicitada
ENV NODE_ENV=production

# ARG con la versión por defecto y la variable de entorno para ajustar la version (ENV APP_VERSION)
ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION}

# directorio de trabajo en esta etapa
WORKDIR /app-reservas-api

# traer las dependencias de la etapa deps y el codigo fuente con permisos de node
COPY --from=deps --chown=node:node /app-reservas-api/node_modules ./node_modules
COPY --chown=node:node . .

# usuario no privilegiado que ya provee la imagen oficial
USER node

# puerto documentado
EXPOSE 3000

# comando a ejecutar cuando se lanza una instancia
CMD ["npm", "start"]
