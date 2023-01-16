# Skaffold <!-- omit in toc -->

# Install
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64 && \
sudo install skaffold /usr/local/bin/

# Clone repo
git clone https://github.com/cachac/skaffold.git

# Dockerfile build

```dockerfile
FROM node:14.11.0-alpine3.12 as base

WORKDIR /app

#builder stage
FROM base as builder
COPY package*.json ./

RUN npm ci
#--only=production
COPY ./src ./src

RUN npm run build
RUN npm prune --production

#----------------release-----------------
FROM base AS release
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./

ARG APP_ENV
ENV APP_ENV=${APP_ENV}

ARG PRIVATE_API
ENV PRIVATE_API=${PRIVATE_API}

EXPOSE 3000

CMD ["node", "./dist/main.js"]
```

