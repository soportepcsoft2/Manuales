# Windows PowerCFG — Cheatsheet y Guía de Diagnóstico

> Guía práctica de `powercfg` para diagnosticar **batería, consumo energético, Sleep, Modern Standby y problemas de Wake** en Windows.

---

## Índice

1. [Introducción](#1-introducción)
2. [Requisitos](#2-requisitos)
3. [Battery Report](#3-battery-report)
4. [SleepStudy](#4-sleepstudy)
5. [Energy Report](#5-energy-report)
6. [Diagnóstico de Sleep y Wake](#6-diagnóstico-de-sleep-y-wake)
7. [Configuración de energía](#7-configuración-de-energía)
8. [Workflow completo de troubleshooting](#8-workflow-completo-de-troubleshooting)
9. [Script de recolección de reportes](#9-script-de-recolección-de-reportes)
10. [Cómo interpretar los reportes](#10-cómo-interpretar-los-reportes)
11. [Cheatsheet rápido](#11-cheatsheet-rápido)

---

# 1. Introducción

`powercfg` es una herramienta incluida en Windows para consultar y diagnosticar la configuración y el comportamiento relacionado con energía.

Es especialmente útil para investigar problemas como:

* La batería dura poco.
* La batería pierde carga mientras el equipo está suspendido.
* El equipo no entra correctamente en Sleep.
* El equipo se despierta solo.
* Modern Standby consume demasiada batería.
* Una aplicación o driver mantiene activo el sistema.
* Un dispositivo provoca eventos de Wake.
* La capacidad de la batería se ha degradado.
* Existen problemas de configuración de energía.

Los comandos más importantes para troubleshooting son:

```text
/batteryreport
/sleepstudy
/energy
/a
/requests
/lastwake
/devicequery wake_armed
/getactivescheme
```

---

# 2. Requisitos

## Ejecutar como Administrador

Se recomienda abrir:

* Command Prompt (CMD) como Administrador
* PowerShell como Administrador

Para comprobar la ayuda general:

```cmd
powercfg /?
```

Ayuda específica:

```cmd
powercfg /batteryreport /?
powercfg /sleepstudy /?
powercfg /energy /?
```

---

# 3. Battery Report

## 3.1 ¿Para qué sirve?

`Battery Report` genera un informe HTML con información sobre la batería.

Es útil para analizar:

* Capacidad original de la batería.
* Capacidad máxima actual.
* Degradación.
* Historial de capacidad.
* Historial reciente de uso.
* Estimaciones de autonomía.
* Ciclos de carga, cuando el firmware expone este dato.

---

## 3.2 Generar Battery Report

Comando básico:

```cmd
powercfg /batteryreport
```

Windows mostrará la ubicación donde guardó el archivo.

Normalmente será una ruta similar a:

```text
C:\Windows\System32\battery-report.html
```

---

## 3.3 Elegir la ubicación del reporte

Es más práctico guardar el archivo en el Desktop:

```cmd
powercfg /batteryreport /output "%USERPROFILE%\Desktop\battery-report.html"
```

O crear una carpeta específica:

```cmd
mkdir "%USERPROFILE%\Desktop\PowerReports"
```

Y guardar allí:

```cmd
powercfg /batteryreport /output "%USERPROFILE%\Desktop\PowerReports\battery-report.html"
```

---

## 3.4 Abrir el reporte

```cmd
start "" "%USERPROFILE%\Desktop\PowerReports\battery-report.html"
```

---

## 3.5 Installed batteries

Una de las secciones más importantes es:

```text
Installed batteries
```

Normalmente contiene información como:

```text
NAME
MANUFACTURER
CHEMISTRY
DESIGN CAPACITY
FULL CHARGE CAPACITY
CYCLE COUNT
```

### Design Capacity

Es la capacidad nominal/original de la batería.

Ejemplo:

```text
DESIGN CAPACITY    50,000 mWh
```

---

### Full Charge Capacity

Es la capacidad máxima que Windows reporta actualmente.

Ejemplo:

```text
FULL CHARGE CAPACITY    42,000 mWh
```

---

## 3.6 Estimación simple de Battery Health

Una forma sencilla de estimar la capacidad restante es:

```text
Battery Health (%) =
Full Charge Capacity / Design Capacity × 100
```

Ejemplo:

```text
Design Capacity:       50,000 mWh
Full Charge Capacity:  42,000 mWh
```

Cálculo:

```text
42,000 / 50,000 × 100 = 84%
```

Por tanto:

```text
Battery Health ≈ 84%
```

> Esta es una estimación basada en capacidad. No debe interpretarse como un diagnóstico absoluto del estado físico de la batería.

---

## 3.7 Battery Capacity History

Esta sección permite observar cómo cambia la capacidad máxima reportada a lo largo del tiempo.

Comparar:

```text
DESIGN CAPACITY
FULL CHARGE CAPACITY
```

Por ejemplo:

```text
Fecha        Design       Full Charge
---------    --------     -----------
2026-01      50,000       48,500 mWh
2026-03      50,000       46,200 mWh
2026-06      50,000       43,500 mWh
2026-08      50,000       42,000 mWh
```

Una tendencia descendente sostenida puede indicar degradación de la batería.

---

## 3.8 Recent Usage

La sección:

```text
Recent usage
```

muestra el uso reciente del equipo.

Puede incluir estados como:

```text
ACTIVE
CONNECTED STANDBY
SLEEP
HIBERNATE
SHUTDOWN
```

Esta información ayuda a relacionar:

```text
Estado del sistema
        +
Duración
        +
Consumo de batería
```

---

## 3.9 Battery Usage

Permite analizar períodos específicos de descarga.

Es especialmente interesante comparar:

```text
Duration
Energy Drain
```

Por ejemplo:

```text
Equipo activo durante 2 horas
→ consumo 25%

Equipo en Modern Standby durante 8 horas
→ consumo 18%
```

El segundo caso puede justificar una investigación con `SleepStudy`.

---

## 3.10 Battery Life Estimates

Windows también proporciona estimaciones de autonomía.

Estas pueden ayudar a comparar:

```text
Autonomía estimada con capacidad actual
```

contra:

```text
Autonomía estimada con capacidad de diseño
```

No deben considerarse mediciones exactas, ya que la autonomía depende de:

* Carga de CPU.
* Brillo.
* Aplicaciones.
* Red.
* Dispositivos conectados.
* Estado de energía.
* Temperatura.
* Drivers.
* Configuración del sistema.

---

# 4. SleepStudy

## 4.1 ¿Para qué sirve?

`SleepStudy` analiza el comportamiento del sistema durante **Modern Standby**.

Es especialmente útil en equipos que utilizan:

```text
S0 Low Power Idle
```

en lugar de los estados tradicionales de Sleep.

Para comprobarlo:

```cmd
powercfg /a
```

Si aparece algo similar a:

```text
Standby (S0 Low Power Idle)
```

el equipo utiliza Modern Standby.

---

# 4.2 Generar SleepStudy

Comando básico:

```cmd
powercfg /sleepstudy
```

El resultado normalmente será:

```text
C:\Windows\System32\sleepstudy-report.html
```

---

## 4.3 Guardar SleepStudy en Desktop

```cmd
powercfg /sleepstudy /output "%USERPROFILE%\Desktop\sleepstudy-report.html"
```

---

## 4.4 Analizar los últimos 7 días

```cmd
powercfg /sleepstudy /duration 7
```

Con ubicación personalizada:

```cmd
powercfg /sleepstudy /duration 7 /output "%USERPROFILE%\Desktop\sleepstudy-report.html"
```

Para 30 días:

```cmd
powercfg /sleepstudy /duration 30 /output "%USERPROFILE%\Desktop\sleepstudy-30days.html"
```

---

# 4.5 SleepStudy Summary

La sección de resumen permite identificar las sesiones de Modern Standby.

Prestar atención a:

```text
Sleep Start
Sleep End
Duration
Energy Change
Energy Change %
```

La combinación más importante es:

```text
Duration
+
Energy Change
```

---

# 4.6 Ejemplo de interpretación

Supongamos:

```text
Sleep duration: 8 hours
Battery drain: 2%
```

Esto representa un consumo relativamente bajo.

Pero si vemos:

```text
Sleep duration: 8 hours
Battery drain: 20%
```

merece una investigación.

No existe un único porcentaje "correcto" para todos los equipos: el consumo depende del hardware, firmware, drivers, conectividad y actividad permitida durante Modern Standby.

---

# 4.7 Top Offenders

Una de las secciones más importantes de SleepStudy es:

```text
Top Offenders
```

Puede mostrar componentes responsables de actividad o consumo durante Modern Standby.

Por ejemplo:

```text
Applications
Drivers
Devices
System components
```

Si un componente aparece repetidamente asociado con consumo elevado, debe investigarse.

---

# 4.8 Qué buscar en SleepStudy

Buscar especialmente:

* Sesiones excesivamente largas.
* Alto porcentaje de batería perdido.
* Aplicaciones activas durante Modern Standby.
* Drivers con actividad.
* Dispositivos que permanecen activos.
* Componentes que aparecen repetidamente en `Top Offenders`.

---

# 5. Energy Report

## 5.1 ¿Para qué sirve?

`powercfg /energy` realiza un diagnóstico general de eficiencia energética.

Es útil para detectar problemas como:

* Configuraciones incorrectas.
* Problemas de energía.
* Dispositivos que permanecen activos.
* Problemas relacionados con timers.
* Procesos o configuraciones que afectan el consumo.

---

## 5.2 Generar Energy Report

Comando básico:

```cmd
powercfg /energy
```

---

## 5.3 Especificar duración

Por ejemplo, ejecutar el análisis durante 60 segundos:

```cmd
powercfg /energy /duration 60
```

Guardar el reporte:

```cmd
powercfg /energy /duration 60 /output "%USERPROFILE%\Desktop\energy-report.html"
```

---

## 5.4 Importante

Durante el análisis conviene utilizar el equipo normalmente.

Evitar generar artificialmente una carga extrema si el objetivo es investigar un problema de consumo cotidiano.

---

# 6. Diagnóstico de Sleep y Wake

Esta familia de comandos es fundamental para investigar:

```text
¿Por qué no entra en Sleep?
```

y:

```text
¿Por qué se despierta solo?
```

---

# 6.1 ¿Qué está impidiendo Sleep?

```cmd
powercfg /requests
```

Muestra solicitudes activas realizadas por componentes del sistema.

Puede mostrar categorías como:

```text
DISPLAY
SYSTEM
AWAYMODE
EXECUTION
PERFBOOST
ACTIVELOCKSCREEN
```

Si aparece:

```text
None.
```

significa que no hay una solicitud activa de ese tipo en el momento de ejecutar el comando.

---

# 6.2 ¿Qué despertó el equipo?

```cmd
powercfg /lastwake
```

Este comando intenta identificar el último evento responsable de despertar el sistema.

Es especialmente útil cuando:

```text
El equipo entra en Sleep
        ↓
se despierta inesperadamente
```

---

# 6.3 ¿Qué dispositivos pueden despertar el equipo?

```cmd
powercfg /devicequery wake_armed
```

Puede mostrar dispositivos como:

```text
Keyboard
Mouse
Network adapter
USB controller
Bluetooth device
```

Un dispositivo listado aquí tiene permiso para generar un evento de Wake.

---

# 6.4 Diferencia entre los comandos

| Problema                              | Comando                            |
| ------------------------------------- | ---------------------------------- |
| ¿Qué impide Sleep ahora?              | `powercfg /requests`               |
| ¿Qué despertó al equipo?              | `powercfg /lastwake`               |
| ¿Qué dispositivos pueden despertarlo? | `powercfg /devicequery wake_armed` |
| ¿Qué estados Sleep soporta?           | `powercfg /a`                      |

---

# 7. Configuración de energía

## 7.1 Ver el plan de energía actual

```cmd
powercfg /getactivescheme
```

---

## 7.2 Listar todos los planes

```cmd
powercfg /list
```

Puede devolver algo como:

```text
Power Scheme GUID: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
    (Balanced)
```

---

## 7.3 Consultar estados Sleep disponibles

```cmd
powercfg /a
```

Este comando es fundamental antes de diagnosticar Sleep.

Puede indicar, por ejemplo:

```text
Standby (S0 Low Power Idle)
Hibernate
Fast Startup
```

o estados tradicionales como:

```text
Standby (S3)
Hibernate
Fast Startup
```

---

# 8. Workflow completo de Troubleshooting

La mejor estrategia es no ejecutar solamente un reporte.

Primero hay que determinar **qué tipo de problema existe**.

---

## Caso A — La batería dura poco

### Paso 1

Generar Battery Report:

```cmd
powercfg /batteryreport /output "%USERPROFILE%\Desktop\battery-report.html"
```

### Paso 2

Revisar:

```text
Design Capacity
Full Charge Capacity
Cycle Count
Battery Capacity History
Battery Life Estimates
Recent Usage
Battery Usage
```

### Paso 3

Calcular aproximadamente:

```text
Full Charge Capacity / Design Capacity × 100
```

---

# Caso B — La laptop pierde mucha batería mientras está suspendida

### Paso 1

Comprobar estados:

```cmd
powercfg /a
```

### Paso 2

Si aparece:

```text
Standby (S0 Low Power Idle)
```

generar:

```cmd
powercfg /sleepstudy /duration 7 /output "%USERPROFILE%\Desktop\sleepstudy-report.html"
```

### Paso 3

Revisar:

```text
SleepStudy Summary
Top Offenders
Energy Change
Energy Change %
```

### Paso 4

Relacionar los resultados con:

```text
Battery Report
```

---

# Caso C — La laptop no entra en Sleep

Ejecutar:

```cmd
powercfg /requests
```

Después:

```cmd
powercfg /a
```

Y finalmente:

```cmd
powercfg /energy /duration 60 /output "%USERPROFILE%\Desktop\energy-report.html"
```

---

# Caso D — La laptop se despierta sola

Ejecutar:

```cmd
powercfg /lastwake
```

Después:

```cmd
powercfg /devicequery wake_armed
```

También:

```cmd
powercfg /requests
```

El objetivo es identificar:

```text
Wake event
      ↓
Dispositivo / driver / componente
      ↓
Posible causa
```

---

# Caso E — Consumo energético general

Ejecutar:

```cmd
powercfg /energy /duration 60 /output "%USERPROFILE%\Desktop\energy-report.html"
```

Después revisar el HTML y prestar atención a:

```text
Errors
Warnings
Power efficiency problems
Devices
Timers
CPU activity
```

---

# 9. Script de recolección de reportes

Para recopilar rápidamente información para troubleshooting:

```cmd
mkdir "%USERPROFILE%\Desktop\PowerReports"

powercfg /a

powercfg /getactivescheme

powercfg /requests

powercfg /devicequery wake_armed

powercfg /lastwake

powercfg /batteryreport /output "%USERPROFILE%\Desktop\PowerReports\battery-report.html"

powercfg /sleepstudy /duration 7 /output "%USERPROFILE%\Desktop\PowerReports\sleepstudy-report.html"

powercfg /energy /duration 60 /output "%USERPROFILE%\Desktop\PowerReports\energy-report.html"

start "" "%USERPROFILE%\Desktop\PowerReports"
```

Esto crea:

```text
Desktop
└── PowerReports
    ├── battery-report.html
    ├── sleepstudy-report.html
    └── energy-report.html
```

---

# 10. Cómo interpretar los reportes

## Battery Report

### Preguntas principales

```text
¿Cuál era la capacidad original?
        ↓
DESIGN CAPACITY

¿Cuál es la capacidad actual?
        ↓
FULL CHARGE CAPACITY

¿Está disminuyendo con el tiempo?
        ↓
BATTERY CAPACITY HISTORY

¿Cuánto consume durante uso normal?
        ↓
BATTERY USAGE

¿Cuál es la autonomía estimada?
        ↓
BATTERY LIFE ESTIMATES
```

---

# SleepStudy

### Preguntas principales

```text
¿El equipo utiliza Modern Standby?
        ↓
powercfg /a

¿Cuánto tiempo permanece en Standby?
        ↓
Duration

¿Cuánta batería pierde?
        ↓
Energy Change / Energy Change %

¿Qué componentes consumen?
        ↓
Top Offenders
```

---

# Energy Report

### Preguntas principales

```text
¿Hay problemas de eficiencia?
        ↓
Errors / Warnings

¿Hay componentes activos innecesariamente?
        ↓
Power requests / devices

¿Hay configuraciones que afectan el consumo?
        ↓
Energy report
```

---

# Sleep / Wake

### Preguntas principales

```text
¿Por qué no duerme?
        ↓
powercfg /requests

¿Por qué despertó?
        ↓
powercfg /lastwake

¿Qué dispositivos pueden despertarlo?
        ↓
powercfg /devicequery wake_armed
```

---

# 11. Cheatsheet rápido

## 🔋 Battery

```cmd
powercfg /batteryreport /output "%USERPROFILE%\Desktop\battery-report.html"
```

---

## 💤 SleepStudy

Últimos 7 días:

```cmd
powercfg /sleepstudy /duration 7 /output "%USERPROFILE%\Desktop\sleepstudy-report.html"
```

Últimos 30 días:

```cmd
powercfg /sleepstudy /duration 30 /output "%USERPROFILE%\Desktop\sleepstudy-30days.html"
```

---

## ⚡ Energy

```cmd
powercfg /energy /duration 60 /output "%USERPROFILE%\Desktop\energy-report.html"
```

---

## 💤 Sleep States

```cmd
powercfg /a
```

---

## ⚙️ Power Plan

```cmd
powercfg /getactivescheme
```

Listar planes:

```cmd
powercfg /list
```

---

## 🚫 Power Requests

```cmd
powercfg /requests
```

---

## 🔔 Last Wake

```cmd
powercfg /lastwake
```

---

## 🖱️ Wake-Armed Devices

```cmd
powercfg /devicequery wake_armed
```

---

# 12. Tabla de referencia rápida

| Objetivo              | Comando                            |
| --------------------- | ---------------------------------- |
| Ver ayuda             | `powercfg /?`                      |
| Ver estados Sleep     | `powercfg /a`                      |
| Ver plan activo       | `powercfg /getactivescheme`        |
| Listar planes         | `powercfg /list`                   |
| Battery Report        | `powercfg /batteryreport`          |
| SleepStudy            | `powercfg /sleepstudy`             |
| SleepStudy 7 días     | `powercfg /sleepstudy /duration 7` |
| Energy Report         | `powercfg /energy`                 |
| Energy Report 60 s    | `powercfg /energy /duration 60`    |
| Ver Power Requests    | `powercfg /requests`               |
| Ver último Wake       | `powercfg /lastwake`               |
| Ver dispositivos Wake | `powercfg /devicequery wake_armed` |

---

# 13. Regla mental para troubleshooting

```text
                         POWERCFG
                            │
             ┌──────────────┼──────────────┐
             │              │              │
          BATTERY          SLEEP          WAKE
             │              │              │
             ▼              ▼              ▼
      /batteryreport   /sleepstudy     /lastwake
             │              │              │
             │              ▼              ▼
             │        Modern Standby   Wake source
             │              │
             ▼              ▼
       Battery Health    Consumption
       Capacity          Top Offenders
       Usage
             │
             └──────────────┬──────────────┘
                            │
                            ▼
                     GENERAL DIAGNOSTICS
                            │
                            ├── /energy
                            ├── /requests
                            ├── /a
                            ├── /getactivescheme
                            └── /devicequery wake_armed
```

---

# 14. Flujo recomendado

Para un diagnóstico completo de un laptop con problemas de batería o Sleep:

```text
1. powercfg /a
        ↓
2. powercfg /getactivescheme
        ↓
3. powercfg /batteryreport
        ↓
4. powercfg /sleepstudy
        ↓
5. powercfg /requests
        ↓
6. powercfg /lastwake
        ↓
7. powercfg /devicequery wake_armed
        ↓
8. powercfg /energy
        ↓
9. Correlacionar resultados
```

La idea es correlacionar **capacidad de batería + consumo + estado de Sleep + actividad + eventos de Wake**, en lugar de analizar cada reporte de forma aislada.

---

# 15. Comando de recopilación rápida

Si necesitas recopilar evidencia para enviar a soporte o analizar un problema:

```cmd
mkdir "%USERPROFILE%\Desktop\PowerReports"

powercfg /a
powercfg /getactivescheme
powercfg /requests
powercfg /devicequery wake_armed
powercfg /lastwake

powercfg /batteryreport /output "%USERPROFILE%\Desktop\PowerReports\battery-report.html"

powercfg /sleepstudy /duration 7 /output "%USERPROFILE%\Desktop\PowerReports\sleepstudy-report.html"

powercfg /energy /duration 60 /output "%USERPROFILE%\Desktop\PowerReports\energy-report.html"

start "" "%USERPROFILE%\Desktop\PowerReports"
```

## Resultado

```text
PowerReports/
│
├── battery-report.html
├── sleepstudy-report.html
└── energy-report.html
```

Además, en la consola quedarán los resultados de:

```text
powercfg /a
powercfg /getactivescheme
powercfg /requests
powercfg /devicequery wake_armed
powercfg /lastwake
```

---

# 16. Resumen

```text
BATTERY HEALTH
    → /batteryreport

MODERN STANDBY
    → /sleepstudy

GENERAL POWER EFFICIENCY
    → /energy

¿NO ENTRA EN SLEEP?
    → /requests

¿SE DESPIERTA SOLO?
    → /lastwake

¿QUÉ DISPOSITIVOS PUEDEN DESPERTARLO?
    → /devicequery wake_armed

¿QUÉ ESTADOS DE SLEEP SOPORTA?
    → /a

¿QUÉ PLAN DE ENERGÍA ESTÁ ACTIVO?
    → /getactivescheme
```

> **Idea clave:** `Battery Report` responde principalmente **"¿cómo está la batería y cómo se ha utilizado?"**; `SleepStudy` responde **"¿qué ocurre con el consumo durante Modern Standby?"**; y `/requests`, `/lastwake`, `/devicequery` y `/energy` ayudan a encontrar **qué está provocando o afectando el comportamiento de energía**.
