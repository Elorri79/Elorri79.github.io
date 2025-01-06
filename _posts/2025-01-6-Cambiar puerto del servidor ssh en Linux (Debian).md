---
layout: post
title: Cambiar puerto del servidor SSH en Linux (Mint)
---

En ocasiones, por razones de seguridad, es necesario cambiar el puerto por defecto en el que el servidor SSH escucha. El puerto predeterminado para SSH es el 22, pero cambiarlo puede ayudar a reducir la exposición a ataques automatizados. A continuación, te muestro cómo hacerlo en un sistema Linux con Mint.

¿Has editado el archivo /etc/ssh/sshd_config y no consigues cambiar el puerto de acceso al servicio SSH? ¡Yo también! pero aquí te dejo un PASO A PASO facilito que te ayudará a cambiar el dichoso puerto y evitarte perder puntos de cordura, horas de sueño y días de vida: 

### Paso a paso para cambiar el puerto de SSH:

1. **Crear el directorio de configuración adicional:**
   Primero, debemos crear un directorio donde se almacenarán las configuraciones adicionales para el servicio `ssh.socket`. Ejecuta el siguiente comando:
   
   ```
   mkdir -p /etc/systemd/system/ssh.socket.d
   ```

Este comando crea el directorio `ssh.socket.d` dentro de `/etc/systemd/system/` si no existe previamente. Este directorio es utilizado por `systemd` para almacenar configuraciones específicas de los sockets.

2. **Crear el archivo de configuración `listen.conf`:** Ahora, necesitamos crear y configurar un archivo llamado `listen.conf` en el directorio recién creado. Usamos el siguiente comando:
    
    ```
    cat >/etc/systemd/system/ssh.socket.d/listen.conf <<EOF
    [Socket]
    ListenStream=
    ListenStream=1234
    EOF
    ```
    
    Este comando hace lo siguiente:
    
    - **`ListenStream=`**: Borra cualquier valor anterior de la opción `ListenStream` que pueda estar configurado.
    - **`ListenStream=1234`**: Configura el socket de SSH para que escuche en el puerto 1234 en lugar del puerto predeterminado 22. Puedes elegir cualquier puerto disponible, pero asegúrate de que el puerto que elijas no esté en uso por otro servicio.
3. **Recargar la configuración de `systemd`:** Después de modificar los archivos de configuración, es necesario recargar `systemd` para que reconozca los cambios. Utiliza el siguiente comando:
    
    ```
    sudo systemctl daemon-reload
    ```
    
4. **Reiniciar el servicio SSH:** Finalmente, reinicia el servicio `ssh.socket` para aplicar la nueva configuración. Usa el siguiente comando:
    
    ```
    sudo systemctl restart ssh.socket
    ```
    

### Resumen:

Con estos pasos, has configurado el servicio SSH para que escuche en el puerto 1234, en lugar del puerto 22. Este cambio se realiza a través de la configuración de `systemd` en el archivo `listen.conf`, recargando los servicios y reiniciando el servicio SSH. 

Asegúrate de abrir el nuevo puerto en tu firewall y de probar la conexión SSH para verificar que todo funcione correctamente.

¡Listo! Ahora tu servidor SSH está protegido con un puerto no estándar.
