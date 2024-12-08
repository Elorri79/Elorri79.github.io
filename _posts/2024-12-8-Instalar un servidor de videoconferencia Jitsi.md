---
layout: post
title: Instalar un servidor de videoconferencia Jitsi
---

Instalaremos Jitsi Meet en un servidor basado en Debian 12.

En un servidor nuevo, tendríamos que instalar Java OpenJDK, un servidor web (en esta ocasión usaremos nginx), y algunos otros paquetes necesarios para que Jitsi funcione correctamente:

```
$ sudo apt install apt-transport-https gnupg2 nginx-full sudo curl -y
```

Este comando instala varios paquetes en el sistema. Aquí tienes un desglose de cada uno:

- **`apt-transport-https`:**  
    Proporciona soporte para que APT gestione repositorios a través de HTTPS. Es útil para garantizar que las descargas de paquetes sean seguras.
    
- **`gnupg2`:**  
    Conjunto de herramientas para manejar claves GPG y criptografía. Es necesario para verificar la autenticidad de los paquetes y repositorios.
    
- **`nginx-full`:**  
    Instala la versión completa de Nginx, un servidor web y proxy inverso de alto rendimiento.
    
- **`sudo`:**  
    Permite a los usuarios ejecutar comandos como superusuario o con permisos de otro usuario.
    
- **`curl`:**  
    Herramienta de línea de comandos para transferir datos desde o hacia un servidor utilizando diversos protocolos (HTTP, HTTPS, FTP, etc.).
    
- **`-y`:**  
    Indica que se debe asumir una respuesta afirmativa (`yes`) para cualquier pregunta durante la instalación.

```
$ sudo apt install default-jdk -y
```

- **`default-jdk`:**  
    Instala el kit de desarrollo de Java (JDK) predeterminado para tu distribución de Debian. Incluye:
    
    - Herramientas para desarrollar aplicaciones en Java.
    - El compilador `javac`.
    - El entorno de ejecución de Java (JRE) necesario para ejecutar aplicaciones Java.
- **`-y`:**  
    Igual que en el comando anterior, evita que el sistema pregunte si deseas continuar con la instalación.


A continuación, nos aseguramos de tener instalado un cortafuegos y de abrir los puertos que va a necesitar el servidor de videoconferencia:

```
$ sudo apt install ufw -y
$ sudo ufw allow 22/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 443/tcp
$ sudo ufw allow 3478/udp
$ sudo ufw allow 5349/tcp
$ sudo ufw allow 10000/udp
$ sudo ufw enable
```

### Explicación de cada paso:

1. **Reglas de puertos:**
    
    - `22/tcp`: Permite conexiones SSH.
    - `80/tcp`: Permite tráfico HTTP.
    - `443/tcp`: Permite tráfico HTTPS.
    - `3478/udp`: Puerto común para STUN/TURN, cuya misión es determinar la dirección IP pública y el puerto de un cliente para que este pueda conectarse a otros clientes.
    - `5349/tcp`: Puerto común para STUN/TURN sobre TLS. Respaldo para las comunicaciones de audio y video cuando los puertos UDP están bloqueados.
    - `10000/udp`: Puerto usado por algunas aplicaciones de videoconferencia.
2. **Habilitar `ufw`:** El comando `sudo ufw enable` activa el cortafuegos, aplicando todas las reglas configuradas.

Para las aplicaciones Java se recomienda establecer un máximo de 65 000 procesos, tareas y archivos abiertos, por lo que deberemos copiar los siguientes parámetros en el archivo de configuración del sistema, en /etc/systemd/system.conf:

```
DefaultLimitNOFILE=65000
DefaultLimitNPROC=65000
DefaultTasksMax=65000
```
![terminal]({{ site.baseurl }}/images/Pasted image 20241208144407.png)

Será necesario reiniciar el servidor para que los cambios surtan efecto.

Instalamos Jitsi Meet. Para ello, lo primero es añadir los repositorios de Prosody y Jitsi, así como sus respectivas claves GPG:

```
$ echo deb http://packages.prosody.im/debian
$(lsb_release -sc) main | sudo tee -a
/etc/apt/sources.list
```

- **`echo deb http://packages.prosody.im/debian $(lsb_release -sc) main`:**  
    Genera una línea de repositorio para el archivo de fuentes de APT.
    
    - `http://packages.prosody.im/debian`: URL del repositorio de Prosody.
    - `$(lsb_release -sc)`: Sustituye este comando con el nombre de la versión de Debian actual (por ejemplo, `bookworm` o `bullseye`).
    - `main`: Es el componente del repositorio.
- **`sudo tee -a /etc/apt/sources.list`:**  
    Añade (`-a`) esta línea al archivo `/etc/apt/sources.list`, que define los repositorios de APT.


```
curl https://download.jitsi.org/jitsi-key.gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/jitsi-keyring.gpg
```

- **`curl https://download.jitsi.org/jitsi-key.gpg.key`**: Descarga la clave pública del repositorio de Jitsi.
- **`sudo gpg --dearmor -o /usr/share/keyrings/jitsi-keyring.gpg`**: Convierte la clave de texto a formato binario (`--dearmor`) y la guarda en `/usr/share/keyrings/jitsi-keyring.gpg`.


```
echo "deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/" | sudo tee /etc/apt/sources.list.d/jitsi-stable.list > /dev/null
```

- **`deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/`**: Indica que APT debe usar la clave almacenada en `/usr/share/keyrings/jitsi-keyring.gpg` para verificar los paquetes del repositorio de Jitsi.
- **`sudo tee /etc/apt/sources.list.d/jitsi-stable.list`**: Añade el repositorio a un archivo en `/etc/apt/sources.list.d/`, que es el lugar donde se almacenan los repositorios adicionales de APT.


Actualizamos la lista de paquetes e instalamos Jitsi:

```
sudo apt update
sudo apt install jitsi-meet
```

Verificaremos que todos los servicios requeridos están funcionando correctamente:

```
sudo systemctl status coturn && sudo systemctl status jicofo && sudo systemctl status jitsi-videobridge2 && sudo systemctl status prosody && sudo systemctl status nginx
```

![terminal]({{ site.baseurl }}/images/Pasted image 20241208150205.png)

![terminal]({{ site.baseurl }}/images/Pasted image 20241208150230.png)

![terminal]({{ site.baseurl }}/images/Pasted image 20241208150301.png)

![terminal]({{ site.baseurl }}/images/Pasted image 20241208150328.png)

![terminal]({{ site.baseurl }}/images/Pasted image 20241208150350.png)


Jitsi Meet está instalado, y para acceder a él, basta con configurar el sistema para que resuelva correctamente el nombre de anfitrión; por ejemplo: 

```
$ sudo hostnamectl set-hostname meet.midominio.es
$ sudo echo "127.0.0.1 meet.midominio.es" >> /etc/hosts
```

![terminal]({{ site.baseurl }}/images/Pasted image 20241208151107.png)

![terminal]({{ site.baseurl }}/images/Pasted image 20241208151202.png)

