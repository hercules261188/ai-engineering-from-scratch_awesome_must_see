# Reconstruir y probar iguales: pruebas diferenciales, comparación de byte y el ciclo de pruebas

> Pasar sus pruebas demuestra que están de acuerdo en sus pruebas, así que la igualdad necesita una cuadrícula que llegue a todos los bordes y bytes que coincidan después de la reubicación enmascarado.

**Type:** Build
**Languages:** Python, C
**Prerequisites:** Phase 13 · 34 (lifting to pseudocode), Phase 13 · 35 (evidence, confidence, and unknowns)
**Time:** ~110 minutes

## Objetivos de aprendizaje

- Escriba una reconstrucción de la lista de instrucciones y nombre el registro de evidencia de donde proviene.
- Construye una red de entrada con una clase por peligro: cero guardia, valores negativos, división truncada, desbordamiento de 32 bits y una trampa de división.
- ejecuta el original y la reconstrucción en un programa y graba cada célula, incluyendo una trampa.
- Recompillar la reconstrucción con las banderas originales y comparar los bytes de función después de ocultar los campos de reubicación.
- Lea las constantes que una reubicación apunta y compártala, para que no pase una letra errónea.
- Envía el servidor MCP de rea desde un pequeño cliente de estudio y graba cada paso con su identificación de evidencia.
- La acuña un registro de prueba cerrado por fallo cuya identidad es un hash sobre sus afirmaciones y vínculos de evidencia.

## El problema

Usted pidió a rea el expediente de funciones de `percent_change`en nuestro `ledger-x86_64-O1.o`Ghidra 12.1.4 devolvió un pseudocodo, y el agente escribió este C de él:

```c
int percent_change(int before, int after) {
    if (before == 0) return 0;
    return (int)(((long)after - before) * 100 / before);
}
```

Pasaron cuatro pruebas:`(100, 150)`da 50, `(200, 100)`Da -50, `(0, 5)`da 0, `(50, 100)`El agente también se reconstruyó.`grade`como cuatro `if`Se han pasado seis pruebas más, el informe dice "verificado".

Ese informe no responde a ninguna de estas preguntas:

- ¿ Qué regresa el original cuando ?`(after - before) * 100`¿No encaja en 32 bits?
- ¿Qué devuelve el original para una puntuación de 110?
- ¿La reconstrucción se compiló a los mismos 16 bytes que el original?
- Cuando dos funciones compiladas tienen los mismos bytes, ¿son sus reubicaciones el punto en las mismas constantes?
- ¿De qué registro proviene la reconstrucción, y el agente leyó la lista o el pseudocodo?
- ¿Qué estado obtiene la prueba en un host sin un compilador C?

Cada objetivo de esta lección es un programa que escribimos y compilamos.

```figure
re-36-prove-equal
```

## Lo que dice la evidencia

El bucle de agente en `code/main.py`Ha hecho seis llamadas a rea 6.5.0 por el estudio.`code/fixtures/runs/agent-run.json`El servidor respondió .`initialize`con la versión de protocolo `2025-11-25`Tres llamadas devolvieron los registros de la evidencia:

| Step | Tool | Time | Evidence id |
|---|---|---|---|
| 2 | `analyze_function percent_change` | 6.31 s | `ev_e88b10dc0c6400e51bb6f7fdc3eaea8d2c97cc02463f0db3e6afeefc273197cd` |
| 3 | `analyze_function grade` | 0.02 s | `ev_67b6d3f992eaf49a2e16de084b6ebfdd9884e8c5c7f75e175b7bb20f4f748ce1` |
| 4 | `read_bytes 0x1000b4, 20` | 0.00 s | `ev_c26146cfedf4c1ee146d8ff00cfa9a5d3f5335bdd9c97ccf4e1af5f6ea3ed6bc` |

La primera llamada pagó por la importación y el autoanálisis de Ghidra.`confidence: "observed"`¿ Qué ?`authority: "shipped-artifact"`, y el SHA-256 del objeto,`9a4a8082ba5e285a732552981a6935e0c5255725664f380e37c44ebfb04d803f`Ghidra carga el objeto trasladable con `.text`En el`0x100000`, así que`percent_change`Se sienta en`0x100000`y `grade`En el`0x100010`¿ Qué ?

El pseudocodo para `percent_change`Es donde el `long`proveniente de:

```c
undefined1  [16] percent_change(int param_1,int param_2,ulong param_3)
{
  long lVar1;
  undefined1 auVar3 [16];
  if (param_1 != 0) {
    lVar1 = (long)((param_2 - param_1) * 100);
    auVar3._0_8_ = lVar1 / (long)param_1 & 0xffffffff;
    auVar3._8_8_ = lVar1 % (long)param_1 & 0xffffffff;
    return auVar3;
  }
  ...
}
```

