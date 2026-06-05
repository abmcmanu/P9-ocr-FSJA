# ── Stage 1 : build Angular ───────────────────────────────────────────────────
FROM node:20-alpine AS build

WORKDIR /src

# Installation des dépendances en premier (cache Docker)
COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npx @angular/cli build --configuration=production

# ── Stage 2 : serveur Caddy (minimaliste et sécurisé) ────────────────────────
FROM caddy:2-alpine AS runtime

WORKDIR /app

COPY --from=build /src/dist/microcrm/browser /app/front

# Caddyfile embarqué : SPA avec gzip et fallback index.html
RUN printf ':80 {\n\troot * /app/front\n\tencode gzip\n\ttry_files {path} /index.html\n\tfile_server\n}\n' \
    > /etc/caddy/Caddyfile

EXPOSE 80

CMD ["caddy", "run", "--config", "/etc/caddy/Caddyfile"]
