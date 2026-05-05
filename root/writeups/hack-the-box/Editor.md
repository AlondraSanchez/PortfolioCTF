# Resumen

**Nombre del reto:** [Editor](https://app.hackthebox.com/machines/Editor)

**Dificultad:** Fácil

**Objetivos:**
1. Conseguir acceso como usuario y obtener la `User Flag` 
2. Escalar privilegios y obtener la `Root Flag`
# Reconocimiento

El primer paso fue realizar tareas de **reconocimiento** utilizando `nmap`, con el objetivo de identificar puertos abiertos y servicios activos en la máquina. Comencé con un escaneo completo de puertos:

```bash
nmap -sV -sC -p- TARGET_IP
```

El resultado arrojó la existencia de una web alojada en el puerto 80, la cual redirige a la url `http://editor.htb/`, así como la existencia de un servidor Jetty, aparentemente versión `10.0.20`

![Escaneo con nmap](root/assets/editor/SS1.png)

Para acceder a la web es necesario agregar el dominio en el archivo hosts. Una vez hecho esto, se muestra una web.

![Web alojada en servidor](root/assets/editor/SS2.png)

Lo interesante se encuentra en el apartado `Docs` que redirige a una web `XWiki`.
Al observar el pie de página, se muestra información de la tecnología que sirve esta web, específicamente que se trata de `XWiki Debian 15.10.8`

![XWiki](root/assets/editor/SS3.png)

# Acceso al sistema

Esta versión de `Xwiki` se ve afectada por la vulnerabilidad [CVE-2025-24893](https://www.incibe.es/index.php/incibe-cert/alerta-temprana/vulnerabilidades/cve-2025-24893), donde un invitado puede ejecutar código de forma remota.

En mi caso, aproveche dicha vulnerabilidad utilizando el siguiente [exploit](https://github.com/dollarboysushil/CVE-2025-24893-XWiki-Unauthenticated-RCE-Exploit-POC?tab=readme-ov-file), ejecutándolo desde mi máquina atacante, colocando los parámetros solicitados y abriendo un puerto para recibir la *reverse shell*.

![[root/assets/editor/SS4.png]]
![[root/assets/editor/SS5.png]]

Una vez dentro de la máquina, utilicé un script para mejorar el entorno y hacer un poco más fácil la navegación

```bash
script /dev/null -c bash
```

![[root/assets/editor/SS6.png]]

Tras revisar los diferentes archivos de configuración accesibles con el usuario `xwiki`, encontré un grupo de credenciales.

![[root/assets/editor/SS7.png]]

Con las credenciales obtenidas es posible acceder a la base de datos, sin embargo, tras inspeccionarla no encontré información que pudiera conducirme hacia la captura de las banderas.

Al revisar el sistema, encontré la existencia de un usuario.

![[root/assets/editor/SS8.png]]

El siguiente paso fue probar las contraseñas encontradas anteriormente con el usuario existente en el sistema mediante SSH, y la respuesta fue positiva. De este modo conseguí acceso al sistema como usuario.

![[root/assets/editor/SS9.png]]

Al igual que en otros retos de este tipo, la `user flag` se encontraba en el `home` del usuario.

![[SS10.png]]

**Primera bandera capturada 🎉**

# Escala de privilegios

Para escalar privilegios, revisé si existían binarios con permisos SUID de los cuales pudiera aprovecharme, para lo cual utilicé el comando

```bash
find / -type f -perm -04000 -ls 2>/dev/null
```

El resultado mostró una lista de binarios inusuales, provenientes de la herramienta netdata.

> **Netdata** es una herramienta para visualizar y monitorear métricas en tiempo real, optimizada para acumular todo tipo de datos, como uso de CPU, actividad de disco, consultas SQL, visitas a un sitio web, etc.
*Fuente: Wikipedia*

![[SS11.png]]

En algunas versiones de **NetData**, existe una vulnerabilidad ([CVE-2024-32019](https://nvd.nist.gov/vuln/detail/CVE-2024-32019)) que permite a un atacante ejecutar programas arbitrariamente con permisos de super usuario.

Para aprovechar dicha vulnerabilidad, utilicé el siguiente [exploit](https://github.com/AliElKhatteb/CVE-2024-32019-POC), cuyo funcionamiento consiste en un código escrito en lenguaje C, el cual, al ejecutarse, se autoconfigura para ejecutarse con permisos de administrador, y después abre una shell reversa con el usuario **root**.
 
![[SS12.png]]

Para asegurar el funcionamiento del exploit, lo descargué, edité y compilé siguiendo las instrucciones del autor en mi máquina atacante. Una vez compilado lo envié hacia la máquina víctima por medio de `scp`.

![[SS13.png]]

Una vez teniendo el binario en la máquina víctima, sólo queda ejecutarlo correctamente: Primero debe abrirse un puerto a la escucha en la máquina atacante, donde se recibirá la shell reversa, y posteriormente se lanza el binario en la máquina víctima.

Como usuario root, es posible leer la flag ubicada en `/root/root.txt`.

![[SS14.png]]

**Segundo objetivo conseguido 🎉**

# Conclusiones

Este reto demuestra principalmente la potencial amenaza que representa el uso de tecnologías desactualizadas, pero también nos refleja uno de los hábitos más comunes y menos seguros: la reutilización de contraseñas.

La máquina **Editor** me ha servido para fortalecer mi habilidad en la búsqueda de vulnerabilidades conocidas, y aunque en esta ocasión me he tenido que valer de *exploits* creados por otras personas, estudiarlos me ayuda también a entrenar mi capacidad de comprender código para cuando llegue el momento de programar mis propias herramientas.