El descompilador vio .`cltd`y `idivl`Escriba ambos .`edx`y `eax`y se modela un valor de retorno de 16 bytes con una división de 64 bits.

```
0x100000: TEST EDI, EDI
0x100002: JZ 0x0010000d
0x100004: SUB ESI, EDI
0x100006: IMUL EAX, ESI, 0x64
0x100009: CDQ
0x10000a: IDIV EDI
0x10000c: RET
0x10000d: XOR EAX, EAX
0x10000f: RET
```

Cada operación tiene 32 bits de ancho.`IMUL EAX, ESI, 0x64`se multiplica en 32 bits y mantiene la mitad baja. `CDQ`signos de extensión `eax`en el`edx`, y `IDIV EDI`divide el par de 64 bits por un divisor de 32 bits. Una reconstrucción que sigue a la lista es `100 * (after - before) / before`En el`int`¿ Qué ?

El `grade`El pseudocodo nombra una tabla,`*(uint *)(&DAT_001000b4 + (ulong)uVar2 * 4)`, y una expresión predeterminada ilegible construida a partir de `0x66666667`La lista muestra la división por 10 como `IMUL RAX, RAX, 0x66666667`¿ Qué ?`SAR RCX, 0x22`¿ Qué ?`SHR RAX, 0x3f`¿ Qué ?`ADD ECX, EAX`Entonces ...`ADD ECX, -0x6`¿ Qué ?`CMP ECX, 0x4`, y `JA`Mantenga decenas de 6 a 10.`XOR EAX, EAX`antes de la comparación es el valor predeterminado de 0. `LEA RCX, [0x1000b4]`y `MOV EAX, dword ptr [RCX + RAX*0x4]`leer una tabla de cinco entradas de 4 bytes.

Paso 4 lee esos 20 bytes: `01000000 02000000 03000000 04000000 04000000`La mesa es`1, 2, 3, 4, 4`para diez, 6, 7, 8, 9, 10. Eso es el `switch`En el`ledger.c`, con `case 10`y `case 9`compartiendo el valor 4.

Ambas pruebas en las páginas de pruebas de rea se iniciaron en la lista y en `read_bytes`El primer pseudocodo de DX-Ball fue`FUN_00406400(void)`No hay entrada de pila, y la vista de instrucciones lo corrigió. Lea la lista primero. Cite la identificación de registro, luego escriba el C.

## Escriba a los candidatos

La carpeta`code/fixtures/candidates/`El código de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de la lista de datos de datos de la lista de datos de datos de la lista de datos de datos de la lista de datos de datos de la lista de datos de datos de datos de la lista de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de la red de datos de datos de la red de datos de datos de datos de la red de datos de datos de la red de datos de datos de datos de la red de datos de datos de datos de la red de datos de datos de datos de la red de red de datos de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red

| File | Source | What it does |
|---|---|---|
| `percent_change_mul_first.c` | `100 * (after - before) / before` | the listing in C |
| `percent_change_split.c` | `(after * 100 - before * 100) / before` | different text, same bytes after clang folds it |
| `percent_change_wide.c` | `(int)(((long)after - before) * 100 / before)` | the pseudocode's `long`, divides before it wraps |
| `percent_change_div_first.c` | `(after - before) / before * 100` | divides first, so rounding differs |
| `grade_switch.c` | `switch (score / 10)` with the five cases | the original source |
| `grade_table.c` | `unsigned slot = score / 10 - 6; slot < 5 ? points[slot] : 0` | the listing in C |
| `grade_thresholds.c` | `if (score >= 90) return 4; ...` | no upper bound on the top band |
| `grade_case10_wrong.c` | `case 10: return 3;` | one table entry wrong |

Cada archivo se compiló con el comando de fijación, `clang -target x86_64-unknown-linux-gnu -O1 -fno-asynchronous-unwind-tables -c`Los objetos compilados se componen junto a las fuentes para que las pruebas se ejecuten en un host sin clang.

## Primera pierna: ejecutar y comparar

La definición de McKeeman de 1998 de pruebas diferenciales requiere "dos o más sistemas comparables" alimentados con "casos de prueba generados mecánicamente", y trata un resultado diferente, una colisión o un accidente como candidato a un error. Él nombró la parte difícil: "El problema más feo en la prueba es evaluar el resultado de una prueba". La función original es el oráculo aquí. No necesitamos saber la respuesta correcta. Necesitamos el original y la reconstrucción para no estar en desacuerdo en ninguna parte de una red que alcance todos los peligros.

### Diseño de la red

`input_grid()`En el`code/main.py`construye 92 células.`percent_change`Se necesita cada par de diez .`before`valores y siete `after`valores:

```
before: 0, 1, -1, 3, -3, 7, 100, -100, 2147483647, -2147483648
after:  0, 5, -5, 105, 536870911, 2147483647, -2147483648
```

Para`grade`Se necesitan 22 puntuaciones: ambos lados de cada banda, negativos, 0, 110, y los dos `int`Los límites.`classify()`La truncation negativa tiene 5, redondeando 2, más allá del interruptor 2, y la trampa 1.

Cada clase existe por una razón.`TEST EDI, EDI`Las células negativas comprueban la división truncante: C y `idiv`redondo hacia cero, así que `-9 / 10`es 0 y `-10 / 10`es -1 . Las células redondas comprueban que la multiplicación ocurre antes de la división, así que `(105 - 100) * 100 / 100`es 5 y no 0.`(after - before) * 100`La única célula de trampa hace que el producto envuelto sea igual.`INT_MIN`con`before`igual a -1.

Yang, Chen, Eide y Regehr construyeron Csmith para generar programas sin comportamiento indefinido. Un desacuerdo del oráculo sobre comportamiento indefinido es ruido y nunca un informe de error.

### El conductor y la trampa

`code/driver.c`lee una célula por línea de stdin e imprime las entradas, el resultado original y el resultado de reconstrucción. `build_harness()`compone al candidato con `-Dpercent_change=cand_percent_change -Dgrade=cand_grade`En un host x86-64 Linux el original es el objeto fijo en sí.`ledger.c`Compilado de forma nativa, y los registros del informe `original_kind: fixture-source`¿ Qué ?

El conductor instala un `SIGFPE`el manipulador con `sigsetjmp`y `siglongjmp`- Para .`before = -1`y `after = 536870911`, la diferencia es 536870912 y el producto de 32 bits se envuelve a -2147483648.`idiv`El coeficiente no encaja en 32 bits (Intel SDM Volumen 2, IDIV).`trap`¿ Qué ?

En el AArch64, `sdiv`El oráculo depende de la arquitectura, y el informe dice cuál se ejecutó.

### Resultados

`prove_behaviour(PLAUSIBLE)`construye el `long`Los diez casos ingenuos pasan, la red falla en 28 celdas:

| Function | Inputs | Class | Original | Rebuild |
|---|---|---|---|---|
| `percent_change` | 3, 2147483647 | overflow | -133 | -1431655899 |
| `percent_change` | 100, 2147483647 | overflow | -101 | 2147483547 |
| `percent_change` | 2147483647, 5 | overflow | 0 | -99 |
| `grade` | 110 | beyond the switch | 0 | 4 |
| `grade` | 2147483647 | beyond the switch | 0 | 4 |

Todos los 26 .`percent_change`las fallas son células de sobrecarga con `|before| > 1`- Para .`before`igual a 1 o -1 el cuotiente de 64 bits se reduce a la cantidad de contenido envuelto, por lo que estas células coinciden accidentalmente.`grade`fracasa porque el interruptor original no tiene ningún caso para decenas de 11 y superiores, y `110 / 10`Es el 11.

`prove_behaviour(EQUAL)`construye`percent_change_mul_first.c`y `grade_switch.c`Todas las 92 células están de acuerdo.`percent_change_div_first.c`No se puede encontrar en dos de los cuatro casos ingenuos, que es el único caso que un examen escrito a mano encuentra.

`oracle_disagreements()`- ¿Qué? - ¿Qué?`percent_change_model()`y `grade_model()`son modelos Python de la lista, con `wrap32()`para el multiplicador de 32 bits y `idiv32()`El informe compara el modelo con el original observado en cada célula.

## Segunda pierna: recopila y compara

El segundo paso pregunta si la reconstrucción, compilada con el mismo compilador y banderas, produce los mismos bytes. El proyecto DX-Ball informó "63 bytes coincidentes" para su ayudante. El proyecto TH04 informó un match de 2.139 bytes. Cada uno usó el compilador de su propio período. rea no puede hacerlo. su propia `compare_functions`el texto del proveedor de line-diffs y se adjunta la limitación "El ensamblaje es texto opaco del proveedor; la normalización de la reubicación no está disponible".

### Lea el objeto

`ElfObject`En el`code/main.py`se lee un objeto relocable ELF64 con `struct.unpack_from`Verifica la magia, la clase de 64 bits, los datos de pequeños indios,`e_machine`62, y `e_type`Luego se recorren los encabezados de la sección, la tabla de cuerdas,`.symtab`, y cada `.rela`En el dispositivo,`percent_change`comienza en`.text`de compensación 0 con tamaño 16 y `grade`En el`0x10`con tamaño 46. `total`comienza en`0x40`con tamaño 31 y `grade_label`En el`0x60`con el tamaño 63.

