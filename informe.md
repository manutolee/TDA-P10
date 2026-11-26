# TP10 - Taller de Arquitectura - Grupo 4

Procesador TDA 1819 con las instrucciones de pila que faltaban (ej1):
pushb/popb, pushw/popw, pushfl/popfl, pushfp/popfp y pushra/popra.

## Como correrlo en Active-HDL

1. Descomprimir el zip y abrir `TDA_1819.wsw` (File > Open Workspace).
2. Design > Compile All. Tiene que terminar sin errores.
3. En `TDA_1819/src/Usuario/usuario.vhd` elegir el programa y el modo del cauce:
   - `ProgName`: `pila_sin.asm`, `pila_con.asm` o `pila_esp.asm`
   - `Pipelining`: `true` (cauce habilitado) o `false` (deshabilitado)
4. Guardar, compilar `usuario.vhd` y correr la simulacion (Simulation > Initialize Simulation y despues `run`).

Si la consola dice "la instruccion declarada no es valida" en una instruccion de pila,
es que quedo compilada una version vieja: hacer Compile All de nuevo.

## Tiempos esperados (consola)

| Programa | Cauce habilitado | Cauce deshabilitado |
|---|---|---|
| pila_sin.asm | ensamblado 221 ns, fin 3300 ns | ensamblado 221 ns, fin 10300 ns |
| pila_con.asm | ensamblado 157 ns, fin 2600 ns | ensamblado 157 ns, fin 7700 ns |
| pila_esp.asm | ensamblado 253 ns, fin 5200 ns | ensamblado 253 ns, fin 14400 ns |

## Archivos modificados

- `src/Packages/const_ensamblador.vhd`
- `src/Packages/tipos_ensamblador.vhd`
- `src/Usuario/PC/1. Ensamblador/ensamblador.vhd`
- `src/Usuario/PC/2. CPU/2. Etapas/1. Fetch/fetch.vhd`
- `src/Usuario/PC/2. CPU/2. Etapas/2. Decode/decode.vhd`
- `src/Usuario/PC/2. CPU/4. Atascos/2. Datos/raw_waw_detector.vhd`
- `src/Usuario/PC/2. CPU/7. Registros/registros.vhd`
- `src/Usuario/usuario.vhd`

Programas: `src/Assembler/pila_sin.asm`, `pila_con.asm`, `pila_esp.asm`
y las pruebas auxiliares `prueba_popfl.asm`, `prueba_popfp.asm` y `prueba_raw.asm`.

La carpeta `1. Maquina de Estados` esta sin tilde a proposito: con la tilde el nombre
se rompia al comprimir/descomprimir y el proyecto no compilaba.

## Tutorial: ver todo en el waveform y comprobar que coincide con el informe

### 1. Preparar el proyecto

1. Abrir `TDA_1819.wsw` y hacer **Design > Compile All** (tiene que terminar sin errores).
2. Si en el Design Browser `usuario` no aparece como top-level: clic derecho sobre
   `usuario (USUARIO_ARCHITECTURE)` > **Set as Top-Level**.

### 2. Elegir el programa

En `TDA_1819/src/Usuario/usuario.vhd` (lineas 42-43):

```vhdl
ProgName   : STRING  := "pila_con.asm";  -- pila_sin.asm / pila_con.asm / pila_esp.asm
Pipelining : BOOLEAN := true;            -- true = cauce habilitado, false = deshabilitado
```

Guardar y compilar `usuario.vhd` (clic derecho > Compile). Cada vez que se cambia el
programa hay que hacer **Simulation > End Simulation** y volver a inicializar.

### 3. Armar el waveform

1. **Simulation > Initialize Simulation**.
2. **File > New > Waveform**. Se abre una pestana vacia.
3. En el Design Browser, pestana **Structure**, abrir la jerarquia:
   `usuario > UUT : pc > UUT2 : cpu > UUT7 : registros`.
   Abajo aparece la lista de senales. Arrastrar al waveform:
   `SP`, `FLAGS`, `FPFLAGS`, `RA`, `R1`, `R2`, `R5`, `R6`.
