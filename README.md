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
export APP_ENV_DEV="{ \"APP_NAME\":\"private-API\", \"NODE_ENV\":\"dev\", \"NODE_PORT\":\"3002\", \"TOKEN_LIMIT\":\"7d\", \"TOKEN_SECRET\":\"PASS\" }"

echo $APP_ENV_DEV
```
```yaml
    docker:
		  ...
			buildArgs:
				APP_ENV: "{{.APP_ENV_DEV}}"
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

# 11. Analyze
```vim
  skaffold init --analyze | jq
```

# 12. Concurrency & Cache
```yaml
build:
  local:
    concurrency: 0
```
```vim
 cat ~/.skaffold/cache
```
## 12.1. delete
```vim
skaffold delete
```
## 12.2. cached
```vim
skaffold dev --no-prune=false --cache-artifacts=false


```
> and then exit:
Cleaning up...
 - namespace "private" deleted
 - deployment.apps "deploy-private-api" deleted
 - service "svc-private-api" deleted
Pruning images...

# 13. port-forward
```vim
skaffold dev --port-forward
```
# 14. Run & Logs
```vim
skaffold run
skaffold run --tail

k get pods -n private
```

# 15. Tagging
```yaml
build:
  tagPolicy:
    envTemplate:
      template: "{{.APP_VERSION}}"
```
```
export APP_VERSION=4.0.0

skaffold dev
```
 - cachac/kubelabs_privateapi_skaffold: Found. Tagging

## 15.1. Check registry

## 15.2. --tag
```
skaffold dev --tag=4.0.1
```

## 15.3. Git tag
```yaml
  tagPolicy:
    gitCommit:
      prefix: "api-"
      variant: AbbrevCommitSha
```
Generating tags...
 - cachac/kubelabs_privateapi_skaffold -> cachac/kubelabs_privateapi_skaffold:api-51b89d6

## 15.4. Custom tags
```yaml
  tagPolicy:
    customTemplate:
      template: "{{.PREF}}_{{.SUFF}}"
      components:
        - name: PREF
          dateTime:
            format: "2006-01-02"
            timezone: "Local"APP_ENV_DEV
```

# 16. Profile
## 16.1. dev
```yaml
profiles:
  - name: dev
    # automatically activate this profile when current context is "ssh-microk8s"
    activation:
      - kubeContext: ssh-microk8s
    build:
      artifacts:
        - image: cachac/kubelabs_privateapi_skaffold
          docker:
            dockerfile: Dockerfile
            buildArgs:
              APP_ENV: "{{.APP_ENV_DEV}}"
```

## 16.2. stage
```yaml
  - name: stage
    activation:
      - kubeContext: ssh-microk8s-stage
    build:
      artifacts:
        - image: cachac/kubelabs_privateapi_skaffold
          docker:
            dockerfile: Dockerfile
            buildArgs:
              APP_ENV: "{{.APP_ENV_STAGE}}"
```

```
export APP_ENV_STAGE="{ \"APP_NAME\":\"private-API\", \"NODE_ENV\":\"stage\", \"NODE_PORT\":\"3002\", \"TOKEN_LIMIT\":\"7d\", \"TOKEN_SECRET\":\"PASS\" }"

echo $APP_ENV_STAGE
```
## 16.3. Optional: env variable
```yaml
activation:
	- env: MY_ENV=stage

```
## 16.4. test
```
skaffold dev
```
> --profile dev
> --profile stage

## 16.5. Profile patching

```yaml
    patches:
      - op: replace
        path: /build/artifacts/0/docker/buildArgs/APP_ENV
        value: "{{.APP_ENV_DEV}}"
```

# 17. kustomize
## 17.1. Install
> [Info](https://kubectl.docs.kubernetes.io/installation/kustomize/)
## 17.2. Create Kustomize structure: ./kustomize/private-api
- base
- overlays/dev
- overlays/stage

## 17.3. build
```
cd kustomize/private-api
kustomize build overlays/stage

# optional
kustomize build overlays/stage > build.yaml
```

## 17.4. Merge: replicas in dev = 2
> strategic merge
```yaml
patchesStrategicMerge:
  - deploy-strategic-patch.yaml
```
## 17.5. Merge: replicas in stage = 3
> strategic merge

## 17.6. In-line Patches: stage service on port 4000
> [Info](https://fabianlee.org/2022/04/15/kubernetes-kustomize-transformations-with-patchesjson6902/)
```yaml
patches:
  # In-line patch
  - target:
      kind: Service
      name: svc-private-api
    patch: |
      - op: replace
        path: /spec/ports/0/port
        value: 4000
```


# 18. Skaffold deploy + Kustomize

## 18.1. Set Dev profile:
```yaml
profiles:
  - name: dev
    activation:
      - kubeContext: ssh-microk8s
    patches:
      - op: replace
        path: /build/artifacts/0/docker/buildArgs/APP_ENV
        value: "{{.APP_ENV_DEV}}"
    manifests:
      kustomize:
        paths:
          - overlays/dev
```
```
kubectl config set-context --current --namespace=default
skaffold dev --profile dev
kubectl get ns
kubectl get pods -n private-dev
```

## 18.2. Set stage profile:
```
kubectl describe svc -n private-stage
kubectl get pods -n private-stage
```
> Port:              http-privateapi  4000/TCP
