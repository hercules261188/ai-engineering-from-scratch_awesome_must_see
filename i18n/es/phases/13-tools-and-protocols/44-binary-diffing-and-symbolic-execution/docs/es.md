# Differencia binaria y ejecución simbólica: función de coincidencia y solución de entradas

> Una coincidencia entre dos funciones es una afirmación hecha por una firma con una calidad, y una entrada resuelta es un modelo de una condición de camino; esta lección construye ambas y las verifica contra radiff2, Ghidra, angr y z3.

**Type:** Build
**Languages:** Python, C, x86-64 assembly, Java (one Ghidra script)
**Prerequisites:** Phase 13 · 33 (disassembly and control flow), 34 (lifting to pseudocode), 36 (rebuild and prove equal)
**Time:** ~150 minutes

## Objetivos de aprendizaje

- Construye cinco firmas de funciones de un objeto y su listaje: bytes exactos, bytes con campos de reubicación enmascarados, secuencia mnemónica, forma de flujo y nombre del símbolo.
- Ejecutar un juego codicioso en escena en el estilo BinDiff, donde una firma coincide solo cuando es única en ambos lados, y grabar el escenario y la calidad detrás de cada partido.
- Reproduce el caso en el que dos funciones idénticas de byte falten un correlador de byte en imágenes cargadas y lo pase después de la mascarada, con la aritmética de reubicación que explica los cuatro bytes cambiados.
- Reproduce las relaciones de similitud de radiff2 desde la distancia de Levenshtein y explique por qué una relación sólo aparece cuando la comparación utiliza los 82 bytes de radar2 alcanzados.
- Enumera las rutas de una función en un pequeño IR, lleva una condición por ruta y resuelve cada hoja para entradas con una búsqueda limitada.
- Medir la explosión de la trayectoria en un bucle con un límite simbólico y comparar los recuentos, clases y modelos con angr 10.0.3 y z3 5.1.0.

## El problema

Usted reconstruyó .`ledger.c`En el`-O2`y preguntó a un agente si algo cambió desde el`-O1`El agente hizo un difer y informó que`grade`El programa de programación de la red de datos de la red de datos de la red de datos de la red de red de datos de la red de red de datos de la red de red de red de datos de la red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red`grade`Las imágenes comparadas después de la reubicación son idénticas en los dos archivos objetos.`.rodata`se encuentra 0x70 bytes más adelante en el `-O2`programa, por lo que un desplazamiento en `grade`creció en 0x70.

La misma tarde que preguntaste qué puntaje gana`grade`La función devuelve 4 para cada puntaje de 90 a 109, y la pregunta no tenía límite, por lo que la respuesta del agente era una muestra donde necesitabas un conjunto.

Las dos fallas provienen de la misma brecha. Una coincidencia y una entrada resuelta son cada una una reclamación con un método detrás de ella. Esta lección construye los dos métodos en Python y ejecuta las herramientas de producción al lado de ellos. Cuando las herramientas y el código coinciden, tienes dos derivadas. Cuando no coinciden, tienes una dirección, un byte o un vínculo para mirar.

```figure
re-44-matches-and-paths
```

## Una firma de función es una reivindicación de calidad

Flake enmarcó la diferencia binaria en 2004 como la construcción de un isomorfismo entre los conjuntos de funciones de dos ejecutables similares. Dullien y Rolles lo extendieron en 2005 a bloques y instrucciones básicas. La página de conceptos de BinDiff describe la estrategia que salió de ese trabajo. Computa un atributo para cada función en ambos lados. Un valor que aparece exactamente una vez en cada lado es una coincidencia. Un valor que aparece más de una vez es ambigüo y se abre el siguiente atributo dentro de esa clase de equivalencia. Un valor sin contraparte se mantiene sin igual. Después de cada paso, los llamadores y llamados de nuevos partidos forman pequeños conjuntos donde es probable que el siguiente atributo sea único, y el proceso se repite con el siguiente atributo.

Cada algoritmo de coincidencia tiene una cualidad. La página clasifica la coincidencia hash de bytes de función bruta y la coincidencia hash de nombre como muy buena, el índice MD del gráfico de flujo y la coincidencia de firmas primas (cada mnemónica un pequeño primer, producto sobre la función) como buena, las referencias de cadena como medio y el recuento de bucles o secuencia de direcciones como pobre. La confianza de una coincidencia es la calidad promedio de los algoritmos que la producen, ponderada por un sigmoide. La similitud de la función es de aproximadamente 25% de los bordes de flujo-grafo coincidentes, 15% de los bloques básicos coincidentes, 10% de las instrucciones coincidentes y 50% de la diferencia en el índice MD del flujo-grafo, multiplicado por la confianza.

Esta lección mantiene cinco de esos atributos y la única regla que importa: único en ambos lados o no coincide. Se deja de lado el índice MD, porque las páginas de conceptos lo nombran sin una fórmula, y utiliza una secuencia de grados simples en su lugar. Las etiquetas de calidad permanecen, porque un coincidencia es tan buena como el hecho más débil que la produjo.

## Cinco firmas de un registro de funciones

El diferencial lee dos cosas por objeto. El ELF da bytes, tamaños de símbolos y `.rela.*`Los registros.`llvm-objdump -d`La lista da límites de instrucción y mnemónica, y la lección 33 demostró un decodificador igual a la lista en cada instrucción de estos objetos. `function_record()`verifica que los bytes de cada instrucción en la lista son iguales a los bytes del objeto en esa dirección y aumenta de otra manera.

¿ Qué ?`grade`En el`ledger-x86_64-O1.o`El símbolo dice que se compensa 0x10 y tamaño 46, por lo que el registro cubre 0x10 a 0x3e y excluye el de dos bytes `nop`que alinea la siguiente función.

La firma de bytes exactos es la SHA-256 de los 46 bytes. La firma de bytes enmascarados cero primero cada campo de reubicación. Hay un campo: el registro`R_X86_64_PC32 .rodata+0x10`en el desplazamiento 0x36, cuatro bytes de ancho, el desplazamiento de la `leaq (%rip), %rcx`En un objeto RELA esos cuatro bytes ya tienen cero, por lo que las dos firmas coinciden en cada función de cada objeto aquí.

