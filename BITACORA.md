# Primero: Configuracion de Almacenamiento Persistente (RAID 1 y LVM)

## 1. Verificar discos disponibles
	comando: 
	- lsblk -f
	* verificamos que:
	/dev/sda tiene el sistema instalado.
	/dev/sdb a /dev/sdg estan vacios.

## 2. Instalacion de paquetes para gestionar RAID y LVM 
	comandos: 
	- sudo apt update 
	- sudo apt install  -y mdadm  lvm2
	* mdadm: administra RAID por software
	* lvm2: Permite la creacion de LVM

## 3. Creacion de los RAID 1
	
## RAID 1 para Apache (/de/md0)
	comando:
	- sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
	* Se crea un raid de nivel 1, con dos discos y escogemos los discos fisicos.

## RAid 1 para Mysql (/dev/md1)
	comando:
	- sudo mdadm --create /dev/md1 --level=1 --raid-devices=2 /dev/sdd /dev/sde

## RAID 1 para Nginx (/dev/md2)
	comando: 
	- sudo mdadm --create /dev/md2 --level=1 --raid-devices=2 /dev/sdf /dev/sdg

## Verificacion de los RAID
	comando:
	- cat /proc/mdstat
	* se espera la siguiente salida: 
	md0 : active raid1 sdb[0] sdc[1]
	md1 : active raid1 sdd[0] sde[1]
	md2 : active raid1 sdf[0] sdg[1]

## 4. Crear PV sobre los RAID
	comando:
	- sudo pvcreate /dev/md0
	- sudo pvcreate /dev/md1
	- sudo pvcreate /dev/md2
	* verificamos con:
	- sudo pvdisplay 
	* Vemos /dev/md0, /dev/md1, /dev/md2 como PV

## 5. Crear VG 
	comandos: 
	- sudo vgcreate vg_apache /dev/md0 
	- sudo vgcreate vg_mysql /dev/md1
	- sudo vgcreate vg_nginx /dev/md2
	* verificamos con:
	- sudo vgdisplay 
	* Vemos nombre y tamano real 

## 6. Crear LV
	comandos: 
	- sudo lvcreate -n lv_apache -l 100%FREE vg_apache
	- sudo lvcreate -n lv_apache -l 100%FREE vg_mysql
	- sudo lvcreate -n lv_create -l 100%FREE vg_nginx
	* verificamos con: 
	- sudo lvdisplay
	
## 7. Formatear los LV
	comandos: 
	- sudo mkfs.ext4 /dev/vg_apache/lv_apache
	- sudo mkfs.ext4 /dev/vg_mysql/lv_mysql
	- sudo mkfs.ext4 /dev/vg_nginx/lv_nginx
	* Se formatea cada LV con el sistema de archivos ext4

## 8. Crear carpetas de montaje
	comandos: 
	- sudo mkdir -p /srv/apache
	- sudo mkdir -p /srv/mysql
	- sudo mkdir -p /srv/nginx
	* /srv para los datos del contnedor correspondiente

## 9. Montaje de los volumenes
	comandos: 
	- sudo mount /dev/vg_apache/lv_apache /srv/apache
	- sudo mount /dev/vg_mysql/lv_mysql /srv/mysql
	- sudo mount /dev/vg_nginx/lv_nginx /srv/nginx

## 10. Configurar montaje automatico
	comando: 
	- sudo nano /etc/fstab
	* Agregamos los UUID de cada LV 

## 11. Probar montaje automatico
	comando:
	- sudo mount -a
	* si no muestra error, todo esta correcto


# Segundo: Configuracion de Docker y Podman

## 1. Instalacion de Docker
	comandos: 
	- sudo apt update
	- sudo apt install -y docker.io
	- sudo systemcl enable --now docker
	- sudo usermod -aG docker $USER
	* Instalamos y habilitamos Docker

## 2. instalacion de Podman
	comando: 
	- sudo apt install -y podman


# Tercero: Creacion de imagen personalizada para Apache 

## 1. Configuracion de Apache en Docker y Podman 
	comandos:
	- mkdir -p /home/andres/proyectoFinalInfra/containers/apache/public-html
	* Estructura del proyecto
	- echo "<h1>Saludos desde Apache en Docker y Podman</h1>" > /home/andres/proyectoFinal/apache/public-html/index.html
	* Se crea el archivo HTML
	- sudo cp /home/andres/proyectoFinalInfra/containers/apache/public-html/index.html /srv/apache/	
	* Se mueve al LVM

# 2. Crear Dokerfile y Containerfile para Apache
	comando: 
	- nano /home/andres/proyectoFinalInfra/containers/apache/Dockerfile
	* contenido para Dockerfile:
	- FROM httpd:2.4 
	  COPY public-html/ /usr/local/apache2/htdocs

	* Contenido para Containerfile
	- FROM docker.io/library/httpd:2.4
	  COPY public-html/ /usr/local/apache2/htdocs/

