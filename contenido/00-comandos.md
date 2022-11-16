# 00 - Comandos

En este archivo sólo se listan los comandos que serán pedidos durante el workshop y que son largos como para dictar o copiar manualmente.

## 01 - Preparación - Instalación de docker-ce

```sh
# Actualizamos apt con los ultimos paquetes
sudo apt update
# Instalamos dependencias
sudo apt install apt-transport-https ca-certificates curl software-properties-common
# Agregamos la llave GPG del repo oficial de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
# Agregamos el repo de Docker a las fuentes de APT
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
# Instalamos docker-ce
sudo apt install docker-ce
# Revisemos que tiene el servicio corriendo
sudo systemctl status docker
# Darle permiso a nuestro usuario para correr docker
# Por defecto docker sólo corre con root
sudo usermod -aG docker ${USER}
# Nos relogueamos a la terminal o saltamos a una sub shell
su - ${USER}
# Verificamos que entre los grupos tenemos a docker
groups
```

## 01 - Jugando con docker - Obtener y ejecutar imagen

```bash
# obtener imagen de alpine
docker pull alpine:latest
# correr imagen de alpine localmente
docker run --name docker_demo --rm -i -t alpine:latest sh 
```

## 01 - Mejor armemos una imagen propia

```bash
# Parados en root de este repo
docker build -t workshop-uns:latest extras/01-contenedores
# Conectarnos a imagen
docker run --name alpine_demo --rm -i -t my-image:latest sh
# Corremos un comando cualquiera
curl google.com
# Salimos
exit
```

## 01 - Pusheando nuestra imagen

```bash
# Personalizar comando
DOCKER_USER="myuser"
# Login
docker login
# Tagueamos la imagen con el nombre del repo
docker tag workshop-uns:latest ${DOCKER_USER}/workshop-uns:latest
# Hacemos push
docker push ${DOCKER_USER}/workshop-uns:latest
```

## 02 - Crear imagen de nuestra aplicación

```bash
# Ver docker file
cat extras/02-imagenes/v0.1.0/Dockerfile
# Ver codigo de la app simulada
cat extras/02-imagenes/v0.1.0/src/index.php
# Crear imagen
docker build -t my-app extras/02-imagenes/v0.1.0/    
# Probar
docker run --name test --rm -i -t my-app sh 
# Revisar contenido dentro de contenedor
ls /var/www/html/
# Deberíamos ver un "index.php", luego salimos
exit
# Ahora corremos la imagen pero exponiendo puerto 80
docker run -p 80:80 my-app
```

## 02 - Tagging de imágenes

```bash
# Agregar tag v0.1.0 a nuestra app
docker tag my-app:latest my-app:v0.1.0
# Revisar tags en nuestras imagenes locales
docker images | grep my-app
```

## 02 - Actualizando imágen

```bash
# Ver cambios
cat extras/02-imagenes/v0.2.0/Dockerfile
# Crear nueva imagen
docker build -t my-app:v0.2.0 extras/02-imagenes/v0.2.0/
# Ver imagenes locales
docker images | grep my-app
# Actualizar latest
docker tag my-app:v0.2.0 my-app:latest
# Comprobar versiones locales 
docker images | grep my-app    
```

## 02 - Subiendo nuestra imagen a un registry

```bash
# Personalizar comando
DOCKER_USER="myuser"
# Login
docker login
# Tagueamos la imagen con el nombre del repo
docker tag my-app:v0.2.0 ${DOCKER_USER}/workshop-uns:v0.2.0
docker tag my-app:v0.2.0 ${DOCKER_USER}/workshop-uns:latest
# Hacemos push
docker push ${DOCKER_USER}/workshop-uns:v0.2.0
docker push ${DOCKER_USER}/workshop-uns:latest
```

## 02 - Multiple instancias de la misma imagen

```bash
# Correr en puerto 80
docker run -p 80:80 my-app:latest
# Correr en puerto 8080
docker run -p 8080:80 my-app:latest
```

## 03 - Instalacion de Kubernetes en local - Instalar k3d

