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

# Reconocimiento
### SO de la maquina
Lo primero seria entender bien con que tipo de maquina estamos tratando recordar que según el TTL, osease el tiempo en el cual se almacenan los datos cache en servidor o maquina, en este caso dará un *ttl de 63*, debido a que es una maquina Linux, pero esta posee un intermediario y por ello se ve reducido el tiempo de respuesta.

| Sistema operativo | TTL |
| ----------------- | --- |
| Linux             | 64  |
| Windows           | 128 |

```bash
ping -c 1 10.10.11.8
```

[Imagen 2]