 -------------------
 - Dificultad: Medio
 - OS: Linux
![](../Adjuntos/Pasted%20image%2020260215145333.png)
-------------------

# Reconocimiento

### 1. Comprobación de conectividad

Se realiza una comprobación de conectividad usando la herramienta ping. Se comprueba que existe conectividad con la máquina objetivo y que por el TTL su S.O es Linux.

```bash
ping -c 1 172.17.0.2
```

![](../Adjuntos/Pasted%20image%2020260215145619.png)

### 2. Escaneo de puertos

Se realiza un escaneo de todos los puertos TCP mediante un TCP SYN scan usando la herramienta nmap.

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2 -oG alltcp_ports.txt
```

![](../Adjuntos/Pasted%20image%2020260215145903.png)

Se vuelve a utilizar la herramienta nmap para extraer información acerca de la versión de los servicios descubiertos y lanzar una serie de scripts de reconocimiento de nmap.

```bash
nmap -p22,80 -sCV 172.17.0.2 -oN tcp_ports.txt
```

![](../Adjuntos/Pasted%20image%2020260215150157.png)

Se descubren dos servicios corriendo en la máquina objetivo:

- 22/tcp (SSH)
- 80/tcp (HTTP)

Sistema operativo utilizado: Ubuntu

### 3. Acceso inicial al servicio HTTP

Se accede al servicio HTTP a través de un navegador web.

![](../Adjuntos/Pasted%20image%2020260215150821.png)

Se encuentra un panel de login donde mirando su código fuente se comprueba que no existe ninguna validación real y se puede pasar el login sin necesidad de introducir credenciales.

Al hacer clic en "Entrar" te redirige a un archivo index.php con la apariencia de un dashboard.

![](../Adjuntos/Pasted%20image%2020260215151352.png)

Se comprueba su código fuente y en el footer se encuentra un cádena codificada en base64 de color negro, de modo que se camufla con el background y no se ve a simple vista.

![](../Adjuntos/Pasted%20image%2020260215151536.png)

Se procede a decodificar la cadena en base64 y se obtiene la cádena: "recuerda... tu contraseña es tu usuario".

![](../Adjuntos/Pasted%20image%2020260215151758.png)

SI hacemos clic en "Cerrar sesión" nos redirige a un archivo admin.php donde aparece otro panel de login diferente.

![](../Adjuntos/Pasted%20image%2020260215151908.png)


# Explotación

En la parte inferior aparece un nombre "notadmin" el cual es un posible usuario.

Se comprueba a acceder mediante una inyección SQL poniendo como usuario notadmin@test.com y como contraseña ' OR 1=1-- -. y se obtiene acceso.

![](../Adjuntos/Pasted%20image%2020260215152623.png)
Se hace clic en "Acceso al portal2" y redirige a un archivo .php que procesa XML.

![](../Adjuntos/Pasted%20image%2020260215152816.png)

Se comprueba si es vulnerable a una inyección XXE (XML External Entity) creando una entidad externa llamada test apuntando al archivo /etc/passwd con éxito.

![](../Adjuntos/Pasted%20image%2020260215153931.png)

![](../Adjuntos/Pasted%20image%2020260215154021.png)

Con esto se consiguen dos usuarios del sistema. 

- jeremias
- ezequiel

Anteriormente vimos una cádena que decia "La contraseña es el nombre de usuario", por lo que se comprueba si aplica en uno de estos usuarios a través de SSH.

Se consigue acceso a la máquina con el usuario jeremias.

![](../Adjuntos/Pasted%20image%2020260215154234.png)

# Escalada de privilegios

En el directorio de trabajo del usuario jeremias existe un archivo python compilado donde el usuario propietario es  jeremias y tiene permiso de lectura y escritura.

![](../Adjuntos/Pasted%20image%2020260215154554.png)

Se procede a descargar el archivo en local con scp para analizarlo.

```bash
scp jeremias@172.17.0.2:/home/jeremias/ezequiel.pyc .
```


![](../Adjuntos/Pasted%20image%2020260215155821.png)

Para decompilar el archivo se utiliza la página www.pylingual.io.

![](../Adjuntos/Pasted%20image%2020260215160000.png)

Una vez se decompila el programa y se lee el código, encontramos dos variables que contienen una posible contraseña hard codeada.

![](../Adjuntos/Pasted%20image%2020260215161343.png)

Se procede a juntar los dos fragmentos y se prueban con el usuario ezequiel con éxito.

![](../Adjuntos/Pasted%20image%2020260215162141.png)

Se procede a enumerar el directorio personal del usuario ezequiel y se encuentra con un archivo .txt con el siguiente contenido:

![](../Adjuntos/Pasted%20image%2020260215162231.png)

Se listan los permisos a nivel de sudoers del usuario ezequiel y éste tiene permiso para ejecutar el binario /usr/local/bin/croc como root sin contraseña.

![](../Adjuntos/Pasted%20image%2020260215162323.png)

Croc es un binario de transferencia de archivos de forma segura.

El usuario ezequiel tiene permisos para ejecutar el binario croc como root sin contraseña, se procede a ejecutarlo poniendo a disposición el archivo /root/passw0rd_r00t.txt y desde el usuario jeremias se descarga introduciendo la contraseña que se crea para su descarga.

![](../Adjuntos/Pasted%20image%2020260215164008.png)

El archivo descargado contiene la contraseña de root que se utiliza para entrar como usuario root y de esta forma tener el control total de la máquina.

![](../Adjuntos/Pasted%20image%2020260215164249.png)

# Mitigaciones

- Validar credenciales en el servidor e implementar un control de sesión robusto.
- Para evitar inyecciones SQL utilizar consultas preparadas con parámetros en lugar de usar directamente los valores del usuario en la consulta SQL.
- No almacenar información confidencial en el código fuente de la página.
- Desactivar DTD y entidades externas en parses XML.
- No harcodear credenciales en el código de un programa.
- Aplicar el principio de mínimo privilegio en donde cada usuario tenga únicamente los privilegios necesarios para realizar sus tareas.
