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
    --port "30000-30100:30000-30100@server:0" \
    --registry-create mycluster-registry:0.0.0.0:5432
INFO[0000] Prep: Network                                
INFO[0000] Re-using existing network '10.100.0.0/16' (1636c41e7ed1e271585f9e007877fb23f83b8fa9906e85f71db6839ab68776ec) 
INFO[0000] Created image volume k3d-mi-cluster-images   
INFO[0000] Creating node 'mycluster-registry'           
INFO[0000] Successfully created registry 'mycluster-registry' 
INFO[0000] Starting new tools node...                   
INFO[0000] Starting Node 'k3d-mi-cluster-tools'         
INFO[0001] Creating node 'k3d-mi-cluster-server-0'      
INFO[0003] Creating node 'k3d-mi-cluster-agent-0'       
INFO[0006] Creating node 'k3d-mi-cluster-agent-1'       
INFO[0009] Creating node 'k3d-mi-cluster-agent-2'       
INFO[0012] Creating LoadBalancer 'k3d-mi-cluster-serverlb' 
INFO[0012] Using the k3d-tools node to gather environment information 
INFO[0012] HostIP: using network gateway 192.168.48.1 address 
INFO[0012] Starting cluster 'mi-cluster'                
INFO[0012] Starting servers...                          
INFO[0014] Starting Node 'k3d-mi-cluster-server-0'      
INFO[0022] Starting agents...                           
INFO[0025] Starting Node 'k3d-mi-cluster-agent-1'       
INFO[0025] Starting Node 'k3d-mi-cluster-agent-2'       
INFO[0025] Starting Node 'k3d-mi-cluster-agent-0'       
INFO[0031] Starting helpers...                          
INFO[0032] Starting Node 'mycluster-registry'           
INFO[0033] Starting Node 'k3d-mi-cluster-serverlb'      
INFO[0045] Injecting records for hostAliases (incl. host.k3d.internal) and for 6 network members into CoreDNS configmap... 
INFO[0057] Cluster 'mi-cluster' created successfully!   
INFO[0060] You can now use it like this:                
kubectl cluster-info
```

> Si intentan mapear el rango completo de NodePorts (30000-32767) puede que falle Docker, más info en los [enlaces sugeridos](#enlaces-sugeridos).

Lo primero que haremos es ver que pods se han creado.

> ¿Qué es un pod?
> Es la unidad más pequeña de Kubernetes y representa a uno o más contenedores que comparten almacenamiento y red entre ellos.
> Un pod modela un *host lógico*, lo cual desde dentro del pod se representa como un host individual.
> Todos los contenedores dentro de un pod son coubicados, coprogramados y ejecutados en un contexto compartido.

```sh
╰─ kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   local-path-provisioner-7b7dc8d6f5-ww8sn   1/1     Running     0          89s
kube-system   coredns-b96499967-sfbxz                   1/1     Running     0          89s
kube-system   helm-install-traefik-crd-hd288            0/1     Completed   0          90s
kube-system   helm-install-traefik-4hnhr                0/1     Completed   1          90s
kube-system   metrics-server-668d979685-xllxq           1/1     Running     0          89s
kube-system   svclb-traefik-51682730-5ff6c              2/2     Running     0          50s
kube-system   svclb-traefik-51682730-7rj7f              2/2     Running     0          50s
kube-system   svclb-traefik-51682730-vxftk              2/2     Running     0          50s
kube-system   svclb-traefik-51682730-smxvc              2/2     Running     0          50s
kube-system   traefik-7cd4fcff68-tbrn7                  1/1     Running     0          50s
```

Tardarán un rato hasta ponerse en Running o Completed, dos de los muchos estados en los que puede terminar un pod.

## Primera prueba de pod

```bash
# Creamos un pod "mi-pod" usando la imagen de "nginx", un conocido web server
╰─ kubectl run mi-pod --image=nginx --restart=Never
pod/mi-pod created
# Luego procedemos a conectarnos a ella por shell y ejecutar algunos comandos
╰─ kubectl exec -ti mi-pod -- /bin/bash
# En este momento estamos dentro del contenedor principal del pod
root@mi-pod:/# hostname
mi-pod
root@mi-pod:/# uptime
bash: uptime: command not found
root@mi-pod:/# ps 
bash: ps: command not found
# Vemos que el root luce muy parecido a cualquier Linux
root@mi-pod:/# ls /    
bin  boot  dev  docker-entrypoint.d  docker-entrypoint.sh  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
# Si revisamos los filesystems montados, vemos algunos detalles interesantes
root@mi-pod:/# df     
Filesystem     1K-blocks      Used Available Use% Mounted on
overlay        244506940 206016964  25996920  89% /
tmpfs              65536         0     65536   0% /dev
tmpfs            8011508         0   8011508   0% /sys/fs/cgroup
/dev/nvme0n1p2 244506940 206016964  25996920  89% /etc/hosts
shm                65536         0     65536   0% /dev/shm
tmpfs            1073744        12   1073732   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs            8011508         0   8011508   0% /proc/acpi
tmpfs            8011508         0   8011508   0% /proc/scsi
tmpfs            8011508         0   8011508   0% /sys/firmware
# Y si buscamos aquellos puntos de montaje con el flag remount-ro, nos damos con otros detalles
root@mi-pod:/# mount | grep remount-ro
/dev/nvme0n1p2 on /etc/hosts type ext4 (rw,relatime,errors=remount-ro)
/dev/nvme0n1p2 on /dev/termination-log type ext4 (rw,relatime,errors=remount-ro)
/dev/nvme0n1p2 on /etc/hostname type ext4 (rw,relatime,errors=remount-ro)
/dev/nvme0n1p2 on /etc/resolv.conf type ext4 (rw,relatime,errors=remount-ro)
root@mi-pod:/# exit
```

Como verán, una vez dentro del pod, el host se reconoce a si mismo como "mi-pod", igual que el nombre de nuestro pod. Por otro lado, varios comandos fallan, porque en la imagen no fueron incluidos (más adelante explicaremos los motivos). Si corremos un `df`, veremos varias cosas intersantes como que el espacio que refleja el `/` es el mismo que nuestro ordenador, y que cuenta con un `/etc/hosts` montado, que está por encima del archivo real de nuestro host. Si observamos los puntos de montaje, vemos otras cosas que se montan por encima para permitir que el contenedor corra como un host lógico.

El siguiente paso es exponer nuestra aplicación de forma fácil, con un `port-forward`.

```bash
# Con 8080 especificamos el puerto local, y 80 el puerto del pod al que conectamos
╰─ kubectl port-forward mi-pod 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
# Una vez probado el acceso via browser, podemos matar el proceso con CTRL+C
```

Abrimos <http://localhost:8080> en nuestro ordenador y veremos un NGINX funcionando.

## Nuestra app en un Pod

Ya que tenemos nuestro propia imágen de contenedor, vamos a ponerla a prueba.

Podemos hacerlo del mismo modo que con la imagen de nginx, o podemos ir un poco más allá y hacerlo de forma declarativa, por medio del lenguaje YAML. Para empezar, usaremos `kubectl` para que nos de una mano.

```bash
# Con dry-run=client, simulamos una acción, y con -o yaml exportamos la salida a ese formato
kubectl run mi-pod --image=fedek3/workshop-uns:latest --restart=Never --dry-run=client -o yaml > mi-pod.yaml
# Esto nos genera el archivo: mi-pod.yaml
```

El contenido de este YAML nos da una base como para empezar a tener nuesta aplicación de forma declarativa en Kubernetes. De este modo, podemos iterar sobre el YAML para ir mejorando nuestro despliegue.

Nuestro siguiente paso es probar dicho pod.

```bash
# Al ser un YAML/declarativo, con apply -f podemos aplicar el manifiesto
kubectl apply -f extras/03-kubernetes/deploy-01/mi-pod.yaml
# Luego podemos ver el estado del pod
kubectl get pod my-pod
```

Esto significa que ¡ya tenemos nuestra aplicación corriendo en Kubernetes! y lo mejor aún, ahora tenemos una definición de dicha definición.

## Entendiendo mi-pod.yaml

Cuando corrimos el comando con `--dry-run=client`, se nos generó un YAML con el siguiente contenido.

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: mi-pod
  name: mi-pod
spec:
  containers:
  - image: fedek3/workshop-uns:latest
    name: mi-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

En la primera sección, se le dice al API server de Kubernetes, que versión y tipo de recurso usar:

```yaml
apiVersion: v1
kind: Pod
```

Al decir `v1`, y no `nombre_api/version`, esto nos indica que el recurso del tipo `Pod` que estamos creando, es una definición base en Kubernetes.

> Pueden obtener el listado de `apiVersion` y `Kind` reconocido por el cluster actual usando el comando `kubectl api-resources`.
> Tengan en cuenta que tanto las API y los tipos de objetos pueden cambiar con el tiempo (nuevas versiones, tipos deprecados).
> Otro tipo de objetos llamado `CRD` o `Custom Resource Definition` también pueden aparecer en esta lista.

Siguiendo con el YAML, tenemos la `metadata` del pod. Esta información es usada por Kubernetes durante la creación y tiene, entre sus tantos atributos, la posibilidad de fijar el nombre del recurso (`name`), las etiquetas asociadas a él (`labels`), entre otros.

A lo último del YAML, tenemos el `status: {}`, que como veremos, está vacío. Este atributo del spec no es necesario declararlo, ya que Kubernetes lo agregará una vez que el objeto haya sido instanciado, y usará dicha información para exponer el estado de un objeto.

En el medio, contamos con el `spec` del objeto, en este caso, de un `pod`:

```yaml
spec:
  containers:
  - image: fedek3/workshop-uns:latest
    name: mi-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
