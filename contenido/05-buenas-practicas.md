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

## Aplicar resources al spec

//TODO

## SecurityContext

//TODO

##  Optimizar imagen

//TODO


### Tamaño

//TODO

### Layers

//TODO

### Alpine

//TODO

##  Readiness, Liveness

//TODO

## Enlace sugeridos

- [Namespaces](https://kubernetes.io/es/docs/concepts/overview/working-with-objects/namespaces/)
- [Kustomization](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Kustomization file](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/)
- [Kubetools](https://collabnix.github.io/kubetools/)