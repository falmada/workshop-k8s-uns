# Monitoreo

- Prometheus
  - Queries
- Grafana
  - Dashboards públicos

Una vez que hemos logrado tener nuestra aplicación siguiendo las buenas prácticas y actualizando la misma de forma frecuente por medio de su *deployment* como también su imagen, es necesario empezar a pensar en el Día 2, es decir, qué pasa una vez que nuestra aplicación está productiva. Para esto, debemos considerar inicialmente el monitoreo de la aplicación, ya que si bien Kubernetes hace su parte con el *self-healing* para evitar que haya *downtime*, puede que nuestra aplicación sea afectada por otras cuestiones que necesitan ser monitoreadas.

## Prometheus

Esta herramienta es una de las más utilizadas hoy en día en el mercado, dado que cuenta con el soporte de la Cloud Native Computing Foundation (CNCF) como también porque provee una solución bastante completa en cuanto al monitoreo de nuestro clúster, sus componentes críticos y las cargas de trabajo que usan Kubernetes como plataforma.

Para hacer uso de Prometheus, tenemos varias opciones disponibles. La más compleja es simplemente instalar [Prometheus](https://prometheus.io/) e ir agregando configuraciones específicas. Rara vez usaremos esta opción a menos que queramos usar la herraimenta para multiples tecnologías fuera de Kubernetes. Otra opción es hacer uso de [Prometheus Operator](https://prometheus-operator.dev/), un operador que se encarga de hacer lo difícil sencillo, y nos da un punto de partida siguiendo las buenas prácticas de esta herramienta. Este último, cuenta con un proyecto asociado que recibe el nombre de [Kube Prometheus](https://github.com/prometheus-operator/kube-prometheus), el cual por medio de Helm nos permite hacer una instalación de todos los componentes necesarios siguiendo las buenas prácticas y permitiendo iterar en las configuraciones de manera rápida y fácil.

En su uso más sencillo, podremos instalarlo usando los siguientes comandos:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus prometheus-community/kube-prometheus-stack
# este último paso demorará bastante ya que descarga el chart y lo instala
```

Una vez instalado, podremos ver todos los componentes por medio del comando que nos brinda la salida anterior.

```bash
╰─ kubectl --namespace default get pods -l "app.kubernetes.io/instance=kube-prometheus"
NAME                                                   READY   STATUS    RESTARTS   AGE
kube-prometheus-prometheus-node-exporter-dl7vp         1/1     Running   0          65s
kube-prometheus-prometheus-node-exporter-9fz64         1/1     Running   0          65s
kube-prometheus-prometheus-node-exporter-62fmz         1/1     Running   0          65s
kube-prometheus-prometheus-node-exporter-2zswm         1/1     Running   0          65s
kube-prometheus-kube-prome-operator-74f8447468-4hxjr   1/1     Running   0          65s
kube-prometheus-kube-state-metrics-54fcd44ccd-nlw9k    1/1     Running   0          65s
```

Los `node-exporter` se encargarán de exportar métricas relacionadas a los nodos. Como tenemos 4 nodos (1 master, 3 workers), la cantidad de estos pods se corresponde. El `operator` será encargado de interpretar nuestra configuración y tratar de desplegar los pods necesarios. Finalmente, el `kube-state-metrics` se encargará de exportar métricas de estado relacionadas a Kubernetes.

Otros componentes que se desplegan por defecto son `grafana`, un visualizador de métricas muy conocido; `alert-manager`, un administrador de alerta que hace uso de las métricas como fuente de información, y `prometheus` en sí mismo, es decir, el servicio que permite hacer consultas vía web.

### Consultas en Prometheus

Para poder hacer consultas (*queries*) en nuestro cluster usando Prometheus, primero vamos a tener que hacer un `port-forward` sobre el servicio.

```bash
kubectl port-forward svc/prometheus-operated 9090
# Luego en el navegador, abrimos http://localhost:9090/
```

Una vez que estamos dentro de la interfaz gráfica, podemos escribir `kube` en el campo de *Expression* y ver cómo el autocompletado nos ayuda a encontrar métricas relacionadas a Kubernetes o sus componentes.

Usaremos el ejemplo con una consulta de pods:

```promql
# Obtener todos los pods
kube_pod_info
# Obtener sólo los pods del namespace default
kube_pod_info{namespace="default"}
# Obtener cantidad de pods por namespaces
sum(kube_pod_info{}) by (namespace)
```

Pero dijimos antes que nuestro propósito era usar esta herramienta para monitorear, por lo que supongamos que queremos asegurarnos de que la cantidad de replicas de nuestro *deployment*, en realidad se mantiene en un número tal como nos promete Kubernetes.

```promql
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

Hacer consultas complejas en Prometheus no es fácil, requiere tiempo y mucha paciencia. Si bien es un lenguaje bastante amigable, la curva de aprendizaje al comienzo puede ser un poco frustrante. Por suerte, encontraremos muchos proyectos, que tienen despliegues pre-armados para Kubernetes, con las consultas más usuales para agregar o incluso la definición ideal para usarlas en AlertManager.

## Grafana

Como mencionamos anteriormente, Prometheus recolecta métricas, y luego Grafana es capaz de visualizarlas.

Nuevamente, abramos un `port-forward`, esta vez apuntando al servicio de Grafana... aunque primero vamos a necesitar la clave para acceder.

```bash
kubectl get secret kube-prometheus-grafan -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
kubectl port-forward svc/kube-prometheus-grafana 3000:80
```

Luego abramos nuestro navegador en <http://localhost:3000> y usemos de usuario `admin` y de clave la que hayamos obtenido antes.

Una vez dentro de la GUI, podemos acceder al menú de `Dashboards (4 cuadrados) > Browse > General` y elegir alguno de los que vienen por defecto, para visualizar información de nuestro cluster mucho más colorida y personalizable que en Prometheus.

### Dashboards

Los dashboards se pueden hacer de forma manual, aunque toman demasiado tiempo y requieren un buen conocimiento de las métricas a analizar, como también de las funciones a utilizar para compactar esas métricas (avg, sum, rate, entre otros). Una de las mejores fuentes de dashboards es el [repositorio público de Grafana](https://grafana.com/grafana/dashboards/), aunque tengan en cuenta que en muchos casos son esfuerzos comunitarios y/o personales, por lo que es necesario hacer algunas correciones según como esté configurado nuestro cluster de Kubernetes (en cuanto a métricas) o los componentes que utilizamos y queremos monitorear.

## Enlaces sugeridos

- [Tools for monitoring resources](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)
- [Prometheus](https://prometheus.io/)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus)
- [Grafana dashboards](https://grafana.com/grafana/dashboards/)