```

Lo más simple a la vista es reconocer el uso de nuestra imagen dentro de `containers`. En ese mismo array, tenemos la declaración del nombre del contenedor y los recursos (`resources`), los cuales de momento están vacíos.

Más abajo nos encontramos con `dnsPolicy`, lo cual nos permite a nivel de pod definir cómo se comportará una petición de resolución de nombre, y `restartPolicy`, que nos permite controlar que pasa cuando el pod falla.

## Exponer y probar nuestra app

De la imsma forma que antes, vamos a exponer nuestra aplicación usando `port-forward`, lo cual es bastante sencillo de lograr.

```bash
kubectl port-forward mi-pod 8080:80
```

Navegamos hacia <http://localhost:8080/> y veremos a nuestra aplicación corriendo sobre Kubernetes y exponiendo el puerto del servidor web.

## Formas de exponer servicios

Para un cluster productivo, no podremos hacer uso de `port-forward`, ¡ni hablar si queremos mostrarle lo que logramos a nuestros amigos!

Kubernetes cuenta con varias formas de exponer las cargas de trabajo que corre, entre las que nos encontramos con:

- Servicios de tipo NodePort
- Servicios de tipo LoadBalancer
- Objetos de tipo Ingress

### Servicios de tipo NodePort

Comencemos con la opción más sencilla y disponible en cualquier cluster. El `NodePort` permite hacer uso de un puerto de numeración alta, para exponer servicios en cualquier otro puerto a nivel de pod.

Podemos usar nuevamente el comando `kubectl create` con `--dry-run=client -o yaml` para producir un YAML válido que nos permite iterar. Del mismo modo que cuando creamos el *pod*.

```bash
# Dado que cuando creamos el cluster en k3d, especificamos un rango de 30000 a 30100, vamos a usar el más alto
# Los servicios de NodePort asignan un puerto al azar entre 30000 y 32767, por lo que hay altas chances de que 
# no esté dentro de nuestro rango a menos que lo especifiquemos
kubectl create service nodeport mi-pod --tcp=80 --node-port=30100 --dry-run=client -o yaml > svc-my-pod.yaml
```

Paso siguiente, aplicamos ese manifiesto en Kubernetes

```bash
╰─ kubectl apply -f extras/03-kubernetes/deploy-02/svc-my-pod.yaml
╰─ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.43.0.1      <none>        443/TCP        8m34s
mi-pod       NodePort    10.43.99.120   <none>        80:30100/TCP   4m47s
```

Siguiente paso, probamos en nuestro navegador apuntando a ese puerto <http://localhost:30100/>

¿Que pasó?

En la definición de un servicio, es necesario especificar a quien vamos a apuntar con el servicio. Para esto, Kubernetes utiliza **labels** (etiquetas) y **selectors**. Del lado del recurso a identificar, aplicaremos **labels** a los elementos (en este caso, un pod) y del lado del objeto que quiere asociarse, el servicio, un selector.

Una forma de ver que un servicio no está apuntando a donde debe, es simplemente revisar los endpoints del mismo. Veamos en nuestro caso:

```bash
╰─ kubectl get endpoints
NAME         ENDPOINTS           AGE
kubernetes   192.168.48.2:6443   16m
mi-pod       <none>              12m
```

Aquí podemos ver dos servicios, uno `kubernetes` que se crea por defecto y apunta al API server de un cluster, mientras que también tenemos `mi-pod` que apunta a nuestro servicio generado. Si vemos en la columna de `endpoints`, está vacía, por lo cual no hay forma que el servicio llegue a destino.

Ahora bien, si vemos la [definición de nuestro pod](../extras/03-kubernetes/deploy-01/mi-pod.yaml), éste ya tiene un label definido (`run: mi-pod`), mientras que si vemos la [definición de nuestro servicio](../extras/03-kubernetes/deploy-02/svc-my-pod.yaml) vemos que el `selector` apunta a `app: mi-pod`. El uso del label `app` es una buena práctica, por lo que actualizaremos el código de nuestro pod para que refleje el label correcto.

> El *debugging* en Kubernetes es complicado al comienzo. Hay muchas aristas a considerar, y a veces puede ser frustrante. En este mismo taller, más sobre el final, daremos un par de tips sobre cómo hacer *troubleshooting* de una forma genérica que te servirá para identificar casi cualquier problema.

Una vez que actualizamos los `labels` del pod a `app: mi-pod`, procedemos a aplicar el nuevo manifiesto. Si revisamos nuevamente los `endpoints`, nos damos con que ahora ya tiene un ip y puerto:

```bash
╰─ kubectl get ep 
NAME         ENDPOINTS           AGE
kubernetes   192.168.48.2:6443   22m
mi-pod       10.42.1.6:80        19m
```

Consecuentemente, si revisamos que ip tiene nuestro pod, nos daremos cuenta que el selector está funcionando correctamente.
```bash
╰─ kubectl get pod mi-pod -o wide
NAME     READY   STATUS    RESTARTS   AGE   IP          NODE                     NOMINATED NODE   READINESS GATES
mi-pod   1/1     Running   0          21m   10.42.1.6   k3d-mi-cluster-agent-0   <none>           <none>
```

> El uso de `-o wide` en `kubectl` es bien útil para obtener más información sobre un objecto. Por simplicidad, la mayoría de los comandos muestran muy poca información por defecto, pero si queremos saber más, siempre podemos usar este parámetro o hacer uso de `kubectl describe` que brinda aún más información!

Hagamos nuevamente la prueba de conectarnos al servicio por medio del NodePort visitando <http://localhost:30100/>

Si bien suena mágico que por medio de `NodePort` logremos exponer nuestra aplicación en unas pocas líneas de código, hay que ser realista y pensar en un entorno productivo. Hay muchas limitaciones para el uso de NodePort si nuestra aplicación vive en Internet, ¿acaso alguna vez visitaron google tipeando http://www.google.com:443? imagínense usando en cambio http://www.mi-app.com:30100! Más de uno desconfiaría y directamente ni visitaría nuestra aplicación. Por otro lado, hay un rango de 2768 puertos disponibles por defecto, y cada puerto implica una mayor complejidad en nuestra red para administrarlo (con posibles colisiones).

> Para laboratorios y/o redes internas, el uso de NodePort es más que suficiente. En Producción, si no tenemos otra alternativa que el uso de NodePort, siempre podemos poner un *Application Load Balancer* enfrente que se haga cargo de exponer un puerto más estándar, aunque debemos considerar las otras limitaciones de los NodePort.

### Servicios de tipo LoadBalancer

#TODO explicar issue de usar nodeport
#TODO explicar load balancer, clusterip
#TODO explicar ingress

## Controladores

#TODO matar pod, que paso?! AUTO RECOVERY? -> Controladores! spec, template de pods
#TODO ejemplos de como se comportan, daemonset, sts, replica, deployment
#TODO replicas
#TODO matar pods y mostrar con varias replicas asi ven como se nombran
#TODO agregar resources al deployment, que pasa?

## Enlaces sugeridos

- [K3d](https://k3d.io)
- [Comparación de varias distro livianas de Kubernetes](https://www.linkedin.com/pulse/run-kubernetes-locally-minikube-microk8s-k3s-k3d-kind-sangode/?trk=portfolio_article-card_title)
- [Pods](https://kubernetes.io/es/docs/concepts/workloads/pods/pod/)
- [Controladores](https://kubernetes.io/es/docs/concepts/workloads/controllers/)
- [Servicios](https://kubernetes.io/es/docs/concepts/services-networking/service/)
- [Exposing services k3d](https://k3d.io/v5.4.6/usage/exposing_services/)
- [When mapping too many ports then daemon failed #22614](https://github.com/moby/moby/issues/22614)