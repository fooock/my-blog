+++
title = "AdGuard Home en Raspberry Pi"
date = 2020-12-28T17:31:12+01:00
author = "Javi"
draft = false
tags = ["adguard", "privacidad", "raspberry", "fedora"]
description = "Nuestra casa libre de trackers y ads gracias a `Adguard Home`, una solución open source para proteger nuestra privacidad y nuestros dispositivos. Lo montaremos todo sobre una RaspberryPi que usa Fedora como sistema operativo."
+++

Si estás cansado de que cada vez que entras en un periódico online aparezcan grandes cantidades de publicidad que ocupan más espacio que las propias noticias, sigue leyendo porque esto es lo que necesitas.

# Objetivo

En este post te enseñaré a montar el sistema operativo que irá en nuestra Raspberry, preparar la configuración de red, poner en funcionamiento el software `AdGuard Home` para que no vuelvas a ver la molesta publicidad, mejorar la privacidad de los miembros del hogar (evitando los trackers) y de paso, ahorrar en ancho de banda.

![AdGuard dashboard](/dashboard.jpg)

## Requisitos

Antes de continuar asegúrate de que tienes todo lo necesario para montar el sistema:

* Una Raspberry Pi (yo tengo la `rpi 3 model B`)
* Memoria SD para la rpi
* Mínimos conocimientos de redes y linux

