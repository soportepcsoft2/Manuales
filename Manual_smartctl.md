# Cheatsheet de smartctl

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
