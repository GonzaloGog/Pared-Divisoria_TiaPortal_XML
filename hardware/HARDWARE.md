# Hardware — PLC de Pared Divisoria (KA1-A1)

Fuente: export Openness desde TIA V15.1 (`hardware/modulos.csv`, `red.csv`, `safety.csv`,
`atributos.csv`) y bloques en `xml/`. Backup base: **05/07/2023** (pedir uno más nuevo).
`PRINT_PLC.pdf` llegó vacío (0 bytes).

---

## 1. CPU

| | |
|---|---|
| Nombre | **KA1-A1** (`SIMATIC 300_1` en el proyecto) |
| CPU | **CPU 319F-3 PN/DP** `6ES7 318-3FL01-0AB0`, FW **V3.2** — S7-300, la misma que tiene hoy laterales |
| Interfaces | PROFINET `ka1-a1` 10.51.2.21/24 · DP (dir. 2) · MPI/DP (dir. 2) |
| Safety | Habilitada. Dirección F-origen central **200**, tiempo de vigilancia por defecto **150 ms** |

**Rack central:**

| Slot | Módulo | E / A | F-destino |
|---|---|---|---|
| 4 | F-DI 24×24 V `326-1BK02` | E 0–9 / A 0–3 | 200 |
| 5–7 | F-DO 10×24 V/2 A `326-2BF10` ×3 | 10–27 | 201–203 |
| 8–10 | DI32 `321-1BL00` ×3 | E 28–39 | — |
| 11 | DO32 `322-1BL00` | A 28–31 | — |

---

## 2. Qué controla este PLC

Dos estaciones (más la que se suma):

| Prefijo | Estación | Periferia |
|---|---|---|
| **`60` / `ST_060`** | Estación 060: OP 60+10, 60+20, robot **60-R1** | ET200pro `60+10-A1`, `60+20-A1`, `60+R1H1_2_3-A1` · CPX `60+10-CPX1/2`, `60+20-CPX1` · ET200eco `60+RIP1`, `60+R1KF1` · ET200S `60+KLSG1` · Murrelektronik MVK `60+R1KC1` |
| **`PD`** | Pared Divisoria: **OP10LH, OP10RH, OP30LH, OP30RH**, robot **PD-R1**, celda | ET200pro `PD-A1` (celda, F-DI), `PD+OP30L-A1`, `PD+OP30R-A1` · CPX `PD+OP10LH`, `PD+OP10RH`, `PD-CPX1` · ET200eco `PD+R1RIP1/2` · ET200S `PD+KLSG1` · Murrelektronik MVK `PD+R1KF1/KF2` |
| Común | Tableros **KA1**, **KLI04**, **KLI05** | ET200S `KLI04-A1` y `KLI05-A1` (con F) · HMI `KLI04-MP277` (es un **KP1200 Comfort**) y `KLI05-MP277` · switches SCALANCE |

Además, un **acoplador DP/DP** `6ES7 158-0AD01` (PROFIBUS dir. 9, E/A 500–505) hacia otro PLC.

---

## 3. Red PROFINET — `PN/IE_1`, 10.51.2.0/24

38 dispositivos. Convención de nombres: `1++<estación>+<equipo>-<item>` (ej. `1++pd+op30l-a1`).

| Rango | Uso |
|---|---|
| .21 | CPU KA1-A1 |
| .44 / .45 | HMI KLI04 / KLI05 |
| .56, .63–.70 | Switches (KA1-SW1 X308, KA1-SW3, KLI04-SW1/2, KLI05-SW1/2, 60-R1-SW1, PD-R1-SW1) |
| .84 / .85 | ET200S KLI04-A1 / KLI05-A1 |
| .100–.127 | Periferia de estación (ET200S, ET200eco, ET200pro, Murrelektronik) |
| .151–.156 | CPX Festo |
| .221 / .222 | Robots 60-R1 / PD-R1 (KUKA, PROFIsafe 64 E/S seguras) |

**IPs libres** para la celda nueva: .1–.20, .22–.43, .46–.55, .71–.83, .86–.99, .128–.150,
.157–.220, .223–.254.