La firma mnemónica hashes la secuencia de 14 mnemónicos:

```text
movslq imulq movq sarq shrq addl addl xorl cmpl ja movl leaq movl retq
```

La firma de forma del gráfico de flujo proviene de los bloques básicos y los bordes tipografados de la lección 33, restringidos a los bloques que la entrada puede alcanzar. `grade`En el`-O1`Tiene tres bloques y tres bordes.`ja 0x3d`Con un borde tomado y un borde de caída, el bloque 0x31 cae a través de 0x3d, y 0x3d devuelve. La secuencia de grados enumera (en, fuera) por bloque en orden orden ordenado, por lo que la firma se lee `3b 3e [0/2,1/1,2/0]`- Para .`grade`En el`-O0`El registro resuelve el salto indirecto a través de los cinco `.rela.rodata`Las entradas son exactamente como lo hizo la lección 33, que da ocho bloques y once bordes con cuatro `table`los bordes.

La firma del nombre es el nombre del símbolo, es el hecho más fuerte cuando existe y nada cuando se despoja la construcción.

Un campo más se corre para el caso de parche más tarde.`lea`traslado a la`.rodata`, un control de límites, y una carga indexada,`referenced_table()`lee la tabla de la manera que la lección 36 hizo: el `cmpl $0x4`antes de`ja`da cinco entradas y la carga `movl (%rcx,%rax,4), %eax`Da cuatro bytes cada uno.`grade`En el`-O1`la mesa en `.rodata+0x14`Leía`1, 2, 3, 4, 4`¿ Qué ?

## El codicioso empate

`match_functions()`El juego registra la etapa, la calidad de la etapa, cada otra firma que también coincide y una relación de similitud.

```python
for stage in order:
    for signature, candidates in left_groups.items():
        partners = right_groups.get(signature, [])
        if len(candidates) != 1 or len(partners) != 1:
            continue
        a, b = candidates[0], partners[0]
        agreeing = [s.id for s in STAGES if a.signatures()[s.id] == b.signatures()[s.id]]
        matches.append(Match(a, b, stage, agreeing, radare2_ratio(a.raw, b.raw), data_equal))
```

El orden de las lecciones es el primero de las cuatro etapas estructurales y los nombres son los últimos, por lo que las etapas estructurales se prueban y los nombres los clasifican después.`-O1`contra`-O2`¿Qué es esto ?

| Function | Stage | Agreeing signatures | Similarity |
|---|---|---|---|
| `percent_change` | exact bytes | all five | 1.000 |
| `grade` | exact bytes | all five | 1.000 |
| `total` | name | name only | 0.171 |
| `grade_label` | exact bytes | all five | 1.000 |

`total`El clang vectorizó el bucle con SSE2 y mantuvo un bucle escálar restante. Su forma pasó de cinco bloques y cinco bordes a diez bloques y trece bordes. Nada estructural sobrevive, y sólo el símbolo une los dos.

Para`-O0`contra`-O1`el mismo orden produce una coincidencia incorrecta:

| Function at -O0 | Matched at -O1 | Stage | Names agree |
|---|---|---|---|
| `percent_change` | `grade_label` | flow-graph shape | no |
| `grade` | `grade` | name | yes |
| `total` | `total` | name | yes |
| `grade_label` | unmatched | | |

`percent_change`En el`-O0`es un diamante de cuatro bloques: una entrada que se ramifica en `before == 0`, dos brazos, y una unión que regresa.`0/2, 1/1, 1/1, 2/0`- ¿ Qué ?`grade_label`En el`-O1`¿ Qué ?`grade`La forma es única en ambos lados y la regla lo acepta. La etiqueta de calidad media es la advertencia, y el nombre de etapa, ejecutado después, contradice.`grade_label`a la derecha, así que la verdadera `percent_change`y `grade_label`Los pares permanecen sin igual.

El orden BinDiff ejecuta la coincidencia de nombres como la segunda etapa en la que existen nombres reales, y `BINDIFF_ORDER`En el código hace lo mismo.`-O0`las funciones coinciden con sus `-O1`Las versiones por nombre, que es el resultado Ghidra informa a continuación. La lección mantiene ambos órdenes en la demostración para que el fracaso sea visible.`if`y `else`que se une antes del regreso.

## Bytes idénticos, operandos reubicados

Ghidra aplica relocaciones cuando importa un objeto. su correlador de byte compara imágenes cargadas y lección 36 campos de relocalización enmascarados por esta razón. `load_image()`Reproduce la importación.`sh_addralign`, y los establece desde 0x100000. Luego aplica cada registro:`S + A - P`escrito como un valor de 32 bits firmado para `R_X86_64_PC32`y `R_X86_64_PLT32`¿ Qué ?`S + A`como 64 bits para `R_X86_64_64`¿ Qué ?

El diseño está de acuerdo con Ghidra 12.1.4 en cada sección de inicio en ambos programas, que la suite de pruebas verifica con la salida de seguimiento de versión capturada:

| Section | `-O1` program | `-O2` program |
|---|---|---|
| `.text` | 0x100000, 159 bytes | 0x100000, 271 bytes |
| `.rodata` | 0x1000a0 | 0x100110 |
| `.rodata.str1.1` | 0x1000f0 | 0x100160 |

`.text`es 112 bytes más largo en `-O2`, el crecimiento de `total`, así que`.rodata`comienza 0x70 bytes más adelante. `grade`Se encuentra en 0x100010 en ambos programas.`leaq`En el desplazamiento 0x33 tiene su campo de desplazamiento en 0x36 y termina en 0x3a.`.rodata+0x14`¿ Qué ?

En el`-O1`imagen: `0x1000a0 + 0x14 - 0x10003a = 0x7a = 122`En el`-O2`imagen: `0x100110 + 0x14 - 0x10003a = 0xea = 234`Los bytes de campo son`7a 00 00 00`en una imagen y `ea 00 00 00`En el otro, un byte de 46 difiere, la relación de Levenshtein es de 0,978, y la etapa de bytes exactos falla.`grade_label`se desplaza por el mismo 0x70 que su tabla, por lo que sus desplazamientos se mantuvieron iguales y pasa bytes exactos en ambas vistas.