Los 16 bytes de `percent_change`¿ Qué ?`85 ff 74 09 29 fe 6b c6 64 99 f7 ff c3 31 c0 c3`La función no tiene traslados.`grade`tiene uno: compensación `0x26`dentro de la función, tipo `R_X86_64_PC32`, símbolo`.rodata`, añadido`0x10`Pertenece a los 7 bytes .`leaq`en compensación `0x23`, cuyos últimos cuatro bytes son el campo de desplazamiento.

### Máscara, luego compara los registros

En un objeto RELA el agregado vive en el registro de reubicación, y los cuatro bytes de campo son cero.`grade_switch.o`Un simple byte comparar pasa mientras las dos reubicaciones nombran lugares diferentes. `mask_relocations()`cero el campo en ambos lados de todos modos, por lo que el mismo método se aplica para los objetos REL y para Mach-O, donde el campo lleva el adendo.

Entonces ...`prove_bytes()`compara los registros de reubicación por sección de compensación, tipo y objetivo.`.rodata+0x10`y el candidato dice:`.rodata-0x4`El adjunto difiere porque el objeto candidato no tiene `labels.rel`Una función diferente es una diferencia de compensación, no una función diferente.

### Lea las constantes los puntos de reubicación en

Para un campo relacionado con el PC que termina su instrucción, la CPU agrega la siguiente dirección de instrucción, por lo que el objetivo es símbolo más adendo más 4. En el original que es `.rodata+0x14`, y en el candidato es`.rodata+0x0`- ¿ Qué ?`read_table()`Aquí hay cinco entradas de 4 bytes.`1, 2, 3, 4, 4`¿ Qué ?

El número cinco viene de la lista:`CMP ECX, 0x4`con`JA`admite índices de 0 a 4, y `[RCX + RAX*0x4]`Lee 4 bytes por índice. `CLAIMS`En el`code/main.py`registros que tienen la forma de `grade`y registros de cero traslados para `percent_change`¿ Qué ?

Este paso existe debido a un error documentado por el proyecto TH08. Su comparación anterior "replayó la dirección de destino en el campo de reubicación de la instrucción, luego comparó bytes de instrucción sin verificar los datos literarios flotantes en esa dirección. Fuente `0.0`por lo tanto podría parecer exacto en contra del objetivo `128.0`." La corrección auditó 1.548 referencias literarias flotantes y corrigió otras doce. `grade_case10_wrong.c`Su código de 46 bytes es igual al original, su registro de reubicación tiene la misma forma, y su tabla es`1, 2, 3, 4, 3`Sólo el chequeo constante lo falla.

### Resultados

| Candidate | Size | Masked bytes | Relocation records | Table | Status |
|---|---|---|---|---|---|
| `percent_change_mul_first` | 16 vs 16 | equal | none | none | pass |
| `percent_change_split` | 16 vs 16 | equal | none | none | pass |
| `percent_change_wide` | 16 vs 26 | size differs | none | none | fail |
| `percent_change_div_first` | 16 vs 18 | size differs | none | none | fail |
| `grade_switch` | 46 vs 46 | equal | same shape | `1,2,3,4,4` both | pass |
| `grade_table` | 46 vs 46 | equal | same shape | `1,2,3,4,4` both | pass |
| `grade_thresholds` | 46 vs 39 | size differs | 1 vs 0 | none | fail |
| `grade_case10_wrong` | 46 vs 46 | equal | same shape | `1,2,3,4,4` vs `1,2,3,4,3` | fail |

Dos resultados merecen una segunda mirada.`percent_change_split.c`tiene texto fuente diferente y bytes idénticos, porque el clang trata el desbordamiento firmado como indefinido y se pliega `after * 100 - before * 100`en un multiplicador.`grade_table.c`También coincide byte por byte, por lo que el compilador llegó a la misma tabla desde un índice de matriz y desde un `switch`. Una coincidencia de byte prueba que este compilador, con estas banderas, produjo el mismo código del candidato.

El tamaño es un diagnóstico y no una aceptación. las páginas de evidencia de rea registran un candidato de 1,825 bytes con la longitud correcta y 467 bytes diferentes.`wide`reconstruir es 26 bytes porque `movslq`¿ Qué ?`imulq`¿ Qué ?`cqto`, y `idivq`El listado nos había dicho el ancho antes de cualquier compilación.

## El agente Loop Over rea

