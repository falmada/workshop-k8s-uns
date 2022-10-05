# 03 - Kubernetes

- Instalación de Kubernetes en local
- Primera prueba de pod
- Nuestra app en un pod
- Exponer y probar nuestra aplicación
- Controladores

## Intro

Ya tenemos una imagen, pudimos correr un contenedor con ella y ahora queremos llevarla a Kubernetes, ¿cómo seguimos?

Kubernetes puede ser instalado de muchas formas, e incluso instanciado en servicios de las distintas nubes públicas como AWS Elastic Kubernetes Services (EKS), Azure Kubernetes Service (AKS), Google Kubernetes Engine (GKE), entre otros. Dado que los servicios en la nube cuestan dinero, podemos empezar con algo sencillo de forma local.

Poder hacer uso de Kubernetes de forma local en tu ordenador es sorprendente. Primero porque te acerca a la tecnología sin una barrera inicial alta (como sería el costo de usarlo), y segundo porque cada vez el proceso es más sencillo, al punto tal que con un comando podemos crear un cluster con varios nodos... todo en un solo ordenador.

## Instalacion de Kubernetes en local

Kubernetes tiene muchísimas opciones para instalar en local, de forma fácil y rápida. Si van a cualquier guía, seguramente se encontrarán con las opciones de microk8s, minikube, kind, k3d, k3s.

Para este ejemplo, haremos uso de k3d, una distribución provista por Rancher Labs, que básicamente hace uso de otra solución de ellos (k3s) y le agrega un wrapper de Docker por encima para crear nodos multi-cluster de una forma super sencilla.

> Del mismo modo que con las batallas de Windows/Linux, Mac/PC, River/Boca, cada cual es libre de elegir la distro que desee en base a sus necesidades y los recursos disponibles en su ordenador.

Para poder hacer uso de k3d, primero debemos instalarlo. Hay muchas formas de hacerlo, las dos más comunes son:

```sh
# Hacerlo con el script de instalación
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
# O bajarse el último release binario y ubicarlo en algún directorio dentro de nuestro $PATH
# Releases en: https://github.com/k3d-io/k3d/releases
```

Una vez que tenemos k3d, podemos explorar las opciones base que nos ofrece y los parámetros:

```sh
╰─ k3d
Usage:
  k3d [flags]
  k3d [command]

Available Commands:
  cluster      Manage cluster(s)
  completion   Generate completion scripts for [bash, zsh, fish, powershell | psh]
  config       Work with config file(s)
  help         Help about any command
  image        Handle container images.
  kubeconfig   Manage kubeconfig(s)
  node         Manage node(s)
  registry     Manage registry/registries
  version      Show k3d and default k3s version

Flags:
  -h, --help         help for k3d
      --timestamps   Enable Log timestamps
      --trace        Enable super verbose output (trace logging)
      --verbose      Enable verbose output (debug logging)
      --version      Show k3d and default k3s version

Use "k3d [command] --help" for more information about a command.

╰─ k3d version
k3d version v5.4.6
k3s version v1.24.4-k3s1 (default)
```