Ghidra informa de la misma división.`-O1`contra`-O2`, Función exacta Bytes Match encontrado `percent_change`y `grade_label`, Exact Function Instructions Match encontró esos dos más`grade`La correlación de instrucciones enmascara las operandas, que cubre el desplazamiento. La vista de imagen de la lección da coincidencias de bytes exactos para exactamente las dos funciones que el correlador de bytes de Ghidra encontró y el acuerdo de bytes enmascarados para exactamente las tres que su correlador de instrucciones encontró.

Las rea 6.5.0 `compare_functions`El texto del proveedor de line-diffs. En la dimensión de montaje se adjunta la limitación "normalización de reubicación no está disponible".

## ¿Qué medidas radiff2

Radar2 6.2.4 barcos `radiff2`- Con`-AC`Analiza los archivos y imprime una fila por función: el tamaño izquierdo y la dirección, MATCH o UNMATCH, una relación, y el tamaño correcto y la dirección.

| Pair | `percent_change` | `grade` | `total` | `grade_label` |
|---|---|---|---|---|
| `-O0` vs `-O1` | UNMATCH 0.130435 | UNMATCH 0.426829 | UNMATCH 0.106061 | UNMATCH 0.269841 |
| `-O1` vs `-O2` | MATCH 1.000000 | MATCH 1.000000 | UNMATCH 0.171429 | MATCH 1.000000 |
| `-O1` vs `-O1` | MATCH 1.000000 | MATCH 1.000000 | MATCH 1.000000 | MATCH 1.000000 |

La proporción en la fuente de radar2 es `1 - distance / max(len_a, len_b)`, donde la distancia es la distancia de Levenshtein entre las dos huellas dactilares de la función y una huella dactilar es la concatenation de los bloques básicos de la función en orden de direcciones.`radare2_ratio()`En el`code/main.py`Implementa la misma fórmula sobre los bytes de la función en bruto y reproduce 0.130435, 0.106061, 0.269841, y 0.171429 a seis decimales.

No se reproduce en 0.426829 para `grade`En el`-O0`. Sobre los 118 bytes de la función la relación es de 0,296610.`118`Sin reubicaciones aplicadas no puede resolver la tabla de salto, por lo que su análisis llega al bloque de entrada, el bloque que termina en `jmpq *%rax`, el caso predeterminado, y la devolución: 82 bytes, el mismo 82 que el descenso recurrente de la lección 33 reclamó.`-O1`es 47, y `1 - 47/82 = 0.426829`- Con`-e bin.relocs.apply=true`radar2 encuentra todos los 118 bytes y impresa 0,288136, lo que equivale a la relación de la lección sobre las imágenes relocadas, donde los cuatro bytes de desplazamiento difieren de los campos cero del objeto.

`-O`La información de la información de la base de datos de la base de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de cuyo`-O0`En comparación`-O1`Las relaciones aumentan a 0,239130, 0,512195, 0,303030 y 0,523810, y `total`En el`-O1`En comparación`-O2`Las distancias de archivo entero también existen: `-s`(Myers) informa de similitud 0.956 y distancia 162 entre los dos objetos,`-ss`(Levenshtein) 0,927 y 137. Estos números incluyen encabezados, tablas de símbolos y secciones de reubicación y no dicen nada sobre funciones.

## Correlatores de seguimiento de la versión de Ghidra

La herramienta de seguimiento de versiones de Ghidra combina un programa fuente con un programa de destino con correlatores. Los correladores de función son el Match de Bytes de Función Exactas, el Match de Instrucciones de Función Exactas (operandos enmascarados), el Match de Mnemónicas de Función Exactas y el Match de Instrucciones de Función Duplicadas (1:N y N:M). Los correladores de datos son Exact, Duplicado y Similar Data Match. Los correlatores de símbolos coinciden exactamente con los nombres, como duplicados o por similitud. Los correladores de referencia utilizan coincidencias aceptadas para encontrar más. Las funciones deben definirse en el programa de destino, y la longitud mínima de la función se establece por defecto en 10 bytes. Auto Version Tracking ejecuta el nombre exacto del símbolo, datos exactos, bytes de función exacta, instrucciones de función exacta, mnemónicas de función exacta, instrucciones de función duplicadas, luego un correlador de referencia, y crea coincidencias implícitas para las aceptadas. Se añadió un correlador BSim en el punto 11.2.

La lección capturó a los correladores sin cabeza.`capture_ghidra_vt.sh`importar los tres objetos en un proyecto y ejecutar `ListVersionTrackingMatches.java`, un guión corto que se extiende`AbstractGhidraVersionTrackingScript`, abre los programas de origen y destino por camino, ejecuta los seis correladores no referentes en el orden de seguimiento de la versión automática, e imprime una línea por bloque, función, correlador y coincidencia.

| Correlator | `-O0` to `-O1` | `-O1` to `-O2` |
|---|---|---|
| Exact Symbol Name Match | 4 functions | 4 functions, 5 data items (`labels.rel`, `.L.str.1` to `.L.str.4`) |
| Exact Data Match | 0 | 0 |
| Exact Function Bytes Match | 0 | 2 (`percent_change`, `grade_label`) |
| Exact Function Instructions Match | 0 | 3 (adds `grade`) |
| Exact Function Mnemonics Match | 0 | 3 |
| Duplicate Function Instructions Match | 0 | 0 |

Cada función cambió entre `-O0`y `-O1`En la página de ayuda se describen los correlatores exactos como similitud 1.0 y confianza 1.0, por lo que la escala de la confianza impresa es una pregunta abierta que esta lección registra y no resuelve.

Los programas de diferencias, en las herramientas, es algo diferente. Compara dos programas con el mismo espacio de direcciones en bytes, etiquetas, unidades de código, referencias, comentarios y funciones en direcciones iguales. Es un diferente alineado con direcciones y no coincide con nada que se movió.

## Diafora y Ninja binario, de sus documentaciones

