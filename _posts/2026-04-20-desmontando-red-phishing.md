---
title: Operación de Inteligencia y Caída de una Estafa de 800 Mil Víctimas
author: ch4r0niv
date: 2026-04-20 00:00:00 -0600
categories: [Investigación, Ciberseguridad]
tags: [OSINT, Phishing, Telegram, Estafa]
math: true
mermaid: true
image: /assets/img/port.jpeg
---

Hace unos días, un compañero me envió la captura de un típico mensaje de texto de estafa que le llegó a su madre. Últimamente se habla mucho de cómo identificar estas amenazas, pero esta vez decidimos ir un paso más allá: queríamos ver hasta dónde podíamos llegar tirando del hilo de este **phishing** tan mal diseñado y, de paso, ponerle un alto a estos ciberdelincuentes novatos.

En este análisis, detallaré cómo, partiendo de un engaño telefónico muy básico, logramos infiltrarnos en el grupo privado de Telegram de los atacantes. Logramos ver todos sus mensajes en tiempo real sin que se dieran cuenta, y todo esto abusando de un **token** que dejaron expuesto debido a un código fuente pésimamente configurado.

Descubrimos que la red estaba compuesta por **cuatro personas: dos menores de edad operando de forma local en españa y dos perfiles técnicos desde el extranjero** (un alemán y un inglés). Aunque toda esta información y la **evidencia recopilada ya ha sido entregada en manos de la policía**, el objetivo de hacer público este reporte es mostrarles **el proceso técnico paso a paso**. Así, si alguna vez reciben un mensaje fraudulento de este tipo, sabrán exactamente **cómo se analiza y cómo operan estos grupos desde las sombras**.


## La trampa: Cómo empieza todo con un simple SMS

Todo comenzó hace unos días, cuando la madre de un compañero recibió un mensaje de texto fraudulento. El SMS exigía el pago de una supuesta tasa de aduanas de 3,45 € para liberar un paquete retenido. En lugar de simplemente ignorarlo, decidimos tirar del hilo.

![Imagen: Captura del SMS pidiendo el pago de aduanas](/assets/img/cibercholos/sms.png)

## Abriendo la puerta: Superando las barreras de los estafadores

El enlace del mensaje nos dirigía a un dominio fraudulento relacionado con la gestión de paquetería. Al intentar acceder por primera vez desde el navegador, nos topamos con una barrera: un bloqueo geográfico gestionado por Cloudflare.

![Imagen: El sitio con Cloudflare bloqueando la carga de la página](/assets/img/cibercholos/cloudflared.png)

Para evadir esta restricción, utilizamos una conexión VPN, simulando que nos conectábamos desde la región permitida. Sin embargo, la página nos arrojó un nuevo error, indicando que el acceso estaba denegado y que la página solo era visible desde dispositivos móviles.

![Imagen: Mensaje de Access Denied, Mobile devices only tras usar la VPN](/assets/img/cibercholos/3-access-denied.png)

Los atacantes validaban el *User-Agent* del visitante para asegurarse de que estaba accediendo desde un teléfono. Para saltar esta segunda restricción, abrimos las Herramientas de Desarrollador del navegador (DevTools). En la pestaña de *Network Conditions*, desmarcamos la opción automática y forzamos la simulación de un teléfono "Nexus S". 

![Imagen: Pasos de configuración en DevTools simulando un teléfono Nexus S](/assets/img/cibercholos/4-devtools-config.png)

Al recargar la página, la web fraudulenta, que suplantaba a la perfección a la empresa de paquetería UPS, finalmente cargó.

![Imagen: La página de phishing de UPS cargada correctamente](/assets/img/cibercholos/5-ups-phishing.png)

## Siguiendo el juego: Enviamos datos falsos para ver qué hacían

La interfaz nos indicaba que el envío había sido interrumpido y mostraba un botón para realizar el pago de una pequeña tarifa de gestión (48 centavos en este punto de la página). 

