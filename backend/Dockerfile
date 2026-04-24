# ---- Étape 1 : build ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY server.js ./

# ---- Étape 2 : image finale ----
FROM node:20-alpine
WORKDIR /app

# Créer un user non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY package*.json ./
RUN npm ci --omit=dev

COPY --from=builder /app/server.js ./

USER appuser

EXPOSE 3001

CMD ["node", "server.js"]