Diaphora es un plugin IDA Pro, lanzado por primera vez en 2015 y en la versión 3.4.2 a partir de 2026-08-18. Es AGPL-3.0 desde 2.0 y necesita IDAPython, por lo que se ejecuta en IDA Home o IDA Pro y no en IDA Free. Exporta cada base de datos a SQLite y ejecuta sus heurísticas como consultas SQL. Cada heurística tiene un nombre, una categoría de Mejor, Partial o No confiable, un tipo de relación y banderas opcionales como SAME_CPU o SLOW. Los nombres en la fuente actual se leen como las etapas de esta lección con más de ellos: hash de bytes, la misma dirección y mnemónica, la misma ensamblaje limpio, Mnemonics pequeño-primes-producto, los mismos nodos, bordes, bucles y componentes fuertemente conectados, topológico ordenar hash, y pseudo-código fuzzy hashes que necesitan el Hex-Rays descompilador. Sus notas de lanzamiento muestran el historial de sintonización: las heurísticas lentas solo funcionan por debajo de unas 2.000 funciones, y "El mismo gráfico" se trasladó a la categoría poco confiable. Ese movimiento es el diamante de cuatro bloques de esta lección, visto en escala.

Binary Ninja 6.0, lanzado 2026-09-03, añadió Binary Similarity como un marco con proveedores y resolvadores. Envía a BinDiff como proveedor estructural y WARP para coincidencias exactas de funciones que toleran operands relocables, y su guía dice que se utilice WARP cuando las funciones coinciden aparte de las instrucciones relocables y BinDiff cuando permanecen estructuralmente relacionadas después de la recompilación. La función está en la edición final sólo. Binary Ninja no tiene un gestor de paquetes instalado aquí y no se ejecutó, así que este párrafo proviene de su documentación y nada más.

BinDiff en sí mismo ha sido de código abierto desde 2023-09-25 bajo Apache-2.0, con la última versión v8. Necesita un desmonteador y el plugin BinExport, que admite IDA Pro 9.4 o posterior, Binary Ninja y Ghidra. El camino libre para un aprendiz es Ghidra más BinExport más la línea de comandos BinDiff. La lección no lo ejecutó, y su Practice Lab le pide que lo haga.

## El mismo código, diferente constante

La lección 36 mantuvo un candidato a reconstruir el`grade`cuya tabla tiene 3 como última entrada, donde el original tiene 4. `grade_case10_wrong.o`tiene los mismos 46 bytes de código que `grade`En el`-O1`El parador los empareja en bytes exactos con cada firma de acuerdo.`1, 2, 3, 4, 4`a través de una reubicación y `1, 2, 3, 4, 3`a través de los demás y los informes `dataEqual: false`Los otros tres .`-O1`las funciones son incomparables a la izquierda, porque el objeto candidato tiene una función.

Esta es la forma de parche-análisis. Una función diferente que se detiene en bytes dice que nada cambió. Ghidra's Exact Data Match y BinDiff's correlación de referencia de cadena existen por la misma razón. La versión de la lección es pequeña: para cada par coincidente, comparar los bytes de cada función relocaciones punto en, con el recuento de los límites de control y el ancho de la carga.

## Ejecución simbólica: caminos, condiciones, soluciones

King describió la ejecución simbólica en 1976. Las entradas se convierten en símbolos. Cada declaración actualiza una tienda simbólica. Cada rama agrega una condición al camino, y un solvente decide si el camino es factible y produce una entrada concreta que lo sigue. Una función con ramas se convierte en un árbol de caminos, y una hoja es un retorno con una condición unida.

Baldoni, Coppa, D'Elia, Demetrescu y Finocchi estudiaron el campo en 2018. Su sección 5 llama la explosión de ruta como el costo central: un ejecutor puede forjarse en cada rama, por lo que el conteo de estado puede crecer exponencialmente en el número de ramas, y un bucle cuyo límite depende de una entrada simbólica da un conjunto ilimitado de caminos. Las mitigations que enumeran son poda de caminos irrealizables resolviendo en cada rama, resúmenes de funciones y bucles, estado de fusión, y sub-aproximación. La ejecución concólica ejecuta el programa de forma concreta y simbólica a la vez, registra las condiciones a lo largo de la ejecución concreta, niega una y pide al solvente la próxima entrada. DART (2005), CUTE (2005), y SAGE (2008) son los sistemas nombrados.

KLEE (Cadar, Dunbar, Engler, OSDI 2008) es una máquina virtual simbólica sobre el bitcode LLVM, por lo que su entrada es `clang -emit-llvm`En los 89 programas de GNU coreutils alcanzó una cobertura de línea del 84.5% frente al 67.7% para las pruebas de los desarrolladores y encontró 56 errores graves. KLEE 3.2 fue lanzado 2025-12-23 bajo la licencia NCSA, con LLVM 16 recomendado.

Angr trabaja en binarios. `angr.Project`tiene un cargador CLE que mapea el objeto y aplica reubicaciones, un elevador que convierte un bloque en VEX IR (`block.vex`) con código de tubería como alternativa, y una fábrica para los estados.`SimState`contiene registros, memoria y un solvente.`SimulationManager`escalones de estados nombrados `active`¿ Qué ?`deadended`¿ Qué ?`found`¿ Qué ?`avoided`, y `errored`, y `explore(find=..., avoid=...)`En cada paso los motores se ejecutan en orden: fallo, syscall, gancho, unicornio para datos concretos, y VEX como la caída. claripy es el solver frente sobre Z3, y en angr 10.0 es una implementación vendida de Rust importada como `from angr import claripy`. angr 10.0.3 fue subido el 2026-10-10, es BSD-2-Clause, y necesita Python 3.12 o posterior. Z3 5.1.0, lanzado 2026-08-16 bajo MIT, es el solver detrás de angr, Triton y backend opcional de KLEE.

## Un pequeño ejecutor sobre la lección 34 IR

El ejecutor en `code/main.py`utiliza una pequeña IR con las formas de la declaración de la lección 34: `Assign`¿ Qué ?`Branch`¿ Qué ?`Jump`¿ Qué ?`Return`, más el `Switch`El terminador que el laboratorio de práctica de la lección 34 pidió.`Const`¿ Qué ?`Var`¿ Qué ?`Bin`con un ancho, `Cast`¿ Qué ?`Load`de una mesa de hormigón con nombre, y `Not`. Las cuatro funciones de fijación se levantan a mano en el nivel que la lección 34 alcanza después de la simplificación, donde la `0x66666667`multiplicarse se ha convertido en `score / 10`, y cada bloque IR enumera las direcciones de instrucciones de donde proviene. `check_ir_against_listing()`afirma que las direcciones cubren la lista de la función exactamente una vez.

