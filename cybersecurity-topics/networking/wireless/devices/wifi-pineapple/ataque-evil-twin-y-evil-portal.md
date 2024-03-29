---
description: >-
  En este tutorial, vamos a ejecutar un ataque Evil Twin para atraer usuarios a
  una red WiFi falsa y luego usar un Evil Portal para recopilar sus
  credenciales.
cover: https://img-c.udemycdn.com/course/750x422/2948934_985b_4.jpg
coverY: 42
layout:
  cover:
    visible: true
    size: hero
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Ataque Evil-Twin y Evil-Portal

## ¿Como funciona un ataque Evil Twin?

<figure><img src="https://www.researchgate.net/publication/321122614/figure/fig5/AS:631949064421377@1527679806852/Illustration-of-an-Evil-Twin-Attack-The-attacker-can-successfully-lure-a-victim-into.png" alt=""><figcaption><p>Evil Twin</p></figcaption></figure>

Un ataque "Evil Twin" (Gemelo Malvado) es un tipo de ataque de ciberseguridad que se lleva a cabo creando un punto de acceso WiFi que imita o clona una red WiFi legítima. Este tipo de ataque tiene como objetivo engañar a los dispositivos para que se conecten a la red maliciosa en lugar de la red legítima. Una vez que un usuario se ha conectado al Evil Twin, el atacante puede llevar a cabo diversas acciones maliciosas, desde espionaje hasta ataques más elaborados como Man-in-the-Middle (MitM).

## ¿Cómo funciona un ataque Evil Portal?

Un ataque Evil Portal consiste en presentar al usuario una página de inicio o portal cautivo falso después de que se haya conectado a una red WiFi. Esta página podría imitar la de un proveedor de servicios de Internet, un hotel, un café o incluso una actualización de sistema, y por lo general solicita la entrada de datos sensibles como credenciales, números de tarjeta de crédito, etc. Los pasos generalmente involucran lo siguiente:

1. **Establecimiento de Conexión**: Primero, el atacante debe conseguir que el objetivo se conecte a una red WiFi controlada, que podría ser un Evil Twin o cualquier otro tipo de red no segura.
2. **Redirección de Tráfico**: Una vez que el usuario está conectado, el tráfico web se redirige mediante técnicas como la manipulación de DNS o reglas de iptables, forzando al usuario a una página web específica: el Evil Portal.
3. **Presentación del Portal**: El usuario ve la página de inicio falsa y, si el ataque es exitoso, introduce sus datos sensibles creyendo que está en un portal legítimo.
4. **Captura de Datos**: El portal falso captura los datos introducidos y los envía al atacante, quien puede usarlos para fines maliciosos.

## ¿Cómo redirige una red a un cliente al portal cautivo?

La redirección a un portal cautivo se realiza generalmente a través de una combinación de reglas de firewall y manipulación de DNS en el punto de acceso o el servidor que controla la red. Aquí hay un resumen de alto nivel:

1. **Reglas de Firewall**: Cuando un nuevo dispositivo se conecta a la red, las reglas de firewall redirigen todo el tráfico HTTP/HTTPS del dispositivo a la dirección IP del portal cautivo. Esto se hace a menudo usando iptables u otras herramientas similares de manipulación de paquetes.
2. **Manipulación de DNS**: Además de las reglas de firewall, se suele configurar un servidor DNS falso o manipulado para resolver todas las consultas de nombres a la dirección IP del portal cautivo. De esta manera, cuando el usuario intenta navegar a cualquier sitio web, la consulta DNS lo redirige al portal.
3. **Sesión y Autenticación**: Una vez que el usuario completa la interacción con el portal cautivo (ya sea introduciendo información o aceptando términos y condiciones), las reglas de firewall y DNS se actualizan para permitir al dispositivo acceder a la red como de costumbre.

## Módulo Evil Portal

El módulo Evil Portal nos provee de una interfaz gráfica para la configuración y despliegue de un ataque Evil-Portal de forma mucho mas automatizada. Para usarlo primero debemos instalarlo desde la sección de módulos del panel izquierdo.

Una vez instalado el módulo, debemos clonar el siguiente repositorio

<figure><img src="../../../../../.gitbook/assets/imagen (1).png" alt=""><figcaption></figcaption></figure>

