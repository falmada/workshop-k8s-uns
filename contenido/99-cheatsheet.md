# 99 - Cheatsheet

## k3d

- Iniciar el cluster creado: `k3d cluster start mi-cluster`
- Frenar el cluster creado: `k3d cluster start mi-cluster`

## Docker

- Limpiar registro local de imágenes: `docker image prune`

## Kubernetes

- Obtener todos los pods y deployments de un namespace: `kubectl get pods,deployments`
- Describir un pod: `kubectl describe pod-name`
- Obtener logs de un pod: `kubectl logs pod-name`
- Conectarse a un pod con shell: `kubectl exec -ti pod-name -- /bin/sh` (asume que imagen contiene /bin/sh)
