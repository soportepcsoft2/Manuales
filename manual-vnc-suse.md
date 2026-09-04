# Manual: Configurar VNC Server en SUSE para acceso remoto desde Windows

Este manual documenta paso a paso cómo instalar, configurar y dejar funcionando de forma permanente un servidor VNC en SUSE Linux, accesible desde una computadora Windows, incluyendo la solución a los problemas más comunes (locks huérfanos, firewall, GNOME Shell fallando, y persistencia al reiniciar).

---

## 1. Instalación del servidor VNC

**¿Qué hace esto?** Instala el paquete que permite compartir el escritorio de Linux por red usando el protocolo VNC.

```bash
sudo sudo zypper install tigervnc xorg-x11-Xvnc
```

En SUSE también puedes hacerlo desde una interfaz gráfica con **YaST > Software Management**.

---

## 2. Configurar un entorno de escritorio

**¿Qué hace esto?** VNC por sí solo no dibuja ventanas ni menús — necesita un gestor de ventanas o un entorno de escritorio completo que se lance dentro de la sesión virtual.

El archivo que controla qué se lanza al iniciar la sesión VNC es:
```
~/.vnc/xstartup
```
(o `/root/.vnc/xstartup` si usas el usuario root).

> ⚠️ **Nota importante:** GNOME Shell (el escritorio completo por defecto de SUSE) generalmente **falla dentro de una sesión VNC** porque necesita aceleración 3D que un framebuffer virtual (Xvnc) no puede ofrecer. El error típico es una pantalla de *"Algo salió mal"*. Por eso, en este manual usamos **IceWM**, un gestor de ventanas mucho más liviano y estable en este contexto, y lanzamos **GNOME Terminal** encima para seguir teniendo una terminal familiar.

---

## 3. Establecer una contraseña de acceso VNC

**¿Qué hace esto?** Crea una contraseña *distinta* a la del sistema operativo, exclusiva para las conexiones VNC.

```bash
vncpasswd
```

Esto guarda la contraseña cifrada en `~/.vnc/passwd`.

---

## 4. Primer inicio manual del servidor (prueba inicial)

**¿Qué hace esto?** Levanta manualmente una sesión VNC para probar que todo funciona antes de dejarlo permanente.

```bash
vncserver :2
```

El número después de los dos puntos (`:2`) es el **número de display**. Se traduce a un puerto de red sumando 5900 + número de display:

| Display | Puerto |
|---|---|
| `:1` | 5901 |
| `:2` | 5902 |

**Comprobación — ver qué sesiones están corriendo:**
```bash
vncserver -list
```

---

## 5. Problema común: "A VNC server is already running as :N"

**¿Por qué pasa esto?** Cuando una sesión VNC se cierra de forma abrupta (crash, corte de energía, etc.), quedan archivos de bloqueo ("lock files") que engañan al sistema haciéndolo creer que el servidor sigue activo, aunque no haya ningún proceso real corriendo.

**Comprobación — confirmar si realmente hay un proceso activo:**
```bash
ps aux | grep Xvnc
```
Si el display en conflicto (ej. `:1`) **no aparece** en esta lista, es un lock huérfano y es seguro limpiarlo.

**Solución — limpiar los archivos huérfanos:**
```bash
# Intento rápido primero:
vncserver -kill :1

# Si no alcanza, borrar manualmente:
sudo rm /tmp/.X11-unix/X1
sudo rm /tmp/.X1-lock
```

> 💡 Nota: dentro de `/tmp/.X11-unix/` el archivo se llama simplemente `X1` (sin punto delante), mientras que en `/tmp/` directamente es `.X1-lock` (con punto). Son dos archivos distintos.

---

## 6. Abrir el puerto en el firewall

**¿Qué hace esto?** Por defecto, el firewall de SUSE bloquea puertos que no estén explícitamente permitidos. Aunque el servidor VNC esté corriendo, sin este paso las conexiones externas nunca llegarán.

```bash
sudo firewall-cmd --add-port=5902/tcp --permanent
sudo firewall-cmd --reload
```

**Comprobación — confirmar que el puerto quedó abierto:**
```bash
sudo firewall-cmd --list-ports
```

**Comprobación — confirmar que el servicio realmente escucha en la red (no solo localhost):**
```bash
ss -tlnp | grep 5902
```
Debe mostrar `0.0.0.0:5902` (todas las interfaces). Si mostrara `127.0.0.1:5902`, significaría que solo acepta conexiones locales.

---

## 7. Instalar un cliente VNC en Windows

Descarga e instala **TigerVNC Viewer** (gratuito y compatible) u otra alternativa como RealVNC Viewer.

**Conectarte desde Windows:**
```
IP_DEL_SERVIDOR:2
```
o con el puerto completo:
```
IP_DEL_SERVIDOR:5902
```

---

## 8. Diagnosticar problemas de conexión (timeout / no conecta)

