# Cambios en las versiones recientes de TigerVNC

Las versiones anteriores de TigerVNC tenían un script de shell llamado `vncserver`. Este script podía ejecutarse manualmente como usuario para iniciar el proceso *Xvnc*. El uso era bastante simple, ya que solo había que ejecutar:

```
$ vncserver :x [opciones de vncserver] [opciones de Xvnc]
```

y con eso bastaba. Funcionaba bien en algunos casos, pero no en todos. Había problemas cuando los usuarios querían combinarlo con *systemd*. Por eso, la implementación tuvo que cambiarse para cumplir con las reglas de *SELinux* y *systemd*.

# Cómo iniciar el servidor TigerVNC

## Agregar una asignación de usuario

Con esto puedes asignar un usuario a un puerto en particular. La asignación debe hacerse en el archivo de configuración `vncserver.users`. Es bastante sencillo: al abrir el archivo verás algunos ejemplos, pero básicamente la asignación tiene la forma:

```
:x=usuario
```

Por ejemplo, puedes tener:

```
:1=test
:2=vncuser
```

## Configurar las opciones de Xvnc

Para configurar los parámetros de Xvnc, debes ir al mismo directorio donde hiciste la asignación de usuario y abrir el archivo de configuración `vncserver-config-defaults`. Este archivo contiene la configuración predeterminada de Xvnc y se aplicará a todos los usuarios, a menos que se cumpla alguna de las siguientes condiciones:

* El usuario tiene su propia configuración en `$XDG_CONFIG_HOME/tigervnc/config` o en `$HOME/.config/tigervnc/config`.
* La misma opción con un valor distinto está configurada en el archivo `vncserver-config-mandatory`, el cual reemplaza la configuración predeterminada y tiene incluso mayor prioridad que la configuración por usuario. Esta opción está pensada para administradores de sistemas que quieran forzar determinadas opciones de *Xvnc*.

El formato del archivo de configuración también es bastante simple, ya que la configuración tiene la forma:

```
opcion=valor
opcion
```

por ejemplo:

```
session=gnome
securitytypes=vncauth,tlsvnc
geometry=2000x1200
localhost
alwaysshared
```

Consulta la siguiente página del manual para más detalles: Xvnc(1).

### Nota:

Se recomienda establecer la opción que especifica la sesión que quieres iniciar. Por ejemplo, si quieres iniciar el escritorio GNOME, debes usar:

```
session=gnome
```

Esto debe coincidir con el nombre de un archivo de escritorio de sesión ubicado en el directorio `/usr/share/xsessions`. Si no especificas la sesión, TigerVNC intentará usar la primera que encuentre, lo cual puede funcionar correctamente o no.

## Establecer la contraseña de VNC

Debes establecer una contraseña para cada usuario con el fin de poder iniciar el servidor TigerVNC. Para crear una contraseña, simplemente ejecuta:

```
$ vncpasswd
```

Debes ejecutarlo como el usuario que va a correr el servidor.

### Nota:

Si ya habías usado TigerVNC antes con tu usuario y ya habías creado una contraseña, debes asegurarte de que la carpeta (heredada, si se usó) `$HOME/.vnc` creada por `vncpasswd` tenga el contexto correcto de *SELinux*. Puedes eliminar esta carpeta y volver a crearla generando la contraseña nuevamente, o bien puedes ejecutar:

```
$ restorecon -RFv /home/<USUARIO>/.vnc
```

## Iniciar el servidor TigerVNC

Finalmente, puedes iniciar el servidor usando el servicio de systemd. Para ello, simplemente ejecuta:

```
$ systemctl start vncserver@:x
```

Ejecuta esto como usuario root, o bien:

```
$ sudo systemctl start vncserver@:x
```

Ejecútalo como usuario normal en caso de que dicho usuario tenga permisos para usar `sudo`. No olvides reemplazar `:x` por el número real que configuraste en el archivo de asignación de usuarios. Por ejemplo:

```
$ systemctl start vncserver@:1
```