```bash
# Descargar e instalar script de instalacion
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
# Ejecutar k3d para ver opciones
k3d
# Ver version actual
k3d version
# Crear cluster local
k3d cluster create "mi-cluster" \
    --api-port 0.0.0.0:6443 \
    --network 10.100.0.0/16 \
    --servers 1 \
    --servers-memory 256Mi \
    --agents 3 \
    --agents-memory 1Gi \
    --port "30000-30100:30000-30100@server:0" \
    --registry-create mycluster-registry:0.0.0.0:5432
```

## 03 - Veamos nuestro mini cluster andando

```bash
# Ver info del cluster
kubectl cluster-info
# Ver pods
kubectl get pods -A
```

## 03 - Primera prueba de pod

```bash
# Creamos un pod "mi-pod" usando la imagen de "nginx", un conocido web server
kubectl run mi-pod --image=nginx --restart=Never
# Luego procedemos a conectarnos a ella por shell y ejecutar algunos comandos
 kubectl exec -ti mi-pod -- /bin/bash
# Dentro del pod podemos correr comandos para ver donde estamos
hostname
uptime
ps 
ls /    
df     
mount | grep remount-ro
# cuando terminamos de indagar, salimos de la shell hacia el pod
exit
# Exportamos un pod usando el local 8080
kubectl port-forward mi-pod 8080:80
# Una vez probado el acceso via browser, podemos finalizar el proceso con CTRL+C
```

## 03 - Nuestra app en un Pod

```bash
# Personalizar comando
DOCKER_USER="myuser"
# Con dry-run=client, simulamos una acción, y con -o yaml exportamos la salida a ese formato
kubectl run mi-pod --image=${DOCKER_USER}/workshop-uns:v0.2.0 --restart=Never --dry-run=client -o yaml > mi-pod.yaml
# Esto nos genera el archivo: mi-pod.yaml, veamoslo
cat mi-pod.yaml
# Al ser un YAML/declarativo, con apply -f podemos aplicar el manifiesto
kubectl apply -f extras/03-kubernetes/deploy-01/mi-pod.yaml
# Luego podemos ver el estado del pod
kubectl get pod my-pod
```

## 03 - Exponer y probar nuestra app

```bash
kubectl port-forward mi-pod 8080:80
```

## 03 - Servicios de tipo NodePort

```bash
# Dado que cuando creamos el cluster en k3d, especificamos un rango de 30000 a 30100, vamos a usar el más alto
# Los servicios de NodePort asignan un puerto al azar entre 30000 y 32767, por lo que hay altas chances de que 
# no esté dentro de nuestro rango a menos que lo especifiquemos
kubectl create service nodeport mi-pod --tcp=80 --node-port=30100 --dry-run=client -o yaml > svc-my-pod.yaml
# Paso siguiente, aplicamos ese manifiesto en Kubernetes
kubectl apply -f extras/03-kubernetes/deploy-02/svc-my-pod.yaml
# Vemos estado del servicio
kubectl get svc
# Vemos los endpoints
kubectl get endpoints
# Corregimos el servicio
# Vemos los endpoints nuevamente
kubectl get ep 
# Comprobamos ip del pod
kubectl get pod mi-pod -o wide
```

## 03 - Objetos de tipo Ingress

``` bash
# Aplicamos
kubectl apply -f extras/03-kubernetes/deploy-03/mi-app-svc.yaml
# Podemos confirmar que encontró los pods
kubectl get endpoints
# Ahora creamos el Ingress también
kubectl apply -f extras/03-kubernetes/deploy-03/mi-app-ingress.yaml
# Validamos ingress
kubectl get ingress
# Editamos nuestros hosts
sudo vi /etc/hosts
# Agregar línea con alguno de los IPs del paso previo y el FQDN mi-app.com, ejemplo
192.168.48.2 mi-app.com
# Luego hacemos un ping para validar
ping -c1 mi-app.com
```

## 04 - Self-healing

```
# Visualizamos pods
kubectl get pods
# Eliminammos pod 
kubectl delete pod mi-pod
# Comprobamos pods
kubectl get pods
```

## 04 - DaemonSet

