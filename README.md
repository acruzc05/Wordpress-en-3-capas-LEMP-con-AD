# Wordpress en 3 capas LEMP con Alta disponibilidad - Antonio Cruz Clavel - 13/12/2023
# Índice
# 1. Descripción del proyecto
Con esta práctica se quiere dar a conocer como implementar una pila LEMP en una infraestructura de tres capas y desplegar el CMS "Wordpress".  
La estructura sería la siguiente:  
### **Capa 1**:  
**Balanceador de carga nginx** con interfaz de red pública que distribuye las solicitudes entre los servers web llamada *BalanceadorAntonioCruz*  
* IP: **10.0.5.2**

### **Capa 2**: 
Backend con:  
#### Dos máquinas con un servidor Nginx cada una llamadas *serverweb1AntonioCruz* y *serverweb2AntonioCruz*.
  ##### Serverweb1AntonioCruz:  
 * IP red1: **10.0.5.4**
 * IP red2: **192.168.2.16**  
  ##### Serverweb2AntonioCruz:
 * IP red1: **10.0.5.6**  
 * IP red2: **192.168.2.18**
#### Una máquina con un servidor NFS y motor PHP-FPM llamada *serverNFSAntonioCruz*.
 * IP: **192.168.2.14**  

### **Capa 3**: Máquina servidor de base de datos llamada *serverdatos1AntonioCruz*.  
 * IP: **192.168.2.20**  
 <br/>
Las capas 2 y 3 no están expuestas a red pública. Los servidores web utilizan una carpeta compartida por NFS desde el serverNFS y además utilizan el motor PHP-FPM instalado es una misma máquina.  



