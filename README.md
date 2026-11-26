[README.md](https://github.com/user-attachments/files/35471789/README.md)
# Procesador TDA 1819 — Extensión de Instrucciones de Pila (TP10)

Extensión de la arquitectura del procesador segmentado **TDA 1819** en **VHDL**, desarrollada para la cátedra de **Taller de Arquitectura** — Ingeniería en Computación, **UNLP** (Grupo 4).

El proyecto incorpora al conjunto de instrucciones (ISA) las operaciones de manejo de pila de 1 byte, 4 bytes y registros especiales que no estaban soportadas en la versión base del procesador, garantizando su correcta ejecución tanto con **cauce (pipeline) habilitado** como **deshabilitado**, e incluyendo detección de riesgos de datos y validación mediante simulación en **Active-HDL**.

---

## Resumen técnico

El procesador original únicamente disponía de `pushh` y `poph` (2 bytes sobre registros de propósito general). En esta extensión se implementaron **10 nuevas instrucciones**:

| Categoría | Instrucciones | Tamaño | Operando / Registro destino |
|---|---|---|---|
| Pila de propósito general (1 byte) | `pushb`, `popb` | 3 bytes | Registro de propósito general (`r1`–`r15`) |
| Pila de propósito general (4 bytes) | `pushw`, `popw` | 3 bytes | Registro de propósito general (`r1`–`r15`) |
| Pila de banderas enteras | `pushfl`, `popfl` | 2 bytes | Registro especial `FLAGS` |
| Pila de banderas de punto flotante | `pushfp`, `popfp` | 2 bytes | Registro especial `FPFLAGS` |
| Pila de dirección de retorno | `pushra`, `popra` | 2 bytes | Registro especial `RA` |

### Principales cambios en la arquitectura

- **Ensamblador e ISA** (`const_ensamblador.vhd`, `tipos_ensamblador.vhd`, `ensamblador.vhd`): incorporación de opcodes, tamaños de instrucción y parseo para instrucciones de pila de 3 bytes (con registro) y 2 bytes (sin operandos explícitos).
- **Etapa Decode** (`decode.vhd`): decodificación de las 10 nuevas instrucciones, generación de señales de control para transferencias de 1 y 4 bytes sobre `Stack_Memory` y actualización del puntero `SP`.
- **Banco de registros** (`registros.vhd`): soporte de escritura en la etapa **Writeback** sobre `FLAGS`, `FPFLAGS` y `RA` mediante señales auxiliares coordinadas con `ALU_Write` y `FPU_Write`, evitando conflictos de múltiples drivers (`'X'`) en simulación.
- **Control de riesgos (Hazards) y Fetch** (`raw_waw_detector.vhd`, `fetch.vhd`):
  - Contadores de escrituras pendientes para `FLAGS`, `FPFLAGS` (diferenciando escrituras provenientes de FPU y de `popfp` en WB) y `RA`.
  - Corrección en la etapa **Fetch** para sostener el freno (`while (StallRAW = '1')`) cuando una misma instrucción sufre atascos RAW consecutivos sin perder instrucciones siguientes.
- **Compatibilidad de compilación**: normalización de rutas sin tilde (`1. Maquina de Estados`) para evitar errores al empaquetar y compilar el workspace en Active-HDL.

---

## Equipo y distribución del trabajo

> Trabajo realizado de forma colaborativa por el **Grupo 4**. Dado que la integración al repositorio se consolidó a través de ramas puntuales, se detalla a continuación la responsabilidad principal de cada integrante por módulo:

| Integrante | Área / Módulo principal | Archivos clave |
|---|---|---|
| **Manuel Toledo** | Control de riesgos de datos (RAW/WAW) en pipeline y corrección de atascos consecutivos en Fetch | `raw_waw_detector.vhd`, `fetch.vhd` |
| **Rocío** | Programas de verificación en Assembler, simulación de waveforms (con/sin cauce) y documentación | `pila_sin.asm`, `pila_con.asm`, `pila_esp.asm`, `prueba_popfl.asm`, `prueba_raw.asm` |
| **Francisco** | Extensión del Ensamblador, definición de constantes/tipos del ISA e integración del proyecto | `const_ensamblador.vhd`, `tipos_ensamblador.vhd`, `ensamblador.vhd` |
| **Ludmila** | Decodificación de instrucciones de pila de 1/4 bytes y control de accesos a `Stack_Memory` / `SP` | `decode.vhd` |
| **Martín** | Banco de registros especiales (`FLAGS`, `FPFLAGS`, `RA`) en Writeback y pruebas de integración | `registros.vhd`, `usuario.vhd` |

---

## Resultados de verificación y rendimiento

Se diseñaron programas de prueba en Assembler para validar cada instrucción y comparar el desempeño frente a la implementación equivalente usando solo `pushh`/`poph` (`pila_sin.asm`):

| Programa | Qué valida | Ensamblado | Fin (cauce habilitado) | Fin (cauce deshabilitado) | Aceleración con pipeline |
|---|---|---|---|---|---|
| `pila_sin.asm` | Referencia usando solo `pushh`/`poph` | 221 ns | 3300 ns | 10300 ns | ~3.12× |
| `pila_con.asm` | Misma tarea con `pushw`/`popw`, `pushb`/`popb` y `pushfl`/`popfl` | 157 ns | 2600 ns | 7700 ns | ~2.96× |
| `pila_esp.asm` | Registros especiales (`pushfp`/`popfp`, `pushra`/`popra`, `pushfl`/`popb`) y salto `bfpt` | 253 ns | 5200 ns | 14400 ns | ~2.77× |

- Comparando `pila_con.asm` frente a `pila_sin.asm` con pipeline habilitado, el uso de las nuevas instrucciones redujo el tiempo total de ejecución en **~21%** (de 3300 ns a 2600 ns) y el tamaño/tiempo de ensamblado en **~29%**.
- Se incluyeron además tres pruebas unitarias auxiliares: `prueba_popfl.asm` (restauración de banderas tras operaciones aritméticas/comparación), `prueba_popfp.asm` (preservación y restauración de `FPFLAGS` frente a comparaciones flotantes) y `prueba_raw.asm` (doble atasco RAW consecutivo en Fetch).

---

## Cómo ejecutarlo en Active-HDL

1. Abrir el workspace `TDA_1819.wsw` (**File > Open Workspace**).
2. Ejecutar **Design > Compile All** y verificar que compile sin errores. (Si `usuario` no figura como top-level en el Design Browser: clic derecho sobre `usuario (USUARIO_ARCHITECTURE)` > **Set as Top-Level**).
3. En `TDA_1819/src/Usuario/usuario.vhd` (líneas 42–43), configurar el programa y el modo de ejecución:
   ```vhdl
   ProgName   : STRING  := "pila_con.asm";  -- pila_sin.asm | pila_con.asm | pila_esp.asm
   Pipelining : BOOLEAN := true;            -- true = cauce habilitado | false = deshabilitado
   ```
4. Guardar y compilar `usuario.vhd`. (Cada vez que se cambia el programa, ejecutar **Simulation > End Simulation** antes de volver a inicializar).
5. Iniciar la simulación (**Simulation > Initialize Simulation**), abrir **File > New > Waveform** y arrastrar desde la pestaña **Structure**:
   - Desde `usuario > UUT : pc > UUT2 : cpu > UUT7 : registros`: `SP`, `FLAGS` (Radix: Binary), `FPFLAGS`, `RA`, `R1`, `R2`, `R5`, `R6` (Radix: Hexadecimal).
   - Desde `usuario > UUT : pc > UUT4 : memoria`: `Data_Memory` y `Stack_Memory`.
6. Ejecutar `run` en la consola.

---

## Comportamiento esperado en simulación (cauce habilitado)

- **`pila_con.asm` (fin ~2600 ns):** `SP` evoluciona `8000 → 7FFC (pushw) → 7FFB (pushb) → 7FFA (pushfl)` y retorna a `8000`; `FLAGS` restaura `ZZ000001` tras `popfl`; `R1` recupera `000083EB` y `R2` recupera `00000007`.
- **`pila_esp.asm` (fin ~5200 ns):** `FPFLAGS` restaura `?5` tras `popfp` habilitando el salto condicional `bfpt` (`R5 = 00000007`); `RA` y `R2` cargan `00002030` mediante `popra` y `pushra`/`popw`.
- **Contraste con procesador original:** ejecutando `pila_con.asm` y `pila_esp.asm` sobre el TDA 1819 base de cátedra, la simulación se detiene por instrucción inválida (`pushw` a los 59 ns y `pushfp` a los 71 ns).

### Problemas comunes

| Problema | Solución |
|---|---|
| `"la instruccion declarada no es valida"` en una instrucción de pila | Quedó compilada una versión anterior: ejecutar **Design > Compile All** |
| El waveform queda vacío al simular | Las señales se agregaron antes de inicializar: hacer **Initialize Simulation** y arrastrarlas nuevamente |
| Se cambió `ProgName` pero sigue corriendo el programa anterior | Ejecutar **Simulation > End Simulation**, recompilar `usuario.vhd` e inicializar de nuevo |
| Error al localizar la máquina de estados | Verificar que la carpeta sea `1. Maquina de Estados` (sin tilde) |

---

## Tecnologías y herramientas

- **VHDL** (descripción de hardware de procesador segmentado)
- **Active-HDL** (compilación, simulación funcional y análisis de waveforms)
- **Assembler TDA 1819** (batería de pruebas funcionales y benchmarks)