Si Windows muestra un error de **timeout** (código 10060) al intentar conectar, el problema casi siempre está en la red, no en VNC.

**Comprobaciones en orden:**

1. **Ping básico** (desde CMD en Windows):
   ```
   ping IP_DEL_SERVIDOR
   ```
2. **Firewall del servidor** — repetir el paso 6.
3. **¿El servicio escucha en la IP correcta?**
   ```bash
   ss -tlnp | grep 5902
   ```
4. **¿Es una máquina virtual?** Si SUSE corre en VirtualBox/VMware/Hyper-V, revisa el modo de red de la VM. Si está en modo **NAT**, la IP interna no es alcanzable desde tu PC Windows directamente — necesitas modo **Bridged/Puente** (o configurar port forwarding).

---

## 9. Hacerlo permanente: iniciar VNC automáticamente al reiniciar

**¿Por qué es necesario esto?** El comando `vncserver :2` que ejecutaste manualmente **no sobrevive a un reinicio**. Para que la sesión VNC se levante sola cada vez que el servidor arranca, hay que registrarla como un **servicio systemd**.

### 9.1. Crear el archivo de servicio

Archivo: `/etc/systemd/system/vncserver@.service`

```ini
[Unit]
Description=Remote desktop VNC service for display %i
After=syslog.target network.target

[Service]
Type=forking
User=root
WorkingDirectory=/root
PIDFile=/root/.vnc/%H%i.pid
ExecStartPre=-/usr/bin/vncserver -kill %i > /dev/null 2>&1
ExecStart=/usr/bin/vncserver %i
ExecStop=/usr/bin/vncserver -kill %i

[Install]
WantedBy=multi-user.target
```

**Cómo crearlo con `vi`:**

```bash
sudo vi /etc/systemd/system/vncserver@.service
```

Una vez dentro de `vi`:

1. Presiona la tecla **`i`** para entrar en **modo inserción** (verás algo como `-- INSERT --` abajo). Sin esto, `vi` interpreta lo que escribas como comandos, no como texto.
2. Escribe o pega el contenido completo:
   ```ini
   [Unit]
   Description=Remote desktop VNC service for display %i
   After=syslog.target network.target

   [Service]
   Type=forking
   User=root
   WorkingDirectory=/root
   PIDFile=/root/.vnc/%H%i.pid
   ExecStartPre=-/usr/bin/vncserver -kill %i > /dev/null 2>&1
   ExecStart=/usr/bin/vncserver %i
   ExecStop=/usr/bin/vncserver -kill %i

   [Install]
   WantedBy=multi-user.target
   ```
3. Presiona **`ESC`** para salir del modo inserción y volver al modo comando.
4. Escribe **`:wq`** y presiona **Enter** para **guardar y salir** (`w` = write/guardar, `q` = quit/salir).

> 💡 Si te equivocas y quieres salir **sin guardar** los cambios, usa `ESC` y luego `:q!` en su lugar.

**Comprobación — confirmar que el archivo se guardó bien:**
```bash
cat /etc/systemd/system/vncserver@.service
```

> ⚠️ Nota sobre `PIDFile`: `%H%i` arma el nombre como `localhost:2`, coincidiendo con el archivo `localhost:2.pid` que TigerVNC crea normalmente en `~/.vnc/`.

### 9.2. Habilitar e iniciar el servicio

```bash
sudo systemctl daemon-reload
vncserver -kill :2
sudo systemctl enable --now vncserver@:2.service
```

**Comprobación — ver el estado del servicio:**
```bash
sudo systemctl status vncserver@:2.service
```
Debe decir `active (running)`.

### 9.3. Prueba real con reinicio

```bash
sudo reboot
```
Después de reiniciar, sin ejecutar nada manualmente en el servidor, intenta conectarte desde Windows.

---

## 10. Problema: GNOME se cae ("Algo salió mal") solo al iniciar por systemd

**¿Por qué pasa esto?** Cuando ejecutabas `vncserver :2` manualmente desde una sesión SSH, heredabas un entorno completo (variables como `XDG_RUNTIME_DIR`, bus de D-Bus, sesión PAM). Cuando **systemd** arranca el servicio automáticamente al iniciar el sistema, no existe esa sesión de login completa — y GNOME Shell, que depende de ese entorno (además de necesitar aceleración 3D), falla.

**Intento de solución (agregar sesión PAM completa):**

En `/etc/systemd/system/vncserver@.service`, dentro de `[Service]`, agregar:
```ini
PAMName=login
```
Luego:
```bash
sudo systemctl daemon-reload
sudo systemctl restart vncserver@:2.service
```

Si **aún así GNOME sigue fallando**, el problema es la falta de aceleración 3D, no el entorno — y la solución real es reemplazar GNOME por un entorno más liviano (siguiente paso).

---

## 11. Solución definitiva: cambiar a IceWM + GNOME Terminal