Seiscientos y cinco de las herramientas MCP de rea no tienen comando de CLI, y `verify_reconstruction`Es uno de ellos.`McpStdioClient`En el`code/main.py`Es el cliente más pequeño que llega a ellos.`npx -y rea-agents@6.5.0 mcp`, escribe una solicitud JSON-RPC por línea a stdin, y lee un objeto JSON por línea de stdout en un hilo. Envía `initialize`con la versión de protocolo `2025-11-25`, entonces`notifications/initialized`, entonces`tools/call`. El plazo predeterminado por llamada es de 330 s, el mismo número rea permite un inicio en frío de Ghidra.

`agent_loop()`hace las seis llamadas y graba cada una:

```json
{
  "step": 2,
  "tool": "analyze_function",
  "arguments": {"procedure": "percent_change"},
  "seconds": 6.31,
  "is_error": false,
  "evidence_id": "ev_e88b10dc0c6400e51bb6f7fdc3eaea8d2c97cc02463f0db3e6afeefc273197cd",
  "confidence": "observed",
  "authority": "shipped-artifact",
  "limitations": ["Resolved reference metadata identifies computed, indirect, external, call, jump, and data edges; unresolved targetless flows remain unknown, and function classifications distinguish thunks and externals.", "..."],
  "result_sha256": "..."
}
```

Los registros completos de la Evidencia se almacenan bajo sus identidades en el mismo archivo.`table_address()`encuentra el `LEA`operand en el `grade`lista y se alimenta a `read_bytes`- ¿ Qué ?`open_binary`¿ Qué ?`get_evidence_bundle`, y `close_binary`no devuelven ningún registro de la Evidencia y se registran con una identificación nula.

`run_agent(live=True)`conducir rea cuando `npx`Está en el camino y `GHIDRA_INSTALL_DIR`En caso contrario, reproduzce el dispositivo y dice así en stderr.`code/tests/fake_rea_server.py`es un servidor alternativo que responde a cada uno `tools/call`El cliente y el bucle se ejecutan contra él en cada prueba, sin rea, Ghidra o Java instalados.

### Lo que el verificador de rea puede y no puede decir

¿Qué es esto?`verify_reconstruction`La prueba y los resultados de la prueba pasan, fallan o son desconocidos por reclamación. su limitación fija dice "Pass significa cada reclamación declarada aprobada; no establece equivalencia global de implementación". Una afirmación sobre una función compara el texto del proveedor de`compare_functions`Una afirmación comportamental compara dos capturas de procesos registradas de `compare_process_captures`Cuando un registro fuente tiene una cobertura incompleta de llamadas, una reclamación sobre llamadas es `unknown`Nada dentro de rea compila una función reconstruida o lo ejecuta sobre su red de entrada.

El libro mayor de obligaciones de rea espera un registro de prueba con `predicate_type: "rea.reconstruction-proof"`¿ Qué ?`operation: "verify_reconstruction_obligations"`¿ Qué ?`confidence: "observed"`, y `normalized_result.passed === true`No hay una herramienta pública que emita una.`build_proof()`En el`code/main.py`La menta:

```json
{
  "predicate_type": "rea.reconstruction-proof",
  "operation": "verify_reconstruction_obligations",
  "confidence": "observed",
  "authority": "controlled-replay",
  "normalized_result": {
    "passed": true,
    "status": "pass",
    "claims": [
      {"claim_id": "percent_change.behaviour", "kind": "behavioral", "status": "pass", "evidence_ids": ["ev_e88b10dc..."]},
      {"claim_id": "grade.behaviour", "kind": "behavioral", "status": "pass", "evidence_ids": ["ev_67b6d3f9..."]},
      {"claim_id": "percent_change.bytes", "kind": "structural-function", "status": "pass", "evidence_ids": ["ev_e88b10dc..."]},
      {"claim_id": "grade.bytes", "kind": "structural-function", "status": "pass", "evidence_ids": ["ev_67b6d3f9..."]},
      {"claim_id": "grade.referenced-constants", "kind": "structural-function", "status": "pass", "evidence_ids": ["ev_67b6d3f9..."]}
    ]
  },
  "evidence_links": ["ev_e88b10dc...", "ev_67b6d3f9..."],
  "evidence_id": "ev_..."
}
```

El ID es`ev_`Además del SHA-256 del registro sin la identificación, sobre la clave ordenada JSON compacto. `aggregate()`se cierra por falla: `fail`gana, entonces `unknown`, y una lista de reclamaciones vacía es `unknown`En un host sin compilador C las dos afirmaciones de comportamiento son `unknown`, así que el registro dice`unknown`Incluso cuando ambas reivindicaciones de byte pasan. `authority`¿ Es verdad ?`controlled-replay`, porque la prueba proviene de nuestro propio arnés y no del archivo del objeto solo.

## Lo que dice la investigación

