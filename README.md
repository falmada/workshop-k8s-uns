# Workshop sobre Kubernetes para la Universidad Nacional del Sur

## Intro

La idea de este workshop nace luego de unas charlas que la UNS nos invitó a realizar en sus aulas. Con el fin de promover el uso de Kubernetes, y hacer llegar esta tecnología a estudiantes que seguramente se cruzarán con ella en sus próximos años, surge la necesidad de armar un Workshop para explorar distintos contenidos relacionados al tema.

La audiencia de este material es todo aquel que tenga una base de conocimientos sobre contenedores, y quiera empezar a experimentar con Kubernetes en su propia computadora.

## ¿Por qué hacemos este taller?

Queremos acercar Kubernetes a distintos grupos que pueden adoptarlo para su trabajo actual o futuro, e incluso hacer uso de esta tecnología en su propio hogar. Creemos que aquellas tecnologías que son accesibles gracias a proyectos como éste, deben ser aprovechados por la comunidad para expandir su conocimiento, como también entender las distintas opciones que existen hoy en día para desplegar sus aplicaciones y administrar sus cargas de trabajo.

Si de este taller surge al menos una persona interesada en usarlo para su trabajo y/o proyectos personales, hemos logrado nuestro objetivo.

### Sobre Kubernetes

Kubernetes es más que un orquestador de contenedores. Durante el workshop, esperamos mostrar tres de sus muchos beneficios:

- Portabilidad
- Alta disponibilidad
- Self-healing

## Objetivo

Durante el Workshop aprenderán lo necesario para llevar una aplicación casera desde su concepción hasta tenerla corriendo en Kubernetes siguiendo buensa prácticas.

## Contenido

### [01 - Contenedores](contenido/01-contenedores.md)

- Instalación de los componentes necesarios para utilizar Docker
- Pull y Push de una imagen
- Container Registry/Docker Registry

### [02 - Imagenes de contenedores](contenido/02-imagenes.md)

- Crear una imagen a partir de una aplicación propia
- Tagging de imágenes
- Update de imágenes
- Correr varias instancias de la misma imagen
- Balanceando tráfico a las imágenes

### [03 - Kubernetes](contenido/03-kubernetes.md)

- Instalación de Kubernetes en local
- Primera prueba de pod
- Nuestra app en un pod
- Exponer y probar nuestra aplicación
- Formas de exponer servicios

### [04 - Preparando nuestra app para el mundo real](contenido/04-hola-mundo-real.md)

- Self-healing
- Controladores
- Balanceo por medio de servicio
- Multiples servicios, mismo label

### [05 - Buenas prácticas](contenido/05-buenas-practicas.md)

- Namespaces
- Kustomization
- Herramientas
- Aplicar resources al spec
- SecurityContext
- Optimizar imagen
  - Tamaño
  - Layers
  - Alpine
- Readiness, Liveness
  
### [06 - Monitoreo](contenido/06-monitoreo.md)

- Prometheus
  - Queries
- Grafana
  - Dashboards públicos

### [07 - Resolución de problemas](contenido/07-troubleshooting.md)

- Ejemplos de situaciones comunes
  - CrashLoopBackOff
  - Pending
  - OOMKill
  - Readiness
  - Terminating

### [99 - Cheatsheet](contenido/99-cheatsheet.md)

- K3d
- Docker
- Kubernetes

## Otros recursos

- [kube.academy](https://kube.academy/)