4. Despues ir a `usuario > UUT : pc > UUT4 : memoria` y arrastrar `Data_Memory` y `Stack_Memory`.
5. Formato de cada senal (clic derecho sobre el nombre > **Radix**):
   - `SP`, `RA`, `R1`, `R2`, `R5`, `R6`: **Hexadecimal**.
   - `FLAGS`: **Binary** (para ver el bit Z, que es el bit 5: `ZZ?ZZZZZ`).
   - `FPFLAGS`: Hexadecimal.
6. En la consola escribir `run` (o el boton Run). La simulacion para sola al terminar el programa.
7. Para no repetir todo: **File > Save As** y guardar el waveform como `waveform.awc`.
   La proxima vez se abre con File > Open y se vuelve a hacer `run`.

Notas para leer los valores:
- La memoria es little endian: 000083EBh se ve como `EB 83 00 00`.
- `U` = sin inicializar, `Z` = alta impedancia, `?` = digito hexa con algun bit U o Z.
  FLAGS empieza en `ZZZZZZZZ` y los bits F e I nunca los escribe la ALU, por eso FLAGS
  se ve como `?1` o `?0` en hexa.
- `Data_Memory` empieza en 4096 (1000h) y `Stack_Memory` llega hasta 32767 (7FFFh).
  Con el `+` al lado del nombre se despliegan las direcciones.

### 4. Que hay que ver en cada programa (cauce habilitado)

Los tiempos son aproximados: puede variar algun ns segun donde se pare el cursor.

**pila_con.asm** (fin a los 2600 ns) - informe 3.3.1, figuras 5, 6 y 7

| Senal | Que pasa |
|---|---|
| R1 | 000083EB (~825 ns) > 00039B6D por el dmul (~1425 ns) > vuelve a 000083EB con popw (~2125 ns) |
| R2 | 00000007 > 00000000 por dsub r2,r2,r2 (~1825 ns) > vuelve a 00000007 con popb (~2025 ns) |
| R5 | 00000006 (7 - 1) |
| SP | 8000 > 7FFC (pushw) > 7FFB (pushb) > 7FFA (pushfl) > 7FFB > 7FFC > 8000 |
| FLAGS (binario) | ZZ000001 > ZZ000ZZ1 durante dmul > ZZ100001 (Z=1) > ZZ000001 despues de popfl |
| Data_Memory | 4096-4099 = EB 83 00 00 (LEGAJO), 4100-4101 = 07 01, 4102-4105 = 6D 9B 03 00 (PROD), 4106-4109 = EB 83 00 00 (COPIA) |
| Stack_Memory | 32764-32767 = EB 83 00 00, 32763 = 07, 32762 = ?1 (FLAGS), 32761 = UU |

**pila_sin.asm** (fin a los 3300 ns) - misma tarea con pushh/poph

| Senal | Que pasa |
|---|---|
| SP | 8000 > 7FFE > 7FFC > 7FFA (tres pushh de 2 bytes) > 7FFC > 7FFE > 8000 |
| R1, R2, R5 | mismos valores finales que pila_con (000083EB, 00000007, 00000006) |
| FLAGS | termina en ZZ000001 porque se repite el dsub al final (no hay pushfl) |
| Data_Memory | PROD = 6D 9B 03 00 y COPIA = EB 83 00 00, igual que pila_con |

**pila_esp.asm** (fin a los 5200 ns) - informe 3.3.2, figuras 8 a 11

