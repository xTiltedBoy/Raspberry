# Configuración Raspberry

### 1. Descargar e instalar Ubuntu en tu micro SD:
   * Descargar https://downloads.raspberrypi.org/imager/imager_latest.exe
   * Seleccionar el SO y la SD
   * Instalar imagen en la SD
  
### 2. Una vez que Ubuntu está instalado:
   * Crear otro usuario y borrar el default

### 3. Instalar paquetes necesarios

```
sudo apt-get update && sudo apt-get install -y \
     apt-transport-https \
     ca-certificates \
     curl \
     software-properties-common \
     fail2ban \
     ntfs-3g \
     linux-modules-extra-raspi \
     rsnapshot
```

### 4. Programar los reinicios:
   * Entrar al fichero crontab del root
   * Escribimos la siguiente linea
```
0 5 * * * reboot
```
   
### 5. Instalar firmas GPG del repo de Docker

```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
```

### 6. Agregar repo de Docker

```
echo "deb [arch=armhf] https://download.docker.com/linux/debian \
     $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list
```

### 7. Instalar Docker

```
sudo apt-get update && sudo apt-get install -y --no-install-recommends docker-ce docker-compose
```

### 8. Agregar usuario al grupo docker y desloguearse y volverse a loguear

```
sudo usermod -a -G docker <usuario>
#(logout and login)
```

### 9. Montar los HDD de Datos y de Backups
   * Miramos los UUID de los discos con el siguiente comando:
   
```
sudo blkid
```

   * Los montamos en el fichero /etc/fstab de la siguiente forma:

```
UUID="" /mnt/datos/ ext4      defaults,user   0       0
UUID="" /mnt/backup/ ext4      defaults,user   0       0
```

   * Hacer propietario al nuevo usuario de los HDD

```
sudo chown <usuario> /mnt/* -R
```

### 10. Crear docker-compose
```
version: "3.9"

services:

  apache:
    image: php:7.4.25-apache-buster
    ports:
      - "80:80"
      - "443:443"
    restart: always
    volumes:
      - /mnt/datos/Apache:/var/www/html

  samba:
    image: dperson/samba:rpi
    restart: always
    command: '-u "usuario;contraseña" -s "public;/media;no;no"'
    stdin_open: true
    tty: true
    ports:
      - "130:103"
      - "445:445"
    volumes:
      - /mnt/datos/Samba/media:/media

  plex:
    image: jaymoulin/plex:1.24.5.5173-arm64v8
    volumes:
      - /mnt/datos/Plex/Series:/Plex/Series
      - /mnt/datos/Plex/Peliculas:/Plex/Peliculas
    restart: always
    network_mode: "host"

  pihole:
    image: pihole/pihole:latest
    environment:
      TZ: "Europe/Madrid"
      WEBPASSWORD: "contraseña"
    volumes:
      - /mnt/datos/pihole/etc-pihole:/etc/pihole
      - /mnt/datos/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    dns:
      - 8.8.8.8
      - 8.8.4.4
      - 1.1.1.1
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
    networks:
      lan:
        ipv4_address: 192.168.0.200

networks:
  lan:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: "192.168.0.0/24"
          gateway: "192.168.0.1"
```


### 11. Iniciar docker-compose
```
docker-compose up -d
```

# Configuración de los directorios compartidos con Samba
   
### 1. Dar acceso de red al directorio compartido

```
sudo chmod 707 /mnt/datos/Samba/* -R
```

# Configuración del docker Apache

### 1. Entrar en la terminal del contenedor

```
docker exec -it Apache bash
```

### 2. Instalar los paquetes necesarios para generar el certificado SSL con Certbot

```
apt-get update && apt-get install -y --no-install-recommends certbot python3-certbot-apache && rm -rf /var/lib/apt/lists/*
```

### 3. Generar el certificado SSL para los dominios que necesitemos

```
certbot --apache -d Dominio1 -d Dominio2
```
### 4. Comprobar que esta corriendo el demonio certbot

```
ls /etc/cron.d
```
