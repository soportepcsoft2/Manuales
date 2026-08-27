# Manual de instalación y configuración de RustDesk en SUSE Linux Enterprise

Guía basada en un proceso real de instalación, configuración y resolución de problemas en una VM con SUSE Enterprise.

---

## 1. Requisitos previos

- SUSE Linux Enterprise con **entorno gráfico instalado** (GNOME o KDE). RustDesk no funciona en instalaciones headless/servidor sin GUI.
- Confirmar la arquitectura del sistema antes de descargar nada:

```bash
uname -m
```

- `x86_64` → arquitectura Intel/AMD de 64 bits (la más común)
- `aarch64` → arquitectura ARM de 64 bits

---

## 2. Descargar el paquete correcto

SUSE Enterprise no incluye RustDesk en sus repositorios oficiales, así que se instala descargando el `.rpm` directamente desde la página de releases del proyecto en GitHub:

👉 https://github.com/rustdesk/rustdesk/releases

Busca el archivo con el patrón `rustdesk-<version>-0.<arquitectura>-suse.rpm`, por ejemplo:

```bash
# Para x86_64
curl -L -o rustdesk.rpm https://github.com/rustdesk/rustdesk/releases/download/1.4.9/rustdesk-1.4.9-0.x86_64-suse.rpm

# Para aarch64 (ARM)
curl -L -o rustdesk.rpm https://github.com/rustdesk/rustdesk/releases/download/1.4.9/rustdesk-1.4.9-0.aarch64-suse.rpm
```

> ⚠️ Verifica siempre el nombre exacto del archivo en la página de releases, ya que puede variar entre versiones.

---

## 3. Instalar el paquete

Usa `zypper` en vez de `rpm -i` directo, para que resuelva automáticamente las dependencias necesarias:

```bash
sudo zypper install ./rustdesk.rpm
```

### Advertencia de firma no verificada

Al ser un `.rpm` descargado directo de GitHub (no de un repositorio zypper formal), es normal que aparezca:

```
package header is not signed
signature verification failed
6 files is unsigned
abort/ retry /ignore
```

Esto ocurre porque el archivo no tiene firma GPG, algo común en builds distribuidos fuera de los repositorios oficiales. Si confirmaste que la descarga viene del repositorio oficial de RustDesk en GitHub, es razonable continuar eligiendo **Ignore**, o evitar el prompt interactivo desde el inicio con:

```bash
sudo zypper --no-gpg-checks install ./rustdesk.rpm
```

---

## 4. Primer inicio y contraseña permanente

Abre la aplicación una vez de forma manual para configurarla:

```bash
rustdesk
```

Se mostrará un **ID numérico** único y una contraseña temporal que cambia en cada intento de conexión fallido.

Para uso persistente (ver sección 6), conviene fijar una **contraseña permanente**:

1. Ve a **Configuración → Seguridad**
2. En el campo de contraseña, selecciona la opción de **contraseña permanente** (no la aleatoria)
3. Anótala en un lugar seguro

Cierra la app después de configurarla (`Ctrl+C` en la terminal, o cerrando la ventana).

---

## 5. Evitar procesos duplicados (causa común de fallos)

**Problema observado:** tener el servicio systemd activo *y* la app abierta manualmente al mismo tiempo genera múltiples procesos de RustDesk compitiendo por los mismos puertos, lo que puede causar errores como `connection refused` al intentar conectarse, incluso con la red y el firewall correctamente configurados.

### Diagnóstico

```bash
ps aux | grep rustdesk
```

Si aparece más de un proceso principal (no solo hilos o procesos `<defunct>`), hay un conflicto.

### Limpieza completa

```bash
sudo systemctl stop rustdesk
sudo systemctl disable rustdesk
sudo pkill -9 -f rustdesk
ps aux | grep rustdesk   # debe quedar limpio, solo el propio grep
```

> **Regla de oro:** nunca corras el servicio systemd y la app manual al mismo tiempo. Si necesitas cambiar configuración, detén el servicio primero, haz el cambio con la app manual, ciérrala, y reinicia el servicio.

---

## 6. Configurar el servicio persistente

Para que RustDesk esté disponible permanentemente (incluso tras reiniciar la VM), sin depender de abrir la app manualmente:

1. Asegúrate de tener fijada la **contraseña permanente** (sección 4) — el servicio no muestra ventana, así que la contraseña temporal no sirve aquí.
2. Cierra cualquier instancia manual abierta.
3. Activa el servicio:

```bash
sudo systemctl enable --now rustdesk
```

4. Verifica que quedó activo y como única instancia:

```bash
systemctl status rustdesk   # debe decir "active (running)"
ps aux | grep rustdesk      # debe mostrar solo el proceso del servicio
```

5. Conecta desde el cliente usando el ID de siempre y la **contraseña permanente**.

---

## 7. Solución de problemas: "Connection refused"

Si al conectar desde otra máquina aparece `connection refused`, revisar en este orden:

| Causa posible | Cómo descartarla |
|---|---|
| Procesos duplicados de RustDesk | Ver sección 5 |
| VM en modo NAT (doble NAT) | Cambiar el adaptador de red de la VM a **Bridged/Puente** en el hipervisor |
| Firewall local de la VM | `sudo systemctl stop firewalld` (temporal, para probar) |
| Firewall del lado del cliente (Windows) | Desactivar temporalmente Windows Defender Firewall / antivirus |
| Conectividad general de la VM | `ping 8.8.8.8` y `curl -I https://rustdesk.com` |
| Servidor ID/Relay mal configurado | Revisar en RustDesk → Red, debe usar el servidor público por defecto salvo que se auto-hospede uno propio |
| Usar IP en vez de ID | RustDesk se conecta por **ID numérico**, no por IP:puerto como RDP/VNC |

> Nota: que la contraseña temporal se regenere tras cada intento fallido es normal — indica que el handshake inicial sí llega al servidor de RustDesk, así que el problema suele estar más adelante en el proceso (procesos duplicados, red de la VM, o firewall).

---

## 8. Reactivar el firewall después de las pruebas

Durante el diagnóstico de conexión (sección 7) es común desactivar `firewalld` temporalmente para descartarlo como causa. **Es fácil olvidar reactivarlo después**, dejando la VM sin firewall de forma permanente.

### Verificar el estado

```bash
sudo systemctl status firewalld
# o
sudo firewall-cmd --state
```

- `active (running)` / `running` → está activo, todo bien
- `inactive (dead)` / `not running` → quedó apagado, hay que reactivarlo

### Reactivarlo

```bash
sudo systemctl start firewalld
sudo systemctl enable firewalld
```

### Si RustDesk deja de conectar al reactivarlo

En vez de dejar el firewall apagado, abre específicamente los puertos que usa RustDesk:

```bash
sudo firewall-cmd --permanent --add-port=21115-21119/tcp
sudo firewall-cmd --permanent --add-port=21116/udp
sudo firewall-cmd --reload
```

> ✅ Buena práctica: cada vez que se desactive el firewall para descartar una causa durante troubleshooting, anotarlo y agregar un recordatorio explícito de reactivarlo al cerrar el diagnóstico.

---

## 9. Notas de seguridad para entornos de prueba expuestos

- Aunque RustDesk no requiere abrir puertos ni exponer una IP pública (usa NAT traversal vía su servidor de rendezvous), la combinación de **ID + contraseña** es la única barrera de acceso.
- Si el servicio queda persistente y accesible por tiempo prolongado, usa una contraseña permanente robusta, no una simple.
- Para mayor control (sin depender del servidor público de RustDesk), es posible auto-hospedar el propio servidor relay (`hbbs`/`hbbr`) en una VPS propia.

---

## 10. Problemas conocidos pendientes

**Síntoma:** el puntero del mouse se mueve correctamente y el teclado funciona, pero los clics no registran — reproducible desde múltiples equipos clientes, lo que apunta a un problema del lado de la VM SUSE (no del cliente).

**Pistas descartadas:**
- No es un tema de permisos entre usuarios (el servicio y la sesión gráfica corren ambos como `root`)
- La sesión es X11 (no Wayland)
- No es específico de un cliente Windows en particular (falla desde varias máquinas)

**Próximas líneas de diagnóstico:**
- Probar clics simulados localmente con `xdotool click 1` para aislar si el problema es de X11/VM o de RustDesk
- Revisar `xinput list` por dispositivos de mouse duplicados o mal configurados
- Revisar la función de "integración de mouse" del hipervisor (VirtualBox/VMware), que puede generar conflictos con la inyección de eventos de RustDesk
- Probar cambiando el tipo de dispositivo apuntador emulado (de USB Tablet/absoluto a PS/2/relativo)

*(Sección a completar una vez resuelto.)*