Liu, Raff y Micinski midieron ocho sistemas de descompilación en nueve configuraciones en 2026. Los candidatos que han superado cada prueba que incluye el índice de referencia todavía divergen en las entradas de descomposición. La tasa fue de 4,9 por ciento en general y hasta 13 por ciento para un sistema. En 300 funciones de biblioteca de GitHub y 287 funciones basadas en CVE, el modelo de refinamiento más fuerte elevó la tasa de construcción de Ghidra del 75 por ciento al 90 por ciento. En las mismas funciones la tasa de coincidencia de comportamiento cayó del 74 por ciento al 62 por ciento. Trazaron la causa a campos inventados, tipos, callees y guardias que reemplazaron a los desconocidos visibles que deja un descompilador tradicional.

Nuestro .`percent_change_wide.c`El pseudocodo ofreció una`long`La nueva construcción de los niveles de nivel es una guardia inventada. Ambos se contaría como reejecutable bajo la métrica LLM4Decompile, que recopila la salida y ejecuta las afirmaciones del benchmark.

DecompileBench midió 23.400 funciones de 130 programas reales. Encontró que los métodos LLM eran 52,2% más bajos en la corrección de funcionalidad que los descompiladores industriales, con puntuaciones de comprensibilidad más altas.

Las tres pruebas que el campo utiliza son las tres piernas que puedes construir. Phoenix y DREAM recompilan la salida y ejecutan las pruebas del programa original. McKeeman y Csmith ejecutan entradas generadas en contra del original con entradas mantenidas dentro del comportamiento definido. Ramos y Engler comprueban dos funciones de C con ejecución simbólica limitada, lo que demuestra la igualdad en todos los caminos explorados hasta un tamaño de entrada. Esta lección construye los dos primeros. El tercero es el laboratorio de práctica en el libro de pruebas de la lección 35 cuando una cuadrícula es demasiado pequeña para confiar.

## Construye el mismo

Ejecutar el arnés estándar de la biblioteca:

```bash
cd phases/13-tools-and-protocols/36-rebuild-and-prove-equal
python3 code/main.py
```

Con el clang en el camino, la demostración recopila ambos pares de candidatos y ejecuta 92 celdas para cada uno. Recopila ocho candidatos, repite la ejecución del agente y fabrica el registro de prueba. Sin clang lee los objetos comprometidos y informa las afirmaciones de comportamiento como `unknown`¿ Qué ?

Forma esperada en un arm64 Mac:

```json
{
  "host": "aarch64 Darwin",
  "originalKind": "fixture-source",
  "gridCells": 92,
  "plausible": {"naivePassed": true, "gridMismatches": 28, "oracleDisagreements": 0},
  "equal": {"naivePassed": true, "gridMismatches": 0, "oracleDisagreements": 0},
  "legTwo": {
    "percent_change_mul_first": {"status": "pass", "reasons": []},
    "percent_change_wide": {"status": "fail", "reasons": ["size 16 vs 26 bytes"]},
    "grade_case10_wrong": {"status": "fail", "reasons": ["referenced constants differ: [1, 2, 3, 4, 4] vs [1, 2, 3, 4, 3]"]}
  },
  "agentRun": {"mode": "replayed fixture", "server": {"name": "rea", "version": "6.5.0", "protocolVersion": "2025-11-25"}},
  "proof": {"status": "pass", "evidence_id": "ev_..."}
}
```

Leer .`code/main.py`en este orden:

1. `wrap32()`¿ Qué ?`c_div()`¿ Qué ?`idiv32()`, y los dos .`*_model()`Las funciones contienen la semántica de números enteros de la lista.
2. `input_grid()`y `classify()`construir y etiquetar las células 92.
3. `build_harness()`y `execute()`compilar un programa del original, los candidatos renombrados, y `driver.c`, luego ejecutar las células.
4. `prove_behaviour()`devuelve el informe de primera línea con discrepancias y desacuerdos de modelo.
5. `ElfObject`Leer encabezados, símbolos y `.rela`Registros.`mask_relocations()`y `read_table()`Preparar el comparativo.
6. `prove_bytes()`devuelve el informe de dos piernas con tamaños, rangos enmascarados, forma de reubicación y tablas.
7. `McpStdioClient`y `agent_loop()`conducir rea y grabar cada paso.
8. `build_proof()`y `aggregate()`el registro de fallas cerradas.

## Usalo

Para demostrar que una nueva función es igual, repita estos pasos:

1. Trae .`analyze_function`Para la función y grabar la identificación de la prueba.
2. Escriba el candidato de los anchos de operación de la lista.
3. Enumera cada multiplicador, suma, restar y dividir en la lista.
4. Añadir ambos lados de cada constante de comparación en la lista.`grade`Esos fueron 6 y 4 después de la restancia.
5. Añade todos los divisores que puedan atrapar al objetivo, y registra qué arquitectura ejecutó la cuadrícula.
6. Corre ambas piernas y lee las razones antes del estado.
7. Mantenga el registro de pruebas junto al candidato.`evidence_links`decir a un lector en que se basa el registro de la reclamación.