```bash
# revisamos yaml del daemonset
cat extras/04-hola-mundo-real/deploy-01/daemonSet-mi-app.yaml
# aplicamos yaml
kubectl apply -f extras/04-hola-mundo-real/deploy-01/daemonSet-mi-app.yaml
# vemos pods
kubectl get pods -w
# luego control Z
# revisamos pods actuales
kubectl get pods
# borramos un pod
kubectl delete pod mi-app-XYZ
# revisamos pods actuales
kubectl get pods -w
# control Z y luego revisar en donde esta cada pod
kubectl get pods -o wide
# eliminamos daemonset
kubectl delete -f extras/04-hola-mundo-real/deploy-01/daemonSet-mi-app.yaml
```

## 04 - ReplicaSet

```bash
# revisamos yaml del replicaset
cat extras/04-hola-mundo-real/deploy-02/replicaSet-mi-app.yaml
# aplicamos yaml
kubectl apply -f extras/04-hola-mundo-real/deploy-02/replicaSet-mi-app.yaml
# vemos pods
kubectl get pods
# vemos con info de nodo 
kubectl get pods -o wide
# eliminamos un pod
kubectl delete pod mi-app-XYZ
# vemos pods
kubectl get pods -o wide
# escalamos
kubectl scale --replicas=5 replicaset/mi-app
# vemos pods
kubectl get pods -o wide
# ver cambios
diff extras/02-imagenes/v0.2.0/src/index.php extras/04-hola-mundo-real/deploy-03/v0.3.0/src/index.php
# Generar nueva imagen
docker build -t my-app:v0.3.0 extras/04-hola-mundo-real/deploy-03/v0.3.0
# Personalizar comando
DOCKER_USER="myuser"
# Login
docker login
# Tagueamos la imagen con el nombre del repo
docker tag my-app:v0.3.0 ${DOCKER_USER}/workshop-uns:v0.3.0
docker tag my-app:v0.3.0 ${DOCKER_USER}/workshop-uns:latest
# Hacemos push
docker push ${DOCKER_USER}/workshop-uns:v0.3.0
docker push ${DOCKER_USER}/workshop-uns:latest
# Actualizamos replicaset
kubectl apply -f extras/04-hola-mundo-real/deploy-03/replicaSet-mi-app.yaml
# vemos estado de pods
kubectl get pods -w
# vemos imagen en uso de pods
kubectl describe pod | grep Image:
# eliminamos pod
kubectl delete pod mi-app-XYZ
# comprobamos imagenes en uso
kubectl describe pod | grep Image:
# eliminamos replicaset
kubectl delete replicaSet mi-app
```

## 04 - Deployments

```bash
# Personalizar comando
DOCKER_USER="myuser"
# creamos yaml apuntando a nuestra imagen
kubectl create deployment mi-deploy --image=${DOCKER_USER}/workshop-uns:v0.2.0 --replicas=3 --port=80 --dry-run=client -o yaml > mi-deploy.yaml
# vemos salida
cat mi-deploy.yaml
# aplicamos yaml
kubectl apply -f extras/04-hola-mundo-real/deploy-04/mi-deploy.yaml
# vemos pods
kubectl get pods -w
# vemos replicaset
kubectl get replicaset
# nuevo deploy y visualizar comportamiento de pods
kubectl apply -f extras/04-hola-mundo-real/deploy-05/mi-deploy.yaml; kubectl get pods -w
```

## 04 - Balanceo por medio de servicio

```bash
# veamos la definicion del servicio
cat extras/04-hola-mundo-real/deploy-05/mi-deploy-svc.yaml
# aplicamos el servicio
kubectl apply -f extras/04-hola-mundo-real/deploy-05/mi-deploy-svc.yaml
# vemos los endpoints
kubectl get ep
# aplicamos el ingress 
kubectl apply -f extras/04-hola-mundo-real/deploy-05/mi-deploy-ingress.yaml
# vemos ingress
kubectl get ingress
# borramos ingress
kubectl delete ingress mi-app-ingress
```

## 05 - Kustomization

