# **Reto 3**

**Nombre:** Paulina Ocampo Duque <br>
**Curso:** Tópicos Especiales en Telemática <br>
**Título:** Aplicación Monolítica con Balanceo y Datos Distribuidos (BD y archivos<br>
**Objetivo:** Desplegar un CMS Drupal utilizando la tecnología de contenedores (docker), con su propio
dominio y certificado SSL. El sitio lo llamará: reto3.sudominio.tl<br>

*******

**Tabla de Contenido**

1. [Introducción](#introduction)
2. [Base de datos](#BD)
3. [Archivos NFS](#nfs) 
4. [Drupal](#drupal) 
5. [Balanceador de cargas](#apache)<br>

*******

<div id='introduction'/> 

### **1. Introducción**
El fin de este reto es realizar un CMS drupal utilizando AWS y contenedores, para ello vamos a crear cinco instancias; una instancia para la base de datos, una instancia 
para los archivos NFS, dos instancias para la aplicación drupal las cuales nos van a garantizar la alta disponibilidad y la última para el balanceador de cargas.
*******

<div id='BD'/> 

### **2. Base de datos**
En primer lugar crearemos una instancia EC2 en AWS llamda DBServer y la asociaremos al puerto 3306, iniciamos el ssh e instalaremos docker (instrucciones para instalarlo https://docs.docker.com/engine/install/ubuntu/)
después descargaremos la imagen de docker de MariaDB (que será la base de datos que utilizaremos) desde Docker hub.

```sh
    sudo docker pull mariadb
```
Creamos un contenedor de Docker a partir de la imagen que descargamos 

```sh
    docker run --name dbserve -e MYSQL_ROOT_PASSWORD=password -d mariadb
```
Para verificar que el contenedor este en ejecución 

```sh
    docker ps
```
y se debe ver asi

```sh
    CONTAINER ID   IMAGE     COMMAND                  CREATED        STATUS         PORTS      NAMES
    522b1932c1ac   mariadb   "docker-entrypoint.s…"   20 hours ago   Up 5 seconds   3306/tcp   dbserve
```
*Para inicializar el docker de la base de datos recuerda ejecutar primero

```sh
    sudo systemctl start docker
    sudo docker start dbserve
```
*Para ejecutar la base de datos dentro del contenedor que creamos

```sh
    docker exec -it dbserve mysql -p
```
Una vez dentro de la base de datos, crearemos una nueva base de datos y un usuario que será el que tenga acceso a ella.

```sh
    CREATE DATABASE drupal;
    GRANT ALL PRIVILEGES ON drupal.* TO 'user'@'mpocampod' IDENTIFIED BY '<contraseña>';
```
Para salir de la sesión de la base de datos
```sh
    exit
```
*******

<div id='NFS'/>  

### **3. Archivos NFS**

En primer lugar creamos la instancia EC2 de AWS llamda NFServer y la asociamos al puerto 22, iniciamos el entorno virtual y crearemos el paquete nfs-kernel-server

```sh
    sudo apt-get update
    sudo apt-get install nfs-kernel-server
```
Después se crea el directorio que se compartirá mediante NFS y se cambian los permisos del directorio para que el servidor NFS tenga acceso

```sh
    sudo mkdir /nfs_server
    sudo chown nobody:nogroup /nfs_server
    sudo chmod 777 /nfs_server
```
Para permitir que cualquier cliente tenga acceso al directorio compartido, agregar la siguiente línea al archivo

```sh
  /nfs_server *(rw,sync,no_subtree_check)
```
Por último reiniciaremos el servicio NFS para que los cambios se vean efectuados
```sh
    sudo systemctl restart nfs-kernel-server
```

<div id='drupal'/> 

#### **4. Drupal**

Para la aplicación drupal primero crearemos dos instancias EC2 de AWS y las asociamos al puerto 80, iniciamos la terminal virtual de cada una e instalamos docker como lo hicimos con la base de datos,
luego descargaremos la imagen de Docker de Drupal.
```sh
    sudo docker pull drupal
```
Después crearemos un contenedor de docker de drupal, en el caso de la primera instancia
```sh
    docker run -p 80:80 --name drupal1 -d drupal
```
en el caso de la segunda instancia
```sh
    docker run -p 80:80 --name drupal2 -d drupal
```
Ahora conectaremos la instancia de la base de datos y la de los archivos NFS con las instancias que acabamos de crear de Drupal, en este caso para la instancia del drupal 1
Nota: las ip son temporales por lo tanto no tomarlas como base

```sh
  docker run -p 80:80 --name drupal1 --network=host -e DRUPAL_DATABASE_HOST=172.31.91.32 -e DRUPAL_DATABASE_USER=user@mpocampod -e DRUPAL_DATABASE_PASSWORD=password -e DRUPAL_DATABASE_NAME=drupal -e DRUPAL_FILES_PATH=/mnt/nfs -e DRUPAL_FILES_NFS_SERVER=172.31.93.197 -e DRUPAL_FILES_NFS_PATH=/exports -d drupal
```
Para la instancia del drupal 2
```sh
  docker run -p 80:80 --name drupal2 --network=host -e DRUPAL_DATABASE_HOST=172.31.91.32 -e DRUPAL_DATABASE_USER=user@mpocampod -e DRUPAL_DATABASE_PASSWORD=password -e DRUPAL_DATABASE_NAME=drupal -e DRUPAL_FILES_PATH=/mnt/nfs -e DRUPAL_FILES_NFS_SERVER=172.31.93.197 -e DRUPAL_FILES_NFS_PATH=/exports -d drupal
```
<div id='apache'/> 
#### **5. Balanceador de cargas**
Ahora crearemos la instancia EC2 en AWS del balanceador de cargas y la asociaremos al puerto 443 para permitir peticiones HTTPS utilizando Apache, iniciamos el ssh e instalamos docker como vimos anteriormente y descargamos la imagen de Docker de Apache 
```sh
  sudo docker pull httpd
```
Luego crearemos el directorio que almacenará la configuración
```sh
  sudo mkdir -p /usr/local/apache2/conf
```
Después crearemos y accederemos a un archivo creado en ese directorio
```sh
  sudo nano /usr/local/apache2/conf/httpd.conf
```
Y editaremos el archivo con la siguiente configuración

```sh
  <Proxy "balancer://mycluster">
        BalancerMember "http://172.31.31.147:80/"
        BalancerMember "http://172.31.16.51:80/"
    </Proxy>

    <VirtualHost *:80>
        ProxyPreserveHost On
        ProxyPass "/" "balancer://mycluster/"
        ProxyPassReverse "/" "balancer://mycluster/"
    </VirtualHost>
```
Guarda y cierra el archivo.

Por último ejecuta el siguiente comando para iniciar el contenedor de Apache con el archivo de configuración que acabas de crear

```sh
  sudo docker run -p 80:80 -v /usr/local/apache2/conf/httpd.conf:/usr/local/apache2/conf/httpd.conf -d httpd
```
*Recuerda cuando vayas a inicializar las máquinas virtuales, iniciar el docker y ejecutarlas dentro del contenedor
*En las oportunidades de mejora encontramos que se le puede agregar un dominio y un certificado ssl.
*******

