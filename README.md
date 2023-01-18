# Skaffold <!-- omit in toc -->

# 1. Install
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64 && \
sudo install skaffold /usr/local/bin/

# 2. Clone repo
```vim
git clone https://github.com/cachac/skaffold.git
cd private-api
```

# 3. Dockerfile build

```dockerfile
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

USER appuser

CMD ["node", "./dist/main.js"]

```

# 4. Kubernetes yaml
- ns
- deploy
- service

# 5. Init
```vim
skaffold init
```
> Docker (dockerfile)
> [x]  Buildpacks (package.json)

## 5.1. Skaffold yaml configuration
- skaffold.yaml

# 6. skaffold build

> Build Failed. No push access to specified image repository. Try running with `--default-repo` flag

## 6.1. fix locally
```vim
skaffold build --push=false
```

## 6.2. Build & Push (remote registry)

### 6.2.1. docker login
```vim
docker login
```

### 6.2.2. Check config.json path
 > ~/.docker/config.json

### 6.2.3. Optional: Kaniko
> [tutorial](https://github.com/GoogleContainerTools/kaniko/blob/main/docs/tutorial.md)
> [config](https://skaffold.dev/docs/pipeline-stages/builders/docker/#dockerfile-in-cluster-with-kaniko)



# 7. skaffold dev
Construcción en repositorio local (desarrollo)
```vim
skaffold dev

# or
skaffold dev --kubeconfig ~/.kube/clusters/ssh-microk8s.yaml
```

## 7.1. Check errors
```vim
k get pods

kubectl logs deployment/deploy-private-api-beta
```
> deploy-private-api-beta   0/1     Error


# 8. Environment Variables
```vi
export APP_ENV_KUBE_PRIVATE_API="{ \"APP_NAME\":\"private-API\", \"NODE_ENV\":\"qa\", \"NODE_PORT\":\"3002\", \"TOKEN_LIMIT\":\"7d\", \"TOKEN_SECRET\":\"PASS\" }"

echo $APP_ENV_KUBE_PRIVATE_API
```
```yaml
        buildArgs:
          APP_ENV: "{{.APP_ENV_KUBE_PRIVATE_API}}"
```
```vim
skaffold dev
```

# 9. test
```vi
kubectl -n private port-forward service/svc-private-api 3002:3002
curl http://localhost:3002/healthcheck
```


# 10. change & auto re-build Image
> package.json: "version": "3.0.0",

















templates:
https://stackoverflow.com/questions/50359633/use-of-skaffold-using-minikube-without-registry

build:
  tagPolicy:
    envTemplate:
      template: "{{.IMAGE_NAME}}:{{.APP_VERSION}}"


https://skaffold.dev/docs/tutorials/ci_cd/

    build:
      artifacts:
        # Skaffold will use this as your image name and push it here after building
        - image: asia.gcr.io/my-project/my-image
          # We are using Docker as our builder here
          docker:
            # Pass the args we want to Docker during build
            buildArgs:
              NPM_REGISTRY: '{{.NPM_REGISTRY}}'
              NPM_TOKEN: '{{.NPM_TOKEN}}'
