# Generador de ejercicios — Parcial 1, Sistemas Digitales I (MCF5272)

Especificación para generar y corregir ejercicios del Parcial 1 de Sistemas Digitales I
(ETSIT UPM), basado en el microcontrolador ColdFire **MCF5272**. Cubre los Temas 1, 2 y 3.

El examen se resuelve con una **hoja-resumen del MCF5272** que el alumno tiene permitida: en ella
están los bits de cada registro (no hay que memorizarlos). Las **fórmulas** sí hay que sabérselas.

---

## INSTRUCCIONES PARA EL GENERADOR

- Genera un ejercicio de uno de los 8 tipos de abajo. Da primero el **enunciado** (como en un examen,
  con un sistema inventado coherente y, cuando aplique, un listado parcial en ensamblador).
- El alumno responde; entonces **corrige paso a paso**, campo a campo, señalando el error concreto y
  el reflejo a corregir. No des la solución antes de que el alumno lo intente, salvo que la pida.
- Trabaja en español. Usa hexadecimal con prefijo `$`. Direcciones siempre de 8 dígitos hex.
- Varía los números cada vez. Mantén el nivel de un examen real (ni trivial ni imposible).
- Después de corregir, ofrece el siguiente ejercicio.

---

## DATOS FIJOS DEL MCF5272 (no se dan en el enunciado; hay que saberlos)

- Direcciones de **32 bits** (8 dígitos hex).
- **Módulo SIM**: siempre **64 KB** (`$10000`). Se coloca en la dirección **MBAR_MEM**
  (se reconoce por `MBAR_MEM EQU $...` + `MOVEC D0,MBAR`).
- **RAM interna** del micro: **4 KB** (`$1000`). Se coloca con **RAMBAR**.
- **EPROM/ROM de arranque**: va obligatoriamente en `$0` (ahí están la tabla de vectores y el vector
  de arranque: `ORG $0 / DC.L FINPILA / DC.L INICIO`). Suele colgar de **CS0#**.
- La **anchura de bus de CS0#** se fija por niveles de tensión en terminales **durante el RESET**
  (antes de poder configurar nada por software). Pregunta conceptual típica.

### Conversiones de tamaño ↔ hex (memorizar)
| Tamaño | Hex | Potencia |
|---|---|---|
| 2 KB | $800 | 2^11 |
| 4 KB | $1000 | 2^12 |
| 8 KB | $2000 | 2^13 |
| 16 KB | $4000 | 2^14 |
| 32 KB | $8000 | 2^15 |
| 64 KB | $10000 | 2^16 |
| 128 KB | $20000 | 2^17 |
| 256 KB | $40000 | 2^18 |
| 512 KB | $80000 | 2^19 |
| 1 MB | $100000 | 2^20 |
| 2 MB | $200000 | 2^21 |
| 4 MB | $400000 | 2^22 |

---

## FÓRMULAS DEL PARCIAL 1

- **Fin de un dispositivo:** `fin = inicio + tamaño − 1`
- **Máscara BAM (del CSOR):** `BAM_completo = $FFFFFFFF − (tamaño − 1)`; se toman los bits 31-12
  (5 dígitos hex altos). Atajo: tamaño 2^n → n ceros bajos en la máscara.
- **Cola del CSOR (bits 6-0):** `cola = WS × 4 + RW × 2 + MRW`
- **A7 inicial (pila):** `A7_inicial = última_dirección_RAM + 1`
- **TAMPILA máximo:** `TAMPILA = tamaño_RAM − bytes_de_variables`
- **Dirección efectiva (indexado con escala):** `EA = desplazamiento + reg_base + índice × escala`
  (la escala entra una sola vez; convertir decimales a hex antes de sumar)
- **Vector de interrupción:** `dir_vector = VBR + (PIVR + offset) × 4` (VBR = 0 si no se configura)
- **LSR (desplazamiento derecha):** `resultado = valor / 2^(nº desplazamientos)`

---

## REGISTROS (campos y bits) — referencia para corregir

### CSBR (Chip Select Base Register) — bits 31-0
| Campo | Bits | Valores |
|---|---|---|
| BA | 31-12 | dirección base (5 dígitos hex altos del inicio) |
| EBI | 11-10 | 00 = SRAM/ROM 16/32 bits (casi siempre 00) |
| BW | 9-8 | **00 = 32 bits, 01 = 8 bits, 10 = 16 bits** (contraintuitivo) |
| SUPER | 7 | 1 = solo supervisor; si no se dice → 0 |
| ENABLE | 0 | 1 = habilita este CS (casi siempre 1) |

