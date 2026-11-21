# Despliegue de una aplicación con Argo CD y Kubernetes (GitOps)

Este repositorio es la **fuente de verdad** de una aplicación web desplegada en Kubernetes.
Argo CD vigila la carpeta `k8s/` de la rama `main` y sincroniza automáticamente el clúster
con lo que haya en Git.

## Estructura

```
k8s/
  deployment.yml   # Deployment: 2 réplicas, labels/selectors, requests/limits, puerto 80
  service.yml      # Service NodePort 30080 -> targetPort 80 de los Pods (app=web-app)
  hpa.yml          # HPA: min 2, max 5 réplicas, objetivo 50 % de CPU
argocd/
  application.yml  # Aplicación de Argo CD con Automated Sync (prune + selfHeal)
```

La aplicación usa la imagen `registry.k8s.io/hpa-example` (Apache + PHP), que hace un
cálculo de CPU en cada petición y por eso sirve para demostrar el autoescalado.

## Requisitos

- Docker Desktop
- `kubectl`
- `minikube` (driver `docker`) con el addon `metrics-server`

## Puesta en marcha

```powershell
minikube start --driver=docker --cpus=4 --memory=4096 --addons=metrics-server

# Argo CD
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server

# Aplicación (Argo CD crea el namespace web-app y despliega k8s/)
kubectl apply -f argocd/application.yml
```

Interfaz de Argo CD:

```powershell
kubectl -n argocd port-forward svc/argocd-server 8080:443
# usuario: admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"   # en base64
```

Acceso a la aplicación (con el driver docker en Windows, minikube abre un túnel):

```powershell
minikube service web-app-service -n web-app --url
```

## Prueba del HPA

```powershell
kubectl run load-generator -n web-app --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://web-app-service; done"
kubectl get hpa -n web-app -w
kubectl delete pod load-generator -n web-app   # al parar la carga, vuelve a 2 réplicas
```

## Nota sobre réplicas y HPA

El Deployment declara `replicas: 2` como valor inicial, pero quien decide el número de
réplicas es el HPA. Para que Argo CD no marque la aplicación como *OutOfSync* ni devuelva
las réplicas a 2 cada vez que el HPA escala, la aplicación ignora `/spec/replicas` del
Deployment (`ignoreDifferences` + `RespectIgnoreDifferences=true`).
