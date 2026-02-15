 -------------------
 - Dificultad: Easy
 - OS: LInux

![](../Adjuntos/Pasted%20image%2020260214210056.png)

--------------------------

# Reconocimiento

### 1. Comprobación de conectividad

Se comienza la fase de   reconocimiento con un ping a la máquina objetivo. De esta forma se comprueba que existe conectividad con la máquina y que su S.O es Linux (TTL 64).

```bash
ping -c 1 172.17.0.2
```

![](../Adjuntos/Pasted%20image%2020260214210122.png)
### 2. Escaneo de puertos

Con nmap se realiza un escaneo de todos los puertos TCP mediante un escaneo TCP SYN.

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2 -oG alltcp_ports.txt
```

![](../Adjuntos/Pasted%20image%2020260214210133.png)

Se procede a  un segundo escaneo sobre los puertos descubiertos para detectar versión de los servicios y lanzar una serie de scripts de reconocimiento sobre dichos servicios.

```bash
nmap -p22,80 -sCV 172.17.0.2 -oN tcp_ports.txt
```

![](../Adjuntos/Pasted%20image%2020260214210147.png)

Con este escaneo de puertos se descubre que hay dos servicios corriendo en la máquina:
- 22/tcp SSH
- 80/tcp HTTP

Además, el sistema que se está utilizando es un Debian.

### 3. Acceso inicial al servicio HTTP

Al acceder a el servicio HTTP lo primero que aparece es un panel de login, el cuál después de probar credenciales comunes como admin/admin no se consigue acceder.

![](../Adjuntos/Pasted%20image%2020260214210157.png)

Se prueba a añadir  una comilla en el usuario y aparece un error lo que puede ser un indicio a una vulnerabilidad SQLI ya que el input del usuario se está insertando directamente en la query SQL.

![](../Adjuntos/Pasted%20image%2020260214210208.png)

# Explotación

Se prueba  admin' or 1=1-- - y se acontece un bypass  de autenticación.

![](../Adjuntos/Pasted%20image%2020260214210216.png)

![](../Adjuntos/Pasted%20image%2020260214210224.png)

Esto redirige a un archivo page.php en donde si se aporta el nombre de una ciudad  devuelve el clima.

![](../Adjuntos/Pasted%20image%2020260214210232.png)

El panel de login es vulnerable a  SQLi por lo que se procede a interceptar la petición con burpsuite para despues pasarsela a sqlmap y dumpear la información contenida.

```
sqlmap -r request --dbs --batch --dump
```

![](../Adjuntos/Pasted%20image%2020260214210241.png)

Se descubren las credenciales de 4 usuarios en texto plano:
- admin:chocolateadministrador
- lucas:lucas
- agustin:soyagustin123
- directorio:directoriotravieso

Se prueban las credenciales obtenidas de la base de datos para  acceder por SSH sin éxito.

![](../Adjuntos/Pasted%20image%2020260214210252.png)

Uno de los usuarios tiene como password "directoriotravieso", por lo que mediante una petición por GET se comprueba si existe dicho directorio.

![](../Adjuntos/Pasted%20image%2020260214210303.png)

Dicho directorio tiene un archivo .jpg el cuál nos descargamos con  wget para analizar en local los metadatos en busca información o si se esta implementando esteganografía.

```bash
wget http://172.17.0.2/directoriotravieso/miramebien.jpg
```

![](../Adjuntos/Pasted%20image%2020260214210314.png)

Mediante el uso de la herramienta exiftool se comprueban  los metadatos y no se obtiene información relevante.

![](../Adjuntos/Pasted%20image%2020260214210323.png)

Se usa la herramienta  steghide para comprobar si se esta implementando esteganografía y solicita un salvoconducto. Se prueba con las contraseñas obtenidas previamente en la base de datos sin éxito.

![](../Adjuntos/Pasted%20image%2020260214210335.png)

 Se procede a crear  un pequeño script para realizar fuerza bruta y comprobar si se obtiene  una contraseña valida.

![](../Adjuntos/Pasted%20image%2020260214210350.png)

Se ejecuta el script y se encuentra la contraseña "chocolatito"

![](../Adjuntos/Pasted%20image%2020260214210400.png)

Con la herramienta file se comprueba el tipo de archivo obtenido de la imagen. Es un .zip.

![](../Adjuntos/Pasted%20image%2020260214210408.png)

Se procede  a descomprimir pero requiere una contraseña.

![](../Adjuntos/Pasted%20image%2020260214210414.png)

Se extrae el hash asociado a la contraseña para posteriormente intentar romperla con john the ripper.

```bash
zip2john prueba.zip > hash
```

![](../Adjuntos/Pasted%20image%2020260214210425.png)

Procedemos a romper la contraseña con John the Ripper.

```bash
john --wordlist=/usr/share/rockyou.txt hash
```

![](../Adjuntos/Pasted%20image%2020260214210434.png)

Descomprimimos el archivo con la contraseña encontrada y obtenemos un archivo con unas credenciales:

![](../Adjuntos/Pasted%20image%2020260214210444.png)

Se prueba a entrar por SSH con esas credenciales y se consigue acceso a la máquina.

![](../Adjuntos/Pasted%20image%2020260214210452.png)

# Escalada de privilegios

Se procede a listar  los grupos en los que está el usuario carlos y no se encuentra ninguno interesante.

![](../Adjuntos/Pasted%20image%2020260214210512.png)

Se realiza una búsqueda de archivos con permiso SUID desde la raíz:

```bash
find / -perm -400 2>/dev/null
```

Encontramos que el binario find tiene permisos SUID. Este binario se puede utilizar para escalar privilegios.

![](../Adjuntos/Pasted%20image%2020260214210520.png)

Mediante la ayuda de GTFOBins vemos que podemos darnos una shell con privilegios del usuario root. 

```bash
find . -exec /bin/bash -p \; -quit
```

![](../Adjuntos/Pasted%20image%2020260214210533.png)



De esta forma se consigue privilegios de root y con ello el control total de la máquina.

# Mitigaciones

- Para evitar inyecciones SQL utilizar consultas preparadas con parámetros en lugar de usar directamente los valores del usuario en la consulta SQL.
- Evitar exponer información sensible en servicios web.
- Evitar el uso de contraseñas débiles que puedan ser fácilmente obtenidas mediante el uso de diccionarios públicos como rockyou.txt
- Aplicar filosofía de privilegio mínimo y conceder a los usuarios del sistema los permisos justos para sus tareas.
- Evitar prácticas de "seguridad por oscuridad" implementando controles de seguridad sólidos y auditables, garantizando que el sistema sea seguro incluso si su funcionamiento interno es conocido.