> Por defecto, k3d va a crear clusters en la última versión que soporta. Si queremos otra versión, debemos usar `--image rancher/k3s:TAG`, usando tags según el [repositorio de k3s](https://hub.docker.com/r/rancher/k3s/tags)

Podemos correr `k3d cluster create -h` para ver todos los parámetros de creación si nos pica la curiosidad, caso contrario procedan a crearlo con el siguiente comando:

```sh
k3d cluster create "mi-cluster" \
    --api-port 0.0.0.0:6443 \
    --network 10.100.0.0/16 \
    --servers 1 \
    --servers-memory 256Mi \
    --agents 3 \
    --agents-memory 1Gi \
    --registry-create mycluster-registry:0.0.0.0:5432
```

## Veamos nuestro mini cluster andando

Un ejemplo de lo que puede pasar (si todo va bien), esto demora aproximadamente 1 o 2 minutos.

```sh
╰─ k3d cluster create "mi-cluster" \
    --api-port 0.0.0.0:6443 \
    --network 10.100.0.0/16 \
    --servers 1 \
    --servers-memory 256Mi \
    --agents 3 \
    --agents-memory 1Gi \
    --registry-create mycluster-registry:0.0.0.0:5432
INFO[0000] Prep: Network                                
INFO[0000] Re-using existing network '10.100.0.0/16' (1636c41e7ed1e271585f9e007877fb23f83b8fa9906e85f71db6839ab68776ec) 
INFO[0000] Created image volume k3d-mi-cluster-images   
INFO[0000] Creating node 'mycluster-registry'           
INFO[0000] Successfully created registry 'mycluster-registry' 
INFO[0000] Starting new tools node...                   
INFO[0001] Creating node 'k3d-mi-cluster-server-0'      
INFO[0002] Pulling image 'ghcr.io/k3d-io/k3d-tools:5.4.6' 
INFO[0004] Pulling image 'docker.io/rancher/k3s:v1.24.4-k3s1' 
INFO[0006] Starting Node 'k3d-mi-cluster-tools'         
INFO[0034] Creating node 'k3d-mi-cluster-agent-0'       
INFO[0036] Creating node 'k3d-mi-cluster-agent-1'       
INFO[0038] Creating node 'k3d-mi-cluster-agent-2'       
INFO[0040] Creating LoadBalancer 'k3d-mi-cluster-serverlb' 
INFO[0042] Pulling image 'ghcr.io/k3d-io/k3d-proxy:5.4.6' 
INFO[0045] Using the k3d-tools node to gather environment information 
INFO[0045] HostIP: using network gateway 192.168.48.1 address 
INFO[0045] Starting cluster 'mi-cluster'                
INFO[0045] Starting servers...                          
INFO[0047] Starting Node 'k3d-mi-cluster-server-0'      
INFO[0056] Starting agents...                           
INFO[0059] Starting Node 'k3d-mi-cluster-agent-2'       
INFO[0059] Starting Node 'k3d-mi-cluster-agent-1'       
INFO[0059] Starting Node 'k3d-mi-cluster-agent-0'       
INFO[0068] Starting helpers...                          
INFO[0068] Starting Node 'mycluster-registry'           
INFO[0069] Starting Node 'k3d-mi-cluster-serverlb'      
INFO[0079] Injecting records for hostAliases (incl. host.k3d.internal) and for 7 network members into CoreDNS configmap... 
INFO[0102] Cluster 'mi-cluster' created successfully!   
INFO[0104] You can now use it like this:                
kubectl cluster-info
```

Lo primero que haremos es ver que pods se han creado

```sh
╰─ kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS              RESTARTS   AGE
kube-system   local-path-provisioner-7b7dc8d6f5-8svgb   1/1     Running             0          53s
kube-system   coredns-b96499967-wmsjp                   1/1     Running             0          53s
kube-system   helm-install-traefik-crd-7qfsr            0/1     Completed           0          53s
kube-system   traefik-7cd4fcff68-zl6vf                  0/1     ContainerCreating   0          10s
kube-system   svclb-traefik-965871f3-r8s6v              0/2     ContainerCreating   0          9s
kube-system   svclb-traefik-965871f3-j4tts              0/2     ContainerCreating   0          9s
kube-system   svclb-traefik-965871f3-rc25b              0/2     ContainerCreating   0          9s
kube-system   svclb-traefik-965871f3-tj2vw              0/2     ContainerCreating   0          9s
kube-system   metrics-server-668d979685-c6jlc           1/1     Running             0          53s
kube-system   helm-install-traefik-glxwx                0/1     Completed           1          53s
```

Tardarán un rato hasta ponerse en Running o Completed, dos de los muchos estados en los que puede terminar un pod.

## Primera prueba de pod

--deployar un nginx pelado y hacer luego portfoward para mostrarlo

## Nuestra app en un pod!

--crear pod con nuestra imagen
## Exponer y probar nuestra app


-- exponer por port forward
-- exponer como node port
-- explicar issue de usar nodeport
-- explicar load balancer, clusterip
-- explicar ingress

## Controladores

-- matar pod, que paso?! AUTO RECOVERY? -> Controladores! spec, template de pods
-- ejemplos de como se comportan, daemonset, sts, replica, deployment
-- replicas
-- matar pods y mostrar con varias replicas asi ven como se nombran
-- agregar resources al deployment, que pasa?

## Enlaces sugeridos

- [K3d](https://k3d.io)
- [Comparación de varias distro livianas de Kubernetes](https://www.linkedin.com/pulse/run-kubernetes-locally-minikube-microk8s-k3s-k3d-kind-sangode/?trk=portfolio_article-card_title)