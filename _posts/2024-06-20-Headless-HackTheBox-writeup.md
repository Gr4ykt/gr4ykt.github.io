---
title: "Headless - HTB Writeup"
layout: single
excerpt: "Headless es una maquina fácil de Hack The Box. Para el acceso inicial robaremos una cookie de administrador gracias a un XSS DOM expuesto que nos permitirá acceder a un dashboard el cual tendrá un RCE a través de su envió de formulario de fechas. Una vez tengamos acceso a la máquina, con una enumeración sencilla al sudo podremos escalar privilegios gracias a un script que se ejecuta sin necesidad de proveer contraseña el cual ejecuta un archivo en el directorio actual de trabajo del cual nos aprovecharemos para obtener root."
show_date: true
classes: wide

toc: true
toc_label: "Contenido"
toc_icon: "cube"
toc_sticky: false


header:
  teaser: https://raw.githubusercontent.com/Gr4ykt/gr4ykt.github.io/master/assets/images/Headless-HackTheBox/Headless-htb-logo.png
  teaser_home_page: true
  icon: https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/2e9afb33-966a-4848-af53-9473877d2350

categories:
  - Writeup
  - Hack The Box
tags:
  - XSS
  - Fuzzing
  - RCE
  - CVE
  - Linux
  - Enumeration
---

![Headless-header](https://raw.githubusercontent.com/Gr4ykt/gr4ykt.github.io/master/assets/images/Headless-HackTheBox/Headless-header.png)

# Reconocimiento
## Sistema operativo de la máquina
Lo primero sería entender bien con qué tipo de máquina estamos tratando recordar que según el TTL, el tiempo en el cual se almacenan los datos cache en servidor o máquina, en este caso dará un *ttl de 63*, debido a que es una máquina Linux, pero esta posee un intermediario y por ello se ve reducido el tiempo de respuesta.

| Sistema operativo | TTL |
| ----------------- | --- |
| Linux             | 64  |
| Windows           | 128 |

```bash
ping -c 1 10.10.11.8
```

[Imagen 2]

# Escaneo y enumeración
## Nmap

```bash
nmap -sS --min-rate 5000 -p- -Pn -n -vvv 10.10.11.8 -oN 1
```

>Parámetros utilizados: 
>* **-sS:** Syn Scan o Stealth Scan, No completa la conexión hacia la máquina, pero de igual modo emite un paquete que descubre si el puerto está abierto o no..
>* **--min-rate 5000:** Decimos que no emita paquetes mas grandes que 5000.
>* **-p-**: Escaneo enfocado en los 65535 puertos.
>* **-Pn:** Evitar realizar rastreo de la version del sistema operativo remoto.
>* **-n:** Evitar la resolución DNS, simplemente para que el escaneo vaya un poco mas rápido.
>* **-vvv:** Triple *verbose* para tener mayor información al momento mientras se realiza el escaneo.
>* **-oN 1:** Almacena los resultados en un archivo llamado `1` en el formato nmap correspondiente, lo cual significa que este archivo entregara la información tal cual se nos fue expuesta durante y al finalizar el escaneo.