* [https://github.com/kleo/evilportals](https://github.com/kleo/evilportals).

Posteriormente debemos seguir los pasos indicados en el repositorio para copiar los portales vía SCP al Wifi Pineapple en la ruta /root/portals. Al realizar esto, podremos ver los portales en la librería de portales en el módulo Evil-Portal.

<figure><img src="../../../../../.gitbook/assets/imagen (1) (1).png" alt=""><figcaption></figcaption></figure>

Ahora para habilitar el Evil Portal, debemos iniciar el servicio web y habilitar el portal al cual queremos redirigir a los clientes que se conecten al fake AP, en este caso se habilitará Starbucks.Login y por último simplemente se debe oprimir el botón Start.

<figure><img src="../../../../../.gitbook/assets/imagen (2).png" alt=""><figcaption></figcaption></figure>

## Configurar el Open AP

Lo primero que se debe realizar una vez instalado y configurado el modulo Evil Portal, es configurar el Open AP. Se debe marcar la opción para esconder el Open Access Point para evitar sospechas y también se debe marcar la opción para hacer que los AP en Spoofed AP Pool respondan a los probe requests. Con esta configuración, los clientes podran conectarse a los AP listados en el Spoofed AP Pool.

<figure><img src="../../../../../.gitbook/assets/imagen (3).png" alt=""><figcaption></figcaption></figure>

## Agregar los SSID al Spoofed AP Pool

Ahora debemos agregar los SSID que queremos personificar al Spoofed AP Pool.\
Cuando agregas un SSID al "Spoofed AP Pool" en un WiFi Pineapple, estás configurando el dispositivo para emitir ese SSID como si fuera una red WiFi legítima. En términos simples, el WiFi Pineapple empezará a transmitir el SSID especificado como una red abierta o como una red que aparenta ser legítima. De esta manera, cualquier dispositivo cercano que esté buscando esa red en particular podría conectarse al WiFi Pineapple pensando que es una red legítima.

Aquí hay una descomposición de lo que sucede a nivel técnico:

1. **Transmisión de Beacon Frames**: El WiFi Pineapple empezará a transmitir "beacon frames" con el SSID que has añadido al Spoofed AP Pool. Estos "beacon frames" son paquetes que anuncian la existencia de una red WiFi.
2. **Escaneo y Conexión**: Los dispositivos cercanos escanean en busca de redes WiFi conocidas. Si han estado conectados previamente a un SSID que coincide con el que has añadido al pool, es probable que intenten conectarse automáticamente.
3. **Establecimiento de Conexión**: Una vez que un dispositivo intenta conectarse al SSID emitido, el WiFi Pineapple puede permitir la conexión, actuando como un punto de acceso.
4. **Intercepción de Tráfico**: Ahora que el dispositivo objetivo está conectado al WiFi Pineapple, todo su tráfico de red pasa a través del dispositivo. Esto ofrece la oportunidad de llevar a cabo ataques MitM, captura de paquetes y otros tipos de actividades maliciosas o de investigación.
5. **Manipulación de Tráfico**: En esta fase, un atacante podría realizar diferentes tipos de ataques, desde simplemente espiar el tráfico hasta modificarlo para inyectar malware o redirigir al usuario a sitios web maliciosos.

En este caso vamos a agregar el SSID "Star Bucks Free Wifi"

<figure><img src="../../../../../.gitbook/assets/imagen (4).png" alt=""><figcaption></figcaption></figure>

Al escanear las redes WiFi cercanas podremos observar que aparece la red que acabamos de agregar al Spoofed AP Pool.

<figure><img src="../../../../../.gitbook/assets/imagen (5).png" alt=""><figcaption></figcaption></figure>

Cuando nos conectemos al Fake AP que acabamos de crear desde otro dispositivo, seremos redirigidos automáticamente al Evil Portal donde nos pediran credenciales para poder acceder.

<figure><img src="../../../../../.gitbook/assets/imagen (6).png" alt=""><figcaption></figcaption></figure>

Al introducir las credenciales, se enviará una notificación al Wifi Pineapple indicandonos que alguien ha caido en la trampa y podremos ver los datos que introdució en los logs del captive portal.

<figure><img src="../../../../../.gitbook/assets/imagen (7).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../../.gitbook/assets/imagen (8).png" alt=""><figcaption></figcaption></figure>