# 2. Esquema del proyecto
![](https://github.com/acruzc05/Wordpress-en-3-capas-LEMP-con-AD/blob/main/Esquema%20NGINX%203capas.drawio.png)
# 3. VagrantFile
```
Vagrant.configure("2") do |config|

  config.vm.box = "debian/buster64"

  # Balanceador
  config.vm.define "balanceadorAntonioCruz" do |bal|
    bal.vm.hostname = "balanceadorAntonioCruz"
    bal.vm.network "private_network", ip: "10.0.5.2"
    bal.vm.network "public_network"
    bal.vm.network "forwarded_port", guest: 80, host: 9000
    bal.vm.provision "shell", path: "balanceadorAntonioCruz.sh"
  end

  # ServidorNFS
  config.vm.define "serverNFSAntonioCruz" do |nfs|
    nfs.vm.hostname = "serverNFSAntonioCruz"
    nfs.vm.network "private_network", ip: "192.168.2.14"
    nfs.vm.provision "shell", path: "serverNFSAntonioCruz.sh"
  end

  # ServidorWeb1
  config.vm.define "serverweb1AntonioCruz" do |sweb1|
    sweb1.vm.hostname = "serverweb1AntonioCruz"
    sweb1.vm.network "private_network", ip: "10.0.5.4"
    sweb1.vm.network "private_network", ip: "192.168.2.16"
    sweb1.vm.provision "shell", path: "serverweb1AntonioCruz.sh"
  end

  # ServerWeb2
  config.vm.define "serverweb2AntonioCruz" do |sweb2|
    sweb2.vm.hostname = "serverweb2AntonioCruz"
    sweb2.vm.network "private_network", ip: "10.0.5.6"
    sweb2.vm.network "private_network", ip: "192.168.2.18"
    sweb2.vm.provision "shell", path: "serverweb2AntonioCruz.sh"
  end

  # Serverdatos1
  config.vm.define "serverdatos1AntonioCruz" do |ssdd|
    ssdd.vm.hostname = "serverdatos1AntonioCruz"
    ssdd.vm.network "private_network", ip: "192.168.2.20"
    ssdd.vm.provision "shell", path: "serverdatos1AntonioCruz.sh"
  end
end
``` 
# 4. Scripts de automatización  
En estos scripts se explica paso a paso como se logra automatizar las máquinas.

## 4.1. Balanceador
```
#! /bin/bash

#Instalar nginx
sudo apt update
sudo apt install -y nginx
check_success

#Iniciar y habilitar servicio Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
check_success

# Configuración del Balanceador

configuracion="
upstream servidoresweb {
    server 10.0.5.4;
    server 10.0.5.6;
}
	
server {
    listen      80;
    server_name loadbalancing.example.com;

    location / {
	    proxy_redirect      off;
	    proxy_set_header    X-Real-IP \$remote_addr;
	    proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header    Host \$http_host;
        proxy_pass          http://servidoresweb;
	}
}"

# Borrar plantilla default
if [ -f /etc/nginx/sites-enabled/default ]; then
    sudo rm /etc/nginx/sites-enabled/default
fi

# Crear nuevo archivo de configuración
if [ ! -f /etc/nginx/conf.d/balanceador.conf ]; then
    sudo touch /etc/nginx/conf.d/balanceador.conf

    # Insertar la configuración en el archivo previamente creado
    echo "$configuracion" > /etc/nginx/conf.d/balanceador.conf

    # Reiniciar el servicio
    sudo systemctl restart nginx
fi
```

## 4.2 Server NFS
```
#! /bin/bash

sudo apt update

#Instalar nfs server
sudo apt install -y nfs-kernel-server

#Instalar php
sudo apt install -y php-fpm
sudo apt install -y php-mysql

# Crear carpeta compartida
if [ ! -d /var/nfs/cms ]; then
    sudo mkdir -p /var/nfs/cms
    sudo chown -R nobody:nogroup /var/nfs/cms
    sudo chmod -R 755 /var/nfs/cms
    sudo echo "/var/nfs/cms     192.168.2.16(rw,sync,no_subtree_check) 192.168.2.18(rw,sync,no_subtree_check)" >> /etc/exports
    sudo systemctl restart nfs-kernel-server
fi


# Configurar php e instalar Wordpress
if [ ! -f $HOME/.flag ]; then
    # Creo la bandera para evitar que se vuelva a ejecutar la configuración.
    sudo touch $HOME/.flag
    sudo chmod +t $HOME/.flag

    # Editar el listen de fmp
    sudo sed -i 's|listen = /run/php/php7.3-fpm.sock|listen = 192.168.2.14:9000|' /etc/php/7.3/fpm/pool.d/www.conf

    # Instalar wordpress
    sudo wget https://wordpress.org/latest.tar.gz
    sudo tar -xzvf latest.tar.gz
    sudo mv wordpress/* /var/nfs/cms

        # Editar config.php
    sudo cp /var/nfs/cms/wp-config-sample.php /var/nfs/cms/wp-config.php
    sudo sed -i "s/define( 'DB_NAME', 'database_name_here' );/define( 'DB_NAME', 'wpdb' );/" /var/nfs/cms/wp-config.php
    sudo sed -i "s/define( 'DB_USER', 'username_here' );/define( 'DB_USER', 'antonio' );/" /var/nfs/cms/wp-config.php
    sudo sed -i "s/define( 'DB_PASSWORD', 'password_here' );/define( 'DB_PASSWORD', '1234' );/" /var/nfs/cms/wp-config.php
    sudo sed -i "s/define( 'DB_HOST', 'localhost' );/define( 'DB_HOST', '192.168.2.20' );/" /var/nfs/cms/wp-config.php

        # Transferir propiedad
    sudo chown -R www-data:www-data /var/nfs/cms
    sudo chmod -R 755 /var/nfs/cms

    # Reiniciar el servicio
    sudo systemctl restart php7.3-fpm.service

fi
```
## 4.3 Server web 1
```
#! /bin/bash
sudo apt update

# Instalar nginx y módulos de php
sudo apt install -y nginx
sudo apt install -y php-fpm
sudo apt install -y php-mysql

# Instalar nfs-common
sudo apt install -y nfs-common

# Creación de punto de montaje
if [ ! -d /var/nfs/cms ]; then
    sudo mkdir -p /var/nfs/cms
fi
sudo mount 192.168.2.14:/var/nfs/cms /var/nfs/cms

# Configurar de nginx
wordpress="
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/nfs/cms;

        index index.php index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                try_files \$uri \$uri/ =404;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass 192.168.2.14:9000;
        }
        location ~ /\.ht {
                deny all;
        }
}"

if [ -f /etc/nginx/sites-enabled/default ]; then
    # Eliminar plantilla predeterminada y crear una nueva
    sudo rm /etc/nginx/sites-enabled/default
    sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/wordpress
    
    # Configurar y activar el archivo conf
    echo "$wordpress">/etc/nginx/sites-available/wordpress
    sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/

    # Reiniciar el servicio
    sudo systemctl restart nginx
fi
    #Instalar mysql-client
    sudo apt-get install -y mariadb-client
```
## 4.4 Server web 2
```
#! /bin/bash
sudo apt update

# Instalar nginx y módulos de php
sudo apt install -y nginx
sudo apt install -y php-fpm
sudo apt install -y php-mysql

# Instalar de nfs-common
sudo apt install -y nfs-common

# Crear punto de montaje
if [ ! -d /var/nfs/cms ]; then
    sudo mkdir -p /var/nfs/cms
fi
sudo mount 192.168.2.14:/var/nfs/cms /var/nfs/cms

# Configurar nginx
wordpress="
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/nfs/cms;

        index index.php index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                try_files \$uri \$uri/ =404;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass 192.168.2.14:9000;
        }
        location ~ /\.ht {
                deny all;
        }
}"

if [ -f /etc/nginx/sites-enabled/default ]; then
    # Eliminar la plantilla de configuración predeterminada y crear una nueva llamada "wordpress"
    sudo rm /etc/nginx/sites-enabled/default
    sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/wordpress
    
    # Insertar la configuración anterior en el archivo nuevo creado. Crear un enlace blando.
    echo "$wordpress">/etc/nginx/sites-available/wordpress
    sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/

    # Reiniciar el servicio
    sudo systemctl restart nginx
   
fi
    #Instalar mysql-client
    sudo apt-get install -y mariadb-client
```

## 4.5 Server BBDD
```
#Instalar mariadb-server
sudo apt-get update
sudo apt-get install -y mariadb-server

#!/bin/bash

# Variables
DB_NAME="wpdb"
DB_USER="antonio"
DB_PASSWORD="1234"
DB_HOST="192.168.2.20"
BIND_ADDRESS="192.168.2.%"

# Cambiar dirección bindaddress
sudo sed -i "s/bind-address = 127.0.0.1/bind-address = ${BIND_ADDRESS}/" /etc/mysql/mariadb.conf.d/50-server.cnf

# Reiniciar servicio MYSQL
sudo systemctl restart mariadb

# Esperar a que MYSQL se reinicie
sleep 5

# Crear BBDD y usuario
sudo mysql -u root -e "CREATE DATABASE IF NOT EXISTS wpdb;"
sudo mysql -u root -e "CREATE USER IF NOT EXISTS 'antonio'@'192.168.2.%' IDENTIFIED BY '1234';"
sudo mysql -u root -e "GRANT ALL PRIVILEGES ON wpdb.* TO 'antonio'@'192.168.2.%';"
sudo mysql -u root -e "FLUSH PRIVILEGES;"
```
 
