# 02 - Imágenes de Contenedores

## Crear imagen de nuestra aplicación

Vamos a crear una aplicación sencilla. Para este ejemplo, utilizaremos PHP pero pueden hacer uso de cualquier lenguaje y framework con el que se sientan cómodos, con la única salvedad que los paquetes a instalar serán distintos.

Nuestra [primera versión será sencilla](../extras/02-imagenes/v0.1.0/src/index.php), con el conocido Hola Mundo adaptado a este workshop.

Luego, volvemos a Docker para generar una imagen.

```bash
╰─ docker build -t my-app extras/02-imagenes/v0.1.0/    
Sending build context to Docker daemon  3.584kB
Step 1/2 : FROM php:7.2-apache
7.2-apache: Pulling from library/php
6ec7b7d162b2: Pull complete 
db606474d60c: Pull complete 
afb30f0cd8e0: Pull complete 
3bb2e8051594: Pull complete 
4c761b44e2cc: Pull complete 
c2199db96575: Pull complete 
1b9a9381eea8: Pull complete 
fd07bbc59d34: Pull complete 
72b73ab27698: Pull complete 
983308f4f0d6: Pull complete 
6c13f026e6da: Pull complete 
e5e6cd163689: Pull complete 
5c5516e56582: Pull complete 
154729f6ba86: Pull complete 
Digest: sha256:4dc0f0115acf8c2f0df69295ae822e49f5ad5fe849725847f15aa0e5802b55f8
Status: Downloaded newer image for php:7.2-apache
 ---> c61d277263e1
Step 2/2 : COPY src/ /var/www/html/
 ---> 7fbe18f0b4d4
Successfully built 7fbe18f0b4d4
Successfully tagged my-app:latest
Execution time: 0h:01m:07s sec
```

Una vez generada, deberíamos poder correrla y ver que nuestra aplicación está efectivamente en su lugar

```bash
╰─ docker run --name test --rm -i -t my-app sh 

# ls /var/www/html/
index.php
# exit
```

Ahora correremos la aplicación exportando el puerto 80 localmente para poder acceder.

```bash
docker run -p 80:80 my-app
```

Desde un navegador nos aseguramos que podemos verla, visitando <http://localhost/>. Luego volvemos a la terminal y con CTRL+C procedemos a matar el proceso de Docker.

## Tagging de imágenes

Si volvemos a la salida de docker build, veremos que nuestra imagen por defecto recibe un tag "latest" cada vez que la buildeamos:

> Successfully tagged **my-app:latest**

Esto es óptimo para un entorno de desarrollo, donde siempre queremos poder fácilmente probar la última versión, pero no es una buena práctica usar dicho tag en un entorno productivo. Entre los motivos contra esta práctica, nos encontramos con la imposibilidad de determinar que la misma imágen está corriendo en multiples entornos (si la imágen se actualiza y un entorno se reinicia, puede recibir la nueva imágen por error); la posibilidad de corromper entornos (las bases de datos necesitan procesos especiales para actualizar determinadas versiones, como saltos en versiones mayores); no es fácil replicar un entorno en otro en caso de problemas, entre otros.

Con estas consideraciones, es necesario entonces que adoptemos versionado en nuestros tags, que tengan sentido con el ciclo de vida del desarrollo de nuestra aplicación. Entre varias formas, existe una en particular llamada **Versionado Semántico** que, a grandes rasgos, nos permite tener un control sobre la versión mayor (generas incompatibilidad en el API), versión menor (nuevas funciones compatibles con anteriores) y parches (corrección de errores de versiones anteriores). Si bien el versionado semántico está enfocado en servicios que declaren un API público, su uso se puede traspolar durante la etapa de desarrollo para nuestras aplicaciones y sacarnos un poco de encima la decisión de cuando cambiar de versión.

Entendiendo esta necesidad, procedemos entonces a crear nuestro primer tag de versión para nuestra precaria aplicación.

```bash
╰─ docker tag my-app:latest my-app:v0.1.0
╰─ docker images | grep my-app
my-app                  latest              7fbe18f0b4d4   15 minutes ago   410MB
my-app                  v0.1.0              7fbe18f0b4d4   15 minutes ago   410MB
```

## Actualizando imágen

Para poder ver los beneficios de agregarle tags a las imagenes, podemos entonces generar una nueva versión de la misma.

Vamos a hacer algo sencillo y [agregarle información sobre PHP](../extras/02-imagenes/v0.2.0/src/index.php) a la salida que aparece en pantalla.

Luego, generamos nuevamente la imagen pero esta vez usando la versión (v0.2.0) en vez de latest

