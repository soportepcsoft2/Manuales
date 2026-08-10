# Manual: Unificar disco con particiones protegidas (recuperación, MSR, dinámico) usando DiskPart

## Contexto

Disco secundario viejo que tuvo Windows instalado, con particiones de recuperación y de sistema que no se podían eliminar desde el Administrador de discos de Windows. El disco además estaba configurado como **disco dinámico**, lo cual complica aún más el proceso estándar.

## Situación inicial

En el Administrador de discos se veían particiones bloqueadas:

- 450 MB — Partición de recuperación
- 100 MB — Partición de sistema
- 194.80 GB — No asignado
- 523 MB — Partición de recuperación
- Volumen NTFS de 269.89 GB (G:)

Estas particiones no se podían eliminar desde la interfaz gráfica porque tienen un atributo de protección que el Administrador de discos respeta.

## Paso 1: Identificar si el disco es dinámico o básico

En el Administrador de discos, revisar la etiqueta debajo del nombre del disco (ej. "Disco 2"). Si dice **"Dinámico"**, el procedimiento de DiskPart cambia: se trabaja con `volume` en vez de `partition`.

## Paso 2: Abrir DiskPart como administrador

```
diskpart
```

## Paso 3 (si es disco dinámico): listar y borrar por volumen

```
list disk
select disk 2
list volume
```

Identificar los volúmenes del disco en cuestión por tamaño (no por número, ya que en discos dinámicos los números de volumen son globales al sistema).

Intentar borrar:

```
select volume 1
delete volume
```

**Posible error:**

```
Error del Servicio de disco virtual:
El objeto no admite la operación.
El comando o los parámetros especificados no se admiten en este sistema.
```

Esto ocurre porque DiskPart no puede eliminar ciertas particiones protegidas de un disco dinámico con `delete volume`.

## Paso 4: Convertir el disco dinámico a básico

Desde el Administrador de discos:

- Clic derecho sobre el disco dinámico completo → **Convertir en disco básico**

⚠️ Esto requiere que los volúmenes de datos ya estén vacíos/eliminados primero (hacer backup antes de cualquier borrado).

## Paso 5: Confirmar el disco y listar particiones (ahora como disco básico)

```
list disk
select disk 2
list partition
```

Ejemplo de salida:

```
N-m Partición   Tipo          Tamaño   Desplazamiento
-----------   ------------  -------  --------------
Partición 1   Recuperación   450 MB   1024 KB
Partición 2   Sistema        100 MB   451 MB
Partición 3   Reservado       16 MB   551 MB
Partición 5   Recuperación   523 MB   195 GB
```

**Nota:** la Partición 3 de 16 MB es la **MSR (Microsoft Reserved Partition)**, una partición reservada a nivel de tabla GPT que Windows oculta intencionalmente. No aparece en el Administrador de discos porque no es un volumen con sistema de archivos utilizable, pero sí ocupa espacio en el disco y debe eliminarse para unificarlo.

## Paso 6: Eliminar cada partición protegida con `override`

Para cada partición identificada, repetir:

```
select partition 1
delete partition override
```

```
select partition 2
delete partition override
```

```
select partition 3
delete partition override
```

```
select partition 5
delete partition override
```

El parámetro `override` es la clave: le indica a DiskPart que ignore el atributo de protección/solo-lectura que impide el borrado normal.

**Si `delete partition override` sigue fallando**, limpiar atributos antes de borrar:

```
select partition X
attributes volume clear readonly
attributes partition clear gpt_attribute
delete partition override
```

## Paso 7: Verificar que el disco quedó vacío

```
list partition
```

Debe mostrar que no quedan particiones, solo espacio no asignado.

## Paso 8: Crear un volumen único unificado

Ejecutar cada línea por separado, presionando Enter después de cada una:

```
create partition primary
```

```
format fs=ntfs quick label="Disco2"
```

```
assign
```

- `create partition primary` crea una única partición usando todo el espacio disponible.
- `format fs=ntfs quick label="Disco2"` formatea en NTFS con formato rápido (puede tardar unos segundos en discos grandes).
- `assign` asigna automáticamente la siguiente letra de unidad disponible (opcional: `assign letter=Z` para especificar una letra).

## Paso 9: Confirmar resultado final

```
list volume
```

Debe mostrar el disco como un único volumen NTFS con la letra asignada.

## Resumen de comandos clave

| Comando                                    | Uso                                                                               |
| ------------------------------------------ | --------------------------------------------------------------------------------- |
| `list disk` / `select disk N`              | Identificar y seleccionar el disco físico                                         |
| `list volume`                              | Ver particiones en disco dinámico                                                 |
| `list partition`                           | Ver particiones en disco básico                                                   |
| `delete volume`                            | Borrar volumen en disco dinámico (puede fallar en particiones protegidas)         |
| `delete partition override`                | Borrar partición protegida en disco básico, ignorando el atributo de solo-lectura |
| `attributes partition clear gpt_attribute` | Limpiar atributo GPT de protección cuando `override` no es suficiente             |
| `create partition primary`                 | Crear partición usando todo el espacio libre                                      |
| `format fs=ntfs quick label="X"`           | Formatear rápido en NTFS                                                          |
| `assign`                                   | Asignar letra de unidad                                                           |

## Notas importantes

- **Siempre verificar el tamaño del disco antes de seleccionar**, no confiar solo en el número (`disk 2`, `volume 1`, etc.), ya que estos índices pueden no coincidir con lo mostrado en el Administrador de discos, especialmente tras conversiones.
- Este proceso es **irreversible** — cualquier dato en las particiones borradas se pierde permanentemente.
- Todo esto se hizo con herramientas nativas de Windows (DiskPart), sin necesidad de software de terceros ni un USB booteable de Linux.
