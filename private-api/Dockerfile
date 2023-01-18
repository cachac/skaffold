FROM node:14.11.0-alpine3.12 as base

WORKDIR /app

#builder stage
FROM base as builder
COPY package*.json ./

RUN npm install
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

EXPOSE 3002 3082

RUN adduser -D appuser
USER appuser

CMD ["node", "./dist/main.js"]