**¿Por qué IceWM?** Es un gestor de ventanas minimalista que no requiere aceleración 3D ni composición gráfica avanzada, por lo que funciona de forma estable dentro de sesiones VNC. Como ya viene instalado por defecto en muchas instalaciones de SUSE, no requiere agregar repositorios nuevos.

### 11.1. Editar el archivo `xstartup`

Archivo: `/root/.vnc/xstartup`

```bash
sudo vi /root/.vnc/xstartup
```

Dentro de `vi`:

1. Si el archivo ya tiene contenido y quieres borrarlo todo antes de escribir el nuevo, en **modo comando** (sin haber presionado `i` todavía) escribe `:%d` y presiona Enter — esto borra todas las líneas del archivo.
2. Presiona **`i`** para entrar en **modo inserción**.
3. Escribe o pega:
   ```bash
   #!/bin/sh

   export SHELL=/bin/bash
   [ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources

   # Iniciar sesión de D-Bus (necesaria para que gnome-terminal pueda lanzarse)
   eval $(dbus-launch --sh-syntax --exit-with-session)

   # Cargar el escritorio IceWM en segundo plano
   icewm-session &

   # Esperar un momento a que el WM termine de cargar antes de abrir la terminal
   sleep 2

   # Abrir la terminal de GNOME
   gnome-terminal &

   wait
   ```
4. Presiona **`ESC`** para salir del modo inserción.
5. Escribe **`:wq`** y presiona **Enter** para guardar y salir.

**Aplicar permisos de ejecución (obligatorio, o el script no se ejecutará):**
```bash
sudo chmod +x /root/.vnc/xstartup
```

### 11.2. Reiniciar el servicio y probar

```bash
sudo systemctl restart vncserver@:2.service
```

Conéctate desde Windows: deberías ver IceWM cargar y, dos segundos después, la ventana de GNOME Terminal abrirse sola.

---

## Apéndice: conceptos básicos usados en este manual

### El símbolo `&` (segundo plano)
Ejecuta un comando **sin bloquear** el script — es decir, el script sigue a la siguiente línea sin esperar a que ese comando termine. Es indispensable cuando quieres lanzar varios programas "al mismo tiempo" desde un mismo script (por ejemplo, el escritorio y luego una terminal).

- **Sin `&`**: el script se queda esperando a que el comando termine antes de continuar.
- **Con `&`**: el comando corre "de fondo" y el script continúa de inmediato. La ventana del programa sigue apareciendo normalmente en pantalla — "segundo plano" se refiere al script, no a si la ventana es visible.

### El comando `wait`
Espera a que **todos** los procesos lanzados en segundo plano (con `&`) dentro del mismo script terminen. Se usa al final de `xstartup` para que la sesión no se cierre apenas termina de ejecutarse la última línea del script.

### El comando `exec`
Reemplaza el proceso actual del shell por el programa indicado, en lugar de crear un proceso hijo nuevo. Es útil como **último comando** de un script (nada se ejecuta después de un `exec`, porque el shell literalmente se convierte en ese otro programa). Por eso no se puede usar `exec` para lanzar algo y luego seguir ejecutando más líneas — solo sirve para el final del script.

### El editor `vi`
Es un editor de texto que funciona con **modos**: en **modo comando** (el que ves al abrir el archivo) las teclas se interpretan como órdenes, no como texto. Para escribir contenido, primero hay que entrar en **modo inserción** con la tecla `i`. Los comandos más usados en este manual:

| Tecla/comando | Función |
|---|---|
| `i` | Entra en modo inserción (para poder escribir) |
| `ESC` | Sale del modo inserción y vuelve al modo comando |
| `:wq` + Enter | Guarda los cambios y cierra el archivo (write + quit) |
| `:q!` + Enter | Sale sin guardar los cambios |
| `:%d` + Enter | Borra todo el contenido del archivo (en modo comando) |

### `dbus-launch`
Inicia una sesión de **D-Bus**, un sistema de comunicación entre procesos que usan muchas aplicaciones modernas (como `gnome-terminal`) para "activarse" correctamente. Sin una sesión de D-Bus activa, este tipo de aplicaciones puede fallar silenciosamente al intentar abrirse.

### Systemd y los servicios (`.service`)
Es el sistema que administra qué programas se inician automáticamente al arrancar Linux. Un archivo `.service` describe cómo iniciar, detener y supervisar un programa. Al "habilitarlo" (`enable`), le decimos a Linux que lo levante solo en cada arranque, sin intervención manual.

### Los "lock files" (archivos de bloqueo)
Son archivos que un programa crea para indicar "ya estoy usando este recurso, nadie más puede tomarlo". El problema es que si el programa se cierra mal (crash, corte de luz), a veces el archivo de bloqueo queda ahí aunque el programa ya no esté corriendo — hay que borrarlo manualmente para "liberar" el recurso.
