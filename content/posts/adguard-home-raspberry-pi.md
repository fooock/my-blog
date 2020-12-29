+++
title = "Adguard Home en Raspberry Pi"
date = 2020-12-28T17:31:12+01:00
author = "Javi"
draft = true
tags = ["adguard", "privacidad", "raspberry", "vodafone"]
description = "Nuestra casa libre de trackers y ads gracias a `Adguard Home`, una solución open source para proteger nuestra privacidad y nuestros dispositivos. Lo montaremos todo sobre una RaspberryPi."
+++

Si estás cansado de que en tu Smart TV se muestren anuncios cada vez que pones un video de YouTube, o cada vez que entras en un periódico online aparezcan grandes cantidades de publicidad que ocupan más espacio que las propias noticias, sigue leyendo porque esto es lo que necesitas.

# Objetivo

En este post te enseñaré a montar el sistema operativo que irá en nuestra Raspberry, preparar la configuración de red, y poner en funcionamiento el software `Adguard Home` para que no vuelvas a ver la molesta publicidad, mejorar la privacidad de los miembros del hogar (evitando los trackers) y de paso, ahorrar en ancho de banda.

## Requisitos

Antes de continuar asegúrate de que tienes todo lo necesario para montar el sistema:

* Una Raspberry Pi (yo tengo la `rpi 3 model B`)
* Memoria SD para la rpi
* Mínimos conocimientos de redes y linux

>Si no tienes una RaspberryPi y quieres una a buen precio, mira en [PC Componentes](https://www.pccomponentes.com/raspberry), que siempre tienen buenas ofertas.

## Configuración de red

Lo primero será entrar en nuestro router para revisar la configuración y dejar todo preparado para poder asignarle a nuestra rpi una IP estática dentro de nuestra red local. El objetivo final será configurar nuestra rpi con `Adguard` como servidor DNS en el router.

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

Descargate la distribución usando el siguiente comando:

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

Una vez finalizado el proceso, desmonta el volumen de la tarjeta SD e introducela en la Raspberry Pi. Conectale los periféricos necesarios e inicia la instalación.

#### Instalar el sistema

Es importante configurar el sistema con nuestras preferencias, poner la zona horaria correcta y la contraseña para el usuario root. Acuérdate de actualizar la configuración de red para asignarle a tu máquina la IP estática que vimos anteriormente, en mi caso `192.168.0.14`.

En caso de no configurar la red en la instalación del sistema, siempre podemos usar la herramienta `nmcli` para cambiar el método de red a manual, asignar el gateway, cambiar la IP y reiniciar la interfaz de red para aplicar los cambios.

Una vez instalado todo, actualizamos el sistema. Esto llevará un tiempo, así que hazte un café.
