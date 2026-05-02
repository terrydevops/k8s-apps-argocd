# k8s-apps-argocd

ArgoCD-managed deployment of a Java/Spring microservice estate on Kubernetes. Eight backend services behind an API gateway, with Eureka for discovery, RabbitMQ + Kafka for async, Zipkin for tracing, and Linkerd injected into every pod.

Sanitized reference layout. Image refs point at `ghcr.io/terrydevops/<service>` placeholders. Swap them for your own registry path before applying.

## Services

```
apigw           gateway (port 8083)
eureka-server   service discovery
api-users       (port 9090)
api-customers   (port 8080)
api-messages    (port 8090)
api-notifications (port 8088)
api-fraud       (port 8081)
api-posts       (port 8089)
api-security    (port 8000) - auth backend behind /auth ingress
```

All in the `asset` namespace. Linkerd injection is on by default via the `linkerd.io/inject: enabled` annotation. If you don't have Linkerd installed, comment those out.

## Layout

```
argocd-installation/    bootstrap manifests for ArgoCD itself
applications/           four ArgoCD Application CRs (services, rabbitmq, kafka, zipkin)
bootstrap/              cluster dependencies (rabbitmq, kafka, zipkin, postgres, otel, prom-graf-loki-tempo)
services/               per-service Deployments, Services, Ingresses
```

## Install

ArgoCD first:

```bash
kubectl create ns argocd
kubectl apply -n argocd -f ./argocd-installation/install.yaml
kubectl apply -f ./argocd-installation/argocd-ingress.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

Initial admin password (do this once, then change it via the UI):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

The four `applications/*.yaml` files point at `repoURL: https://github.com/terrydevops/k8s-apps-argocd.git`. If you forked, edit them to point at your fork. Then:

```bash
kubectl apply -f ./applications/
```

ArgoCD picks them up and syncs.

## Images

Every `services/*/deployment.yml` references `ghcr.io/terrydevops/<service>:<tag>`. Those don't actually exist on ghcr.io. Build your own:

```bash
docker build -t ghcr.io/<you>/api-users:1.0 services/api-users
docker push ghcr.io/<you>/api-users:1.0
```

Then update the `image:` line in each `services/*/deployment.yml`. For automation, wire it into a GitHub Actions workflow on each service's source repo and let `argocd-image-updater` bump the tag.

## A few notes

- The `cosmos-db-secret` references in deployments are Azure Cosmos DB (the document store). Nothing blockchain-related. Adjust `SPRING_PROFILES_ACTIVE` and the secret refs for your own data tier.
- `bootstrap/postgres/configmap.yml` declares a Secret with `POSTGRES_PASSWORD: REPLACE_ME`. Change it before applying. In production prefer SealedSecrets / external-secrets-operator.
- Ingress hosts are all `*.example.com`. Point them at your own domain.
- TLS uses cert-manager with a `letsencrypt` ClusterIssuer. Set that up separately if you don't have it.

## License

MIT.