`grade`En el`-O0`en ese IR:

```text
0x30: local_8 = score; local_10 = score / 10 - 6; if (local_10 >u 4) goto 0x9a else 0x62
0x62: switch local_10 -> 0x91, 0x88, 0x7f, 0x76, 0x76
0x76: local_4 = 4; goto 0xa1        0x7f: local_4 = 3; goto 0xa1
0x88: local_4 = 2; goto 0xa1        0x91: local_4 = 1; goto 0xa1
0x9a: local_4 = 0                    0xa1: return local_4
```

Los objetivos del cambio provienen de `jump_table_targets()`, la misma aritmética de reubicación que la lección 33, así que el IR lee la fijación.

`execute()`Un estado tiene el bloque actual, el rastro de bloques, la lista de condiciones y un entorno que mapea cada variable a una expresión sobre las entradas. `Assign`El medio ambiente se sustituye por la fuente y almacena el resultado. `Branch`Se puede encontrar una parte que el solvente no puede satisfacer, que es la poda ansiosa que describe la encuesta. `Switch`Los grupos de entradas de la tabla por objetivo y bifurcaciones una vez por objetivo distinto con la disyunción de las igualdades de entrada, que es la forma en que Angr trata un objetivo de salto simbólico también. `Return`Una pista más larga que el límite de paso marca el camino truncado y lo detiene.

Sigue el camino hacia `return 4`La entrada asigna `local_10 = score / 10 - 6`- El estado de la rama`local_10 >u 4`es falso en este camino, por lo que la lista de condiciones se mantiene `!(score / 10 - 6 >u 4)`. El interruptor entradas 3 y 4 ambos puntos en 0x76, por lo que el tenedor a 0x76 se agrega `score / 10 - 6 == 3 || score / 10 - 6 == 4`. Bloqueo 0x76 asigna `local_4 = 4`, salta a 0xa1, y el reemplazo de retorno a `4`La hoja es el par de esa condición y ese valor.

El solucionador es`BoundedSearch`. Tiene un dominio finito por entrada, enumera el producto de los dominios y evalúa cada restricción con semántica C: 32 bits de envoltura, división truncada, comparaciones firmadas y sin firmas, y cambios aritméticos o lógicos. Una división por cero o un índice de tabla fuera de la tabla hace que esa asignación no sea la restricción.`score`en -200 a 1200 los cinco caminos de `grade`El resultado de la evaluación de cada hoja para el conjunto de entradas es:

| Leaf | Returns | Condition | Scores in the bound |
|---|---|---|---|
| 0x76 | 4 | `!(q >u 4) && (q == 3 \|\| q == 4)` | 90 to 109, 20 inputs |
| 0x7f | 3 | `!(q >u 4) && q == 2` | 80 to 89, 10 inputs |
| 0x88 | 2 | `!(q >u 4) && q == 1` | 70 to 79, 10 inputs |
| 0x91 | 1 | `!(q >u 4) && q == 0` | 60 to 69, 10 inputs |
| 0x9a | 0 | `q >u 4` | 1,351 inputs, the rest of the bound |

`q`significa`score / 10 - 6`Las mismas clases salen de`grade`En el`-O1`, donde el IR tiene dos caminos, el camino de la tabla regresando `lookup_14[q]`y el camino predeterminado devuelve 0, y la pregunta se convierte en una restricción en el valor devuelto, porque no hay un bloque de caso para alcanzar.`lookup_14`se lee desde el objeto `.rodata`en 0x14.

El límite es el límite del método. Una búsqueda limitada se completa dentro de su dominio y silenciosa fuera de él. Dentro de -200 a 1200 ningún puntaje en 60 a 109 devuelve 0 y ningún puntaje de 110 o más devuelve un valor no cero. z3 prueba ambos para todos los 2 ^ 32 puntajes, y la siguiente sección muestra donde los dos métodos devuelven diferentes testigos.

## Se midió la explosión del camino

`total`En el`-O1`es el ciclo de la lección 33: una prueba en `count`, un cuerpo de bucle que agrega `values[rdx]`y incrementos `rdx`, y un `jne`De vuelta al cuerpo mientras`rdx != rcx`En el IR`rcx`¿ Es verdad ?`zext64(count)`y `rdx`comienza en 0. Cada paso a través de la parte posterior de la bifurcación en `rdx != count`con`rdx`una constante, así que el camino para `count = c`dirige el cuerpo `c`Tiempo y salida.`count`limitado a 0 a 2, el ejecutor encuentra 3 caminos. Con 0 a 4 encuentra 5, y con 0 a 8 encuentra 9. Cada camino devuelve la suma triangular de su conteo: (0, 0), (1, 1), (2, 3), hasta (8, 36). angr informa estados 3, 5 y 9 muertos y los mismos nueve pares de conteo y suma en los tres niveles de optimización, incluyendo `-O2`, cuyo bucle vectorial necesita un recuento de al menos 8 y por lo tanto nunca cambia el recuento de trayectoria en este rango.

Sin ningún límite el ejecutor nunca termina, porque cada cuenta de iteraciones es un camino más factible.`count`El informe da esos tres números. El costo de solución también crece: 33 evaluaciones en la línea 2, 115 en 4, 531 en 8, y 6.900 para la carrera limitada.

## Angr y z3 en las mismas preguntas

`capture_angr.py`CLE cargó cada objeto relocable con el backend ELF a 0x400000, aplicó 13 relocalizaciones para`-O0`y 8 para los demás, y resuelto los cuatro símbolos.`grade`con un argumento simbólico de 64 bits, `run()`terminó con 5 estados en desfallecimiento en`-O0`y 2 en `-O1`y `-O2`, en 0,475, 0,408, y 0,318 segundos.`eax == v`En el caso de los Estados miembros, el valor de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los valores de los cuyo de los cuyo de los valores de los cuyo de los valores de los cuyo de la cuenta de los valores de los valores de los cuyo de los cuyo de la cuenta de los cuyo de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de la cuenta de`v`En todos los niveles, los controles cruzados fueron completados sin ningún resultado.