---

## 4. Seguridad (Safety Advanced, S7-300F)

Programa F en LAD: `OB 35` → `FC 70 Main_Safety` (F_FBD) → `FB 70 SAFETY PROGRAM`,
`FB 84 ZONA05_ST_060`, `FB 85 ZONA06_PD_FB`, más el DB F global `F_ZONE`. **Cada estación es una
zona**: laterales entraría como una zona más.

19 módulos F, todos con vigilancia 150 ms y passivación por canal:

| F-destino | Módulo | DB F-I/O |
|---|---|---|
| 200–203 | Rack central KA1-A1 (F-DI + 3 F-DO) | 8000–8003 |
| 204 | ET200pro `60+20-A1.1` (F-DI) | 8073 |
| 207, 220, 221 | ET200S `KLI04-A1` (2 F-DI, 1 F-DO) | 8074, 8010, 8011 |
| 290, 291 | ET200S `60+KLSG1` | 8018, 8019 |
| 600, 601 | ET200S `PD+KLSG1` | 8036, 8037 |
| 602, 603 | ET200pro `PD-A1` celda (F-DI ×2) | 8065, 8066 |
| 604–606 | ET200S `KLI05-A1` | 8039, 8040, 8063 |
| 1000 | Robot 60-R1 (64 E/S seguras, E/A 3000–3011) | 8029 |
| 1100 | Robot PD-R1 (E/A 3100–3111) | 8035 |

**Lo que no salió:** los parámetros por canal (evaluación 1oo1/1oo2, tiempo de discrepancia,
test de cortocircuito). Openness solo expone los parámetros F del módulo. Para verlos: captura de la
pestaña "Parámetros F" / canales de cada módulo, o la impresión del proyecto.

---

## 5. Qué implica para integrar laterales

1. **CPU.** El EPLAN nuevo pide una **CPU 1517F-3 PN/DP**; este PLC es una **319F-3 (S7-300)**.
   Si laterales va en este PLC, hay dos caminos:
   - **Cambiar KA1-A1 a 1517F** y migrar las tres estaciones (060, PD, laterales) a S7-1500. El
     programa estándar (LAD/STL) pasa con la migración de TIA; el F se regenera (F-I/O DB nuevos,
     Safety Advanced para 1500). Es lo que sugiere el EPLAN.
   - **Quedarse con la 319F** y sumar laterales. Funciona, pero la celda nueva suma 4 G120D, 2
     MOVIDRIVE, KR C5, Bosch, 2 ET200pro, 6 CPX y ET200SP F, más su programa F: revisar memoria de
     trabajo y tiempo de ciclo online antes de decidirlo.

   **Hay que definirlo con el cliente antes de seguir.**
2. **Nombres repetidos.** PD ya tiene estaciones **OP10LH/RH y OP30LH/RH**, y los nombres PROFINET
   `pd+op30l-a1`, `pd+op30r-a1`. La celda nueva usa OP10, OP20, OP30, OP31 y OP40: necesita un
   prefijo propio (ej. `1++PL+…`) en EPLAN, PROFINET, tags y bloques.
3. **Red.** Este PLC vive en **10.51.2.x/24**. Los G120D del EPLAN nuevo están en 10.51.5.177/.178
   (red de traseras, y además duplicadas). Hay que asignarlos dentro de 10.51.2.x (rangos libres
   arriba) o agregar una segunda subred.
4. **Direcciones F.** Ocupadas: 200–207, 220–221, 290–291, 600–606, 1000, 1100. Para laterales
   conviene reservar un bloque libre (ej. **700–799**) para ET200SP F, ET200pro F, los 4 G120D,
   los 2 MOVIDRIVE (`CSB21A`) y el KR C5.
5. **Programa F.** Estructura por zonas: se agrega `ZONA07` con su FB y DB, llamada desde
   `Main_Safety`.
6. **HMI.** KLI04 está declarado como MP277 pero es un **KP1200 Comfort** con versión 12.0: es lo
   que bloquea el paso a V20 (ver nota de migración).
