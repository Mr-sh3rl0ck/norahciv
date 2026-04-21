---
title: Environment - Hack The Box
author: r4v3nC0d3
date: 2025-05-29 11:00:00 +0800
categories: [Hacking, HackTheBox]
tags: [Laravel]
# pin: true
math: true
mermaid: true
---


![](/assets/img/HackTheBox/Environment/init.png)

----------------------

## Reconocimiento Inicial

Se realizó un escaneo completo de puertos (-p-) con scripts por defecto y detección de versiones (-sCV) utilizando una tasa minima de envío de paquetes elevada para acelerar el proceso:

```bash
nmap -p- --open -sS -sCV --min-rate 5000 -vvv -oN targeted 10.10.11.67
```
![](/assets/img/HackTheBox/Environment/recon.png)



## Enumeracion Web

El puerto 80/tcp está corriendo un servidor nginx 1.22.1, sirviendo una página web con el título:

`Save the Environment | environment.htb`

![](/assets/img/HackTheBox/Environment/80.png)

Este es el primer punto de entrada accesible públicamente, por lo que se procede a realizar una revisión manual y luego automatizada del sitio.

#### Navegación Manual
Al ingresar a http://environment.htb, se carga una página simple con temática ecológica. A primera vista, no hay formularios de login ni enlaces evidentes a funcionalidades administrativas.

⚠️ Se recomienda revisar el contenido fuente de la página (Ctrl+U) y observar cualquier comentario HTML o ruta interna interesante (robots.txt, rutas ocultas, JS, etc.).



![](/assets/img/HackTheBox/Environment/techstack.png)
