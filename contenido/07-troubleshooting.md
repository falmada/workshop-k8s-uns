# 07 - Resolución de problemas

En toda tecnología es necesario contar con algún manual que nos permita resolver los problemas más frecuentes, como también entender las distintas formas de obtener información para poder solucionar aquellos problemas no tan comunes. Kubernetes, debido a su complejidad, hace esto mucho más sencillo gracias a opciones de su herramienta de cliente nativa `kubectl`, lo cual facilita enormemente la tarea. Existen varias guías (ver [Enlaces sugeridos](#enlaces-sugeridos)) que nos orientan en la resolución de problemas.

## Ejemplos de situaciones comunes

### CrashLoopBackOff

Si han visto memes de Kubernetes, el *CrashLoopBackOff* es muy usado como contenido debido a que es un error muy común que nos encontraremos cuando empezamos a trabajar en esta tecnología.

Cuando un *pod* entra en un estado de *CrashLoopBackOff*, este mismo inicia sus contenedores, pero en determinado momento se reinicia, no pudiendo llegar a un estado de *Running*. Este estado puede aparecer por múltiples motivos, el más usual estando relacionado a errores de configuración o haciendo uso de algún *flag* no soportado al momento de inicializar uno de los contenedores del *pod*.

La forma más rápida de comprender cuál es el problema es haciendo uso de `kubectl describe pod nombre_pod`, aunque a veces también `kubectl logs nombre_pod` y `kubectl get events` pueden darnos algunas pistas.

Si estamos haciendo uso de Prometheus, las siguientes expresiones pueden ayudarnos a encontrar estos problemas:

- `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1`
- `rate(kube_pod_container_status_restarts_total[5m]) > 0`

### ImagePullBackOff

Otro error muy común es cuando estamos intentando hacer uso de una imágen, o de un *tag* de imágen, que no está disponible. Esto usualmente se debe a un *typo* al escribir estos valores, o bien cuando intentamos accedr a una imagen en un registro que no está público o al cual no tenemos los permisos necesarios para hacer un *pull*.

Lo ideal en estos casos es revisar la definición de nuestro *deployment* con `kubectl describe` y a su vez ver los eventos con `kubectl get events`. En este caso, y en los casos en donde el *pod* sólo tenga un contenedor, los *logs* no estarán disponibles dado que nunca llegó a ejecutarse.

### Pending

Cuando un *pod* entra en estado *Pending* es porque está esperando que algo suceda. Este error no es muy común pero puede ser un poco frustrante de resolver debido a que muchas veces no está relacionado a nuestro despliegue sino más bien a recursos que consumimos.

Haciendo uso de `kubectl describe` y `kubectl get events` podemos tener una idea general de que recurso nos está frenando el despliegue. En los casos que un *pod* no pueda ser asignado a un nodo por el *scheduler*, la resolución será complicada dado que tendremos que entender si no hay capacidad suficiente o más bien hay un problema a nivel cluster que debe ser atendido previamente. En otros casos, puede darse por un problema de acceso al almacenamiento, lo cual puede ser relacionado a capacidad o a un problema de configuración al solicitar un *persistent volume claim*.

### OOMKill

Este error no es propio de Kubernetes, sino más bien que es heredado del sistema operativo. OOM significa *Out of Memory*, y el evento de *OOMKill* es muy común cuando nuestro *pod* decide consumir más memoria que la que tiene asignado.

El error en estos casos puede darse porque nuestra aplicación simplemente está teniendo un problema de manejo de memoria y ha llegado al límite establecido en la especificación de *resources* de nuestro *template* de *pod*, o simplemente el límite está muy bajo y está provocando un problema cuando simplemente debería tener permitido más memoria.

Hay que tener en cuenta que el total de *resources.limits* que asignemos en nuestros *pods*, será comparado con la *quota* que tengamos asignada en el namespace. Esta última puede verse afectada según el tipo de despliegue que hagamos, ya que si bien podemos tener asignado 10 unidades de memoria y 10 pods usando una unidad por cada uno, al momento de hacer un despliegue con un pod extra, estaremos solicitando 11 unidades de memoria cuando nuestra *quota* es de máximo 10.

### Readiness

Hablamos sobre el *Readiness* en el tema de las [buenas prácticas](05-buenas-practicas.md). Si un *pod* no logra validar el *readinessProbe* satisfactoriamente, fallará en fijarse como Ready, lo cual implica que si hay algún servicio que lo tiene como objetivo en el *selector*, no podrá hacer uso de este *pod* en dicho estado. Si ningún *pod* está disponible, el servicio fallará.

Entender cómo fijar un buen *readiness* no es fácil, requiere de comprender bien el momento en que nuestra aplicación está lista para recibir peticiones. Esta validación puede ser tan simple como ver si nuestra aplicación está ejecutando, o ser compleja al punto de requerir validar una conexión a una base de datos o un servicio externo que nuestra aplicación necesita para funcionar.

### Terminating

Eliminamos un *pod*, esperamos... esperamos... y esperamos. El *pod* sigue en estado *Terminating*, ¿pero que pasó?

Si hacemos uso de `kubectl get pod nombre_pod -o yaml` podemos ver una parte de la especificación llamada *finalizers*. En este atributo, podemos ver específicamente que está esperando el *pod* antes de declararse como muerto. En algunos casos, esto puede estar relacionado al almacenamiento, aunque algunas veces veremos que simplemente Kubernetes no considera seguro finalizar el *pod* porque algún recurso aún sigue vivo, y espera que nosotros tomemos esa decisión (correctiva, o simplemente forzada) para cumplir con la tarea.

## kubectl explain

Si bien la opción `explain` de `kubectl` no es explícitamente para resolución de problemas, esta nos puede ayudar a entender más sobre una mala definición de un YAML al momento de aplicarlo al cluster.

Si estamos creando un pod, por ejemplo, podríamos correr `kubectl explain pod` para entender los campos que utiliza y para que son.

```bash
╰─ kubectl explain pod
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion	<string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind	<string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata	<Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec	<Object>
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

   status	<Object>
     Most recently observed status of the pod. This data may not be up to date.
     Populated by the system. Read-only. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
```

Luego, si nos interesa saber dentro de *pod* cómo se puede configurar el `spec`, entonces utilizaremos `kubectl explain pod.spec`:

```bash
╰─ kubectl explain pod.spec
KIND:     Pod
VERSION:  v1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

     PodSpec is a description of a pod.

FIELDS:
   ... editado ...

   containers	<[]Object> -required-
     List of containers belonging to the pod. Containers cannot currently be
     added or removed. There must be at least one container in a Pod. Cannot be
     updated.

   ... editado ...

   volumes	<[]Object>
     List of volumes that can be mounted by containers belonging to the pod.
     More info: https://kubernetes.io/docs/concepts/storage/volumes
```

Lo mejor de este comando es que estamos consultando directamente a la API de Kubernetes, por lo que siempre tendremos documentación al día sin el menor esfuerzo.

## kubectl debug

Los contenedores nacen siendo livianos, o al menos eso se intenta como buena práctica. Cuando los creamos, buscamos que tengan la menor cantidad de componentes, lo cual no es fácil de mantener en el tiempo. Eventualmente, surge la necesidad de hacer uso de herramientas específicas para resolver algún problema complejo en un contenedor, y como respuesta a eso, editamos la imágen del contenedor para que tenga un conjunto de herramientas con las que estamos familiarizados (tcpdump, dig, curl, bash, ip, ss, entre otras). Estas herramientas son utilizadas por medio de `kubectl exec`. Al agregar más paquetes, inevitablemente incrementamos el tamaño de la imagen, y peor aún, convertimos nuestros contenedores en un recurso muy valioso en caso de una intrusión a nuestro entorno, dado que el atacante ya tiene todas las herramientas necesarias para inspeccionar nuestra red o cluster. Por este motivo, y algunos avances en cuanto a los *runtimes* y las imágenes base de contenedores (como *distroless*), a partir de la v1.23 se introdujo una beta de `kubectl debug` (ya *stable* en v1.25) que nos permite depurar nuestros pods en ejecución sin necesidad de instalar componentes en las imágenes de nuestras aplicaciones.

Para poder hacer uso de `kubectl debug`, lo mejor es tener una imagen lista específicamente con nuestras herramientas más usadas, y luego hacer uso de la misma sólo en caso de emergencia. El contenedor que creemos con nuestra imagen, se adjuntará al *pod* que estamos intentando analizar, y luego de terminado nuestro trabajo, simplemente eliminaremos el *pod* de depuración.

## Operadores y complejidad

Hemos intentado omitir a los operadores en este tema, pero vale la mención. Dado que los operadores implican una abstracción sobre los recursos desplegados de nuestras cargas de trabajo, el flujo usual de resolución de problemas pasa primero no por revisar los logs del *pod* afectado, sino más bien los logs del *pod* del operador, dado que este es el encargado de mantener todos los recursos hijos en buen estado, por ende tiene información de contexto mucho más útil que cada *pod* por separado.

Es importante entender, al hacer uso de operadores, qué tipo de recursos crea, cúales son sus limitaciones y qué tipo de operaciones son gestionadas por el operador y no necesitan intervención humana. Ejemplo, la gran mayoría de los operadores se encargan de gestionar el *image* y su versión específica, definiendo la versión de forma manual (por medio de un CRD) o según la versión del operador, por lo que si actualizamos un componente a nivel operador, las instancias harán sus cambios sin necesidad de tomar acción manual.

## Enlaces sugeridos

- [Debug Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Debug Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Kubernetes Troubleshooting - The Complete Guido](https://komodor.com/learn/kubernetes-troubleshooting-the-complete-guide)
- [A visual guide on troubleshooting Kubernetes deployments](https://learnk8s.io/troubleshooting-deployments)
- [Debug Kubernetes CrashLoopBackOff](https://sysdig.com/blog/debug-kubernetes-crashloopbackoff/)
- [Debugging with an ephemeral debug container](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container)
