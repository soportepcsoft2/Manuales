# Manual de Particionamiento e Instalación — SLES for SAP Applications 15 SP7 + SAP HANA

**Servidor de referencia:** 64 GB RAM · 5 discos físicos de 960 GB en RAID 5 (único arreglo disponible) · Controlador con batería (BBU)

> 📘 Los bloques marcados así contienen la explicación teórica de **por qué** se tomó cada decisión. Si solo necesitas ejecutar la instalación, puedes saltarlos y seguir únicamente los bloques marcados con ✅ **Pasos**.

---

## 1. Diseño general del almacenamiento

> 📘 **Teoría — Por qué un solo RAID dividido en dos discos virtuales**
> Al contar con un único conjunto de 5 discos físicos (sin discos de arranque separados), no es posible aislar físicamente el sistema operativo de los datos de HANA. La solución es un **único arreglo RAID físico**, dividido lógicamente en **discos virtuales (VD)** independientes a nivel de controlador:
> - **VD1**: sistema operativo (tamaño fijo, sin necesidad de crecer).
> - **VD2**: todo el almacenamiento de HANA vía LVM, con espacio libre reservado para expansión futura.
>
> Esto da aislamiento de capacidad y de administración, aunque ambos VD comparten el mismo rendimiento físico de fondo (mismos 5 discos).

> 📘 **Teoría — RAID 5 vs RAID 10**
> RAID 5 usa paridad distribuida, lo que penaliza escrituras pequeñas y frecuentes (patrón típico de `/hana/log`, que exige latencia <1 ms). RAID 10 (espejo + stripe) no tiene ese penalti y es la opción de mejor rendimiento para el log, pero requiere número par de discos y sacrifica capacidad útil.
> Con 5 discos de 960 GB y una carga **no intensiva** como la de este servidor, RAID 5 es aceptable: se prioriza capacidad sobre el margen extra de rendimiento que daría RAID 10, ya que la carga de trabajo no es exigente. Es recomendable validar después de la instalación con el **HWCCT (HANA Hardware Configuration Check Tool)** que la latencia del log cumple el KPI de SAP.

### ✅ Pasos
1. Crear un único arreglo **RAID 5** usando los 5 discos físicos (≈3.84 TB útiles).
2. Dividir ese arreglo en **2 discos virtuales (VD)** desde el controlador RAID:
   - **VD1**: 65 GB — sistema operativo.
   - **VD2**: el resto del espacio (~3.78 TB) — almacenamiento de HANA.

---

## 2. Configuración avanzada del controlador RAID

> 📘 **Teoría — Políticas de escritura, lectura y caché de disco**
> - **Política de escritura (Write Policy):** con batería (BBU) presente, se usa **Write-Back** ("Escritura no simultánea" en la interfaz del controlador). El controlador confirma la escritura apenas queda en su caché protegida, y la vuelca al disco después — más rápido, y seguro porque la batería protege esa caché ante un corte de energía. Sin batería, se debería usar **Write-Through** para no arriesgar datos.
> - **Política de caché de disco (Disk Cache Policy):** debe quedar **Desactivada**, incluso con batería en el controlador. La batería solo protege la caché del controlador, **no** la caché individual de cada disco físico — dejarla activada introduce el mismo riesgo de pérdida de datos que un Write-Back sin protección.
> - **Política de lectura (Read Policy):** se recomienda **Lectura anticipada adaptativa**. El controlador detecta automáticamente si el patrón de acceso es secuencial o aleatorio y ajusta la lectura por adelantado en consecuencia — ideal para HANA, que mezcla ambos patrones (carga inicial secuencial, consultas aleatorias en operación normal).

### ✅ Pasos
Aplicar en **ambos discos virtuales** (VD1 y VD2):
1. Política de escritura → **Escritura no simultánea (Write-Back)**.
2. Política de caché de disco → **Desactivado**.
3. Política de lectura → **Lectura anticipada adaptativa**.

---

## 3. Sistema operativo — SUSE Linux Enterprise Server for SAP Applications 15 SP7

> 📘 **Teoría — Por qué estas particiones**
> - **`/` (raíz):** contiene el sistema base, paquetes y logs del SO. 60 GB es más que suficiente y no requiere crecimiento posterior.
> - **`/boot`:** almacena kernel, initrd y archivos de GRUB. No crece con el uso — se mantiene fijo en 2 GB. Formatearlo en Ext4 (en vez de XFS) es una práctica común y válida para esta partición específica.
> - **`/boot/efi`:** requerida solo si el servidor arranca en modo **UEFI** (no BIOS legacy). Es un requisito del estándar UEFI, no de la guía de SAP — por eso debe formatearse obligatoriamente en **FAT32**, sin excepción.
> - **`/home` independiente:** no se requiere, según la guía del proveedor.
> - **Swap:** para hosts dedicados a SAP HANA, SAP indica (SAP Note 1944799) usar un valor **fijo de 2 GB**, sin importar la cantidad de RAM — a diferencia de la regla general antigua de Linux (2× RAM). Un swap grande es contraproducente: es preferible que un proceso falle con "out of memory" a que el sistema degrade su rendimiento paginando a disco silenciosamente.

### ✅ Pasos (en VD1, dentro de YaST Partitioner)
| Partición | Tamaño | Filesystem | Punto de montaje | Función en YaST |
|---|---|---|---|---|
| 1 | 60 GB | XFS | `/` | Sistema operativo |
| 2 | 2 GB | Ext4 | `/boot` | Sistema operativo |
| 3 | ~500 MB | FAT | `/boot/efi` | Partición de arranque EFI |
| 4 | 2 GB | Swap | `swap` | (tipo Swap, sin punto de montaje) |