Una coincidencia de byte es una declaración sobre un compilador con un conjunto de banderas.

## Laboratorio interactivo

### Laboratorio A: el modelo y la ejecución

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/36-rebuild-and-prove-equal/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
percent_change_model(3, INT_MAX)
percent_change_model(-1, 536870911, "x86_64")
percent_change_model(-1, 536870911, "aarch64")
report = prove_behaviour(PLAUSIBLE)
report.naive_passed, len(report.mismatches), report.mismatches[0]
```

La primera llamada da -133, el producto envuelto dividido por 3. La segunda da `'trap'`El informe muestra que diez casos ingenuos pasaron y 28 celdas de red fallaron.`report.oracle_disagreements`con una lista vacía.

### Laboratorio B: leer el objeto

```python
obj = ElfObject(ORIGINAL_OBJECT)
grade = obj.function("grade")
grade.size, grade.data[0x26:0x2a].hex(), grade.relocations
read_table(obj, grade.relocations[0], 5, 4)
```

Los bytes de campo son `00000000`Los nombres de las reubicaciones`.rodata`con aditivo`0x10`, y la mesa dice`[1, 2, 3, 4, 4]`¿ Qué ?

### Laboratorio C: rompe una constante

```python
report = prove_bytes("grade", CANDIDATES / "grade_case10_wrong.o")
report.masked_equal, report.relocations_match, report.tables_equal, report.reasons()
```

Los bytes coinciden, los registros coinciden, y las tablas no. Ahora edita `grade_case10_wrong.c`¿ Qué ?`case 10`devuelve 4 de nuevo, volver a compilarlo con el comando de fijación, y ver el cambio de estado a `pass`¿ Qué ?

### Laboratorio D: rea de conducción

Con rea 6.5.0 y Ghidra 12.1.4 instaladas:

```bash
GHIDRA_INSTALL_DIR=/opt/homebrew/Cellar/ghidra/12.1.4/libexec \
JAVA_HOME=/opt/homebrew/opt/openjdk@21 \
python3 code/main.py --live
```

El primero .`analyze_function`Los datos de la prueba de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de

## Laboratorio de práctica

Extenda las dos piernas hasta`total`y `grade_label`de un mismo objeto.

Requisitos:

- Para`total`, la entrada es una matriz y un recuento. Construir células con recuento 0, un recuento negativo, un elemento, y sumas que se envuelven en 32 bits.
- Para`grade_label`, la función llama`grade`en línea y se lee `labels.rel`, una tabla de cinco offsets de cuerdas relativas con sus propias reubicaciones en `.rela.rodata`¿ Qué ?
- Añadir un`CLAIMS`Entrada para `grade_label`con dos relocalizaciones y una tabla de cinco entradas de 4 bytes, y comparar las cadenas a las que apuntan las entradas.
- Registre qué células el modelo y la carrera no están de acuerdo, si las hay, y explique cada una de ellas desde la lista.
- Añadir una reclamación por pie por función al registro de prueba y mantener el conjunto cerrado.

El éxito significa que el registro de pruebas enumera cuatro funciones, cada afirmación lleva un ID de evidencia, y `grade_label`se falla cuando una cadena de etiquetas difiere.

## Habilidad de salida

Esta lección produce`outputs/skill-rebuild-and-prove-equal.md`. Utilice para convertir la evidencia de la función y reconstruir un candidato en una prueba de dos patas y un registro de prueba. La habilidad requiere la listación antes del pseudocodo, una cuadrícula con cada clase de peligro, la máscaría de reubicación más una verificación constante y un agregado cerrado por falla.

## Verifique el hecho

Ejecutar la demostración y la suite de pruebas:

```bash
cd phases/13-tools-and-protocols/36-rebuild-and-prove-equal
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

Las 35 pruebas demuestran:

- `c_div()`truncates hacia cero y `wrap32()`sigue el complemento de dos
- El modelo da -133 para `(3, INT_MAX)`¿ Qué ?`trap`para la célula de trampa en x86-64 y -2147483648 en AArch64
- La red tiene 92 células y cada clase de peligro, y los casos ingenuos evitan cada clase dura
- El lector de ELF encuentra cuatro funciones en los desplazamientos y tamaños registrados
- `percent_change`es de 16 bytes sin reubicación y `grade`Tiene uno .`R_X86_64_PC32`En el`0x26`
- los bytes del campo de reubicación son cero y la mascarada no cambia nada en este objeto RELA
- los ocho candidatos comprometidos alcanzan el estatus esperado, con las razones registradas
- `grade_case10_wrong`pasa los bytes enmascarados y falla sólo la verificación constante
- Las carreras registradas de la pierna uno muestran diez pases ingenuos, 28 desajustes y cero desacuerdos de modelo
- con un compilador, ambos pares reconstruyen y re-ejecutan con el mismo resultado
- El agente ejecutado se dirige al objeto comprometido por SHA-256, en orden, con `ev_`Identificación en cada paso de la prueba
- el `read_bytes`registro es igual a la tabla que el lector ELF encuentra
- El cliente maneja el servidor falso y registra las mismas identidades de la evidencia que la ejecución real
- un resultado de error del servidor entra en el registro de ejecución
- `aggregate()`se cierra de forma automática y la prueba de identificación se recalcula a partir de su contenido

## Modos de falla de producción

| Failure | What the weak report says | What the harness must prove |
|---|---|---|
| Rebuild keeps the decompiler's `long` | "ten tests pass" | 26 overflow cells differ from the original |
| Thresholds replace a switch | "every grade band passes" | 110 and larger return 0 in the original |
| Divide before multiply | "formula is algebraically equal" | rounding cells differ, naive cases fail |
| Trap cell skipped | "no crashes" | x86-64 raises `#DE`, AArch64 returns `INT_MIN`, the report names the host |
| Byte compare without masking | "bytes equal" | relocation records name different offsets |
| Masking without reading constants | "bytes equal after masking" | `1,2,3,4,3` differs from `1,2,3,4,4` |
| Size taken as acceptance | "same length" | differing bytes inside the same length |
| Different compiler or flags | "bytes differ, rebuild wrong" | the same source gives different bytes, so the compare is void |
| Model treated as observation | "model agrees" | the run is observed, the model is derived, disagreements are listed |
| Claim without evidence id | "verified" | every claim links an `ev_` record of the function it covers |
| No compiler on the CI host | "passed" | behaviour claims are `unknown` and the aggregate is `unknown` |
| Pseudocode read as source | "decompiler says long" | the listing shows `IMUL EAX`, `CDQ`, `IDIV EDI` |

## Regla de funcionamiento

Escriba la reconstrucción de la lista y nombre el registro de donde proviene. Apliquela contra el original en una cuadrícula que envuelve cada intermedio y cruza cada constante de comparación. Recompilla con las banderas originales, enmascara los campos de reubicación, compara los registros y lea las constantes a las que apunta. Registra cada afirmación con su identificación de evidencia, y deja que una afirmación no verificada permanezca desconocida.

## Leer más

- [McKeeman, Differential Testing for Software, Digital Technical Journal 10(1), 1998](https://www.cs.tufts.edu/comp/150FP/archive/bill-mckeeman/DifferentailTesting.pdf)
- [Yang, Chen, Eide, Regehr, Finding and Understanding Bugs in C Compilers, PLDI 2011](https://doi.org/10.1145/1993498.1993532)
- [Ramos, Engler, Practical, Low-Effort Equivalence Verification of Real Code, CAV 2011](https://doi.org/10.1007/978-3-642-22110-1_55)
- [Schwartz, Lee, Woo, Brumley, Native x86 Decompilation Using Semantics-Preserving Structural Analysis, USENIX Security 2013](https://www.usenix.org/conference/usenixsecurity13/technical-sessions/presentation/schwartz)
- [Liu, Raff, Micinski, When LLM Decompilers Recompile More and Preserve Less, arXiv 2609.05370, 2026](https://arxiv.org/abs/2609.05370)
- [Gao et al., DecompileBench, Findings of ACL 2025](https://aclanthology.org/2025.findings-acl.1194/)
- [Tan et al., LLM4Decompile, arXiv 2403.05286, 2024](https://arxiv.org/abs/2403.05286)
- [System V gABI, Chapter 4, Relocation](https://www.sco.com/developers/gabi/latest/ch4.reloc.html)
- [x86-64 psABI repository, relocation types](https://gitlab.com/x86-psABIs/x86-64-ABI)
- [Intel 64 and IA-32 Architectures Software Developer's Manual, Volume 2, IDIV](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Arm Architecture Reference Manual for A-profile, DDI 0487, SDIV](https://developer.arm.com/documentation/ddi0487/latest)
- [rea 6.5.0 MCP contracts](https://github.com/morluto/rea/blob/rea-agents-6.5.0/docs/mcp-contracts.md)
- [rea 6.5.0 reconstruction obligation ledgers](https://github.com/morluto/rea/blob/rea-agents-6.5.0/docs/reconstruction-obligation-ledgers.md)
