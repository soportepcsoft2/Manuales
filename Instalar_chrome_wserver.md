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
