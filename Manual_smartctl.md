# Guia básica de smartctl

Herramienta principal de smartmontools para consultar y controlar el sistema S.M.A.R.T. de discos duros y SSDs.

## 1. Identificar discos disponibles

```bash
smartctl --scan
```

Lista todos los dispositivos detectados, con su tipo de bus (`ata`, `scsi`, `sat`, `megaraid`, etc.). Primer paso obligatorio, sobre todo detrás de un RAID controller.

## 2. Comandos básicos de consulta

| Comando | Qué hace |
|---|---|
| `smartctl -i /dev/sdX` | Info básica: modelo, serie, capacidad, firmware, soporte SMART |
| `smartctl -H /dev/sdX` | Estado de salud rápido (PASSED/FAILED) |
| `smartctl -A /dev/sdX` | Tabla de atributos SMART (solo ATA/SATA) |
| `smartctl -a /dev/sdX` | Todo junto: info + salud + atributos + logs |
| `smartctl -x /dev/sdX` | Superset de `-a` + info extendida (vendor stats, SATA phy events, SCT) |

```bash
sudo smartctl -a /dev/sda
```
Es el comando más usado para diagnóstico completo.

## 3. Discos detrás de RAID controllers (PERC / MegaRAID)

Cuando el disco está detrás de una controladora RAID, hay que indicar el tipo de dispositivo:

```bash
sudo smartctl -a -d megaraid,0 /dev/sda   # disco físico índice 0
sudo smartctl -a -d megaraid,1 /dev/sda   # disco físico índice 1
```

El número tras la coma es el índice físico dentro del array, correspondiente al `EID:Slot` que muestra `perccli /c0/eall/sall show`.

Para forzar tipo SCSI (discos SAS):

```bash
sudo smartctl -a -d scsi /dev/sda
```

## 4. Atributos clave a revisar (ATA/SATA)

| ID | Atributo | Qué indica |
|---|---|---|
| 9 | Power_On_Hours | Horas totales de encendido |
| 5 | Reallocated_Sector_Ct | Sectores reasignados (desgaste) |
| 194 | Temperature_Celsius | Temperatura |
| 241/242 | Total_LBAs_Written/Read | Uso en SSDs y algunos SATA |

**Discos SAS (ej. HGST):** la tabla `-A` normalmente no aplica. Buscar power-on hours con:

```bash
sudo smartctl -a -d scsi /dev/sdX | grep -i "power on"
```

En SAS aparece como "Accumulated power on time", no como atributo numerado.

## 5. Logs

```bash
sudo smartctl -l error /dev/sda      # historial de errores del disco
sudo smartctl -l selftest /dev/sda   # historial de autotests
```

### Estructura del log de selftest

```
Num  Test_Description   Status                     Remaining  LifeTime(hours)  LBA_of_first_error
#1   Short offline      Completed without error     00%        12406            -
```

| Columna | Significado |
|---|---|
| Test_Description | Short offline / Extended offline / Conveyance |
| Status | Completed without error, Completed: read failure, Aborted by host, Interrupted (host reset), In progress |
| Remaining | % faltante del test |
| LifeTime(hours) | Power-on hours al momento del test |
| LBA_of_first_error | Sector exacto donde ocurrió el fallo (si aplica) |

## 6. Ejecutar y controlar autotests

```bash
sudo smartctl -t short /dev/sda   # ~2 min, chequeo rápido
sudo smartctl -t long /dev/sda    # completo, puede tomar horas
```

Ver progreso en tiempo real:

```bash
sudo smartctl -a /dev/sdX | grep -i "self-test execution status"
```

Abortar un test en curso:

```bash
sudo smartctl -X /dev/sdX
```

Modo captive (bloqueante, no recomendado en producción):

```bash
sudo smartctl -t long -C /dev/sdX
```

Congela el disco para cualquier otra I/O hasta terminar el test.

### Short vs Long

- **Short**: revisa electrónica, mecánica y muestra de sectores. Rápido, detecta problemas obvios.
- **Long**: escanea el disco completo sector por sector. Más lento pero exhaustivo.

### ¿El disco sigue usable durante el test?

Sí, corre en background por defecto. En discos con carga, el test puede pausarse/reanudarse según el I/O real, alargando el tiempo total.

### Desconexión de emergencia

- **Con tiempo**: `smartctl -X` primero (queda registrado como "Aborted by host").
- **Sin tiempo**: el test queda como "Interrupted (host reset)". No hay riesgo de daño por el test en sí — el riesgo real es el I/O de escritura en curso o el filesystem sin desmontar.

## 7. Combo de diagnóstico rápido (filtrado)

```bash
sudo smartctl -a -d megaraid,0 /dev/sda | grep -Ei "power_on|reallocated|health|temp"
```

## 8. Referencia de flags

| Flag | Origen | Significado |
|---|---|---|
| `-i` | info | Info básica |
| `-H` | Health | Salud |
| `-A` | Attributes | Tabla de atributos |
| `-a` | all | Todo lo anterior |
| `-x` | extended | Info extendida |
| `-l` | log | Logs (error, selftest) |
| `-t` | test | Tipo de autotest (short/long/conveyance) |
| `-X` | eXit/abort | Cancelar test en curso |
| `-d` | device | Tipo de dispositivo (ata, scsi, sat, megaraid,N) |
| `-C` | Captive | Modo bloqueante para autotest |
| `--scan` | — | Lista dispositivos detectados |