![Imagen: Pantalla de Delivery Interrupted mostrando el botón de Pay](/assets/img/cibercholos/6-delivery-interrupted.png)

Al hacer clic, el sitio nos redirigió a un formulario muy completo donde solicitaban toda nuestra información personal: nombre, dirección, ciudad, código postal y teléfono.

![Imagen: Formulario en blanco pidiendo datos personales](/assets/img/cibercholos/7-forms-datos.png)

Para ver hacia dónde viajaba la información, decidimos completar el formulario con datos totalmente ficticios, inventando un nombre como "John Doe".

![Imagen: El formulario rellenado con los datos falsos](/assets/img/cibercholos/8-datos-falsos.png)

El siguiente paso en la pasarela de los atacantes era la captura de la tarjeta de crédito. Exigían el nombre del titular, el número de la tarjeta, la fecha de expiración y el código de seguridad (CVV).

![Imagen: Formulario pidiendo los datos de la tarjeta de crédito](/assets/img/cibercholos/9-form-tarjeta.png)

Sabiendo que estos sitios suelen tener validadores (checkers) que verifican si el formato de la tarjeta es correcto, utilizamos un servicio llamado *VCC Generator* para generar una tarjeta Mastercard virtual sin fondos, a nombre de un ficticio.

![Imagen: Página de VCC Generator con la información ficticia generada](/assets/img/cibercholos/10-vcc-generator.png)

![Imagen:](/assets/img/cibercholos/data-forms.png)

Ingresamos estos datos y le dimos a confirmar. Tal como esperábamos, en este punto el sitio simplemente se quedó cargando infinitamente; sin embargo, en el fondo, la información ya había sido enviada a los delincuentes.

![Imagen: El sitio colgado/cargando infinitamente después de dar clic a confirmar](/assets/img/cibercholos/11-sitio-cargando.png)


## Buscando pistas: Así encontramos dónde guardaban lo robado

Con nuestros datos falsos inyectados en su sistema, comenzamos a aplicar técnicas de *fuzzing* para descubrir directorios ocultos en su servidor. Rápidamente encontramos una ruta llamada `visitors.html`. Este era un registro público donde los atacantes almacenaban las direcciones IP y los *User-Agents* de todas las víctimas que entraban al enlace.

![Imagen: Archivo visitors.html mostrando IPs y User-Agents de víctimas difuminados](/assets/img/cibercholos/12-visitors-html.png)

Poco después, encontramos otra ruta aún más reveladora: un panel de estadísticas interactivo. Aquí los estafadores visualizaban sus ganancias en tiempo real. En ese momento, la gráfica mostraba que ya habían recolectado los datos de 36 tarjetas de crédito.

![Imagen: Gráfico interactivo del panel de estadísticas mostrando los robos](/assets/img/cibercholos/13-grafico-stats.png)

## Su peor error: Nos dejaron las llaves de su propio sistema

El descubrimiento que cambió el rumbo de la investigación llegó al continuar nuestra enumeración. Nos topamos con el peor error que un administrador puede cometer: dejaron olvidado en la raíz del servidor un archivo llamado `archive.zip`.

![Imagen: Visualización del archivo archive.zip encontrado en el servidor](/assets/img/cibercholos/14-archive-zip.png)

Este archivo comprimido contenía la totalidad del código fuente de su página web. Lo descargamos y lo descomprimimos desde nuestra terminal para analizar a fondo su estructura y funcionamiento interno.

![Imagen: Terminal mostrando cómo se ve la web de ellos descomprimida del ZIP](/assets/img/cibercholos/15-terminal-unzip.png)

Al navegar por la estructura de carpetas, visualizamos directorios clave como `public_html` y la carpeta `app`, donde residía la lógica principal programada en PHP.

![Imagen: Rutas internas expuestas y visualización del directorio app](/assets/img/cibercholos/16-rutas-dir-app.png)