## 3. Construir imagen
	comando: 
	- cd /home/andres/proyectoFinal/apache
	* Para Dockerfile
	- docker build -t mi-apache:1.0 .

	* Para Containerfile 
	- podman build -t mi-apache:1.0 -f Containerfile . 

## 4. Ejecutar Apache en Docker (puerto 8080)
	comando: 
	- docker run -d --name apache-docker -p 8080:80 -v /srv/apache:/usr/local/apache2/htdocs mi-apache:1.0
	* Probar
	- curl http://localhost:8080
	* muestra el HTML 

## 5. Ejecutar Apache en Podman (puerto 8081)
	comando:
	- docker run -d --name apache-podman -p 8081:80 -v /srv/apache:/usr/local/apache2/htdocs:Z mi-apache:1.0
	* Probar
	- curl http://localhost:8081
	* Muestra el HTML

# Cuarto: Creacion de imagen para Mysql usando Docker y Podman

## 1. Creacion de imagen con Docker

## 1.1. Persistencia de datos
	comando: 
	- docker volume create mysql_data
	* directorio interno donde se guardan los datos, garantizando la persistencia


## 1.2. Montar volumen en el contenedor
	comando:
	- docker run -d --name mysql-prod -p 3306:3306 -e MYSQL_ROOT_PASSWORD=1234 -v mysql_data_volume:/var/lib/mysql mysql:latest 
	* Inicia el contenedor con lo necesario para su utilizacion

## 1.3. Acceder a la base de datos
	comandos: 
	- docker exec -it mysql-db bash
	- mysql -u root -p
	* Se conecta al cliente MySQL

## 1.4. Crear base de datos de prueba
	comandooo:
	- CREATE DATABASE persistencia_test;
	- SHOW DATABASES;
	* Sale persistencia_test en la lista

## 1.5. Simular perdida 
	comando:
	- docker stop mysql-prod
	- docker rm mysql-prod
	* Borramos el contenedor, pero el volumen con los datos sigue ahi

## 1.6. Recrear el contendor 
	* Se incia el contenedor y dentro de la base de datos verificamos
	comando:
	- SHOW DATABASES;
	* Vemos persistencia_test


## 2. Creacion de imagen en podman

## 2.1. Persistencia de datos
	comando: 
	- podman volume create podman_mysql_data
	* directorio interno donde se guardan los datos para la persistencia  

## 2.2. Montar volumen en el contendor 
	comado:
	- podman run -d --name mysql-podman -p 3307:3306 -e MYSQL_ROOT_PASSWORD=1234 -v podman_mysql_data:/var/lib/mysql docker.io/library/mysql:latest
	* Inicia el contendor con todo lo necesario para su utulizacion 

## 2.3. Acceder a la basa de datos 
	comando:
	- docker exec -t mysql/podman bash 
	- mysql -u root -p 
	*  se conecta al cliente MySQL

## 2.4. Crear base de datos de prueba
	comando: 
	- CREATE DATABASE podman_test;
	- SHOW DATABASES;
	* sale podman_test en la lista


## 2.5. Simular Perdida
	comando:
	- podman stop mysql-podman
	- podman rm mysql-podman 
	* Borramos el contendor, pero el volumen con los datos sigue ahi 

## 2.6. Recrear el contenedor
	* Se inicia el contendor y verificamos dentro de la base de datos
	comando: 
	- SHOW DATABASES;
	* vemos podman_test




# Quinto: Creacion de imagen para Nginx		

## 1. Creacion del directorio y contenido
	comando:
	- sudo mkdir -p /srv/nginx/html
	* Directorio para el volumen
	- mkdir -p /home/andres/ProyectoFinalInfra/nginx/conf.d
	* Directorio para la configuracion de Nginx

## 2. Archivos de configuracion
	comando:
	- nano /home/andres/ProyectoFinalInfra/nginx/conf.d/default.conf
	* El contenido del archivo es la configuracion minima y estandar para Nginx

## 3. Contenido de Prueba
	comando:
	- echo "<h1>Nginx funciona correctamente</h1>" > /srv/nginx/html/index.html
	* Crea una pagina HTML simple para verificar persistencia

## 4. Ejecutar el contendor docker para Nginx
	comando:
	- docker run -d --name nginx_server -p 80:80 -v /srv/nginx/html:/usr/share/nginx/html:ro -v /home/andres/ProyectoFinalInfra/nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf:ro nginx:latest 
	* Imagen oficial de Nginx y montaje de la configuracion

## 5. Verificar esatdo y contenido de Nginx
	comando:
	- docker ps
	* Verificar que el estado este en UP
	- curl localhost 
	* Devuelve el HTML 



## 6. Ejecucion de Podman
	comando:
	- podman run -d --name nginx_server_podman -p 8080:80 -v /srv/nginx/html:/usr/share/nginx/html:ro,z -v /home/andres/ProyectoFinalInfra/nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf:ro,z docker.io/library/nginx:latest
	* Ejecuta la instancia especificando el registro de docker

## 7. Verificacion de estaado y contenido
	comando:
	- podman ps
	* verificamos que el estado este en UP
	- curl localhost
	* Devuelve el HTML