- Firma típica: ROM/RAM de 16 bits habilitada, sin supervisor → CSBR acaba en **`201`** (+ BA delante).
- 32 bits → acaba en **`001`**; 8 bits → acaba en **`101`**.

### CSOR (Chip Select Option Register) — bits 31-0
| Campo | Bits | Valores |
|---|---|---|
| BAM | 31-12 | **máscara del tamaño** (NO la base) |
| WS | 6-2 | nº de estados de espera (en binario, $00–$1E; $1F = según TA#) |
| RW | 1 | 0 = solo lectura, 1 = solo escritura |
| MRW | 0 | 0 = lectura/escritura; 1 = el tipo lo decide RW |

- **RAM libre (lectura y escritura):** MRW = 0 (RW indiferente) → cola acaba en `...0` tras el WS.
- **ROM solo lectura:** RW = 0, MRW = 1 → cola = WS×4 + 1.
- **Pantalla solo escritura:** RW = 1, MRW = 1 → cola = WS×4 + 3.

### RAMBAR — bits principales
| Campo | Bit | Valores |
|---|---|---|
| BA | 31-12 | dirección base de la RAM interna |
| WP | 8 | 1 = protegida contra escritura (0 si se escribe) |
| C/I | 5 | suele 1 |
| SC | 4 | 1 = **prohíbe** código supervisor |
| SD | 3 | 1 = **prohíbe** datos supervisor |
| UC | 2 | 1 = **prohíbe** código usuario |
| UD | 1 | 1 = **prohíbe** datos usuario |
| V | 0 | 1 = valida (enable) |

- **En las máscaras SC/SD/UC/UD el `1` PROHÍBE, el `0` PERMITE.** La 2ª letra = Código/Datos.
- Firmas: "solo datos, ambos modos" → SC SD UC UD = `1 0 1 0` → cola `$35`.
  "solo supervisor, todo" → `0 0 1 1` → cola `$37`. "solo usuario, todo" → `1 1 0 0` → cola `$39`.

### SR (para interrupciones, Tema 2)
- Bit S (supervisor) = 1; máscara I2-I0 ajusta el nivel que pasa. `$2000` o `$2100`, `$2700` enmascara todo.

---

## LOS 8 TIPOS DE EJERCICIO

### Tipo 1 — Mapa de memoria
Dar un sistema con 3-5 dispositivos (EPROM, RAM ext., puertos/ADC/pantalla, SIM, RAM interna) y un
listado parcial con `MBAR_MEM EQU`, `ORG` de variables, `ORG $0` (vectores), y `MOVEC D0,MBAR`.
Pedir rellenar inicio-fin de cada uno.
**Trampas a incluir a veces:** dispositivo sin dirección (va al hueco libre más bajo); `DS.L` (×4);
tamaño de EPROM deducible solo del BAM del CSOR; colisión que obliga a reubicar.
**Solución-tipo:** fin = inicio + tamaño − 1; EPROM en $0; SIM en MBAR_MEM (64 KB).

### Tipo 2 — Configurar CSBR/CSOR
Dar dispositivo (tipo, tamaño, bus, CS#, dirección, estados de espera, lectura/escritura).
Pedir CSBR y CSOR con justificación de cada campo.
**Solución-tipo:** BA del inicio; BW según bus (16→10); BAM del tamaño; cola = WS×4+RW×2+MRW.
**Trampa clásica:** confundir BAM (tamaño) con BA (base); invertir RW/MRW.

### Tipo 3 — RAMBAR
Dar base de la RAM interna y permisos (solo datos / solo supervisor / solo usuario / etc.) y si
se escribe. Pedir RAMBAR campo a campo.
**Solución-tipo:** WP=0 si escribe, C/I=1, V=1, y SC/SD/UC/UD con el 1 prohibiendo.

### Tipo 4 — Dimensionar la pila
Dar rango de RAM y un listado de variables (`DS.L`, `DS.B`) seguido de `PILA DS.B TAMPILA / FINPILA`.
Pedir A7 inicial y TAMPILA máximo.
**Solución-tipo:** A7 = última dir + 1 (sin variables); TAMPILA = tamaño RAM − bytes variables.
**Trampa:** meter variables en A7; olvidar ×4 de DS.L; dar TAMPILA mayor que la RAM.

### Tipo 5 — Dibujar el contenido de la pila
Dar A7 inicial (o rango de RAM) y un trozo de código con BSR/PEA/MOVE.L, y opcionalmente una o varias
**interrupciones** (incl. anidadas). Pedir el dibujo (dirección + contenido por casilla) y A7 final.
**Reglas:** BSR/JSR/PEA/MOVE.L = 4 bytes (PC o registro). Interrupción = **8 bytes** (PC 4 + SR 2 +
Formato/Vector 2). MOVEM = 4 × nº registros. Lo primero metido va abajo (dir. alta); lo último arriba
(dir. baja, donde apunta A7). La instrucción marcada con `****` aún NO se ha ejecutado.
Anidadas: un bloque de 8 por interrupción, en orden temporal (la 1ª abajo).
**Control:** las direcciones acaban en 0/4/8/C; si salen en 3/7/B/F, A7 inicial no llevó el +1.

### Tipo 6 — Dirección efectiva
Dar un operando tipo `d(An,Dm.L*escala)` con valores de An y Dm. Pedir la EA.
**Solución-tipo:** EA = d + An + Dm×escala (escala una vez; pasar decimales a hex).

### Tipo 7 — Trazar ensamblador
Dar memoria con bytes y un bucle (MOVE.B (A0)+, ADD, SUBQ, BNE) o una rutina con NOT/AND/LSR/indexado.
Pedir el valor final de un registro, o **qué hace** la rutina.
**Reglas clave:** `(A0)` = contenido en esa dirección (NO la dirección). La inicialización del contador
va fuera del bucle (antes de la etiqueta). AND = máscara (deja bits con 1). LSR = dividir por 2^n.
LEA = carga la dirección, no el contenido.
**Deducir "qué hace":** leer la condición del if — actualiza si mayor → máximo; si menor → mínimo;
siempre suma → acumulador. Mín suele empezar en $FF.../$FFFFFFFF; máx en 0.
**Saltos con/sin signo:** sin signo (relojes, direcciones, contadores) → BCS (menor), BHI (mayor),
BLS (≤), BCC (≥). Con signo → BLT, BGT, BLE, BGE. `CMP A,B` evalúa B respecto a A.

### Tipo 8 — Análisis de C
Dar una función con parámetros por valor y por referencia (puntero + `*`), y un `main`. Pedir el valor
final de las variables.
**Reglas:** pasado por valor (sin `&`) → la función NO cambia la variable (conserva su valor inicial).
Pasado por referencia (`&x`, dentro `*p`) → sí la cambia. `*=` es multiplicar. `v[i]` recorre arrays.
Trazar con tabla paso a paso; en una línea, usar el valor ya actualizado de la anterior.

---

## EJEMPLO COMPLETO DE EJERCICIO + SOLUCIÓN (para calibrar el nivel)

**Enunciado (Tipo 1):** MCF5272. EPROM 256 KB (CS0, bus 16 bits, programa+vectores);
RAM ext. 1 MB (CS1, bus 32 bits); puerto E/S de 4 bytes (CS2, sin dirección asignada); SIM.
`MBAR_MEM EQU $40000000`; variables en `ORG $00200000`; `ORG $0` con FINPILA e INICIO;
`MOVEC D0,MBAR`. Nota: asignar las menores direcciones posibles.

**Solución:**
- EPROM: $00000000 – $0003FFFF (en $0, 256 KB).
- Puerto E/S: $00040000 – $00040003 (hueco libre más bajo tras EPROM; 4 bytes → +3).
- RAM ext.: $00200000 – $002FFFFF (1 MB).
- SIM: $40000000 – $4000FFFF (MBAR_MEM, 64 KB).

---

## LAS DOS TRAMPAS QUE MÁS SE REPITEN (recordar al alumno)

1. **BAM sale del TAMAÑO, no de la base.** La base va en BA (CSBR); la máscara del tamaño va en BAM
   (CSOR). Empieza por `$FFF...`, nunca por `$00...`.
2. **A7 inicial = última dirección + 1** (nunca el último byte; las variables no entran aquí).

Otras: olvidar el −1 del fin; `DS.L` cuenta ×4; la interrupción son 8 bytes no 4; `(A0)` es contenido
no dirección; EPROM en $0 y SIM en MBAR_MEM (no intercambiarlos).
