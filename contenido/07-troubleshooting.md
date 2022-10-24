# 07 - Resolución de problemas

En toda tecnología es necesario contar con algún manual que nos permita resolver los problemas más frecuentes, como también entender las distintas formas de obtener información para poder solucionar aquellos problemas no tan comunes. Kubernetes, debido a su complejidad, hace esto mucho más sencillo gracias a opciones de su herramienta de cliente nativa `kubectl`, lo cual facilita enormemente la tarea. Existen varias guías (ver [Enlaces sugeridos](#enlaces-sugeridos)) que nos orientan en la resolución de problemas.

## Ejemplos de situaciones comunes

### CrashLoopBackOff

Si han visto memes de Kubernetes, el *CrashLoopBackOff* es muy usado como contenido debido a que es un error muy común que nos encontraremos cuando empezamos a trabajar en esta tecnología.

Cuando un *pod* entra en un estado de *CrashLoopBackOff*, este mismo inicia sus contenedores, pero en determinado momento se reinicia, no pudiendo llegar a un estado de *Running*. Este estado puede aparecer por múltiples motivos, el más usual estando relacionado a errores de configuración o haciendo uso de algún *flag* no soportado al momento de iniciarlizar uno de los contenedores del *pod*.

La forma más rápida de comprender cuál es el problema es haciendo uso de `kubectl describe pod nombre_pod`, aunque a veces también `kubectl logs nombre_pod` y `kubectl get events` pueden darnos algunas pistas.

//TODO EJEMPLO

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

//TODO

## kubectl debug

//TODO

## Operadores y complejidad

//TODO

## Enlaces sugeridos

- [Debug Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Debug Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Kubernetes Troubleshooting - The Complete Guido](https://komodor.com/learn/kubernetes-troubleshooting-the-complete-guide)
- [A visual guide on troubleshooting Kubernetes deployments](https://learnk8s.io/troubleshooting-deployments)
- [Debug Kubernetes CrashLoopBackOff](https://sysdig.com/blog/debug-kubernetes-crashloopbackoff/)