Esto inicia un servidor TigerVNC para el usuario `test` con una sesión de GNOME.

En caso de que quieras que tu servidor se inicie automáticamente al arrancar el sistema, puedes ejecutar:

```
$ systemctl enable vncserver@:1
```

### Nota:

Si anteriormente usabas TigerVNC y solías iniciarlo mediante *systemd*, es posible que necesites eliminar los archivos de configuración previos de *systemd* ubicados en `/etc/systemd/system/vncserver@.service`, para evitar que tengan prioridad sobre los nuevos archivos de servicio de systemd de la última versión de TigerVNC.

# Cómo agregar un usuario con permisos sudo en SUSE SLES

A diferencia de distribuciones como Ubuntu, SUSE Linux Enterprise Server (SLES) no viene con un grupo `sudo` preconfigurado; en su lugar utiliza el grupo `wheel`, siguiendo la convención de las distribuciones basadas en Red Hat.

## 1. Crear el usuario (si aún no existe)

```
# useradd -m nombre_usuario
# passwd nombre_usuario
```

El parámetro `-m` crea el directorio personal en `/home/nombre_usuario`. Luego se define la contraseña del usuario.

## 2. Agregar el usuario al grupo `wheel`

```
# usermod -aG wheel nombre_usuario
```

La opción `-a` (append) agrega el usuario al grupo sin quitarlo de los grupos a los que ya pertenece, y `-G` indica el grupo.

Puedes verificar que el usuario quedó en el grupo con:

```
# groups nombre_usuario
```

## 3. Habilitar al grupo `wheel` en sudoers

Edita el archivo de sudoers de forma segura con `visudo` (nunca lo edites directamente con un editor normal, ya que `visudo` valida la sintaxis antes de guardar):

```
# visudo
```

Busca la línea siguiente y asegúrate de que esté descomentada (sin el símbolo `#` al inicio):

```
%wheel ALL=(ALL) ALL
```

Esto permite que cualquier usuario del grupo `wheel` ejecute cualquier comando como root usando `sudo`.

## 4. Verificar el acceso

Inicia sesión como el nuevo usuario y prueba:

```
$ sudo whoami
```

Si todo está bien configurado, después de ingresar la contraseña del usuario, el comando debería devolver `root`.

# Cómo abrir un puerto en el firewall para TigerVNC

SLES usa `firewalld` como firewall predeterminado. TigerVNC escucha, por cada display, en el puerto TCP `5900 + número de display` (por ejemplo, el display `:1` usa el puerto `5901`, el display `:2` el puerto `5902`, etc.).

## 1. Verificar el estado del firewall

```
# firewall-cmd --state
```

## 2. Abrir el puerto correspondiente al display usado

Por ejemplo, para el display `:1` (puerto 5901):

```
# firewall-cmd --zone=public --add-port=5901/tcp --permanent
```

Si vas a usar varios displays o quieres cubrir un rango, puedes abrir un rango de puertos, por ejemplo del 5900 al 5910:

```
# firewall-cmd --zone=public --add-port=5900-5910/tcp --permanent
```

Ajusta `--zone` a la zona que corresponda en tu configuración si no usas la zona `public`.

## 3. Recargar el firewall para aplicar los cambios

```
# firewall-cmd --reload
```

## 4. Verificar que el puerto quedó abierto

```
# firewall-cmd --zone=public --list-ports
```

Deberías ver el puerto o rango de puertos que agregaste en la salida.

### Nota:

Si no usas `--permanent`, la regla solo será válida hasta el próximo reinicio del servicio de firewall o del sistema. Para reglas persistentes usa siempre `--permanent` seguido de `--reload`.

# Limitaciones

No podrás iniciar un servidor TigerVNC para un usuario que ya haya iniciado sesión en una sesión gráfica. Evita ejecutar el servidor como usuario `root`, ya que no es algo seguro de hacer. Aunque ejecutar el servidor como `root` debería funcionar en general, no se recomienda hacerlo y podría haber cosas que no funcionen correctamente.