## La llave maestra: Descubrimos cómo recibían los datos en Telegram

Al revisar los archivos PHP, notamos que no utilizaban una base de datos tradicional para almacenar las tarjetas robadas. Para entender a dónde iba la información, utilizamos el comando `grep` para buscar la palabra "telegram" dentro de todo el código fuente.

![Imagen: Terminal mostrando el resultado del comando grep 'telegram'](/assets/img/cibercholos/17-grep-telegram.png)

Esta búsqueda nos llevó directamente al archivo `panel.php`. Al abrirlo, descubrimosque usaban un bot de la aplicación Telegram para enviarse los datos de las víctimas directamente a su chat privado. Lo más grave es que dejaron escritos en texto plano el *Token* de acceso del bot y el identificador de su grupo privado (`chat_id`).

![Imagen: Fragmento de código mostrando el Token del bot y el chat_id en texto plano](/assets/img/cibercholos/18-token-chatid.png)

## Espionaje silencioso: Leyendo sus conversaciones sin ser vistos

Con el *Token* y el `chat_id` en nuestro poder, utilizamos la API oficial de Telegram (`api.telegram.org`) a través de nuestro navegador para hacer consultas manuales.

Primero usamos el método `getChat` para obtener información de su grupo. Descubrimos que se llamaba "UPS Proyecto Group". Al ver que el bot tenía permisos de lectura, usamos el método `getUpdates` para interceptar la información.

![Imagen: Paso a paso en el navegador mostrando la URL de la API de Telegram con los datos del grupo y permisos](/assets/img/cibercholos/t1.png)
![Imagen: Paso a paso en el navegador mostrando la URL de la API de Telegram con los datos del grupo y permisos](/assets/img/cibercholos/t2.png)
![Imagen: Paso a paso en el navegador mostrando la URL de la API de Telegram con los datos del grupo y permisos](/assets/img/cibercholos/t3.png)
![Imagen: Paso a paso en el navegador mostrando la URL de la API de Telegram con los datos del grupo y permisos](/assets/img/cibercholos/t4.png)

Para automatizar el espionaje y no perdernos de nada, creamos nuestro propio grupo privado de Telegram y programamos un *script* en Python. Este programa consultaba la API en tiempo real y reenviaba (forwarding) cada mensaje del grupo de los atacantes directamente a nuestro grupo.

Nos habíamos convertido en fantasmas dentro de sus comunicaciones.

![Imagen: Nuestro grupo de Telegram recibiendo los mensajes reenviados por el script](/assets/img/cibercholos/20-grupo-script.png)

Decidimos exportar todo este registro en carpetas organizadas para preservar la evidencia antes de que pudieran borrar algo. Esta es la estructura de los datos que posteriormente se entregaría a la policía.

![Imagen: La data empaquetada que se compartió a la policía](/assets/img/cibercholos/21-data-policia.png)

Además, levantamos un servidor HTTP local para poder leer cómodamente el historial completo de sus conversaciones a través de un archivo `messages.html` generado por nuestra exportación.

![Imagen: Registro web en el navegador de todo lo que los atacantes han hablado](/assets/img/cibercholos/22-registro-web.png)

## Viéndolo todo desde adentro: Su gigantesca lista de víctimas y quiénes eran en realidad

Lo que leímos durante los siguientes días fue alarmante. En el chat, compartieron un archivo de texto (`.txt`) con la lista de objetivos para su campaña de correos y SMS.

![Imagen: El archivo .txt compartido en el grupo de Telegram de los atacantes](/assets/img/cibercholos/23-txt-descarga.png)

Ese documento contenía una inmensa base de datos de números de teléfono a los cuales estaban bombardeando con mensajes.

![Imagen: Varios números de teléfono dentro del archivo, difuminados por seguridad](/assets/img/cibercholos/24-numeros-difuminados.png)

