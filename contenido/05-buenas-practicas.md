# 05 - Buenas prácticas

## Namespaces

> ¿Qué es un namespace? Es una división lógica dentro del mismo cluster que nos permite que múltiples usuarios trabajen sobre el mismo entorno, sin necesariamente interferir con los objetos creados por el otro. Dentro de un namespace, dos objetos del mismo tipo y nombre no podrían convivir, pero en distintos namespaces esto no sería un problema. El uso más usual de namespaces es para dividir el cluster en múltiples proyectos, aunque también se suele usar para dividir los entornos (dev, qa, prod, etc) de un mismo proyecto, o combinaciones a necesidad de los usuarios.

Si bien durante todo el taller hemos ido trabajando sin especificar un namespace, por defecto `default`, esto **no** es una buena práctica, dado que nuestras cargas de trabajo seguramente terminen ejecutándose al lado de otros usuarios que también hayan omitido este consejo.

Lo recomendable es trabajar sobre un namespace específico como, por ejemplo, `mi-app`, a fines de diferenciar dentro del mismo cluster nuestra carga de trabajo de la de otros.

## Kustomization

A medida que nuestra aplicación crece, los objetos que utilicemos también crecerán en cantidad y complejidad. Tendremos configuraciones por entorno para cada objeto, como también convivirán varias versiones de nuestra aplicación en sus distintos estados. Toda esta complejidad, a la larga, frustra.

Una de las opciones que ofrece Kubernetes es hacer uso de Kustomize. En su versión más simple, Kustomize, por medio de un archivo `kustomization.yaml`, nos permite desplegar una aplicación y todos sus objetos de forma fácil por medio de un sólo comando.

```bash
╰─ kubectl apply -k extras/05-buenas-practicas/ejemplo-kustomize

namespace/mi-app-kustomize configured
service/mi-deploy-svc created
deployment.apps/mi-deploy created
ingress.networking.k8s.io/mi-deploy-ingress created

╰─ kubectl get all -n mi-app-kustomize                          
NAME                            READY   STATUS              RESTARTS   AGE
pod/mi-deploy-f4cf76bd9-wffr9   0/1     ContainerCreating   0          5s
pod/mi-deploy-f4cf76bd9-t5z89   1/1     Running             0          5s
pod/mi-deploy-f4cf76bd9-55mq6   1/1     Running             0          5s
```

Una de los aspectos positivos de utilizar un archivo `kustomization.yaml`, es que podemos especificar el `namespace` que se le aplicará a todos los recursos, como también el orden de los objetos a crear. Hacer lo mismo de forma manual implicaría los siguientes pasos:

```bash
kubectl apply -f extras/05-buenas-practicas/ejemplo-kustomize/namespace.yaml
kubectl apply -f extras/05-buenas-practicas/ejemplo-kustomize/mi-deploy-ingress.yaml -n mi-app-kustomize
kubectl apply -f extras/05-buenas-practicas/ejemplo-kustomize/mi-deploy-svc.yaml -n mi-app-kustomize
kubectl apply -f extras/05-buenas-practicas/ejemplo-kustomize/mi-deploy.yaml -n mi-app-kustomize
```

Para poder documentar eso, lo tendríamos que agregar en un archivo de documentación en nuestro repositorio, y un cambio de nombre de namespace implicaría al menos 4 cambios versus 1 en el archivo Kustomization. Nuevamente, a mayor complejidad, se vuelve bastante útil.

## Herramientas

No hay que temer al uso de herramientas para ayudarnos a trabajar. En Kubernetes no son pocas, lo cual nos da un montón de posibilidades para facilitarnos la vida ya sea mientras codeamos nuevas soluciones o bien cuando queremos probarlas.

La idea no es hacer una revisión exhaustiva de herramientas, sino mostrarles algunas que son de digna mención, y queda en ustedes explorar más:

- [yamllint](https://yamllint.readthedocs.io/en/stable/)
  - Dado que Kubernetes, en su forma declarativa, hace uso de YAML, esta herramienta nos permite validar y hacer `linting` de nuestro código previo a commitearlo en un repositorio o en un `pipeline` de desarrollo.
- [krew](https://krew.sigs.k8s.io/)
  - Al usar `kubectl` nos daremos cuenta del potencial de expandirlo. Krew es una de las soluciones más simples para conseguirlo. Cuenta con muchos plugins que nos harán la vida mucho más fácil a la hora de administrar clusters y/o nuestras cargas de trabajo.
- [Visual Studio Code](https://code.visualstudio.com/)
  - Si bien es una herramienta para programar, dispone de muchísimos plugins oficiales y útiles relacionados a Docker, Kubernetes e incluso k3d
- [checkov](https://www.checkov.io/)
  - Permite escanear configuraciones erróneas y problemas potenciales de seguridad
- [Helm](https://helm.sh/)
  - Una de las herramientas más usadas para "empaquetar" aplicaciones en Kubernetes es Helm.
  - Para quienes desarrollan, permite generar un *skeleton* de un *Chart* para luego iterar sobre él y empaquetar nuestra aplicación.
  - Por medio del uso de *values* podemos personalizar una aplicación y ejecutarla en distintos *releases* con distintas configuraciones, dentro de uno o varios clusters.

### Mención especial - checkov

**checkov** es una herramienta muy útil si ya estamos trabajando sobre Kubernetes y nos interesa ir un paso más allá aplicando buenas prácticas. En lo positivo, tiene una imágen de Docker que nos permite escanear siempre con la última versión e incorporar esto en nuestro *pipeline* de integración continua para poder asegurarnos que siempre salimos con una buena base.

Veamos un ejemplo:

```bash
# Obtenemos la última imagen
docker pull bridgecrew/checkov
# Validamos el ejemplo de Kustomization de este capítulo
docker run --tty --volume ${PWD}/extras/05-buenas-practicas/ejemplo-kustomize:/yaml --workdir /yaml bridgecrew/checkov --directory /yaml
# Validamos el mismo ejemplo pero en formato compacto, sin el código
docker run --tty --volume ${PWD}/extras/05-buenas-practicas/ejemplo-kustomize:/yaml --workdir /yaml bridgecrew/checkov --directory /yaml --compact
```

Otros flags que podemos aplicar son:

- `--output csv` para exportar los errores en formato de CSV, lo cual puede ser luego consumido por otra herramienta
- `--quiet` para evitar mostrar las validaciones que salieron bien
- `--check CKV_123` para solamente chequear una validación específica (ideal cuando estamos resolviendo cada ítem)
- `--check CRITICAL` para enfocar nuestra validación en sólo aquellos ítems críticos (puede ser LOW, MEDIUM, HIGH también).
- `--skip-check CKV_123` para evitar chequear un ítem específico que sabemos que no podemos corregir o lo reporta erróneamente
- `--skip-check LOW` para evitar chequeos que no representan un problema grave (puede ser MEDIUM, HIGH, CRITICAL también)
- `--help` para ver todas las opciones disponibles según la versión que estemos usando

## Aplicar resources al spec

Ya que *checkov* nos reportó algunos errores relacionados con los *resources*, vamos a enfocar nuestras mejoras en eso.

```bash
╰─ docker run --tty --volume ${PWD}/extras/05-buenas-practicas/ejemplo-kustomize:/yaml --workdir /yaml bridgecrew/checkov --directory /yaml --compact --check CKV_K8S_11,CKV_K8S_10,CKV_K8S_12,CKV_K8S_13 

       _               _              
   ___| |__   ___  ___| | _______   __
  / __| '_ \ / _ \/ __| |/ / _ \ \ / /
 | (__| | | |  __/ (__|   < (_) \ V / 
  \___|_| |_|\___|\___|_|\_\___/ \_/  
                                      
By bridgecrew.io | version: 2.1.270 

kubernetes scan results:

Passed checks: 0, Failed checks: 4, Skipped checks: 0

Check: CKV_K8S_11: "CPU limits should be set"
	FAILED for resource: Deployment.default.mi-deploy
	File: /mi-deploy.yaml:1-26
	Guide: https://docs.bridgecrew.io/docs/bc_k8s_10
Check: CKV_K8S_10: "CPU requests should be set"
	FAILED for resource: Deployment.default.mi-deploy
	File: /mi-deploy.yaml:1-26
	Guide: https://docs.bridgecrew.io/docs/bc_k8s_9
Check: CKV_K8S_12: "Memory requests should be set"
	FAILED for resource: Deployment.default.mi-deploy
	File: /mi-deploy.yaml:1-26
	Guide: https://docs.bridgecrew.io/docs/bc_k8s_11
Check: CKV_K8S_13: "Memory limits should be set"
	FAILED for resource: Deployment.default.mi-deploy
	File: /mi-deploy.yaml:1-26
	Guide: https://docs.bridgecrew.io/docs/bc_k8s_12
```

Como notarán, han fallado todas las validaciones. En cada ítem también se ve que nos sugiere enlaces para corregir los ítems (algunos con ejemplos, otros con información enlazada de otros sitios).

Corregimos los ítems, agregando un bloque de resources a `mi-deploy.yaml`:

```bash
╰─ diff ${PWD}/extras/05-buenas-practicas/ejemplo-kustomize/mi-deploy.yaml ${PWD}/extras/05-buenas-practicas/corregido-checkov/mi-deploy.yaml 
25c25,31
<         resources: {}
---
>         resources:
>           limits:
>             cpu: 500
>             memory: 512Mb
>           requests:
>             cpu: 100
>             memory: 256Mb
```

Y luego procedemos a verificar de vuelta:

```bash
╰─ docker run --tty --volume ${PWD}/extras/05-buenas-practicas/corregido-checkov:/yaml --workdir /yaml bridgecrew/checkov --directory /yaml --compact --check CKV_K8S_11,CKV_K8S_10,CKV_K8S_12,CKV_K8S_13


       _               _              
   ___| |__   ___  ___| | _______   __
  / __| '_ \ / _ \/ __| |/ / _ \ \ / /
 | (__| | | |  __/ (__|   < (_) \ V / 
  \___|_| |_|\___|\___|_|\_\___/ \_/  
                                      
By bridgecrew.io | version: 2.1.270 

kubernetes scan results:

Passed checks: 4, Failed checks: 0, Skipped checks: 0

Check: CKV_K8S_11: "CPU limits should be set"
	PASSED for resource: Deployment.default.mi-deploy
	File: /mi-deploy.yaml:1-32
	Guide: https://docs.bridgecrew.io/docs/bc_k8s_10
Check: CKV_K8S_10: "CPU requests should be set"
	PASSED for resource: Deployment.default.mi-deploy
	File: /mi-deploy.yaml:1-32
	Guide: https://docs.bridgecrew.io/docs/bc_k8s_9
Check: CKV_K8S_12: "Memory requests should be set"
	PASSED for resource: Deployment.default.mi-deploy
	File: /mi-deploy.yaml:1-32
	Guide: https://docs.bridgecrew.io/docs/bc_k8s_11
Check: CKV_K8S_13: "Memory limits should be set"
	PASSED for resource: Deployment.default.mi-deploy
	File: /mi-deploy.yaml:1-32
	Guide: https://docs.bridgecrew.io/docs/bc_k8s_12
```

## SecurityContext

Otro de los ítems que podemos resaltar es el uso (o falta de) `securityContext` en la definición del deployment (`CKV_K8S_30`)

```bash
Check: CKV_K8S_30: "Apply security context to your pods and containers"
	FAILED for resource: Deployment.default.mi-deploy
	File: /mi-deploy.yaml:1-26
	Guide: https://docs.bridgecrew.io/docs/bc_k8s_28
```

Si vamos a la [documentación de checkov](https://docs.bridgecrew.io/docs/bc_k8s_28) vemos que agregando el bloque de `spec.containers.securityContext` ya debería funcionar, pero no es así. En algunos casos debemos ir a la [documentación oficial](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) de Kubernetes para obtener más información.

Pueden usar el siguiente comando para validar el arreglo que hemos hecho al deployment:
```
docker run --tty --volume ${PWD}/extras/05-buenas-practicas/corregido-checkov-securityContext:/yaml --workdir /yaml bridgecrew/checkov --directory /yaml --check CKV_K8S_30
```

## Optimizar imagen

Si bien esto no es exclusivo de Kubernetes, resulta importante mencionarlo, ya que puede ser la diferencia entre una buena y una mala experiencia cuando un *pod* salta entre nodos. Dado que Kubernetes hace uso de varios nodos, si nuestra aplicación está ejecutando en los nodos 1, 2 y 3, y alguno de estos nodos se cae, el *pod* morirá y de fondo el controlador de *replicaSet* intentará *schedulear* el *pod* en otro nodo. Al hacer esto, se produce inicialmente un *pull* de la imagen que usará el contenedor, por lo que si dicha imagen es muy pesada, difícilmente nuestra aplicación se recupere rápido de esta falla.

Otro beneficio, ajeno a Kubernetes pero que afecta el desarrollo, es que cuando *pusheamos* una imagen a un *registry*, los *layers* que ya existen, no son subidos de vuelta. Si organizamos bien nuestra imagen, es probable que ni nos demos cuenta cada vez que la actualizamos, dado que no tardará mucho. Esto redunda en que si a futuro la imagen se actualiza en el *deployment*, Kubernetes solo descargará los *layers que le faltan*.

No es motivo de este Workshop optimizar las imágenes, pero les dejamos unos enlaces al final para que lean un poco más si están interesados.

### Alpine

Otra mención especial, y es por motivo de un consejo que veremos a menudo en grupos de discusión relacionados al desarrollo de imágenes de contenedor. Alpine es una distribución de Linux, tal como Ubuntu y Red Hat, con la diferencia de que es liviana... muy liviana.

A día de hoy, si nos descargamos la versión `latest` que actualmente corresponde a [3.16.2](https://hub.docker.com/layers/library/alpine/3.16.2/images/sha256-1304f174557314a7ed9eddb4eab12fed12cb0cd9809e4c28f29af86979a3c870?context=explore), su peso es de tan sólo 2.68 MB. Si comparamos con la última imágen de Ubuntu actualmente publicada, [jammy-2022100](https://hub.docker.com/layers/library/ubuntu/jammy-20221003/images/sha256-a8fe6fd30333dc60fc5306982a7c51385c2091af1e0ee887166b40a905691fd0?context=explore)3, esta pesa 29.02 MB... unas 10 veces más pesada.

Nuestras imagenes difícilmente sean tan livianas, ya que luego debemos agregarle las librerías y paquetes necesarias para correr nuestra aplicación, pero no es lo mismo empezar con una imagen de 2.68 MB que con una de casi 30.

Alpine, por su parte, hace uso de `apk` para la instalación de paquetes y por defecto trae muy poco instalado, lo cual requerirá un buen conocimiento de que necesita nuestra aplicación para poder hacer uso de esta distribución.

## Startup, Liveness y Readiness Probes

Esto si es de Kubernetes y es crítico para que nuestra aplicación se comporte y conviva dentro del *cluster*, informando de su estado de una forma clara para que la orquestración funcione correctamente.

Kubernetes cuenta con la posibilidad de agregar *probes* (chequeos de estados) tanto para el inicio de una aplicación, para cuando el *pod* está corriendo, y para cuando el *pod* está listo para ejecutar la tarea para la cual está creado.

### Startup probe

Dado que Kubernetes espera inicializar los *pods* de forma rápida, las aplicaciones *legacy* que sean migradas a esta plataforma pueden sufrir problemas debido a su lentitud al iniciar. El `startupProbe` permite setear `failureThreshold * periodSeconds` para fijar un tiempo de inicio del contenedor lo suficientemente largo para cubrir el peor caso posible.

Detalle al margen, hasta que el `startupProbe` no complete, se deshabilitan los otros dos *probe* en caso de existir.

### Liveness probe

El `livenessProbe` está pensado para recuperar aquellas aplicaciones que eventualmente pueden caer en un estado incorrecto y la única forma de corregirlas es por medio de un *restart*. Dicho de otro modo, el *kubelet* usará esta regla para reiniciar un contenedor si detecta que la condición no se cumple.

De no fijar un `livenessProbe`, estamos a la suerte de que alguien se de cuenta que la aplicación no está respondiendo, para corregirlo. Si consideramos las réplicas como parte del problema, nos daremos cuenta que quizás un *pod* de varios quede corrupto y toda la carga termine en cada vez menos *pods* hasta llegar a la situación crítica de no poder ejecutar la aplicación debido a que todos están afectados. Estos casos son los peores, porque el rendimiento se va lentamente empeorando y es muy complicado medir eso con métricas que puedan alertarnos de forma proactiva.

### Readiness probe

Finalmente, el *readinessProbe* sirve para saber cuando un contenedor está listo para aceptar tráfico. Dado que los servicios de Kubernetes hacen uso de *endpoints* para saber que *pods* están listos para recibir tráfico, necesitamos de alguna forma para saber cuando se cumple dicha condición. Cuando un *pod* se cae, y la condición falla, el servicio de Kubernetes dejará de listarlo como posible objetivo en las conexiones entrantes, evitando un mal servicio al cliente que accede.

## Enlace sugeridos

- [Namespaces](https://kubernetes.io/es/docs/concepts/overview/working-with-objects/namespaces/)
- [Kustomization](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Kustomization file](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/)
- [Kubetools](https://collabnix.github.io/kubetools/)
- [https://vsupalov.com/docker-image-layers/](https://vsupalov.com/docker-image-layers/)
- [What's Up With Multi-Stage Builds?](https://vsupalov.com/multi-stage-builds/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