---

## 4. Almacenamiento de SAP HANA — LVM `vg_hanasap`

> 📘 **Teoría — LVM y la regla de dimensionamiento 1-3-1**
> **LVM** agrupa discos físicos (PV) en un pool flexible (VG) del cual se reparten volúmenes lógicos (LV) — permite crecer un volumen después sin reparticionar el disco.
> La convención de dimensionamiento inicial usada aquí relaciona cada volumen con la RAM instalada:
> - `/hana/log` = 1× RAM (latencia crítica, escrituras pequeñas y síncronas).
> - `/hana/data` = 3× RAM (contiene los datos en disco + espacio para *savepoints* y crecimiento).
> - `/hana/shared` = 1× RAM (binarios, trazas y archivos compartidos entre nodos).
>
> Con 64 GB de RAM: log = 64 GB, data = 192 GB, shared = 64 GB.

> 📘 **Teoría — Por qué `/hana/backup` va dentro del mismo VG**
> El requisito de "filesystem independiente" para los respaldos se cumple con que tenga **su propio punto de montaje y LV separado** — no exige un disco físico distinto. Colocarlo dentro de `vg_hanasap` (en vez de un VD separado) permite expandirlo en caliente con `lvextend` sin tocar el RAID, a cambio de compartir el espacio libre del VG con los demás volúmenes.

> 📘 **Teoría — Volumen normal vs. depósito flexible (thin provisioning)**
> Un **volumen normal (thick)** reserva su espacio de forma fija e inmediata — predecible y aislado. Un **depósito flexible (thin)** asigna espacio "virtual" que varios LV comparten dinámicamente, lo que puede generar quedarse sin espacio físico real aunque cada LV individualmente "diga" tener espacio libre. Para SAP HANA se usa siempre **volumen normal**, evitando esa contención inesperada.

> 📘 **Teoría — XFS y sus límites**
> XFS es el sistema de archivos recomendado por SAP para todos los volúmenes de HANA: buen manejo de archivos grandes y escrituras concurrentes. Soporta **crecer en caliente** (`xfs_growfs`) sin desmontar, pero **no permite reducir (shrink)** bajo ninguna circunstancia — una vez que un LV/filesystem XFS crece, ese tamaño es definitivo salvo un proceso manual de respaldo/recreación/restauración.

### ✅ Pasos
1. En YaST, ir a **Grupos de volúmenes LVM** → **Añadir grupo de volúmenes...**
2. Nombrar el grupo **`vg_hanasap`** y asignar como PV **el VD2 completo** (una sola unidad, sin particionar en varios pedazos).
3. Dejar el tamaño de "physical extent" (PE) en su valor por defecto (4 MiB).
4. Crear cada volumen lógico como **Volumen normal**, en **XFS**, con estos valores:

| LV | Tamaño | Punto de montaje |
|---|---|---|
| `lv_data` | 192 GB (3× RAM) | `/hana/data` |
| `lv_log` | 64 GB (1× RAM) | `/hana/log` |
| `lv_shared` | 64 GB (1× RAM) | `/hana/shared` |
| `lv_usr_sap` | 100 GB | `/usr/sap` |
| `lv_backup` | 250 GB (mínimo) | `/hana/backup` |

5. **No asignar el 100% del VG** — dejar espacio libre sin usar, disponible para `lvextend` futuro sin tocar el RAID.

---

## 5. Expansión futura

> 📘 **Teoría — Por qué expandir LVM y no el RAID**
> Expandir un disco virtual ya creado (OCE) implica una reconstrucción a nivel de controlador, lenta y riesgosa. Expandir un LV dentro de un VG con espacio libre es una operación en caliente, rápida y sin tocar el RAID: `lvextend -L +XG /dev/vg_hanasap/lv_x` seguido de `xfs_growfs /point`.

### ✅ Pasos para crecer un volumen (ejemplo con `lv_data`)
```bash
lvextend -L +50G /dev/vg_hanasap/lv_data
xfs_growfs /hana/data
```

---

## 6. Kdump — advertencia de espacio

> 📘 **Teoría — Qué es un vmcore y por qué pesa tanto**
> Ante un kernel panic, kdump vuelca **toda la memoria RAM** al disco (no es un log de texto) para permitir análisis forense post-mortem. Por eso su tamaño escala casi 1:1 con la RAM instalada (64 GB → advertencia de ~67.5 GB necesarios).

### ✅ Pasos
- Si el espacio en `/` no alcanza para el volcado completo, se puede **ignorar la advertencia** sin riesgo para la operación normal del sistema ni de SAP HANA — solo se pierde la posibilidad de un volcado de memoria completo ante un panic (evento poco frecuente).

---

## 7. Resumen final de la distribución

| Disco virtual | Contenido | Tamaño |
|---|---|---|
| **VD1** (fijo) | `/`, `/boot`, `/boot/efi`, `swap` | 65 GB |
| **VD2** (`vg_hanasap`) | `lv_data`, `lv_log`, `lv_shared`, `lv_usr_sap`, `lv_backup` + espacio libre | ~3.78 TB (670 GB usados, resto libre) |

**Configuración del controlador RAID (ambos VD):** Write-Back (con BBU) · Caché de disco desactivada · Lectura anticipada adaptativa.
