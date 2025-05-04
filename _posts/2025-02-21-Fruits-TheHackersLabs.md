---
title: "Fruits - THL Writeup"
layout: single
excerpt: "Fruits es una máquina de TheHackerLabs. Para el acceso inicial aprovecharemos un LFI expuesto, lo cual nos permitirá obtener nombres de usuarios a través de enumeración de archivos primordiales como los son /etc/passwd, para posteriormente a través de fuerza bruta con ssh obtener la contraseña de un usuario. Una vez tengamos acceso a la máquina, con enumeración básica de sistemas Linux conseguiremos ver que el usuario tiene permisos de sudo con el comando find, lo cual, con ayuda del sitio GTFobins obtendremos acceso a root."
show_date: true
classes: wide

toc: true
toc_label: "Contenido"
toc_icon: "hacker"
toc_sticky: false


header:
  teaser: assets/images/TheHackersLabs/Fruits.png
  teaser_home_page: true
  icon: assets/images/TheHackersLabs/Icon_principal.png

categories:
  - Writeup
  - The Hackers Labs
tags:
  - Linux
  - Fuzzing
  - LFI
  - BruteForce
  - SSH
  - GTFobins
---

[IMAGEN PRESENTACION]

# Reconocimiento
## Sistema operativo de la máquina
Lo primero sería entender bien con qué tipo de máquina estamos tratando recordar que según el TTL, el tiempo en el cual se almacenan los datos cache en servidor o máquina podremos hacernos una idea del sistema operativo utilizado en esta.

| Sistema operativo | TTL |
| ----------------- | --- |
| Linux             | 64  |
| Windows           | 128 |

```bash
ping -c 1 192.168.1.34
```

[IMAGEN 1]

# Escaneo y enumeración
## Nmap

```bash
nmap -sS --min-rate 5000 -p- -Pn -vvv 192.168.1.34
```

>Parámetros utilizados: 
>* **-sS:** Syn Scan o Stealth Scan, No completa la conexión hacia la máquina, pero de igual modo emite un paquete que descubre si el puerto está abierto o no..
>* **--min-rate 5000:** Decimos que no emita paquetes mas grandes que 5000.
>* **-p-**: Escaneo enfocado en los 65535 puertos.
>* **-Pn:** Evitar realizar rastreo de la version del sistema operativo remoto.
>* **-vvv:** Triple *verbose* para tener mayor información al momento mientras se realiza el escaneo.

[IMAGEN 2]