```
# ver codigo
cat extras/05-buenas-practicas/ejemplo-kustomize/kustomization.yaml
# instalar con kustomize
kubectl apply -k extras/05-buenas-practicas/ejemplo-kustomize
# ver servicios relacionados
kubectl get all -n mi-app-kustomize
# aplicar objetos manualmente
kubectl apply -f extras/05-buenas-practicas/ejemplo-kustomize/namespace.yaml
kubectl apply -f extras/05-buenas-practicas/ejemplo-kustomize/mi-deploy-ingress.yaml -n mi-app-kustomize
kubectl apply -f extras/05-buenas-practicas/ejemplo-kustomize/mi-deploy-svc.yaml -n mi-app-kustomize
kubectl apply -f extras/05-buenas-practicas/ejemplo-kustomize/mi-deploy.yaml -n mi-app-kustomize
```

## 05 - checkov

```bash
# Obtenemos la última imagen
docker pull bridgecrew/checkov
# Validamos el ejemplo de Kustomization de este capítulo
docker run --tty --volume ${PWD}/extras/05-buenas-practicas/ejemplo-kustomize:/yaml --workdir /yaml bridgecrew/checkov --directory /yaml
# Validamos el mismo ejemplo pero en formato compacto, sin el código
docker run --tty --volume ${PWD}/extras/05-buenas-practicas/ejemplo-kustomize:/yaml --workdir /yaml bridgecrew/checkov --directory /yaml --compact
```

## 05 - Aplicar resources al spec

```bash
# ver arreglo aplicado
diff ${PWD}/extras/05-buenas-practicas/ejemplo-kustomize/mi-deploy.yaml ${PWD}/extras/05-buenas-practicas/corregido-checkov/mi-deploy.yaml
# comprobar de vuelta
docker run --tty --volume ${PWD}/extras/05-buenas-practicas/corregido-checkov:/yaml --workdir /yaml bridgecrew/checkov --directory /yaml --compact --check CKV_K8S_11,CKV_K8S_10,CKV_K8S_12,CKV_K8S_13
```

## 05 - SecurityContext

```bash
docker run --tty --volume ${PWD}/extras/05-buenas-practicas/corregido-checkov-securityContext:/yaml --workdir /yaml bridgecrew/checkov --directory /yaml --check CKV_K8S_30
```

## 06 - Prometheus

```bash
# agregar repo de prometheus community
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
# obtener ultimas versiones de charts disponibles
helm repo update
# instalar usando valores por defecto
helm install kube-prometheus prometheus-community/kube-prometheus-stack
# este último paso demorará bastante ya que descarga el chart y lo instala
# comprobar instalacion
kubectl --namespace default get pods -l "app.kubernetes.io/instance=kube-prometheus"
```

## 06 - Consultas en Prometheus

```bash
# exponer servicio
kubectl port-forward svc/prometheus-operated 9090
```

```promql
# Obtener todos los pods
kube_pod_info
# Obtener sólo los pods del namespace default
kube_pod_info{namespace="default"}
# Obtener cantidad de pods por namespaces
sum(kube_pod_info{}) by (namespace)
# Primero busquemos que métricas usar. Una simple búsqueda a "replicas" nos da varios resultados, usemos la siguiente
kube_deployment_status_replicas_available
# Ahora bien, esto nos trae una gran lista de métricas con muchos datos, agrupemos las mismas por uno de sus labels, por ejemplo: deployment
sum(kube_deployment_status_replicas_available) by (deployment)
# Pero esto nos trae todos los deployment del cluster, y a nosotros sólo nos interesan los de un namespace llamado default
sum(kube_deployment_status_replicas_available{namespace="default"}) by (deployment)
# ¡Bien! ya tenemos algo útil, hacemos click en Graph para ver la cantidad de réplicas
# Ahora en la consola, ejecutemos: kubectl scale --replicas=5 -n default deployment/mi-deploy
# Si volvemos a la GUI de Prometheus inmediatamente, no veremos las métricas reflejadas aún
# Prometheus tiene un tiempo entre cada recolección de métricas, más el tiempo que tarda en procesarlas
# Si apretamos nuevamente en Execute, ya deberíamos ver que nuestro gráfico refleja que creció el número de métricas
```

## 06 - Grafana

```bash
# obtener clave
kubectl get secret kube-prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
# exportar servicio
kubectl port-forward svc/kube-prometheus-grafana 3000:80
```
