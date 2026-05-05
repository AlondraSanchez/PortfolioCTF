# Resumen

**Nombre del reto:** [Tortuga](https://labs.thehackerslabs.com/machine/131)

**Dificultad:** Principiante

**Objetivos:**
1. Conseguir acceso como usuario y obtener la `Flag de Usuario` 
2. Escalar privilegios y obtener la `Flag de Root`

# Reconocimiento

Para identificar la máquina y obtener su IP, realicé un escaneo ARP:

```bash
sudo arp-scan -l -I eth1
```
Donde:
- `-l`: escanea toda la red local según la configuración de la interfaz.
- `-I eth1`: especifica la interfaz de red a utilizar.

Una vez obtenida la `TARGET_IP`, procedí con un escaneo de puertos completo:

```bash
nmap -sV -sC -p- TARGET_IP
```

El resultado reveló dos puertos abiertos:
- **22** → acceso remoto por SSH
- **80** → servidor web

![Escaneo de puertos](root/assets/tortuga/SS1.png)

Sin credenciales para el acceso por SSH, el siguiente paso fue investigar el contenido del sitio web en el puerto 80.

![Acceso a web Isla Tortuga](root/assets/tortuga/SS2.png)

Se trata de un sitio estático, sin formularios ni interacción directa con el usuario. Incluso tras realizar fuzzing con diccionarios comunes y personalizados, no se revelaron rutas adicionales. La clave estaba oculta en el propio contenido narrativo del sitio.

Al acceder a la sección del mapa, aparece una frase que sugiere la existencia de un usuario accesible y una **nota** en su directorio `/home`.

![Inspección de web mapa](root/assets/tortuga/SS3.png)

# Acceso al sistema

Sin contraseñas disponibles, opté por un ataque de fuerza bruta utilizando `hydra` junto al diccionario `rockyou`:

```bash
hydra -l USUARIO -P /usr/share/wordlists/rockyou.txt ssh://TARGET_IP
```

El ataque fue exitoso y reveló una contraseña para acceder por SSH.

![Ataque de fuerza bruta](root/assets/tortuga/SS4.png)

Una vez dentro, la `Flag de Usuario` se encontraba fácilmente en el directorio personal.

![Acceso remoto y captura de primer flag](root/assets/tortuga/SS5.png)

**Primer objetivo conseguido 🎉**

Sin embargo, el sitio web insinuaba algo más. Explorando archivos ocultos en el sistema, encontré `.nota.txt`:

![Inspección de archivos ocultos](root/assets/tortuga/SS6.png)

La nota contenía una contraseña, presumiblemente perteneciente a otro usuario del sistema.

![Nota oculta](root/assets/tortuga/SS7.png)

# Escalada de privilegios

Tras cambiar de usuario, no se observó una elevación directa de privilegios, por lo que fue necesario buscar vectores de escalada.

Para ello utilicé la herramienta [linpeas](https://github.com/peass-ng/PEASS-ng), que realiza una auditoría profunda del sistema en busca de configuraciones débiles y vulnerabilidades.

Entre los resultados, se detectó una oportunidad de escalada mediante _Linux capabilities_.

![Uso de linpeas para encontrar vulnerabilidades](root/assets/tortuga/SS8.png)

**Las Linux capabilities** son fragmentos de privilegios que normalmente solo tiene el usuario root. Permiten asignar permisos específicos a binarios sin darles acceso total al sistema. Por ejemplo, en vez de permitir que un programa sea root completo, puedes darle solo la capacidad de cambiar de usuario, acceder a la red, o montar sistemas de archivos.

En este caso, el binario `/usr/bin/python3.11` tenía asignada la capability `cap_setuid`, lo que le permite modificar el UID del proceso. Esto puede explotarse para obtener una shell como `root`:

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Al ejecutar este comando, obtuve una sesión como `root`, lo que me permitió acceder a la `Flag de Root`.

![Escala de privilegios y obtención de flag root](root/assets/tortuga/SS9.png)

**Segundo objetivo conseguido 🎉**

# Conclusiones

Este laboratorio pone a prueba tanto habilidades técnicas como blandas, especialmente la capacidad de leer entre líneas y deducir información a partir de pistas narrativas. Personalmente, pasé bastante tiempo en la fase de reconocimiento, enfocándome en fuzzing y búsqueda de vulnerabilidades web, cuando la clave estaba frente a mí.

La escalada de privilegios fue especialmente interesante. Como principiante, conozco pocas técnicas manuales, por lo que herramientas como _linpeas_ resultan esenciales. Eso sí, interpretar su salida requiere paciencia y atención.

De este reto me llevo una mente más abierta, nuevos conocimientos sobre sistemas Linux —como las _capabilities_— y una lección clara: a veces, la mejor pista no está en el código, sino en el contexto.