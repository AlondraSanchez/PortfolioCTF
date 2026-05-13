**Category:** Misc

*Nuestro equipo de inteligencia ha interceptado un archivo sospechoso llamado MasonJar.jar. A simple vista, parece una aplicación inofensiva que requiere una clave para entrar, pero el creador no nos ha proporcionado el acceso.*

*¿Cuál es la flag que se muestra tras introducir la contraseña correcta en el ejecutable Java?*

**Provided resources:**
- JAR file `MasonJar.jar`

---

# Solution

## Static Analysis

Since the challenge provided a Java executable (`.jar`), the first step was to inspect its contents before executing it.  

JAR files can be decompiled using tools such as `jd-gui`, allowing recovery of the original Java source code and enabling static analysis of the application logic.

![MasonJar decompiled](../../../assets/ctf-jnic-2026/MJP1_SS1.png)

The decompiled code revealed a simple authentication flow: the application displays a graphical interface containing an input field and a button.  
  
When the hardcoded string: "AbreElTarroYCogeLaMermelada" is submitted, the program reveals a Base64-encoded value.

This indicates that the application does not implement any real authentication mechanism, but instead relies on client-side validation with a **hardcoded** password.

## Flag recovery

The resulting Base64 string was decoded using standard decoding utilities such as `base64`.

![Decoded text](../../../assets/ctf-jnic-2026/MJP1_SS2.png)

Decoding the value revealed the final flag.

This challenge highlights the risks of relying on client-side validation and hardcoded secrets within distributed applications, as static analysis can easily expose sensitive logic and embedded credentials.