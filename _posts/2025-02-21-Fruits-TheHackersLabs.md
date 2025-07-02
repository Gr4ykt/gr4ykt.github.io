![Pasted image 20241205180242](https://github.com/user-attachments/assets/3defc240-3ad4-419f-b7fb-e398c8845be6)---
title: "Fruits - THL Writeup"
layout: single
excerpt: "Fruits es una máquina de TheHackerLabs. Para el acceso inicial aprovecharemos un LFI expuesto, lo cual nos permitirá obtener nombres de usuarios a través de enumeración de archivos primordiales como los son /etc/passwd, para posteriormente a través de fuerza bruta con hydra obtener la contraseña de un usuario al cual nos conectaremos por SSH. Una vez tengamos acceso a la máquina, con enumeración básica de sistemas Linux conseguiremos ver que el usuario tiene permisos de sudo con el comando find, lo cual, con ayuda del sitio GTFobins obtendremos acceso a root."
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

![Pasted image 20241205150737](https://github.com/user-attachments/assets/b4417714-b1cb-4cf5-9f90-7acb526d4517)

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

![Pasted image 20241205150813](https://github.com/user-attachments/assets/be54431c-8ac9-4550-a928-134e077ab10b)

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

![Pasted image 20241205154059](https://github.com/user-attachments/assets/2a473035-0a82-48b7-aa79-ec7a4a86f44f)

Pues podemos concluir la abertura de dos puertos con el análisis previamente realizado, el puerto `22, 80` están expuestos, posiblemente, se traten de servicio `SSH` y `HTTP`, ya que son los predeterminados para estos servicios. Un posible vector de descubrimiento de vulnerabilidades puede ser con la misma herramienta `nmap` utilizando los parámetros `-sCV` podemos explorar las versiones de estos servicios y hacernos ideas de por donde más explorar.

```bash
nmap -sCV -p22,80 192.168.1.34
```

>Parámetros utilizados:
>* **-sCV:** Este comando es la combinación de dos, pese a ir a algo similar, ya que ambos hacen referencia a escanear las versiones y servicios que corren en el puerto, -sC es un comando enfocado en lanzar una serie de scripts de nmap a través de los propios scripts de la herramienta (NSE), su objetivo es la búsqueda de versiones, identificar vulnerabilidades, entre otras pruebas útiles. Mientras que -sV realiza una detección de servicios y versiones exactas en el puerto especificado.
>* **-p22,80:** Señalamos los puertos a los que realizar el escaneo, en este caso solo son los dos puertos abiertos, en caso de no hacerlo con este parámetro, nmap realizara el escaneo en su top predeterminado de puertos, ya que nmap por detrás, si no se le señalan los puertos específicos, no realiza el escaneo en los 65535 puertos existentes, para ello se utiliza el parámetro `-p-`.

![Pasted image 20241205154146](https://github.com/user-attachments/assets/a7ad9f60-73cb-4fc2-aafb-13f2e785c94f)

Vale, las versiones de estos servicios no son vulnerables, tendremos que ingeniárnosla con otras cosas dentro de estos mismos vectores, al ser una máquina fácil, lo ideal es no buscar vulnerabilidades complejas, ya que suele ser más fácil y externo de lo que se ve.

## Escaneando la Web
En continuación a los escaneos, lo ideal sería tratar de abordarlo vía **HTTP**, puesto que es el vector más posible al tener una versión de SSH no vulnerable, así que no podremos probar mucho más de algún ataque relacionado con este servicio, en este caso, realizaré un ataque sencillo de **Fuzzing**, al fin y al cabo, enfocaremos este tipo de ataque al descubrimiento de rutas con un diccionario de rutas, este tipo de ataque consiste en la exploración rápida y automatizada de rutas, para este caso, pero te invito a investigar más de este tipo de ataques, ya que hasta existen ciertos tipos relacionados con la fuerza bruta vía petición web, descubrimiento de subdominios, etc. En este caso en particular, lo que haremos es tratar de descubrir rutas posibles a acceder con la herramienta `wfuzz` de la siguiente manera:

```bash
wfuzz -c --hc=404 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://192.168.1.34/FUZZ.php
```

>Parámetros utilizados:
>* **-c:** Formato coloreado para los resultados del fuzzeo.
>* **--hc=404:** Esconde el código de estado *404*, ya que de no hacerlo, la pantalla se llenara de rutas con este estado, cosa que es irrelevante si lo que buscamos, son las posibles rutas.
>* **-t 200:** Emplea *200 hilos*, esto con el objetivo de que el escaneo vaya un poco más rápido, en caso de tener tu sistema operativo funcionando con una buena GPU y CPU, puedes tener la posibilidad de emplear muchos más hilos y que el escaneo vaya mucho más rápido.
>* **-w [Ruta diccionario]:** Esta ruta es de un diccionario de fuzzing que viene con la herramienta [dirbuster](https://www.kali.org/tools/dirbuster/) y con su parámetro le damos a entender a la herramienta `wfuzz` que es el diccionario a emplear para las iteraciones en busca de rutas.
>* **URL/FUZZ.extensión:** Señalamos la dirección web finalmente y también donde emplear el fuzzing, con la palabra clave `FUZZ` nosotros le decimos a `wfuzz` que es donde debe aplicar las palabras clave dentro del diccionario, y en conjunto, he aplicado el ataque con la extensión `.php` porque son los archivos que me interesan en este ataque (ya veras el porque).

![Pasted image 20241205160847](https://github.com/user-attachments/assets/e9f904f4-7f23-461f-b038-3377cbb6d6a5)

Lo que pudimos rescatar del fuzzeo es muy relevante, debido a que nos puede llevar a temas bastantes útiles dentro de todo, podemos ver que nos ha detectado la ruta `fruits.php`, la cual nos ha entregado un código de esta `200 OK` señalando que podemos acceder a esta ruta.
# Explotación
## Local File Inclusion
> Una cosa que se me olvido mencionar es que puedes perfectamente realizar este ataque a través del navegador web como Firefox, pero yo preferí realizarlo de esta manera a través de la terminal con `curl`, ya que me acomodaba más ver la información resultado del ataque, de igual manera, te invito que busques de otras maneras que se acomoden a tu manera de hackear.

Pues bien, teniendo en cuenta la ruta que tenemos `fruits.php` podemos tratar de enumerar de dos maneras, una y la más útil de todas es una manera más manual, utilizando técnicas y probando distintos componentes del sitio web en cuestión y utilizar herramientas como [BurpSuite](https://portswigger.net/burp) para analizar las peticiones enviadas al servidor y sus respuestas. O la segunda manera y la que utilizaré en este caso, utilizar herramientas automatizadas para agilizar el proceso de análisis de vulnerabilidades y posibles vectores de ataques, si quieres ver algún caso de ataque web manual, te invito a que veas mi writeup de la máquina retirada de Hack The Box [Headless](https://gr4ykt.github.io/writeup/hack%20the%20box/Headless-HackTheBox-writeup/). Pues bien, volviendo al tema, la herramienta que emplee en este caso fue `nikto`, una muy buena herramienta para análisis rápidos de sitios web o rutas en específico.

> Recomiendo encarecidamente que si estás comenzando a aprender hacking ético, busca comprender los conceptos y el trabajo manual antes de hacer uso de herramientas automatizadas, es importante aprender de todo un poco en este rubro.

```bash
nikto -url http://192.168.1.34/fruits.php
```

![Pasted image 20241205160804](https://github.com/user-attachments/assets/34f59135-dd6b-4697-ad0c-6de78f588422)

Lo que podemos concluir de este resultado con la herramienta `nikto` es que estamos frente a un caso de **Local File inclusion** o *LFI*. Esta vulnerabilidad consiste en que el atacante puede acceder a archivos locales del servidor mediante la manipulación de parámetros de la URL, en este caso, específicamente `?file`, que es el parámetro que busca los datos de las frutas dentro de la página web. Pues bien, este será nuestro vector de ataque, en este caso, haré el intento con la herramienta `curl` para poder enviar la petición y ver los resultados de mejor manera.

```bash
curl -X GET "http://192.168.1.34/fruits.php?file=/etc/passwd"
```

![Pasted image 20241205163705](https://github.com/user-attachments/assets/aa2c6f2d-b8e3-49e3-9363-1d55886158c3)
![Pasted image 20241205163714](https://github.com/user-attachments/assets/f0129fec-0a01-47f8-a018-6cdb26d44412)

## Fuerza bruta
Pues bien, tenemos al usuario `bananaman`, con este usuario trataremos de realizar un ataque de fuerza bruta, pues si bien recuerdas, tenemos el puerto `22` abierto, el cual tiene el servicio `ssh` expuesto, la idea en este caso sería con la herramienta `hydra` tratar de hacer un ataque de fuerza bruta con uno de los mejores diccionarios para este caso que es el **rockyou.txt**, este diccionario es excepcional cuando se trata de contraseñas, por lo menos en lo que respecta en CTFs. Bueno, pues esto lo ejecute de la siguiente manera y me dio resultados muy positivos.

```bash
hydra -l bananaman -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.34 -t 20
```

![Pasted image 20241205175458](https://github.com/user-attachments/assets/5dffa49a-1803-4eaf-b56e-bc42a7be0a95)

Este resultado es bastante positivo, ya que tenemos tanto el usuario como contraseña del usuario `bananaman:celtic` y esto nos da una ventaja en poder conectarnos vía ssh y pasar a la siguiente parte del ataque, con esto, ya habremos obtenido la bandera user

![Pasted image 20241205175726](https://github.com/user-attachments/assets/a3e97974-bc8c-4638-834d-b2e80943c340)

# Escalando privilegios
En este caso, al tener la contraseña del usuario ingresado (el cual es `bananaman`) pues el intento de enumerar el sistema operativo es poco, sabemos que se trata de una máquina `Linux` debido al reconocimiento inicial basado en el `TTL` del servidor. Pues bien, con el comando `sudo -l` y colocamos la contraseña del usuario nos entregará información relacionada a que podemos ejecutar con permisos del usuario `root`.

![Pasted image 20241205175818](https://github.com/user-attachments/assets/3d4df4d2-507d-4aed-85d8-01e5410031a0)

Pues vemos que podemos ejecutar el comando `find` como el usuario `root` con `sudo`, pues bien, en este siguiente proceso la cosa sería bastante sencilla, si ingresamos a la página de [GTFOBins](https://gtfobins.github.io/) y buscamos el comando `find` podremos ver la siguiente posible shell con sudo a través de este comando:

![Pasted image 20241205175934](https://github.com/user-attachments/assets/b82b7ea1-393e-4277-9c3b-cd71c67a2dd4)

Si logramos ejecutar este comando correctamente, como se puede ver, obtendremos una shell y si ejecutamos el comando `whoami` podemos ver que tenemos el acceso como el usuario root, por tanto, la máquina estará completada, tenemos acceso al sistema completo, solo debemos de sacar la flag igualmente del usuario root.

![Pasted image 20241205180217](https://github.com/user-attachments/assets/458d9e53-b5cd-476a-a41e-36f2c012e565)
![Pasted image 20241205180242](https://github.com/user-attachments/assets/143d27d3-eb93-4dd6-91e6-535a7a053d02)