# Guía Técnica Unidades ocultas

Esta guía documenta el procedimiento para identificar, escanear y extraer la salud S.M.A.R.T. de discos **SATA, SAS y NVMe** en estado **UGOOD** (Unconfigured Good) u ocultos detrás de una controladora RAID por hardware LSI MegaRAID / Broadcom (ej. serie Tri-Mode SAS3508).

---

## 1. Identificación Inicial del Hardware
Antes de ejecutar consultas de diagnóstico, es obligatorio identificar el modelo de la tarjeta y mapear los identificadores físicos de las unidades.

### Paso A: Detectar la controladora RAID
```bash
lspci | grep -i -E "raid|storage"
```
*   `lspci`: Lista todos los dispositivos conectados a los buses PCI del servidor.
*   `| grep -i -E "raid|storage"`: Filtra la salida para mostrar únicamente las controladoras de almacenamiento.

### Paso B: Escanear el mapeo físico de unidades (SATA y SAS)
```bash
smartctl --scan-open
```
*   `smartctl`: Utilidad de control y monitoreo para sistemas de almacenamiento S.M.A.R.T.
*   `--scan-open`: Examina los dispositivos del sistema y abre comunicación real con ellos. Fuerza a la controladora a revelar la ruta del bus (ej. `/dev/bus/15`) y el ID físico (ej. `25`) de los discos ocultos.

---

## 2. Comandos de Consulta S.M.A.R.T. según la Tecnología del Disco

### Opción A: Discos SATA
Cuando el escaneo devuelva un formato como `/dev/bus/15 -d sat+megaraid,25`, ejecuta:
```bash
smartctl -a -d sat+megaraid,25 /dev/bus/15
```
*   `-a`: Ordena volcar toda la información disponible (firmware, estado de salud y tabla de atributos).
*   `-d sat+megaraid,25`: Define el tipo de dispositivo (`-d` / `--device`):
    *   `sat`: (SCSI to ATA Translation) Traduce comandos SATA dentro del bus SCSI de la tarjeta. **Obligatorio para SATA**.
    *   `+megaraid`: Cruza la capa propietaria de control de la tarjeta LSI MegaRAID.
    *   `,25`: Especifica la ranura o ID físico exacto del disco en el backplane.
*   `/dev/bus/15`: Nodo de entrada asignado por Linux para comunicarse con esa controladora específica.

### Opción B: Discos SAS
Los discos empresariales SAS no requieren traducción ATA. Si el disco detectado es SAS, utiliza:
```bash
smartctl -a -d megaraid,25 /dev/bus/15
```
*   `-d megaraid,25`: Accede de manera directa al disco SAS físico utilizando únicamente el protocolo nativo de la tarjeta MegaRAID en el ID físico `25`.

### Opción C: Unidades SSD NVMe (Controladoras Tri-Mode)
Las unidades NVMe bajo controladoras Tri-Mode no responden a `smartctl`. Se debe utilizar la herramienta oficial de Broadcom preinstalada en System Rescue:
```bash
storcli /c0 /eall /sall show all
```
*   `storcli`: Utilidad de línea de comandos para la gestión de hardware MegaRAID.
*   `/c0`: Apunta a la primera controladora RAID del sistema (Controller 0).
*   `/eall`: Consulta todos los gabinetes o backplanes conectados (Enclosure All).
*   `/sall`: Consulta todas las bahías físicas de discos (Slot All).
*   `show all`: Vuelca la telemetría interna, contadores de desgaste y estado de salud de la controladora.

---

## 3. Auditoría de Componentes Nuevos (Métricas Clave)
Para validar si un disco entregado como "nuevo" es genuino y no remanufacturado o usado, audita los siguientes valores en los reportes generados:

### En Discos SATA y SAS (Salida de `smartctl`)
*   **ID 09 - Power_On_Hours (Raw Value):** Horas totales de encendido. En un disco nuevo debe marcar entre **`0` y `5` horas** (tiempo de pruebas de control de calidad en fábrica).
*   **ID 12 - Power_Cycle_Count (Raw Value):** Número de veces que el disco se ha encendido/apagado. Debe ser muy bajo, idealmente entre **`1` y `5` ciclos**.
*   **ID 241/242 - Total_LBAs_Written / Read:** Volumen de datos transferidos. Debe estar en **`0` o valores mínimos** de Megabytes generados por el fabricante.
*   **Health Status (SMART overall-health):** Debe dictaminar estrictamente **`PASSED`**.

### En Unidades NVMe (Salida de `storcli`)
*   **Power On Hours:** Debe marcar de **`0` a `2` horas**.
*   **Power Cycles:** Debe registrar un valor menor a **`5` ciclos**.
*   **Percentage Used:** Muestra el desgaste consumido de la vida útil de las celdas flash. En una unidad nueva debe marcar estrictamente **`0%`**. Cualquier valor superior indica que fue usado previamente.
*   **Media Error Count / Pred Fail Count:** Ambas métricas de error deben estar en **`0`**.

---

## 4. Atributos Críticos de Falla (SATA / SAS)
Independientemente de la edad del disco, vigila siempre que estos contadores de degradación física se mantengan limpios:
*   **ID 05 (Reallocated_Sector_Count):** Sectores dañados reemplazados. **Debe estar en 0**.
*   **ID 197 (Current_Pending_Sector):** Sectores inestables en espera de reasignación. **Debe estar en 0**.

