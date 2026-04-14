### **Tutorial: Explotación de la máquina Library en TryHackMe**


## 1. Conexión a la VPN de TryHackMe

Para poder acceder a las máquinas del laboratorio es necesario conectarse primero a la VPN de TryHackMe. Esto crea un túnel cifrado entre la máquina Kali y la red privada del laboratorio.

### 1.1 Conexión mediante OpenVPN

Desde la terminal de Kali ejecutamos el siguiente comando utilizando el archivo .ovpn descargado desde la plataforma:

```bash
sudo openvpn /home/nerea/Descargas/eu-central-1-nereacandonramos-regular.ovpn
```

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

## 3. Escaneo de puertos con Nmap

El siguiente paso consiste en identificar los servicios expuestos en la máquina objetivo utilizando Nmap, una herramienta fundamental para el reconocimiento en auditorías de seguridad.

Se ejecuta el siguiente comando:

```bash
nmap -sC -sV -A -O 10.114.169.81
```
![nmap](Maquina_de_Linux/Library/imagenes/nmap.png)

Los parámetros utilizados permiten obtener información detallada del sistema:

- sC ejecuta los scripts básicos de Nmap para enumeración.

- sV detecta la versión de los servicios que están ejecutándose en los puertos abiertos.

- A activa la detección avanzada de servicios, scripts adicionales y traceroute.

- O intenta identificar el sistema operativo de la máquina objetivo.

Este tipo de escaneo proporciona una visión completa de la máquina antes de comenzar la fase de explotación.

## 4. Acceso a la página web

Tras identificar mediante el escaneo de Nmap que el puerto 80 (HTTP) se encuentra abierto, el siguiente paso consiste en acceder al servidor web desde el navegador.

Para ello introducimos la dirección de la máquina víctima en el navegador:

```bash
http://10.114.169.81:80
```

Al cargar la página se muestra un sitio web con el título:

```bash
Welcome to Blog - Library Machine
```
![pagina](Maquina_de_Linux/Library/imagenes/pagina.png)

Se trata de un blog sencillo relacionado con una biblioteca. Aunque la página es simple, proporciona un punto de partida para continuar con la fase de enumeración del servidor web.

Durante la exploración inicial del sitio también se puede observar que aparece mencionado el usuario:

```bash
meliodas
```

Este tipo de información puede resultar muy útil durante la fase de ataque, ya que los nombres de usuario suelen reutilizarse para servicios como SSH.

## 5. Enumeración del archivo robots.txt

Durante el escaneo con Nmap se detectó que el servidor web contiene un archivo robots.txt. Este tipo de archivo se utiliza para indicar a los motores de búsqueda qué partes del sitio web no deben indexar.

Para revisarlo se accede a la siguiente ruta desde el navegador:

```bash
http://10.114.169.81/robots.txt
```
![robots](Maquina_de_Linux/Library/imagenes/robots.png)

El archivo muestra las rutas que el administrador del sitio ha marcado como no indexables por los buscadores. Sin embargo, en un proceso de enumeración de seguridad, estos archivos pueden revelar directorios o información interesante sobre la estructura del sitio web.

La revisión de este archivo forma parte del proceso de reconocimiento del servidor web, ya que puede proporcionar pistas sobre posibles recursos ocultos o restringidos.

## 6. Ataque de fuerza bruta con Hydra

Durante el escaneo inicial se identificó que el puerto 22 (SSH) está abierto en la máquina objetivo. Además, durante la exploración del sitio web se encontró el posible nombre de usuario meliodas.

Con esta información se puede intentar descubrir la contraseña mediante un ataque de fuerza bruta utilizando la herramienta Hydra y el diccionario de contraseñas rockyou.txt.

El comando utilizado es el siguiente:

```bash
hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://10.114.169.81
```
![hydra](Maquina_de_Linux/Library/imagenes/hydra.png)

Explicación de los parámetros utilizados:

- -l indica el usuario que se va a probar.

- -P especifica el diccionario de contraseñas.

- ssh:// define el servicio objetivo.

- 10.114.169.81 es la dirección IP de la máquina víctima.

Tras ejecutar el ataque, Hydra prueba múltiples combinaciones de contraseñas hasta encontrar una válida para el usuario meliodas.

Una vez encontrada la contraseña correcta, es posible utilizarla para acceder al sistema mediante SSH.

## 7. Acceso al sistema mediante SSH

Una vez obtenidas las credenciales del usuario meliodas, se procede a acceder al sistema utilizando el protocolo SSH.

El comando utilizado es el siguiente:

```bash
ssh meliodas@10.114.169.81
```
![ssh](Maquina_de_Linux/Library/imagenes/ssh.png)

El sistema solicita la contraseña del usuario. Introducimos la contraseña obtenida anteriormente con Hydra:

```bash
iloveyou1
```


## 8. Enumeración para escalada de privilegios

Una vez obtenido acceso al sistema como el usuario meliodas, el siguiente objetivo es intentar conseguir privilegios de administrador (root).

Para ello se comprueba qué comandos puede ejecutar el usuario con sudo utilizando el siguiente comando:

```bash
sudo -l
```

Este comando muestra los programas que el usuario puede ejecutar con privilegios elevados.

Esto indica que el usuario meliodas puede ejecutar el archivo:

```bash
/home/meliodas/bak.py
```
![sudo](Maquina_de_Linux/Library/imagenes/sudo.png)

utilizando Python con privilegios de root, y además sin necesidad de introducir contraseña (NOPASSWD).

Este tipo de configuración es peligrosa porque permite ejecutar un script con privilegios elevados. Si el usuario tiene permisos para modificar ese archivo, puede introducir código malicioso y ejecutarlo como root.

## 9. Modificación del script bak.py

Dado que el archivo bak.py se encuentra en el directorio del usuario meliodas, es posible modificar su contenido.


Primero se elimina el archivo original:

```bash
rm bak.py
```
No saldrá si deseamos eliminar el archivo:

```bash

y
```
Después se crea uno nuevo:

```bash
nano bak.py
```
![bak](Maquina_de_Linux/Library/imagenes/bak.png)

Dentro del archivo se añade el siguiente código:

```bash
import os
os.system("/bin/bash")
```
![scripts](Maquina_de_Linux/Library/imagenes/archivo.png)

Este código ejecuta una shell del sistema.

Se guardan los cambios y se cierra el editor.

## 10. Escalada de privilegios a root

Una vez modificado el archivo bak.py con el código que ejecuta una shell del sistema, el siguiente paso consiste en ejecutarlo utilizando sudo.

Recordemos que anteriormente se comprobó con el comando sudo -l que el usuario meliodas puede ejecutar este script con privilegios de administrador sin necesidad de introducir contraseña.

Para ello se ejecuta el siguiente comando:



```bash
sudo /usr/bin/python3 /home/meliodas/bak.py
```

Al ejecutarse el script, se abre una nueva shell con privilegios elevados. Esto ocurre porque el script se está ejecutando como root.

Para verificar que se han obtenido los máximos privilegios del sistema se utiliza el comando:

```bash
whoami
```

Si la escalada de privilegios ha sido exitosa, el resultado será:

```bash
root
```
![root](Maquina_de_Linux/Library/imagenes/root.png)

Esto confirma que se ha conseguido acceso completo al sistema.
