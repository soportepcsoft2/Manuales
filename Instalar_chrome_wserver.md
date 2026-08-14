# Guía: Instalación Forzada de Winget en Windows Server 2022

Windows Server 2022 no incluye la infraestructura de Microsoft Store de forma nativa. Este procedimiento inyecta los paquetes contenedores y las dependencias de software requeridas para activar `winget`.

---

## Requisitos Previos
* Cuenta con permisos de Administrador en el servidor.
* Conexión activa a Internet.

---

## Paso 1: Abrir PowerShell como Administrador
1. Haz clic en el botón de **Inicio** o presiona la tecla `Windows`.
2. Escribe `powershell`.
3. Haz clic derecho sobre **Windows PowerShell** y selecciona **Ejecutar como administrador**.

---

## Paso 2: Descargar e Inyectar el Instalador de Winget
Copia el siguiente bloque de comandos completo, pégalo en la ventana de PowerShell y presiona `Enter`:

```powershell
# Setea la directiva de ejecución para permitir scripts de la galería oficial
Set-ExecutionPolicy RemoteSigned -Scope Process -Force

# Instala el script automatizado oficial de la galería de PowerShell
Install-Script -Name winget-install -Force

# Ejecuta el script para descargar las dependencias UWP y binarios de Winget
winget-install -Force
```

> **Nota de seguridad:** Si PowerShell te solicita confirmar la instalación de proveedores de paquetes externos (como `NuGet` o `PSGallery`), escribe **`S`** (Sí) y presiona `Enter` para continuar.

---

## Paso 3: Reiniciar el Entorno de Consola
Para que Windows Server reconozca los nuevos componentes en las variables de entorno:
1. **Cierra** la ventana actual de PowerShell.
2. Abre una **nueva ventana** de PowerShell como Administrador.

---

## Paso 4: Verificar la Instalación Exitosa
En la nueva ventana, ejecuta el comando de verificación:

```powershell
winget --version
```

### Resultados esperados:
* **Correcto:** La consola te devolverá un número de versión (por ejemplo, `v1.9.2514` o superior). Ya puedes usar `winget install`.
* **Incorrecto:** Si vuelve a salir un error rojo, es probable que las políticas de ejecución del servidor restringieran la descarga de los paquetes `.msixbundle`.

---

## Paso 5: Instalar Google Chrome
Ahora que dispones de la herramienta oficial, ejecuta el comando estándar para realizar el despliegue de Chrome de forma silenciosa:

```powershell
winget install -e --id Google.Chrome --accept-source-agreements --accept-package-agreements
```

# Caso se instaló Winget pero no se agregó al path

## Paso 1: Verificar información completa del script instalado

Ejecuta el siguiente comando para obtener todos los detalles del script, incluyendo su ubicación:

```powershell
Get-InstalledScript -Name winget-install | Select-Object *
```

Si el campo de ruta no aparece claramente, continúa al siguiente paso.

---

## Paso 2: Localizar el script manualmente

Como el `Install-Script` se ejecutó con permisos de administrador, es probable que se haya instalado en el scope **AllUsers**, cuya ruta por defecto es:

```
C:\Program Files\WindowsPowerShell\Scripts\winget-install.ps1
```

Puedes confirmarlo listando el contenido de la carpeta:

```powershell
dir "C:\Program Files\WindowsPowerShell\Scripts"
```

---

## Paso 3: Ejecutar el script con la ruta completa

Como esa carpeta no está en el `PATH` de tu sesión actual, ejecuta el script indicando la ruta completa:

```powershell
& "C:\Program Files\WindowsPowerShell\Scripts\winget-install.ps1" -Force
```

---

## Paso 4: (Opcional) Agregar la carpeta al PATH del sistema

Para que en el futuro puedas ejecutar `winget-install -Force` directamente sin especificar la ruta completa, agrega la carpeta al `PATH` de forma permanente (requiere permisos de administrador):

```powershell
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Program Files\WindowsPowerShell\Scripts", "Machine")
```

Luego **cierra completamente la ventana de PowerShell** y ábrela de nuevo como administrador para que tome el cambio.

---

## Paso 5: Verificar que Winget quedó instalado

Una vez ejecutado el script, comprueba que winget funcione correctamente:

```powershell
winget --version
```

Si devuelve un número de versión (ej. `v1.x.x.x`), la instalación fue exitosa.

---

## Notas

- Si el `Paso 5` falla, puede tratarse de un problema de permisos en `C:\Program Files\WindowsApps`. En ese caso, ejecuta:
```powershell
  TAKEOWN /F "C:\Program Files\WindowsApps" /R /A /D Y
  ICACLS "C:\Program Files\WindowsApps" /grant Administrators:F /T
```
- Windows Server 2019 no tiene soporte oficial de Microsoft para winget, por lo que este método (script de la comunidad) es la vía recomendada.