>Si no tienes una RaspberryPi y quieres una a buen precio, mira en [PC Componentes](https://www.pccomponentes.com/raspberry), que siempre tienen buenas ofertas.

## Configuración de red

Lo primero será entrar en nuestro router para revisar la configuración y dejar todo preparado para poder asignarle a nuestra rpi una IP estática dentro de nuestra red local (aun que AdGuard también permite configurarlo como [servidor DHCP](https://github.com/AdguardTeam/AdGuardHome/wiki/DHCP)). El objetivo final será configurar nuestra rpi con `Adguard` como servidor DNS en el router (en caso de que el router lo permita, o hacer la configuración manualmente en caso de que no).

>**Nota**: yo este proceso lo voy a hacer con Vodafone ya que es mi ISP. El proceso debe ser similar para el resto de proveedores de internet (Movistar, R... etc).

Entra en [192.168.0.1](192.168.0.1) o la dirección que tenga tu router. Una vez dentro, si estás en Vodafone, tienes que seleccionar el **Modo experto**.

Ahora vete a `Configuración` -> `LAN`, y revisa la configuración del servidor DHCP. Este servidor es el encargado de asignar las IP a los dispositivos conectados a tu red de forma dinámica, y justo eso es lo que queremos evitar, ya que nuestra rpi será el servidor DNS encargado de filtrar y bloquear publicidad, trackers etc, y tiene que tener siempre la misma IP para que nuestro router resuelva las direcciones correctamente.

Fíjate en el grupo de direcciones IP de inicio y fin, yo los tengo configurados de la siguiente forma:

* Inicio del grupo: `192.168.0.20`
* Fin del grupo: `192.168.0.250`

Esto significa que nuestro router empezará a asignar IP desde la `.20` a todos los dispositivos conectados, como por ejemplo ordenadores, tablets, móviles etc. Puedes ver la IP que tienes asignada en tu PC ejecutando el siguiente comando:

```
javi@fooock $ ip a show wlp2s0 | grep inet
  inet 192.168.0.20/24 brd 192.168.0.255 scope global dynamic noprefixroute wlp2s0
  inet6 fe80::8acb:1513:fb25:8813/64 scope link noprefixroute
```

Todo correcto.

>**Importante:** usa el nombre de la interfaz a la que estás conectado para ver los detalles. En mi caso es `wlp2s0`, pero en tu caso puede ser diferente.

A nuestra rpi le vamos a asignar una IP fuera de ese rango para que no haya problemas. En mi caso, le voy a asignar la IP `192.168.0.14`.

En este momento ya podemos continuar con la configuración de nuestra Raspberry.

## Preparar el sistema operativo de la Raspberry

Para este sistema voy a instalar el sistema operativo [`Fedora 33`](https://fedoraproject.org/wiki/Architectures/ARM/Raspberry_Pi) en su versión servidor. 

>Ten en cuenta que el soporte de Raspberry en Fedora está disponible en versiones `Raspberry Pi Model B version 2 and the 3-series of devices (3B, 3B+, 3A+, CM3, CM3+)`, por lo que si tienes una rpi4, tendrás que instalar un sistema operativo diferente.

Descárgate la distribución usando el siguiente comando:

```
wget https://download.fedoraproject.org/pub/fedora/linux/releases/33/Server/aarch64/images/Fedora-Minimal-33-1.3.aarch64.raw.xz
```

Mientras descarga, vamos a introducir la memoria SD en nuestro PC y vamos a borrar todo el contenido que tenga para tener una instalación limpia. Lo primero que tenemos que ver es en que ruta está el dispositivo de la tarjeta SD. Para verlo podemos usar el comando `lsblk`. En mi caso es `/dev/sdc`.

Bien, para borrar todo el contenido de la SD ejecutamos el siguiente comando:

>**Muy importante**: esta es una acción destructiva, así que asegurate de borrar el contenido del dispositivo correcto, en este caso la SD.

```
sudo dd if=/dev/zero of=/dev/sdc bs=4096 status=progress
```

Dependiendo del tamaño de la SD tardará más o menos tiempo. Mientras se elimina todo el contenido vamos a ir instalando la herramienta para quemar nuestra imagen de Fedora en nuestra tarjeta. El paquete que necesitamos instalar se llama `arm-image-installer`. Usando el gestor de paquetes de nuestra distribución lo instalamos:

```
$ dnf install arm-image-installer
$ arm-image-installer --version
arm-image-installer-3.0
```

A partir de aquí, necesitamos esperar a que finalicen las dos operaciones que lanzamos anteriormente, la descarga de Fedora y el borrado de nuestra tarjeta SD, asi que paciencia.

#### Escribir la imagen en la SD

Ahora que ya tenemos todos los requisitos, ejecutamos el siguiente comando que a parte de escribir la imagen en nuestra tarjeta, también copiará nuestra clave pública SSH.

```
$ sudo arm-image-installer \
  --image=Fedora-Minimal-33-1.3.aarch64.raw.xz \
  --target=rpi3 \
  --media=/dev/sdc \
  --addkey=/home/javi/.ssh/rpi_rsa.pub
```

>Revisa y confirma los datos. Espera a que termine.

Una vez finalizado el proceso, desmonta el volumen de la tarjeta SD e introducela en la Raspberry Pi. Conéctale los periféricos necesarios e inicia la instalación.

#### Instalar el sistema

Es importante configurar el sistema con nuestras preferencias, poner la zona horaria correcta y la contraseña para el usuario root. Acuérdate de actualizar la configuración de red para asignarle a tu máquina la IP estática que vimos anteriormente, en mi caso `192.168.0.14`.

En caso de no configurar la red en la instalación del sistema, siempre podemos usar la herramienta `nmcli` para cambiar el método de red a manual, asignar el gateway, cambiar la IP y reiniciar la interfaz de red para aplicar los cambios.

Una vez instalado todo, actualizamos el sistema. Esto llevará un tiempo, así que hazte un café.

Cuando todo finalice, reiniciamos la rpi y verificamos que desde nuestra máquina podemos entrar mediante SSH:

```
javi@fooock ~ $ ssh -i ~/.ssh/rpi_rsa.pub root@192.168.0.14
Last login: Tue Dec 29 19:23:43 2020 from 192.168.0.20
[root@pidora ~]# uname -r
5.9.16-200.fc33.aarch64
```

Todo correcto.

## Configurar DNS en Raspberry

Queremos configurar nuestra Raspberry como servidor de DNS, por lo que necesitamos deshabilitar el servicio `systemd-resolved` y realizar una configuración manual. Empezamos por deshabilitar y parar el servicio de systemd:

```
# systemctl disable systemd-resolved
# systemctl stop systemd-resolved
```

Eliminamos el fichero `/etc/resolv.conf` y creamos uno nuevo con el siguiente contenido:

```
nameserver 192.168.0.14
```

>Cambia la IP por la tuya en caso de ser diferentes.

Después de aplicar estos cambios, reinicia el servicio de red usando el siguiente comando `nmcli con reload eth0`.

## Instalar AdGuard home

Ahora vamos a descargar los binarios oficiales de Adguard en nuestra Raspberry. Ejecutamos los siguientes comandos desde la terminal de nuestra rpi.

>Si has instalado Fedora Minimal en tu Raspberry, tienes que instalar el paquete `tar` para poder continuar. Ejecuta lo siguiente: `dnf install tar`

```
# curl -L -o adguard.tar.gz \
    https://static.adguard.com/adguardhome/release/AdGuardHome_linux_arm.tar.gz
# tar xvf adguard.tar.gz
# rm adguard.tar.gz
```

Ahora tendrás una carpeta llamada `AdGuardHome` con todos los ficheros necesarios. Entra dentro de la carpeta y ejecuta el comando para instalar:

```
# ./AdGuardHome -s install -v
2020/12/29 19:56:54 [info] Service control action: install
2020/12/29 19:56:56 [info] Service has been started
2020/12/29 19:56:56 [info] Almost ready!
AdGuard Home is successfully installed and will automatically start on boot.
There are a few more things that must be configured before you can use it.
Click on the link below and follow the Installation Wizard steps to finish setup.
2020/12/29 19:56:56 [info] AdGuard Home is available on the following addresses:
2020/12/29 19:56:56 [info] Go to http://127.0.0.1:3000
2020/12/29 19:56:56 [info] Go to http://192.168.0.14:3000
2020/12/29 19:56:56 [info] Action install has been done successfully on linux-systemd

```

Si intentamos entrar a la web que nos indica, veremos que no podemos acceder. Si intentamos ver el estado, nos dirá que algo ha fallado:

```
# ./AdGuardHome -s status
2020/12/29 20:11:54 [info] Service control action: status
2020/12/29 20:11:54 [fatal] failed to get service status: the service is not installed
```

Esto es debido a que en nuestra distribución tenemos habilitado SELinux, como es nuestro caso:

```
# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```

Podemos comprobar el contexto de seguridad de SELinux usando el flag `-Z` del comando `ls`:

```
# ls -Z AdGuardHome 
unconfined_u:object_r:admin_home_t:s0 AdGuardHome
```

Para solucionar este problema, vamos a mover el binario `AdGuardHome` al directorio `/usr/local/bin`, y ejecutamos el siguiente comando para cambiar el contexto de seguridad de nuestro binario.

```
# chcon -t bin_t /usr/local/bin/AdGuardHome
```

Si vemos ahora los permisos:

```
# ls -Z /usr/local/bin/AdGuardHome 
unconfined_u:object_r:bin_t:s0 /usr/local/bin/AdGuardHome
```

Ahora si vemos el estado del servicio, nos dirá que ya está funcionando, pero si intentamos entrar en la web del dashboard... NO funciona!.

Esto es debido a que tenemos el firewall activado. Podemos comprobarlo ejecutando el comando:

```
# firewall-cmd --state
running
```

>Si no tienes el firewall activado lo más probable es que puedas ver el dashboard de Adguard, así que puedes saltarte este paso

Vamos a crear una nueva zona para la configuración de AdGuard. Ejecutamos los siguientes comandos:

```
# firewall-cmd --new-zone=adguard --permanent
# firewall-cmd --zone=adguard --add-source=192.168.0.14/24 --permanent
# firewall-cmd --zone=adguard --add-port=3000/tcp --permanent
# firewall-cmd --zone=adguard --add-port=53/udp --permanent
# firewall-cmd --zone=adguard --add-port=80/tcp --permanent
# firewall-cmd --reload
```

Después de todo, si intentamos entrar en la web, nos aparecerá la pantalla de instalación. Es importante seleccionar la IP que configuramos tanto para el acceso web como para el servidor DNS. Una vez terminada la configuración del dashboard, eliminamos el puerto 3000 del firewall.

```
# firewall-cmd --zone=adguard --remove-port=3000/tcp --permanent
# firewall-cmd --reload
```

Después de todo este trabajo, tendremos el servicio `AdGuarHome` instalado y funcionando en nuestra Raspberry Pi 3 model B, usando el sistema operativo Fedora 33.

### Configurar router/clientes

Si el router permite cambiar la dirección del servidor DNS entonces es la mejor solución (**Vodafone** no lo permite en su router). En caso contrario, tendremos que configurar cliente por cliente de forma manual el DNS al que apuntan, y cambiarlo por el nuestro, `192.168.0.14`.

Y eso sería todo, si tienes dudas o tienes comentarios, puedes escribir en esta [issue](https://github.com/fooock/my-blog/issues/3) de Github.


##### Referencias

* [AdGuard home Github](https://github.com/AdguardTeam/AdGuardHome)
* [Fedora Raspberry Pi](https://fedoraproject.org/wiki/Architectures/ARM/Raspberry_Pi)
* [firewalld docs](https://docs.fedoraproject.org/en-US/quick-docs/firewalld/)
* [systemd-resolved](https://fedoraproject.org/wiki/Changes/systemd-resolved)