```bash
╰─ docker build -t my-app:v0.2.0 extras/02-imagenes/v0.2.0/
Sending build context to Docker daemon  3.584kB
Step 1/2 : FROM php:7.2-apache
 ---> c61d277263e1
Step 2/2 : COPY src/ /var/www/html/
 ---> 800130fbe69d
Successfully built 800130fbe69d
Successfully tagged my-app:v0.2.0
```

Si comparamos ambas salidas, nos daremos cuenta que la primera vez tardó bastante más tiempo en general la imagen, pero esta vez fue bastante más rápido. Esto se debe a que las imagenes de contenedores hacen uso de capas, y las que ya estaban descargadas localmente, están disponibles para acelerar el proceso, mientras que todo aquello que haya cambiado, invalida el resto de las capas locales de ahí en más.

Un ejemplo bien sencillo sería:

1. Tenemos una imagen base de Ubuntu
2. Nuestra empresa aplica una serie de buenas prácticas sobre esa imagen base
3. El equipo de seguridad hace un proceso de *hardening* sobre la imagen que ya tiene buenas prácticas
4. Nosotros aplicamos nuestra aplicación sobre la imagen de *hardening*.

Si nuestros cambios se aplican sobre la capa 4, reutilizaremos siempre las capas 1, 2 y 3, ya que al generar una nueva imagen, Docker se dará cuenta que dichas capas están disponibles localmente y no necesita descargarlas (esto lo hace comparando el *checksum*). Pero si la empresa decide hacer una actualización de la capa 2, donde están las buenas prácticas, dicho cambio generará un efecto dominó sobre la capa 3 y la capa 4 cuando alguien quiere generar una imagen nueva de nuestra aplicación.

```bash
╰─ docker images | grep my-app
my-app                  v0.2.0                  800130fbe69d   6 minutes ago    410MB
my-app                  latest                  7fbe18f0b4d4   40 minutes ago   410MB
my-app                  v0.1.0                  7fbe18f0b4d4   40 minutes ago   410MB
```

Si ahora revisamos las imágenes locales, nos daremos con que hay 3 en total. Detalle extra, dado que construimos la imagen haciendo uso de `-t my-app:v0.2.0`, notese que el `checksum` de `latest` no coincide con la última imagen, sino con la anterior. Esto lo podemos corregir fácilmente actualizando la referencia con el comando: `docker tag my-app:v0.2.0 my-app:latest`

```bash
╰─ docker tag my-app:v0.2.0 my-app:latest
╰─ docker images | grep my-app           
my-app                  latest                  800130fbe69d   8 minutes ago    410MB
my-app                  v0.2.0                  800130fbe69d   8 minutes ago    410MB
my-app                  v0.1.0                  7fbe18f0b4d4   42 minutes ago   410MB
```

De este modo nos garantizamos que nuestros desarrolladores siempre estén probando la última versión.

## Multiple instancias de la misma imagen

Si quisieramos correr multiples instancias de esta imagen, y sólo disponemos de nuestro entorno local, podemos simplemente generar dos consolas y correr los siguientes comandos

```bash
docker run -p 80:80 my-app:latest
# Dado que el puerto 80 ya está en uso por la primera instancia, usamos 8080
# Al hacer -p 8080:80, decimos que abrimos nuestro puerto 8080 para exponer el puerto 80 del contenedor
docker run -p 8080:80 my-app:latest
```

Luego podemos visitar <http://localhost/> y <http://localhost:8080/> para acceder a cada instancia, y nos daremos cuenta por la información de `System` que ambas corren en distintos *hostnames*.

## Balancear tráfico a ambas instancias

Supongamos que estamos probando nuestra aplicación y necesitamos poder ver su comportamiento en casos que una de las instancias se caiga o esté saturada, para ver cómo responder la otra. Poder ofrecer alta disponibilidad en nuestras aplicaciones de negocio es casi obligatorio para cualquier tipo de solución (hay excepciones), por lo que pensar en cómo se comporta nuestra aplicación cuando hay "más de una instancia" corriendo, es necesario. Con esto surgen varias dudas, la principal es cómo enviar tráfico a "cada una de las instancias", y lo segundo es el comportamiento de las mismas cuando interactúan con componentes en común con las otras instancias.

Simplemente enfocándonos en lo primero, es necesario considerar una solución de balanceo de carga. En la mayoría de los casos, y gracias a una simple búsqueda de Google, ya entraremos en el mundo de los *reverse proxy* o soluciones pre-armadas que necesitan de ciertas condiciones para que todo corra bien. Dicho de un modo simple... no es fácil, pero tampoco imposible.

A continuación comenzaremos con [Kubernetes](03-kubernetes.md), que nos brinda una solución declarativa a este problema, y viene con otros beneficios por debajo de la manga.

## Enlaces sugeridos

- [Repositorio de Imágenes de PHP](https://hub.docker.com/_/php)
- [Versionado Semántico](https://semver.org/lang/es/)