El formulario de alcance de un bloque utilizó CFGFast en `grade`En el`-O0`, que enumera las ocho bloques de la lección 33 encontrada, entonces `explore(find=0x400076)`En el bloque que almacena 4. Encontró un estado con tres todavía activos en 0,47 segundos, con un mínimo de 90 y un máximo de 109 en el argumento y muestras 91, 93, 94, 97 y 101.

Para`percent_change`, dos estados sin salida con las restricciones `before[31:0] != 0`y `before[31:0] == 0`. Pidiendo el estado no cero para`eax == 50`¿ Qué ?`before = 21055744, after = 332315210`En el`-O0`¿ Qué ?`40566804, 834296877`En el`-O1`, y `-24484867, 993934694`En el`-O2`La búsqueda limitada en -16 a 16 respuestas .`before = -10, after = -15`. z3, hizo la misma pregunta sobre la lista de la lección 34, respuestas `before = -25092599, after = 305712160`. La suite de pruebas evalúa los cinco pares con el evaluador IR y cada uno da 50. Un modelo prueba que existe una entrada.

`capture_z3.py`Modelo de la`-O1`lista de `grade`en 32 bits bit vectores con el multiplicador, los dos turnos, y la tabla, y hace cuatro preguntas más.`switch`Para algunas puntuaciones de 32 bits: no se ha establecido en 0,068 segundos, por lo que la función compilada es igual a la fuente en todas las entradas de 2^32. La lista difiere de la lección 36 de la puesta a punto de la lección 36: se ha establecido en la puntuación 706346536, donde `score / 10`Es 70634653 y el interruptor cae al estado predeterminado mientras los umbrales regresan 4. La misma diferencia restringida a 0 a 109: no satisfecha, por lo que la cuadrícula de la lección 36 no pudo verlo.`percent_change`¿ Qué ?`before == 3`con el resultado -133 se sitúa con `after = 1073741823`, la célula de desbordamiento de la lección 36, y la búsqueda limitada encuentra un segundo testigo dentro de su dominio,`after = -1`, donde `(-1 - 3) * 100 / 3`Truncates a -133 sin sobrecarga en absoluto.

## Construye el mismo

Ejecutar la demostración:

```bash
cd phases/13-tools-and-protocols/44-binary-diffing-and-symbolic-execution
python3 code/main.py
```

La salida es un informe JSON. `diffs`contiene los archivos de objetos y las imágenes cargadas para `-O0`contra`-O1`y `-O1`contra`-O2`, más el `-O0`pareja en el orden BinDiff. `toolComparison`pone la etapa de la lección junto al veredicto y la relación radiff2 y los correlatores de Ghidra para cada función. `patch`¿ Es el `grade_case10_wrong.o`¿Qué es esto?`radare2Ratios`reproducir los números radiff2, `gradeRelocationCase`muestra el diseño y los dos desplazamientos, y `symbolic`tiene los caminos, clases y recuentos de explosiones.

Leer .`code/main.py`en este orden:

1. `ElfObject`Leer secciones, símbolos y `.rela.*`Registros.`parse_listing()`Leemos el `llvm-objdump`¿Qué es eso?
2. `jump_table_targets()`y `build_cfg()`dar bloques accesibles y bordes tipografados. `FunctionRecord.signatures()`construye las cinco firmas.
3. `load_image()`establece las secciones asignadas y se aplica `S + A - P`y `S + A`¿ Qué ?
4. `match_functions()`es el juego en escena. `levenshtein()`y `radare2_ratio()`dar la similitud.
5. `parse_radiff2()`y `parse_ghidra_vt()`leer las salidas de la herramienta capturadas para la comparación.
6. `evaluate()`¿ Qué ?`substitute()`, y `BoundedSearch`son la semántica de las expresiones y el solvente.
7. `execute()`Enumera los caminos. `solve_returns()`solicita a cada hoja un conjunto de entradas.
8. `grade_ir_o0()`¿ Qué ?`grade_ir_o1()`¿ Qué ?`percent_change_ir_o1()`, y `total_ir_o1()`son los levantadores de manos con sus direcciones de instrucción.

Hay tres órdenes más.`python3 code/main.py diff A.o B.o`Diferencia dos objetos cuyas listas se encuentran junto a ellos, y `--image`Difere sus imágenes cargadas. `python3 code/main.py paths grade O0`Imprime las rutas y clases de una función, `paths total 8`el bucle con un límite. `python3 code/main.py figure-data`Imprime los datos que la figura interactiva incorpora.

## Usalo

Utilice el difer y el ejecutor en este orden en dos construcciones de su propio programa.

1. Captura una lista para cada objeto y deja `function_record()`Verifique con los bytes, arregle la lista antes de cualquier otra cosa.
2. Difórmese primero los archivos objetos. Lea la etapa de cada coincidencia y trate las coincidencias de forma del gráfico de flujo como sugerencias hasta que una segunda firma esté de acuerdo.
3. Diferencia las imágenes cargadas. Una función que baja de bytes exactos a bytes enmascarados se mueve en relación con sus datos, y la aritmética de desplazamiento nombra la sección que se movió.
4. Para cada coincidencia, compara las tablas referenciadas. El mismo código con constantes diferentes es una función modificada.
5. Cuando la construcción tiene símbolos, ejecuta el orden BinDiff y usa el resultado estructural sólo para clasificar las etapas.
6. Cuando una herramienta informa una relación, averiguar qué bytes se metieron en ella antes de compararla con la suya.
7. Para la función cuyas entradas desea, escriba la IR con una dirección por instrucción, ejecuta `execute()`con un límite en cada contador de bucle simbólico, y resolver cada hoja.
8. ejecuta la misma consulta a través de angr o z3 cuando estén disponibles y compara los límites de la clase.

## Laboratorio interactivo