| Senal | Que pasa |
|---|---|
| FPFLAGS | ZZ > ?5 despues de ltf > ?0 despues de eqf > ?5 otra vez despues de popfp (~2425 ns) |
| R5 | 00000007 (~3025 ns): bfpt salto porque popfp restauro FPFLAGS (si fallara daria 1) |
| R1 | 00002030 (~3425 ns) |
| RA | 00000000 > 00002030 con popra (~3825 ns) |
| R2 | 00002030 con pushra + popw (~4225 ns) |
| R6 | 000000?0 (byte de FLAGS que se desapilo con popb) |
| SP | 7FFF con pushfp; despues 7FFC (3454-3552 ns), 7FFC (3854-3952 ns) y 7FFF (4356-4452 ns) |
| Data_Memory | 4108-4111 = 07 00 00 00 (RES), 4112-4115 = 30 20 00 00 (COPIARA), 4116 = ?0 (BANDERAS) |

### 5. Medir los tramos de SP con cursores (figura 9 del informe)

1. Hacer zoom en el waveform entre 3400 y 4500 ns (lupa + de la barra del waveform).
2. Seleccionar la fila `SP` y poner un cursor donde baja a 7FFC (clic en el waveform).
3. Agregar otro cursor (clic derecho en la regla de tiempo > Add Cursor) donde vuelve a 8000.
   Para que caiga justo en el cambio se pueden usar los botones Next/Previous Transition.
4. Abajo aparece la diferencia entre cursores. Tiene que dar:
   - pushw/popra: 3454 > 3552 = **98 ns**
   - pushra/popw: 3854 > 3952 = **98 ns**
   - pushfl/popb: 4356 > 4452 = **96 ns**
   - entre popra y pushra (3552 > 3854): **302 ns**, porque pushra espera a que popra escriba RA.

### 6. Con el cauce deshabilitado

Poner `Pipelining := false`, compilar y repetir. Los valores finales son los mismos;
solo cambia cuando termina: pila_sin 10300 ns, pila_con 7700 ns y pila_esp 14400 ns
(figura 12 del informe para pila_esp: R5 = 7, FPFLAGS = ?5, RA = R2 = 00002030).

### 7. Contraste con el procesador original (informe 2.3)

En una copia del TDA_1819 original de la catedra (sin nuestros cambios), copiar los tres
`.asm` en `src/Assembler/` y correrlos:
- pila_con.asm: se frena a los 59 ns, "Error en la linea 12" (pushw r1).
- pila_esp.asm: se frena a los 71 ns, "Error en la linea 16" (pushfp).
- pila_sin.asm: anda igual que en el nuestro (221 ns de ensamblado, fin a los 3300 ns).

### 8. Pruebas auxiliares (opcional)

- **prueba_popfl.asm** (fin a los 2000 ns): FLAGS pasa a ZZ100001 con el dsub, popfl lo
  vuelve a ZZ000001 y despues el daddi lo deja en ZZ000000. R12 = FFFFFFFB (-5),
  R15 = FFFFFFFB (popw) y R14 = 000000FB (popb).
- **prueba_popfp.asm** (fin a los 3100 ns): ltf deja F = 1, pushfp y popfp lo mantienen, y despues eqf lo pone
  en F = 0. bfpt MAL no tiene que saltar, asi que R5 = 00000007 y RES = 7. Si diera 1,
  popfp habria pisado el resultado de eqf.
- **prueba_raw.asm** (fin a los 1400 ns): R5 = 00000006, R9 = 00000005 y R10 = 00000006.
  Prueba el caso del dsub que se atasca dos veces seguidas (fetch.vhd).

### Problemas comunes

| Problema | Solucion |
|---|---|
| "la instruccion declarada no es valida" en una instruccion de pila | Quedo compilada una version vieja: Compile All |
| El waveform queda vacio | Se agregaron las senales antes de inicializar la simulacion: Initialize Simulation y arrastrarlas de nuevo |
| Cambie ProgName y sigue corriendo el programa anterior | End Simulation, compilar usuario.vhd e inicializar de nuevo |
| No encuentra la carpeta de la maquina de estados | Es `1. Maquina de Estados`, sin tilde (ver arriba) |
