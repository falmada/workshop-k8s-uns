# 01 - Contenedores

## Preparación

Para poder ejecutar los pasos de este contenido, necesitamos tener docker-ce instalado. El ejemplo de abajo está pensado para Ubuntu pero pueden adecuar los pasos a su SO de elección:

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

## Jugando con docker

Una vez instalado los paquetes, una prueba base que podemos hacer para probar si funciona es _pullear_ y ejecutar una imagen del repositorio oficial de Docker.

```bash
╰─ docker pull alpine:latest
latest: Pulling from library/alpine
213ec9aee27d: Pull complete 
Digest: sha256:bc41182d7ef5ffc53a40b044e725193bc10142a1243f395ee852a8d9730fc2ad
Status: Downloaded newer image for alpine:latest
docker.io/library/alpine:latest
```

Para ejecutar la imagen, tenemos varias opciones, la más simple es ejecutar:

```bash
╰─ docker run --name docker_demo --rm -i -t alpine:latest sh 
```

Si corremos un comando como `curl` nos daremos con que el mismo no viene instalado por defecto.

## Mejor armemos una imagen propia

Vamos a armar una imagen usando como base `alpine` e instalaremos `curl` dentro de ella para poder hacer peticiones a distintos sitios web como prueba

```bash
╰─ docker build -t workshop-uns:latest ../extras/01-contenedores
Sending build context to Docker daemon  2.048kB
Step 1/3 : FROM alpine:3.16
 ---> 9c6f07244728
Step 2/3 : RUN apk update
 ---> Running in 528623970b69
fetch https://dl-cdn.alpinelinux.org/alpine/v3.16/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.16/community/x86_64/APKINDEX.tar.gz
v3.16.2-221-ge7097e0782 [https://dl-cdn.alpinelinux.org/alpine/v3.16/main]
v3.16.2-227-g7411d9b1c4 [https://dl-cdn.alpinelinux.org/alpine/v3.16/community]
OK: 17033 distinct packages available
Removing intermediate container 528623970b69
 ---> 089941bc6bf9
Step 3/3 : RUN apk add curl
 ---> Running in bc4e017224b5
(1/5) Installing ca-certificates (20220614-r0)
(2/5) Installing brotli-libs (1.0.9-r6)
(3/5) Installing nghttp2-libs (1.47.0-r0)
(4/5) Installing libcurl (7.83.1-r3)
(5/5) Installing curl (7.83.1-r3)
Executing busybox-1.35.0-r17.trigger
Executing ca-certificates-20220614-r0.trigger
OK: 8 MiB in 19 packages
Removing intermediate container bc4e017224b5
 ---> a35e9462a2fd
Successfully built a35e9462a2fd
Successfully tagged workshop-uns:latest
```

Ahora podemos conectarnos a nuestra imagen y hacer finalmente un `curl google.com`

```bash
╰─ docker run --name alpine_demo --rm -i -t my-image:latest sh
/ # curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
/ # 
```

## Pusheando nuestra imagen

Por defecto, cuando construimos la imagen, esta queda almacenada internamente en nuestra computadora. Para poder distribuirla hay muchas opciones, siendo la más conocida Docker Hub. Si quisieramos hospedar un registro de imágenes en nuestra red, podemos ir por opciones como Harbor, GitLab Community Edition, o bien hacer uso de los servicios ofrecidos por los distintos cloud providers (ACR, ECR, GCR, etc).

Vamos a usar Docker Hub como ejemplo, para lo cual [necesitan hacerse una cuenta](https://hub.docker.com/signup), pero es gratis y toma unos minutos.

Una vez que tenemos la cuenta, [creamos un repositorio](https://hub.docker.com/repositories) y luego corremos los siguientes comandos para _pushear_ nuestra imagen. Importante: Para pruebas, hagan el repositorio público, caso contrario van a tener que autenticar más adelante para _pullear_.

```bash
# Login
docker login
# Tagueamos la imagen con el nombre del repo
docker tag workshop-uns:latest fedek3/workshop-uns:latest
# Hacemos push
docker push fedek3/workshop-uns:latest
```

Si visitamos [nuestro repositorio](https://hub.docker.com/repositories), deberíamos poder ver la nueva imagen lista para consumir.

## Enlaces sugeridos

- [Docker Hub](https://hub.docker.com/search?q=) - Hogar de millones de imagenes para usar en contenedores
- [RedHat Quay.io](https://quay.io/search) - Menos conocido pero igual de util para obtener imagenes
- [Harbor](https://goharbor.io/) - Registro de imagenes para nuestro entorno