### Laboratorio A: cinco firmas a mano

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/44-binary-diffing-and-symbolic-execution/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
grade = {r.name: r for r in fixture_records("ledger-x86_64-O1.o")}["grade"]
grade.raw.hex(" ")
grade.masked_bytes() == grade.raw
grade.mnemonics
grade.cfg_shape()
grade.signatures()
```

Escriba los cuatro bytes en el desplazamiento 0x26 de la función antes de imprimir `grade.raw`Entonces sigue las mismas líneas .`grade`de la`ledger-x86_64-O0.o`y contar los `table`los bordes en `grade.edges`¿ Qué ?

### Laboratorio B: ver la reubicación rompe el hash de byte

```python
case = grade_relocation_case()
case["layout"]
case["displacements"], case["rodataShift"]
case["objectBytesEqual"], case["imageBytesEqual"], case["maskedImageBytesEqual"]
```

Computación`0x100110 + 0x14 - 0x10003a`en papel antes de leer.`displacements`Entonces cambia la base en`load_image()`a 0x400000 y confirmar que los desplazamientos no cambian, porque ambos operandos se mueven juntos.

### Laboratorio C: la coincidencia falsa y el orden que la evita

```python
report = diff_fixtures("ledger-x86_64-O1.o", "ledger-x86_64-O2.o")
[(m.left.name, m.right.name, m.stage) for m in report.matches]
report = diff_fixtures("ledger-x86_64-O0.o", "ledger-x86_64-O1.o")
[(m.left.name, m.right.name, m.stage, m.names_agree) for m in report.matches]
report = diff_fixtures("ledger-x86_64-O0.o", "ledger-x86_64-O1.o", stages=BINDIFF_ORDER)
[(m.left.name, m.right.name, m.stage) for m in report.matches]
```

Impresión`records(...)["percent_change"].cfg_shape()`por`-O0`y `grade_label`por`-O1`y leer las dos secuencias de grados. Luego escriba una sexta etapa que hashes la mnemónica ordenada, así que el orden ya no importa, y ver si se separa.

### Laboratorio D: caminos y límites

```python
report = grade_paths("O0")
[(p["trace"], p["condition"], p["returns"]) for p in report["paths"]]
report["returns"]["4"]
total_paths(2), total_paths(4), total_paths(8)
total_paths(64, step_limit=24)["truncated"]
```

Cambiar`SCORE_DOMAIN`¿ Qué ?`range(0, 110)`y volver a correr.`grade_paths("O0")`. La clase para 0 se reduce a 60 entradas y los dos controles cruzados todavía se mantienen.`range(90, 100)`Y leer qué hojas no tienen modelo en absoluto.

## Laboratorio de práctica

Ejecutar BinDiff en los fijos y compararlo con los tres resultados que ya tienes.

Requisitos:

- Instale la extensión BinExport en Ghidra 12.1.4 y exporta los tres programas de libro mayor. Graba la etiqueta de lanzamiento de BinExport y la versión de Ghidra.
- Instale la línea de comando BinDiff 8 y produzca la`-O0`En comparación`-O1`y `-O1`En comparación`-O2`resultados con la opción de similitud única.
- Captura la similitud y la confianza por función en `code/fixtures/`con los comandos exactos en `build_fixtures.sh`¿ Qué ?
- Extenderse`tool_comparison()`con un `bindiff`columna y añadir una prueba que compara el conjunto de BinDiff coincidiendo en `-O1`En comparación`-O2`con el conjunto de bytes enmascarados de la lección.
- Escriba un párrafo que nombre el algoritmo BinDiff utilizado para `grade`y si comparó imágenes cargadas.

El éxito significa que la demostración imprime cuatro columnas de herramientas por función y la nueva prueba establece una igualdad o una diferencia explicada.

## Lección Artefacto

Esta lección incluye `outputs/skill-binary-diffing-and-symbolic-execution.md`. Utilice para convertir dos builds y una función en una diferencia en etapas con una calidad declarada por partido, una explicación de cada función sin igual o cambiada, y un informe de camino con un conjunto de entradas resueltas por hoja y el límite que la produjo.

## Verifique el hecho

Ejecutar la demostración y la suite:

```bash
cd phases/13-tools-and-protocols/44-binary-diffing-and-symbolic-execution
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

La verificación deberá demostrar:

- Las tablas de símbolos dan las cuatro funciones con tamaños 46, 118, 66, 39 en `-O0`, 16, 46, 31, 63 en `-O1`, y 16, 46, 140, 63 en `-O2`
- el `.rela.text`registros de los `-O1`objeto se describe como `llvm-objdump -r`las imprime
- cada instrucción en la lista está en su extensión de símbolo y sus bytes igual al objeto, y un byte cambiado aumenta
- `total`En el`-O1`tiene cinco bloques, cinco bordes y un bucle propio, y su cadena de forma es determinista
- `grade`En el`-O0`Recupera cinco objetivos de la mesa de salto de `.rela.rodata`y tiene ocho bloques y once bordes
- sin la mesa,`grade`En el`-O0`alcanza 82 bytes en cuatro bloques
- `percent_change`En el`-O0`y `grade_label`En el`-O1`tienen la misma forma de cuatro bloques
- hashes exactos y enmascarados coinciden en cada función de un objeto relocable
- `-O1`y `-O2`compartir bytes para tres funciones, `-O0`y `-O1`no comparten
- el cuadro de referencia de `grade`Leía`1, 2, 3, 4, 4`y las lecturas del candidato.`1, 2, 3, 4, 3`
- los lugares del cargador `.text`¿ Qué ?`.rodata`, y `.rodata.str1.1`donde Ghidra los colocó en ambos programas
- `R_X86_64_PC32`se escribe como `S + A - P`y da desplazamiento 122, y `R_X86_64_64`las entradas en `.data`recibir direcciones de cadena
- `grade`los bytes coinciden en los objetos y difieren en las imágenes por un cambio de 0x70, y el enmascaramiento restaura la coincidencia
- `-O1`En comparación`-O2`coincide con tres funciones en bytes exactos y `total`por nombre; la vista de imagen coincide `grade`por bytes enmascarados con similitud 1 - 1/46
- el orden de las lecciones produce una coincidencia falsa de forma del gráfico de flujo en `-O0`En comparación`-O1`, y el orden BinDiff coincide con los cuatro por nombre
- el candidato de parche coincide en bytes y no en los datos referenciados
- La relación Levenshtein reproduce 0.130435, 0.106061, 0.269841, 0.171429, 0.426829 sobre 82 bytes, y las relaciones de bytes reubicados imprimen radiff2 con reubicaciones aplicadas
- el conjunto MATCH de radiff2 es igual al paso de bytes exactos, y los correladores de byte e instrucción de Ghidra son iguales a los conjuntos exactos y enmascarados de la vista de imagen
- Ghidra sólo encontró nombres en `-O0`En comparación`-O1`, con similitud 1.0 y confianza 10.0 en cada partido
- El IR levantado a mano cubre cada instrucción enumerada una vez, y el evaluador sigue la semántica C
- `grade`Tiene cinco caminos en `-O0`con las condiciones de cambio y dos en `-O1`con las mismas clases
- las clases de límites 60 a 69, 70 a 79, 80 a 89, 90 a 109 de angr igual en los tres niveles y z3, con 5 y 2 estados de límite y el rango de alcance de un bloque de 90 a 109
- los dos controles cruzados no tienen modelo en el límite, en angr, o en z3, y z3 demuestra que la lista es igual al interruptor de origen
- cada uno`percent_change`modelo de la búsqueda limitada, angr, y z3 evalúa al valor solicitado
- `total`da 3, 5, 9 caminos con las sumas triangulares, igual a angr, y el paso de la tapa devuelve 22 y truncadas 2
- Los datos de la figura contienen las dos vistas, los números de reubicación y cinco hojas con sus conjuntos de entradas, y el módulo de sitio incorpora los mismos datos

