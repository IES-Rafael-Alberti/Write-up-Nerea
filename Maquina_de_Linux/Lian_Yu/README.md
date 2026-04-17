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

### 14.1 Búsqueda de información con strings

```bash
strings aa.jpg | less
binwalk aa.jpg
exiftool aa.jpg
steghide info aa.jpg
```

## 14.2 Extracción de información oculta (STEGO)

Tras ejecutar:

```bash
steghide info aa.jpg
```

se detecta que el archivo contiene datos ocultos y solicita una contraseña.