Para dimensionar el tamaño de la amenaza, utilizamos la terminal de comandos. Al contar las líneas del documento, nos topamos con la escalofriante cifra de exactamente 831.795 números de teléfono.

![Imagen: Terminal mostrando el conteo de líneas total del archivo: 831,795](/assets/img/cibercholos/25-conteo-lineas.png)

A través de sus conversaciones, perfilamos al grupo. Eran cuatro personas en total. Uno de ellos era considerado la "mano derecha" (al que le decían que sabían dónde vivía). Sin embargo, la revelación más surrealista fue al filtrar conversaciones buscando términos relacionados con la escuela.

Nos dimos cuenta de que dos de ellos eran menores de edad. Hablaban de no poder sacar el teléfono porque el profesor estaba cerca, e incluso, en su exceso de confianza, se enviaban fotografías de ellos mismos dentro del grupo.

![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/26-chats-escuela-fotos.png)
![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/e.png)
![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/e1.png)
![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/e2.png)
![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/e3.png)
![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/e6.png)

Mientras leíamos todo esto, nuestro sistema comenzó a captar cómo los datos robados de nuevas víctimas empezaban a llegar al grupo a través de su bot.

![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/e4.png)

Al ver el éxito de su campaña y el dinero entrando, los atacantes se jactaban de sus habilidades técnicas en el chat, autodenominándose como "puto pro".

![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/e5jaja.png)

Su arrogancia llegó al límite cuando comenzaron a enviar videos personales. En uno de los archivos multimedia, los pudimos ver directamente, dejando sus rostros expuestos.

![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/e7.png)

Analizamos a detalle un video grabado en una especie de cena o reunión de amigos, lo que nos permitió identificar plenamente las caras de los responsables.

![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/cara.png)

En otro de los videos filtrados en el chat, presumían su infraestructura operativa: un montón de teléfonos móviles alineados y operando con tarjetas vinculadas.

![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/phones.png)
![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/phones2.png)

## ¿En qué se gastaban el dinero robado?

¿Qué hacían con el dinero robado? En uno de los mensajes, el grupo celebró una compra exitosa confirmada: una consola PlayStation 5 valorada en 500 dólares, pagada directamente con los fondos de una de sus víctimas.

![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/play.png)
![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/playdecerca.png)

También discutían sobre sus métricas de éxito. Afirmaban que por cada lote de 1.300 mensajes enviados, lograban robar entre tres y cuatro tarjetas de crédito válidas, además de lograr intrusiones en cuentas de Google Pay.

![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/googlepay.png)

Su red no paraba de crecer. En un punto de la infiltración, compartieron un nuevo bloque masivo para futuras campañas, anunciando que acababan de pasarse otros 500.000 números nuevos.

![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/e9.png)

Finalmente, interceptamos conversaciones con los miembros extranjeros del grupo (un individuo alemán que proporcionaba apoyo técnico), discutiendo cómo al enviar 10.000 mensajes lograban sacar, como mínimo, 20 tarjetas de crédito limpias.

![Imagen: Conversaciones sobre la escuela, su mano derecha y compartiendo fotos](/assets/img/cibercholos/e10.png)

## Conclusión: Lo que aprendimos de todo esto

Exportamos el historial completo, descargamos todos sus videos, guardamos la inmensa base de datos y empaquetamos sus verdaderas identidades. Absolutamente toda esta información fue entregada a las autoridades policiales para desmantelar la red y proceder legalmente contra ellos.

![Imagen: Evidencia de los archivos compartidos a la policía](/assets/img/cibercholos/evi-files.png)

Hago público este análisis exhaustivo porque entender cómo operan desde dentro es nuestra mejor defensa. Un atacante siempre dejará un rastro si se sabe dónde buscar. Desconfíen de cualquier mensaje de texto que les exija pagos inesperados; detrás de una simple "tasa de aduanas", suele operar una maquinaria masiva de fraude lista para vaciar sus cuentas.