## Modos de falla de producción

| Failure | What the weak tool reports | What the analysis must prove |
|---|---|---|
| Byte hash on a loaded image | "grade modified" between two builds with identical code | The field offset, the two displacements, and the section shift; the masked hash matches |
| Masked hash on a relocatable object | "masking made no difference" | The fields were already zero; the relocation records and the referenced constants carry the change |
| Flow-graph shape alone | A confident pair of unrelated functions | A second signature agrees, or the names contradict and the match is dropped |
| Names before structure on a stripped build | Nothing matches | The structural stages with their qualities, and the unmatched list |
| A ratio compared across tools | "radiff2 says 0.43 and the lesson says 0.30" | Which bytes each side fingerprinted; 82 bytes without relocations, 118 with |
| Same code, different constant | "exact match" | The table read through the relocation and the bounds check |
| One model reported as the input | "the score is 90" | The class 90 to 109 with its count, and the bound it was found in |
| Bounded search reported as unsat | "no input gives -133" | Unsat inside the bound only; z3 finds 1073741823 outside it |
| Symbolic loop without a bound | A timeout | The bound or the cap, the paths returned, truncated, and never started |
| Closed tool described as run | Numbers with no capture | A capture file with the command and version, or a sentence saying documentation only |

## Regla de funcionamiento

Compare en una firma a la vez, acepta sólo valores únicos en ambos lados, y mantenga el escenario y su calidad con el juego. Diferencia los objetos y las imágenes cargadas y explica cada función que cambia de escenario entre los dos. Enumera los caminos con un solvente en cada rama, enlace cada bucle simbólico, y informe un conjunto de entrada con su límite. Un modelo único sin límite es una muestra.

## Leer más

- [BinDiff concepts: matching strategy, algorithms, and quality ratings](https://github.com/google/bindiff/blob/main/docs/concepts.md)
- [BinDiff repository and the v8 open-source release](https://github.com/google/bindiff/releases/tag/v8)
- [BinExport README: IDA Pro, Binary Ninja, and Ghidra exporters](https://github.com/google/binexport)
- [Flake, "Structural Comparison of Executable Objects" (DIMVA 2004) and Dullien, Rolles, "Graph-based comparison of Executable Objects" (SSTIC 2005), shipped in the BinDiff repository](https://github.com/google/bindiff/tree/main/docs/papers)
- [Diaphora README and heuristics source](https://github.com/joxeankoret/diaphora)
- [Ghidra Version Tracking correlators help page](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/VersionTracking/src/main/help/help/topics/VersionTrackingPlugin/VT_Correlators.html)
- [Ghidra Auto Version Tracking help page](https://www.ghidradocs.com/11.0_PUBLIC/help/VersionTracking/help/topics/VersionTrackingPlugin/VT_AutoVT.html)
- [Ghidra 12.1.4 release](https://github.com/NationalSecurityAgency/ghidra/releases)
- [Binary Ninja 6.0 "Krypton" release notes](https://binary.ninja/2026/09/03/binary-ninja-6.0-krypton.html)y [Binary Similarity user guide](https://docs.binary.ninja/guide/similarity.html)
- [radare2 book: radiff2](https://book.rada.re/tools/radiff2/intro.html)y [radare2 6.2.4 source](https://github.com/radareorg/radare2/releases/tag/6.2.4)
- [King, "Symbolic execution and program testing" (CACM 1976)](https://dl.acm.org/doi/10.1145/360248.360252)
- [Baldoni et al., "A Survey of Symbolic Execution Techniques" (ACM Computing Surveys 2018)](https://dl.acm.org/doi/10.1145/3182657)
- [Cadar, Sen, "Symbolic execution for software testing: three decades later" (CACM 2013)](https://dl.acm.org/doi/10.1145/2408776.2408795)
- [Cadar, Dunbar, Engler, "KLEE: Unassisted and Automatic Generation of High-Coverage Tests for Complex Systems Programs" (OSDI 2008)](https://www.usenix.org/legacy/events/osdi08/tech/full_papers/cadar/cadar.pdf)y [KLEE 3.2 release](https://github.com/klee/klee/releases/tag/v3.2)
- [Shoshitaishvili et al., "SoK: (State of) The Art of War: Offensive Techniques in Binary Analysis" (IEEE S&P 2016)](https://oaklandsok.github.io/papers/shoshitaishvili2016.pdf)
- [angr documentation: loading, simulation managers, and the simulation pipeline](https://docs.angr.io/en/latest/core-concepts/toplevel.html)y [angr 10.0.3 on PyPI](https://pypi.org/project/angr/)
- [Z3 5.1.0 release](https://github.com/Z3Prover/z3/releases/tag/z3-5.1.0)
- [Triton dynamic binary analysis library](https://github.com/JonathanSalwan/Triton)
- [System V Application Binary Interface, AMD64 Architecture Processor Supplement, Table 4.9: Relocation Types](https://gitlab.com/x86-psABIs/x86-64-ABI)
