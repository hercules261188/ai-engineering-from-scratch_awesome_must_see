# Más allá del código nativo: Python, JVM, .NET y Bytecode de WebAssembly

> Un archivo de código de byte es un programa más las notas del compilador al respecto, y esta lección construye los lectores que muestran qué notas cada recipiente guarda.

**Type:** Build
**Languages:** Python, Java, C#, WebAssembly text
**Prerequisites:** Phase 13 · 32 (ELF from scratch), 33 (disassembly and control flow)
**Time:** ~150 minutes

## Objetivos de aprendizaje

- Lea el 16 byte `.pyc`encabezado de PEP 552 y desmarcar el objeto de código sin el `marshal`módulo, para archivos escritos por cuatro versiones de intérprete.
- Parsear un archivo de clase JVM desde el JVMS Capítulo 4: conjunto constante, campos, métodos, la `Code`el atributo, y las tablas que `javac -g`- ¿Qué?
- Desmontar los códigos de opcodes de la JVM en `Ledger.class`y demostrar la inclusión en la lista contra `javap -c`en cuatro construcciones de la misma fuente.
- Lea una sección por sección de un módulo de WebAssembly, decodifique cada cuerpo de función y pruebe el resultado en función de la`wasm-objdump -d`¿ Qué ?
- Pase los metadatos ECMA-335 de un conjunto .NET desde el directorio de datos PE hasta el `MethodDef`las filas y el CIL de cada método, verificado en la biblioteca de lectores que ILSpy utiliza.
- Medir, para una función en cinco contenedores, qué nombres, tipos y estructura de control mantiene y qué pierde cada formato.

## El problema

Usted construyó .`ledger.py`y encontró`ledger.cpython-314.pyc`En el`__pycache__`Le preguntaste a un agente qué contenía el archivo, el agente ejecutó un descompilador.`Unsupported opcode`y una línea que dice`# WARNING: Decompyle incomplete`El agente concluyó que el expediente estaba obfuscado.

El archivo no fue obfuscado. Sus dos primeros bytes son`2b 0e`La clave de la base de datos de la base de datos de la computadora de decompiladores es la de la base de datos de la computadora de decompiladores de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de

La lección 33 mostró lo que pierde el código nativo: nombres de parámetros, tipos y los datos.`switch`que se convirtió en una mesa de saltos. Los contenedores de código de byte pierden menos, y cada uno pierde algo diferente. Un pyc mantiene todos los nombres variables y ningún tipo. Un archivo de clase guarda tipos y nombres locales sólo cuando usted pide. Un módulo de Wassm mantiene la estructura de control y deja caer los nombres a menos que una sección personalizada los lleve. Un conjunto .NET guarda nombres de parámetros en una tabla y deja nombres locales en un archivo separado. No puedes juzgar la salida de un decompiler en ninguno de ellos hasta que sabes con qué notas tuvo que comenzar el contenedor.

Esta lección construye cuatro lectores en la biblioteca estándar de Python. Cada uno lee el contenedor desde su primer byte, enumera una función y se comproba con la propia herramienta del formato en los accesorios que construye.`grade`de la`ledger`lado a lado en los cinco contenedores.

```figure
re-43-five-containers
```

## El encabezado de pyc es cuatro palabras

PEP 552 define el encabezado como cuatro palabras de 32 bits. El primero es el número mágico. El segundo es un campo de bits.`check_source`En el caso de los datos de la fuente, el intérprete hashes la fuente en la importación y compara.

El número mágico vive en`Include/internal/pycore_magic_number.h`. Su regla desde 3.11 es que Python 3.n comienza en 2900 + 50n, y cada cambio de código de byte dentro de un ciclo de liberación añade uno.`\r\n`, por lo que un pyc abierto en modo de texto obtiene una magia equivocada. Los valores finales son 3495 para 3.11, 3531 para 3.12, 3571 para 3.13, 3627 para 3.14, y 3666 para 3.15. Python 3.15.0 fue lanzado el 2026-10-09 y no está instalado en la máquina que construyó estos accesorios, por lo que su número proviene del archivo de encabezado y no de un archivo aquí.

Las instalaciones contienen dos archivos pyc por intérprete. `compile_pyc.py`las escribió con `py_compile.compile(..., dfile="ledger.py")`para que el nombre de archivo almacenado sea `ledger.py`Los primeros 16 bytes de los archivos 3.14:

```text
checked-hash   2b 0e 0d 0a  03 00 00 00  c7 16 c7 fc c5 1d 81 01
timestamp      2b 0e 0d 0a  00 00 00 00  8d 60 cf 6a  f1 01 00 00
```

Lea el encabezado de timestamp palabra por palabra. `03 00 00 00`o `00 00 00 00`es la palabra de bandera.`8d 60 cf 6a`es 0x6acf608d = 1791975565, el mtime de `ledger.py`cuando se construyó el archivo. `f1 01 00 00`El archivo de hash verificado tiene banderas 3 y ocho bytes hash en su lugar.

| Interpreter | Magic bytes | Word | File bytes | `grade.co_code` bytes | `grade` instructions |
|---|---|---|---|---|---|
| 3.11.14 | `a7 0d 0d 0a` | 3495 | 1054 | 80 | 31 |
| 3.12.12 | `cb 0d 0d 0a` | 3531 | 937 | 62 | 26 |
| 3.13.12 | `f3 0d 0d 0a` | 3571 | 943 | 70 | 26 |
| 3.14.6 | `2b 0e 0d 0a` | 3627 | 965 | 96 | 35 |

`read_pyc_header()`En el`code/main.py`devuelve estos campos y nombra la versión de la palabra. `importlib.util.MAGIC_NUMBER`Da los mismos cuatro bytes para el intérprete en funcionamiento. `python3 code/main.py compare`compila la fuente con cada `python3.11`¿ Qué ?`python3.15`Se encontró en`PATH`Para cada uno imprime el encabezado que nuestro lector ve junto a los hechos que el intérprete informa sobre sí mismo.

## El mariscal escribe el código objeto campo por campo

El byte 16 de cada fijación es `e3`Eso es .`0x80 | ord('c')`: el código de tipo marshal para un objeto de código con `FLAG_REF`conjunto, que le dice al lector que recuerde el objeto para más tarde `r`La documentación del marshal dice que el formato no está documentado a propósito y que los objetos de código no son compatibles entre las versiones. Ambas declaraciones son ciertas, y el formato es todavía lo suficientemente pequeño como para leer. `Python/marshal.c`enumera los códigos de tipo: `N`Ninguno,`i`una int de 32 bits, `s`bytes, `z`y `Z`corta cadena ASCII, `(`y `)`Túples, `c`un objeto de código, `r`Los contenedores reservan su espacio de referencia antes que sus hijos, así que el lector hace lo mismo.

`w_complex_object()`Escribe un objeto de código en un orden fijo. para 3.11 a 3.15 es`argcount`¿ Qué ?`posonlyargcount`¿ Qué ?`kwonlyargcount`¿ Qué ?`stacksize`¿ Qué ?`flags`como 32 bits de in, entonces `code`¿ Qué ?`consts`¿ Qué ?`names`¿ Qué ?`localsplusnames`¿ Qué ?`localspluskinds`¿ Qué ?`filename`¿ Qué ?`name`¿ Qué ?`qualname`como objetos, entonces `firstlineno`, entonces`linetable`y `exceptiontable`. Descifrar el código de módulo objeto del archivo 3.14 a mano desde el byte 17:

```text
00 00 00 00   argcount 0
00 00 00 00   posonlyargcount 0
00 00 00 00   kwonlyargcount 0
01 00 00 00   stacksize 1
00 00 00 00   flags 0
f3 22 00 00 00   type 's' with FLAG_REF, 34 bytes of co_code follow
80 00  52 06  74 00  52 01  17 00  74 01 ...
```

Los bytes de código son dos por instrucción: `80 00`¿ Es verdad ?`RESUME 0`¿ Qué ?`52 06`¿ Es verdad ?`LOAD_CONST 6`¿ Qué ?`74 00`¿ Es verdad ?`STORE_NAME 0`, y `17 00`¿ Es verdad ?`MAKE_FUNCTION`. La lista 3.14 en los accesorios imprime las mismas cinco declaraciones a nivel de módulo.`code`vienen las constantes, que contienen los cuatro objetos de código anidados, y después de ellos `names`¿ Qué es esto ?`('LABELS', 'percent_change', 'grade', 'total', 'grade_label')`¿ Qué ?

`Unmarshaller`En el`code/main.py`se lee este diseño y el anterior que 3.9 y 3.10 usaron, donde `nlocals`¿ Qué ?`varnames`¿ Qué ?`freevars`, y `cellvars`Se trata de un campo separado.`CodeObject`con los mismos nombres de atributos que el objeto real tiene. `varnames`filtros `localsplusnames`por el `CO_FAST_LOCAL`Un poco de`localspluskinds`- Para .`grade`En cada uno de los cuatro archivos que devuelve `('score', 'bucket')`¿ Qué ?`firstlineno`10, y un no vacío `linetable`- El examen .`test_unmarshaller_agrees_with_the_running_interpreter`compara cada campo con `marshal.loads()`Los otros tres archivos son legibles por nuestro código y no legibles por`marshal`bajo este intérprete.

## Cuatro intérpretes, cuatro bytecodes para calificación

La desdocumentación contiene una advertencia en cada versión: el código de byte es un detalle de implementación de CPython y no se garantiza que se mantenga igual.`python3.1N -m dis ledger.py`fue capturado para cada intérprete.

En 3.11 `grade`comienza así:

```text
 11           2 LOAD_FAST                0 (score)
              4 LOAD_CONST               1 (10)
              6 BINARY_OP                2 (//)
             10 STORE_FAST               1 (bucket)
 12          12 LOAD_FAST                1 (bucket)
             14 LOAD_CONST               2 (9)
             16 COMPARE_OP               5 (>=)
             22 POP_JUMP_FORWARD_IF_FALSE     2 (to 28)
```

`BINARY_OP`Se encuentra en el offset 6 y la siguiente instrucción en 10. Los dos bytes entre ellos son una entrada en caché en línea de PEP 659, el intérprete adaptativo especializado. `COMPARE_OP`En 16 está seguido por 22, así que posee dos entradas de caché.`show_caches=True`. 3.12 sustituye `LOAD_CONST`Además`RETURN_VALUE`con`RETURN_CONST`, renombrar el salto a `POP_JUMP_IF_FALSE`, y los psicóticos .`grade`de 80 a 62 bytes. 3.13 Impresiones de etiquetas `L1`¿ Qué ?`L4`en lugar de compensar y da la CLI `-C`para cajas y `-O`para compensaciones. 3.14 elimina `RETURN_CONST`de nuevo, añade `LOAD_SMALL_INT`y `LOAD_FAST_BORROW`, y inserciones `NOT_TAKEN`después de cada salto condicional:

```text
 12         20       LOAD_FAST_BORROW         1 (bucket)
            22       LOAD_SMALL_INT           9
            24       COMPARE_OP             188 (bool(>=))
            26       CACHE                    0 (counter: 0)
            28       POP_JUMP_IF_FALSE        3 (to L1)
            30       CACHE                    0 (counter: 0)
            32       NOT_TAKEN
```

El `-C -O`muestra de la lista 13 `CACHE`entradas dentro `grade`: cinco después `BINARY_OP`, uno tras otro .`COMPARE_OP`, uno tras otro .`POP_JUMP_IF_FALSE`El dispositivo adaptativo muestra para qué son los caches.`dis.dis(grade, adaptive=True)`huellas`RESUME_CHECK`y `COMPARE_OP_INT`donde la lista simple tenía `RESUME`y `COMPARE_OP`, y `total`muestra`FOR_ITER_LIST`¿ Qué ?`BINARY_OP_ADD_INT`, y `JUMP_BACKWARD_NO_JIT`Las formas especializadas viven en el mismo`co_code`Una pyc nunca las contiene, porque la especialización ocurre en la memoria.

Es por eso que los descompiladores tienen límites de versión. uncompyle6 3.9.3 dice que acepta código de byte de Python 1.0 a 3.8. decompyle3 3.9.3 lee 3.7 y 3.8. xdis 6.3.0, el desmonteador de versión cruzada detrás de ambos, lee hasta 3.13. pycdc tiene tablas de código optico hasta `python_3_13.cpp`en su archivo de construcción, y en una impresión de código op desconocido `Unsupported opcode`PyLingual reclama todas las versiones publicadas desde 3.6 y utiliza intérpretes de 3.8 a 3.15 para recompilar y verificar su propia salida. Para un archivo 3.14 los tres primeros rechazan o fallan, y solo los dos últimos se aplican. Lea la palabra mágica antes de elegir una herramienta.

## El archivo de clases mantiene un diccionario propio

JVMS Java SE 25, sección 4.1, da el diseño como una estructura: `magic`¿ Qué ?`minor_version`¿ Qué ?`major_version`¿ Qué ?`constant_pool_count`, la piscina constante,`access_flags`¿ Qué ?`this_class`¿ Qué ?`super_class`, interfaces, campos, métodos, atributos. La magia es`0xCAFEBABE`Una máquina virtual Java SE 25 acepta las versiones principales 45 a 69, y una minor de 65535 marca un archivo de función de vista previa.`Ledger-g.class`¿ Qué ?`ca fe ba be 00 00 00 41 00 5e`: mayor 65 es Java 21, y `0x5e`= 94 es el número constante de las cuencas. `Ledger-jdk25.class`, construido por javac 25.0.2 de la misma fuente, comienza `ca fe ba be 00 00 00 45`El mayor 69.

El grupo constante es donde el archivo de clase mantiene su propio diccionario.`#13`en el `-g`construir es un `Methodref`que apunta a la clase `#8`y nombre y tipo `#14`. Siguiendo los índices se da `Ledger.grade:(I)I`, que es exactamente lo que `javap -v`huellas después `//`El descriptor .`(I)I`Es uno .`int`Parámetro y un `int`El resultado.`([I)I`En el`total`dice que el parámetro es un `int`Cada método, campo y clase que el código toca tiene tal entrada, y el verificador en la sección 4.10 rechaza un archivo cuyas instrucciones no están de acuerdo con ellos.`ClassFile`clase en `code/main.py`lee todas las 17 etiquetas constantes de la tabla 4.4 y trata un `Long`o `Double`Se eleva en una etiqueta desconocida y en bytes después del último atributo.

Los atributos se cuelgan de la clase, cada campo, cada método, y el `Code`JVMS dice que la máquina virtual debe ignorar atributos que no conoce, por lo que las tablas de descomposición se pueden agregar o soltar sin cambiar un cuerpo de método.`-g`genera toda la información de depuración, incluidas las variables locales, que el predeterminado genera números de línea y el archivo fuente sólo, y que `-g:none`Los tres construyen la medida que:

| Build | Bytes | Constant pool count | `LineNumberTable` | `LocalVariableTable` | `SourceFile` | `grade` code bytes |
|---|---|---|---|---|---|---|
| `javac -g` | 1832 | 94 | 7 methods | 6 methods | yes | 50 |
| `javac` | 1581 | 82 | 7 methods | none | yes | 50 |
| `javac -g:none` | 1404 | 79 | none | none | no | 50 |

Los 50 bytes de código de `grade`son idénticos en los tres archivos, y la prueba lo verifica.`-g`añade para `grade`Es uno .`LocalVariableTable`fila: comienzo 0, longitud 50, ranura 0, nombre `score`, descriptor `I`Sin él un descompilador ve ranura de tipo 0`int`Y tiene que inventar un nombre.`StackMapTable`se mantiene en cada construcción porque el verificador lo necesita, y el `BootstrapMethods`El atributo permanece porque `main`concatenando las cuerdas a través de `invokedynamic`¿ Qué ?

## Descifrar el interruptor de tablas a mano

El capítulo 6 de JVMS define cada instrucción como un byte de código optico y los operandos que nombran el código optico.`tableswitch`y `lookupswitch`comienzan con cero a tres bytes de relleno para que sus operandos de 32 bits comienzan en una dirección que es un múltiplo de cuatro.

Los bytes de `grade`desde el ordenador 4 en `Ledger-g.class`¿Qué es esto ?

```text
aa                 tableswitch at pc 4
00 00 00           padding to pc 8
00 00 00 2c        default: 4 + 44 = 48
00 00 00 06        low 6
00 00 00 0a        high 10
00 00 00 2a        case 6:  4 + 42 = 46
00 00 00 28        case 7:  4 + 40 = 44
00 00 00 26        case 8:  4 + 38 = 42
00 00 00 24        case 9:  4 + 36 = 40
00 00 00 24        case 10: 4 + 36 = 40
```

36 bytes, una instrucción, y todo.`switch (score / 10)`La construcción nativa de la misma `switch`En la lección 33 se necesitaba una verificación de límites, un `lea`, una `movslq`, un `add`Aquí los objetivos están en la instrucción, y también es el hecho de que pertenecen juntos.

`disassemble_jvm()`cubre todos los 202 códigos opcodes definidos con sus anchos de operand, sigue la regla de relleno y hace operands de la manera `javap`¿Qué es eso?`compare_with_javap()`analiza a los capturados .`javap -c -p`texto, incluido el texto de varias líneas `tableswitch`Los cuatro componentes están de acuerdo en 108 de las 108 instrucciones.`test_branch_offsets_are_relative_to_the_branch_instruction`los cheques `ifne`en el pc 1 aterrizaje en 6, `if_icmpge`en el 13 aterrizaje en el 33, y `goto`Al 30 aterrizar de nuevo en 10.

radar2 6.2.4, construido desde la fuente para estas lecciones, lee el mismo archivo. `rabin2 -I`informes `arch java`¿ Qué ?`lang java 21`¿ Qué ?`linenum true`, y `lsyms true`, que es su lectura de la versión y las dos tablas de descomposición. `pd 14 @ sym.Ledger.grade`Imprima el `tableswitch`Con flechas a sus cinco objetivos.`Unable to parse code attribute: 2`En este archivo, que es una limitación de la herramienta nombra a sí mismo y una razón para mantener `javap`como el oráculo. Los descompiladores para este formato son maduros en comparación con Python. CFR 0.152 está licenciado por el MIT. Procyon 0.6.0 y Vineflower 1.12.0 son Apache-2.0, y Vineflower es un horquilla de Fernflower con soporte Java 21. Fernflower mismo vive dentro del repositorio de la Comunidad IntelliJ. JADX 1.5.6 cubre el formato DEX de Android, y JD-GUI tuvo su última versión en 2019. Harrand, Soto-Valero, Monperrus y Baudry midieron ocho de ellos en 2020. El mejor producido sintácticamente correcto de salida para el 84 por ciento de las clases y semánticamente equivalente de salida para el 78 por ciento. Su meta-descompiler recuperó el 37,6 por ciento de las clases que ninguna herramienta manejaba. Los tipos, nombres y pilas verificadas hacen que el formato sea descompiliable. No hacen que un solo decompiler esté completo.

## Un módulo de la masa es una lista de secciones

La especificación central define un módulo como el preámbulo `00 61 73 6d 01 00 00 00`, la cuerda`\0asm`y la versión 1, seguida de secciones. Cada sección es un byte de ID, un `u32`El número de identificación de los números 1 a 13 es el número de las secciones conocidas, cada una de ellas una vez más y en orden.`e4 00`es de 100 y `80 08`Es 1024.`wasm-objdump -h`En el`ledger.wasm`En el artículo 7 se incluyen siete secciones y `WasmModule`encuentra los mismos siete con los mismos desplazamientos de carga útil:

| Section | Payload | Count | What it holds for ledger |
|---|---|---|---|
| Type | 10 to 26 | 3 | `(i32) -> nil`, `(i32, i32) -> i32`, `(i32) -> i32` |
| Import | 28 to 39 | 1 | `env.log` with type 0 |
| Function | 41 to 46 | 4 | type index for each defined function |
| Memory | 48 to 51 | 1 | one page, no maximum |
| Export | 53 to 110 | 5 | `memory` and the four function names |
| Code | 113 to 271 | 4 | bodies of 23, 63, 44, and 23 bytes |
| Data | 273 to 290 | 1 | `F\0D\0C\0B\0A\0` at offset 1024 |

La división entre las secciones Función y Código existe para que un navegador pueda comenzar a compilar antes de que llegue todo el archivo.`grade`el vector local es `01 7f`: un grupo de uno `i32`Eso es .`bucket`, con su tipo y sin su nombre.

El flujo de control se estructura por construcción.`block`¿ Qué ?`loop`, o`if`Las etiquetas se indexan por profundidad de anidación. `br_table`El sistema de etiquetas de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de red de datos de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de`grade`en el archivo de compensación `0xa3`¿Qué es esto ?

```text
0e                  br_table
0b                  11 labels follow
04 04 04 04 04 04   buckets 0 to 5: label 4, the outermost block, returns 0
03 02 01 00         buckets 6, 7, 8, 9: labels 3, 2, 1, 0
00                  bucket 10: label 0, shared with 9
04                  default label 4
```

Cada etiqueta es una profundidad.`block`, y el código después de su `end`¿ Es verdad ?`i32.const 4`seguido por `br 4`fuera del bloque de resultados.`block`Las instrucciones delante de la mesa, las once etiquetas y las cinco `br`instrucciones después de que son todo `switch`, y un validador rechaza cualquier rama que salga de la estructura. `disassemble_wasm()`rastrea la profundidad de modo que los tiros de la lista como `wasm-objdump`, cubre cada código op de un solo byte desde `unreachable`¿ Qué ?`i64.extend32_s`, y coincide con los capturados `wasm-objdump -d`En 77 de las 77 instrucciones para ambas construcciones.

Los nombres son lo único que cae un módulo simple.`wasm-objdump`huellas todavía`<grade>`porque la sección Export mapea esa cadena para el índice de función 2, y el lector hace lo mismo. Los nombres locales no existen en ninguna parte en el archivo de 290 bytes. `wat2wasm --debug-names`añade una sección personalizada llamada `name`El archivo crece hasta 425 bytes. El lector encuentra `{0: 'score', 1: 'bucket'}`para la función 2 y `{0: 'values', 1: 'count', 2: 'sum', 3: 'i'}`para la función 3. `wasm-strip`La sección de las convenciones de herramientas describe dos portadores más.`.debug_*`Secciones de aduanas.`sourceMappingURL`sección personalizada puede apuntar a un mapa fuente cuyos números de columna son byte offsets en el binario.

La herramienta se movió durante 2026.`wasm-decompile`, y la construcción de Homebrew en esta máquina tiene exactamente las once herramientas restantes.`print`¿ Qué ?`objdump`, y `demangle`. Binaryen version_133 proporciona `wasm-dis`. radar2 6.2.4 también lee el módulo.`rabin2 -I`dice`arch wasm`Su símbolo para`grade`comienza en`0x8e`, donde comienza el vector local, mientras que `wasm-objdump`etiquetado la función en `0x8b`En su lista se utilizan las mnemónicas más antiguas.`get_local`¿ Qué ?`set_local`, y `brtable 11 4 4 ...`con el recuento de etiquetas impresos. Los bytes son los mismos en las tres herramientas y las ortografías difieren, por lo que una comparación debe nombrar la herramienta con la que fue capturada. La investigación para esta lección no encontró ningún módulo de procesador WebAssembly en el directorio de procesadores de Ghidra, y esa lista no se leyó en su totalidad. El plugin de ghidra-wasm de terceros v2.4.0 tiene como objetivo Ghidra 12.0. La especificación en sí misma es un documento vivo. La página del W3C para `wasm-core-2`es un proyecto de recomendación de candidatos de 13 de octubre de 2026 cuyo resumen describe la versión 3.0, y el proyecto de la redacción tiene la misma fecha.

## Los metadatos ECMA-335 son la base de datos del programa

Un conjunto .NET es un archivo PE. `LedgerNet.dll`comienza con `MZ`, tiene tres secciones nombradas `.text`¿ Qué ?`.rsrc`, y `.reloc`, y declara máquina `0x14c`El directorio de datos 14 del encabezado opcional, el encabezado de tiempo de ejecución CLR, apunta a RVA `0x2008`. La ECMA-335 II.25.3.3 define los 72 bytes que se encuentran en el mismo:

```text
48 00 00 00   Cb 72
02 00 05 00   MajorRuntimeVersion 2, MinorRuntimeVersion 5
78 21 00 00   MetaData RVA 0x2178
84 07 00 00   MetaData size 1924
01 00 00 00   Flags: COMIMAGE_FLAGS_ILONLY
05 00 00 06   EntryPointToken 0x06000005, MethodDef row 5, which is Main
```

La raíz de metadatos en RVA `0x2178`comienza con la firma `42 53 4a 42`, que dice `BSJB`, luego la versión 1.1 y la cadena `v4.0.30319`Cinco cabeceras de corriente siguen:`#~`en compensación `0x6c`, entonces`#Strings`¿ Qué ?`#US`¿ Qué ?`#GUID`, y `#Blob`. II.24.2 dice que cada tipo de corriente ocurre una vez como máximo.`#~`El flujo comienza con un `HeapSizes`byte, cero aquí, así que cada cadena, GUID, e índice de manchas es de dos bytes de ancho, y un 64-bit `Valid`mascaras que nombran las tablas presentes.

| Table | Id | Rows | Row size | What the rows say |
|---|---|---|---|---|
| Module | 0x00 | 1 | 10 | the module name and its GUID |
| TypeRef | 0x01 | 21 | 6 | `System.Object`, `System.Console`, 19 more |
| TypeDef | 0x02 | 4 | 14 | `<Module>`, `Ledger`, and two compiler helpers |
| Field | 0x04 | 2 | 6 | `Labels` and a static array initialiser |
| MethodDef | 0x06 | 6 | 14 | the five methods and `.cctor` |
| Param | 0x08 | 5 | 6 | `before`, `after`, `score`, `values`, `score` |
| MemberRef | 0x0a | 16 | 6 | 12 attribute constructors and the 4 methods that `Main` calls |
| CustomAttribute | 0x0c | 12 | 6 | attributes the compiler and the SDK add |
| StandAloneSig | 0x11 | 3 | 2 | the three local variable signatures |
| Assembly, AssemblyRef, ClassLayout, FieldRva, NestedClass | | 1, 2, 1, 1, 1 | | identity, references, layout |

El tamaño de una fila no se almacena en ninguna parte. II.22 da a cada columna un tipo, y II.24.2.6 da la regla de tamaño.`HeapSizes`. Un índice de tabla simple es de dos bytes mientras que esa tabla tiene menos de 65536 filas. Un índice codificado que puede apuntar a n tablas con b bits de etiqueta es de dos bytes mientras que cada una de esas tablas tiene menos de 2^(16-b) filas. `CliAssembly`El sistema.Reflexión.Dump de metadatos en los accesorios informa los mismos tamaños y los mismos desventajas de tablas, que es la forma en que el ensayo demuestra la aritmética.

III.1.9 define el token: el byte alto es la tabla, los tres bytes bajos son la fila basada en 1. `0x06000002`¿ Es verdad ?`MethodDef`fila 2. Esa fila tiene RVA `0x2060`, nombre `Grade`, y una mancha de firma .`00 01 08 08`: convención de llamada por defecto, un parámetro, retorno `int32`, parámetro `int32`- Es un .`ParamList`puntos en la fila 3 de `Param`, cuyo nombre es `score`La mesa de Param es por lo que ILSpy puede imprimir`Grade(int score)`de una construcción de liberación desnudada.`0x2060`el cuerpo del método comienza con un encabezado graso:

```text
13 30         flags 0x13: fat, init locals; size 3 dwords
02 00         MaxStack 2
2d 00 00 00   CodeSize 45
01 00 00 11   LocalVarSigTok 0x11000001
```

`StandAloneSig`la fila 1 se mantiene `07 01 08`¿ Qué es esto ?`LOCAL_SIG`, un local, `int32`. El local tiene un tipo y no tiene nombre. ECMA-335 guarda los nombres locales en el formato de intercambio de desinfecciones de la partición V, y el SDK .NET los escribe a un PDB.`DebugType`Ninguna, así que ninguna tabla y ningún archivo lleva el nombre.`bucket`- ¿ Qué ?`PercentChange`tiene un encabezado pequeño en su lugar: su primer byte `3a`¿ Es verdad ?`0x0e << 2 | 2`, un cuerpo de 14 bytes sin locales y una pila predeterminada de 8.

Los 45 bytes de código de `Grade`Descifrar con la tabla de código optico de la Parte III. `switch`es opcode `0x45`seguida de un `u32`Conto y tantos.`int32`objetivos relativos a la instrucción después de la tabla:

```text
IL_0000: 02           ldarg.0
IL_0001: 1f 0a        ldc.i4.s 10
IL_0003: 5b           div
IL_0004: 0a           stloc.0
IL_0005: 06           ldloc.0
IL_0006: 1c           ldc.i4.6
IL_0007: 59           sub
IL_0008: 45 05 00 00 00 08 00 00 00 06 00 00 00 04 00 00 00 02 00 00 00 02 00 00 00
                      switch (IL_0029, IL_0027, IL_0025, IL_0023, IL_0023)
IL_0021: 2b 08        br.s IL_002b
IL_0023: 1a 2a        ldc.i4.4, ret
```

El compilador restó 6 antes de la `switch`, así que el caso 6 es el primer objetivo.`disassemble_cil()`renders tokens a través de las tablas: `ldsfld Labels`¿ Qué ?`call Grade`¿ Qué ?`call System.Console::WriteLine`, y `ldstr ' '`de la `#US`El oráculo para todo esto es`MetaDump`, un programa C# corto en los accesorios construidos en`System.Reflection.Metadata`Imprime el recuento de filas, tamaño y compensación de cada tabla, el RVA de cada método, la firma, los nombres de parámetros y los bytes de IL, y los 226 campos de `System.Reflection.Emit.OpCodes`- ¿ Qué ?`compare_with_metadump()`La tabla de tiempo de ejecución tiene 218 códigos de opciones más ocho.`prefix`La tabla de la lección tiene 219 entradas porque incluye:`no.`de III.2.2, que la tabla de emisores nunca tuvo.

Herramientas para el formato: ILSpy 11.0 (MIT, lanzado 2026-08-12) se ejecuta en macOS a través de su versión de escritorio Avalonia. `ilspycmd`11.1.0.9782 Impresión de la IL con `-il`y selecciona un método por token con `-m 0x06000002`. dnSpyEx 6.6.0 continúa dnSpy sólo en Windows.`ildasm`las naves como paquetes NuGet en lugar de dentro del SDK. radar2 6.2.4 lee los metadatos también: `rabin2 -I`dice`arch cil`y `class PE32`, y `is`las listas `Ledger.Grade(int:score):int`Con el nombre del parámetro y ambos tipos tomados de las tablas de Param y firma. No tiene desmonteador CIL en esta construcción. Este accesorio fue construido con .NET SDK 8.0.205 porque ese es el SDK instalado en la máquina de construcción. .NET 10 ha estado actuando desde 2025-11-11 y escribe las mismas estructuras ECMA-335 .

## Lo que guarda cada contenedor

Ponga`grade`de los cinco contenedores de una sola mesa.`python3 code/main.py keeps`, que hace que los lectores sobre las lámparas:

| Container | Body bytes | Function name | Parameter names | Parameter and return types | Local names | Line numbers | Control flow |
|---|---|---|---|---|---|---|---|
| ELF object, clang -O0 | 118 | `grade` in `.symtab` | none | none | none | none, no `.debug_*` sections | 5 jumps and 1 indirect jump through a relocation table |
| CPython 3.14 pyc | 96 | `co_qualname` | `score` | none | `bucket` | `co_linetable`, first line 10 | 4 compare and `POP_JUMP_IF_FALSE` pairs |
| JVM class, `javac -g` | 50 | constant pool | `score` in `LocalVariableTable` | `(I)I` | none beyond the parameter | `LineNumberTable`, 6 rows | 1 `tableswitch` |
| WebAssembly | 63 | Export section | only with a name section | `(i32) -> i32` | only with a name section | none | 6 nested blocks, 1 `br_table`, 4 `br` |
| .NET assembly | 45 | `MethodDef` | `score` in `Param` | `static int32(int32)` | none, types only in `StandAloneSig` | none without a PDB | 1 `switch`, 1 `br.s` |

La tabla explica los resultados del descompilador que verá.`int total(const int *, int)`con el `int total(long param_1, uint param_2)`que Ghidra dedujo del código nativo, porque el código nativo no guarda ninguna de estas columnas. Un pyc guarda todos los nombres y ningún tipo, y su flujo de control es saltos cuya forma cambia cada versión, por lo que los descompiladores de Python rompen por versión y recuperan nombres de forma gratuita. Un archivo de clase guarda tipos y pilas verificadas, guarda nombres cuando`-g`es en, y expresa el `switch`Un módulo de Wassm mantiene la estructura por construcción, así que `wasm2wat`Un conjunto .NET mantiene la descripción más completa, porque las tablas de metadatos son normativas, y todavía necesita un PDB para el nombre de un local.

La tabla le dice con qué tuvo que trabajar el descompilador, y cuáles de sus salidas copian un hecho almacenado y cuáles lo deducen. Un nombre de parámetro en la salida ILSpy es una copia. Un nombre local en la misma salida es una inferencia a menos que haya estado presente un PDB. La lección 35 registra esa diferencia como la autoridad de cada afirmación.

## Construye el mismo

Ejecutar la demostración:

```bash
cd phases/13-tools-and-protocols/43-bytecode-and-managed-formats
python3 code/main.py
```

La salida es un informe JSON. Para cada pyc se dan ambos encabezados, el archivo y `grade`Los datos de la lista de instrucciones y nombres de opcodes de la lista capturada, y si el intérprete en ejecución puede leerlo. para cada clase de construcción que da la versión, los tamaños, los recuentos de tablas de desaceleración, y el `javap`Para cada construcción de Wassm se dan secciones, tipos, funciones, contenido de la sección de nombre y el`wasm-objdump`Para el ensamblaje, se da el encabezado de CLI, los flujos, el recuento de filas de tablas, los métodos y el acuerdo de sistema.Reflección.Metadatos.`PATH`¿ Qué ?

Leer .`code/main.py`en este orden:

1. `Reader`tiene el cursor y los anchos enteros. `uleb()`y `sleb()`son los lectores LEB128 que Wasm y el montón de manchas .NET necesitan.
2. `read_pyc_header()`¿ Qué ?`Unmarshaller`, y `PycFile`Leer un guión.`parse_dis_text()`Leía los capturados .`dis`las listas para que la comparación entre versiones cruzadas no necesite cuatro intérpretes en el momento de la prueba.
3. `ClassFile`lee el archivo de clase y `disassemble_jvm()`Descifrará su código.`parse_javap_code()`y `compare_with_javap()`son los cheques del oráculo.
4. `WasmModule`lee las secciones y `disassemble_wasm()`Descifrar una expresión con su profundidad.`compare_with_wasm_objdump()`es el cheque del oráculo.
5. `CliAssembly`camina PE, encabezado CLI, raíz de metadatos, flujos y tablas. `TABLE_SCHEMAS`y `CODED_INDEXES`Se consideran II.22 y II.24.2.6 como datos. `disassemble_cil()`Descifrar un cuerpo y`compare_with_metadump()`- ¿Qué es eso?
6. `NativeObject`lee ELF suficiente para enumerar símbolos de función y toma la lección 33 como su texto. `what_each_keeps()`y `figure_data()`construye la comparación.

Hay cinco órdenes más.`python3 code/main.py pyc FILE`¿ Qué ?`class FILE`¿ Qué ?`wasm FILE`, y `dll FILE`imprimir la estructura y una lista de cada función en un archivo que les proporcione. `python3 code/main.py compare`compilaciones `ledger.py`con cada intérprete en `PATH`y imprime lo que cada uno escribió.

## Usalo

Usa los lectores en este orden en un recipiente que construiste.

1. Lea los primeros bytes y nombra el formato.`0d 0a`es un pyc, `ca fe ba be`un archivo de clases, `00 61 73 6d`un módulo, `4d 5a`con el directorio de datos 14 establecer un conjunto.
2. Lea el encabezado y escriba la versión de productor que nombra. Para un pyc que es la palabra mágica, para un archivo de clase la versión principal, para un conjunto las cadenas de versión de tiempo de ejecución y metadatos.
3. Revisa cada decompiler que planeas ejecutar con esa versión antes de ejecutarla.
4. ejecuta el lector y la propia herramienta del formato en el mismo archivo y comparalas fila por fila. Una incompatibilidad es un error en el lector o un cambio en la herramienta, y cualquiera de ellos debe entenderse antes de que la lista sea evidencia.
5. Llenar la tabla de mantenimiento para la función que le importa, marcar cada hecho como conservado o perdido y nombrar la tabla, el atributo o la sección de la que proviene.
6. Sólo entonces lee la salida del descompilador, marca cada nombre y escribe como una copia de un hecho guardado o como una inferencia.

Mantenga los archivos de texto del oráculo con los contenedores. Cuando un intérprete, JDK, SDK o versión wabt cambia, reconstruya con `build_fixtures.sh`y recapturar, porque una lista cambiada es una fijación cambiada.

## Laboratorio interactivo

### Laboratorio A: leer el encabezado, luego el objeto del código

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/43-bytecode-and-managed-formats/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
pyc = load_pyc("ledger.cpython-311.pyc")
pyc.header.to_dict()
pyc.module.names
pyc.functions()["grade"].varnames
len(pyc.functions()["grade"].code)
```

Escribe la palabra mágica antes de la primera llamada.`ledger.cpython-314.pyc`y comparar los dos `code`Entonces intenta.`import marshal; marshal.loads(pyc.data[16:])`En el archivo 3.11 bajo su intérprete y leer el error.

### Laboratorio B: la cadena constante de la piscina

```python
cls = load_class("Ledger-g.class")
cls.constant(13)
cls.constant(14)
cls.describe_constant(13)
[row["text"] for row in jvm_listing(cls, "grade")]
```

Sigue .`#13`¿ Qué ?`#8`y `#14`, luego a la`Utf8`Las entradas, a mano antes de llamar.`describe_constant`Entonces cargue .`Ledger-gnone.class`y llamar`cls.method("grade").attribute_names()`En ambos.`Code`el array es el mismo, los atributos no lo son.

### Laboratorio C: profundidad y etiquetas

```python
module = load_wasm("ledger.wasm")
[(row["depth"], row["text"]) for row in wasm_listing(module, "grade")]
module.function_name(2)
load_wasm("ledger-names.wasm").names["locals"][2]
```

Encuentra el`br_table`Recuerde su profundidad.`end`filas después de él y coincidir cada uno con un número de etiqueta en la tabla.`ledger.wat`y encontrar el `block $six`Esa etiqueta tiene tres nombres.

### Laboratorio D: desde tokens hasta bytes

```python
asm = load_assembly("LedgerNet.dll")
asm.method("Grade")
body = asm.method_body(asm.method("Grade")["RVA"])
body.fat, body.max_stack, body.code_size, hex(body.local_signature_token)
asm.token_text(0x11000001)
[row["text"] for row in cil_listing(asm, "GradeLabel")]
```

Calcule el descenso de archivo de RVA `0x2060`de la `.text`encabezado de sección a mano y comparar con `asm.rva_to_offset(0x2060)`Entonces mira hacia arriba .`asm.row(0x08, 3)`y encontrar donde el nombre `score`¿Qué es eso?

## Laboratorio de práctica

Añadir un quinto lenguaje fuente a la comparación, o un sexto hecho a la tabla.

Requisitos:

- Escriba una función con un bucle y una variable local en un lenguaje cuyo compilador tiene, por ejemplo Kotlin a un archivo de clase o Rust a un módulo Wasm. Compromete la salida y la lista del oráculo con el comando exacto en `build_fixtures.sh`¿ Qué ?
- Haz que el lector coincida con el oráculo en cada instrucción.
- Extenderse`what_each_keeps()`El nuevo contenedor, y un hecho más para cada contenedor: si el cuerpo lleva una tabla de excepción o `try`región.
- Añadir el nuevo contenedor a `figure_data()`y regenerarse.`site/figures-re-43.js`¿ Qué ?
- Escriba una prueba que falla cuando un lector informa un hecho como se guardó sin nombrar la tabla, el atributo o la sección que lo contiene.

El éxito significa`python3 code/main.py keeps`muestra la nueva fila con evidencia para cada célula, y cada comparación del oráculo todavía está de acuerdo.

## Lección Artefacto

Esta lección incluye `outputs/skill-bytecode-and-managed-formats.md`. Utilice para convertir un contenedor de código de byte en un informe de encabezado, una lista verificada de una función y una tabla de hechos que marca cada nombre, tipo y estructura como guardada o perdida. Requiere la propia herramienta del formato como oráculo, una verificación de versión antes de cualquier descompilador y una declaración de dónde habría vivido cada hecho perdido.

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/43-bytecode-and-managed-formats
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

La verificación deberá demostrar:

- Los cuatro encabezados pyc llevan las palabras mágicas 3495, 3531, 3571, y 3627, banderas 3, y un hash de 8 bytes, y los archivos de timestamp llevan banderas 0, mtime 1791975565, y tamaño 497
- Cada palabra mágica para 3.11 y más tarde se encuentra en el rango de la`2900 + 50n`regla da su liberación
- el unmarshaler devuelve los mismos nombres, constantes, bytes de código, nombres de variables y tabla de líneas como `marshal.loads()`en el archivo el intérprete en ejecución escribió
- Los cuatro archivos guardan los nombres de los módulos,`('score', 'bucket')`por`grade`, primera línea 10, y `ledger.py`como el nombre del archivo
- `grade.co_code`es 80, 62, 70 y 96 bytes en las cuatro versiones y las listas capturadas tienen 31, 26, 26 y 35 instrucciones
- La lista 3.11 tiene `POP_JUMP_FORWARD_IF_FALSE`, 3.12 y 3.13 tienen `RETURN_CONST`, y 3.14 ha `LOAD_SMALL_INT`¿ Qué ?`LOAD_FAST_BORROW`, y `NOT_TAKEN`
- el 3.14 `-C -O`la lista muestra 13 entradas de caché en `grade`, cinco de ellos después .`BINARY_OP`, y los programas de listaje adaptativo `COMPARE_OP_INT`
- un encabezado sin `0d 0a`, un encabezado truncado, y un cuerpo que no es un objeto de código elevar `FormatError`
- una lista de instrucciones sólo existe para el pyc escrito por el intérprete en ejecución y coincide con el capturado `dis`salida
- Los archivos de clase tienen 65 o 69 majors, tamaños 1581, 1832, 1404, y 1581, y cuenta de grupo constante 82, 94, 79, y 82
- constante`#13`se resuelve a `Method Ledger.grade:(I)I`y `#47`¿ Qué ?`String F`
- el desmonteador de la VVM está de acuerdo con `javap -c -p`en 108 de las 108 instrucciones en las cuatro construcciones
- `tableswitch`en pc 4 es 36 bytes con bajo 6, alto 10, por defecto 48, y objetivos 46, 44, 42, 40, 40
- los tres niveles de descomposición cambian el recuento de atributos y dejan los 50 bytes de código de `grade`identico
- `ifne`¿ Qué ?`if_icmpge`, y `goto`los objetivos son relativos al código op de la rama
- una magia equivocada, bajadas atrasadas, y un aumento indefinido de código optico.`FormatError`
- Las siete secciones de Wassm tienen la carga útil compensada `wasm-objdump -h`Imágenes, y los nombres de construcción añade un `name`sección de aduanas en 293 a 425
- Los tipos de`env.log`La información sobre la importación, las cinco exportaciones y el segmento de datos en el decodo 1024 como se escribe en `ledger.wat`
- El desmonteador de Wassm está de acuerdo con `wasm-objdump -d`en 77 de las 77 instrucciones en ambas construcciones
- `grade`tiene 6 bloques anidados, un `br_table`con etiquetas `4 4 4 4 4 4 3 2 1 0 0`y 4 por defecto en profundidad 6, y uno `i32`local
- Los nombres de las funciones provienen de las exportaciones sin una sección de nombres y los nombres locales sólo con él
- LEB128 `e4 00`es 100, `7f`es -1, `80 08`es 1024, y los tipos de bloques se rinden como `wasm-objdump`
- un preámbulo equivocado, versión 2, y un aumento desconocido de código op `FormatError`
- el conjunto es PE32 con máquina `0x14c`, encabezado de CLI en RVA `0x2008`, tiempo de ejecución 2,5, `ILONLY`, y punto de entrada `0x06000005`
- La versión de metadatos es `v4.0.30319`, existen cinco corrientes, y las tablas 14 tienen el recuento de filas y los tamaños de filas que el oráculo informa
- El lector .NET está de acuerdo con System.Reflection.Metadata en 246 de 246 controles
- `Grade`tiene parámetro `score`, firma `static int32(int32)`, un cabezal grueso con `MaxStack`2 y 45 bytes de código, y uno `int32`local sin nombre
- CIL `switch`objetivos, `br.s`¿ Qué ?`ldsfld`¿ Qué ?`call`, y `ldstr`operands resuelven a través de las tablas y montones
- La tabla de código optico de CIL cubre todos los códigos opticos de tiempo de ejecución y solo añade `no.`
- la tabla de mantenimiento informa el nombre del parámetro en cuatro contenedores y en ninguno para el código nativo, y los tipos en tres
- Cada línea del oráculo en la demostración lee n de n
- La figura interactiva incorpora los mismos datos `figure_data()`Produce

## Modos de falla de producción

| Failure | What the weak tool reports | What the analysis must prove |
|---|---|---|
| Decompiler older than the interpreter | `Unsupported opcode`, output marked incomplete, or a claim of obfuscation | The magic word, the release it names, and the tool's documented ceiling |
| pyc read with the wrong interpreter | `marshal.loads` raises or returns garbage | Our unmarshaller reads names, constants, and sizes from any 3.9 to 3.15 file, and instructions come from the matching `dis` |
| Specialised opcodes mistaken for the file's content | `COMPARE_OP_INT` in a listing | A pyc holds the unspecialised form, specialisation happens in memory after warm-up |
| Local names reported for a default javac build | `var1`, `var2` presented as source names | `LocalVariableTable` is present only with `-g`, the 50 code bytes are identical either way |
| Branch target off by the instruction length | A JVM listing whose jumps land mid-instruction | Offsets relative to the branch opcode, checked against `javap` |
| Switch padding skipped | A `tableswitch` with a wrong default and shifted cases | Zero to three padding bytes to a 4-aligned pc, 36 bytes for five cases |
| Wasm names assumed stripped | "no symbols" | Exports name every exported function, locals need the `name` section, `wasm-strip` removes only that section |
| Three tools, three spellings | `get_local` versus `local.get`, a function at `0x8b` versus `0x8e` | Same bytes, and the listing that the comparison was captured with |
| Table row size guessed | Every row after the first table misread | Row sizes from `HeapSizes`, row counts, and coded index widths, equal to the oracle's |
| Local names expected in a release assembly | `V_0` presented as the source name | Locals have types in `StandAloneSig` and names only in a PDB that `DebugType` none did not write |
| Reader guesses an unknown opcode or type code | A plausible mnemonic | `FormatError` with the offset, and the oracle comparison stops there |
| Oracle captured from another build | Tests pass on stale text | `build_fixtures.sh` names every tool version and writes `versions.txt` and `sha256.txt` |

## Regla de funcionamiento

Leer el encabezado primero y nombrar al productor al que apunta. Leer el contenedor con un lector que rechaza lo que no sabe, y comprobar su listaje con la propia herramienta del formato en el mismo archivo. Luego digamos, para cada nombre y tipo en la salida de un descompilador, ya sea que el contenedor lo almacenó o la herramienta lo deducido.

## Leer más

- [PEP 552: Deterministic pycs](https://peps.python.org/pep-0552/)
- [CPython `Include/internal/pycore_magic_number.h`](https://github.com/python/cpython/blob/main/Include/internal/pycore_magic_number.h)
- [Python `marshal` module documentation](https://docs.python.org/3/library/marshal.html)
- [CPython `Python/marshal.c`](https://github.com/python/cpython/blob/main/Python/marshal.c)
- [Python 3.14 `dis` module documentation](https://docs.python.org/3.14/library/dis.html)
- [PEP 659: Specializing Adaptive Interpreter](https://peps.python.org/pep-0659/)
- [PEP 626: Precise line numbers for debugging and other tools](https://peps.python.org/pep-0626/)
- [PEP 790: Python 3.15 Release Schedule](https://peps.python.org/pep-0790/)
- [uncompyle6 on PyPI](https://pypi.org/project/uncompyle6/)¿ Qué ?[decompyle3 on PyPI](https://pypi.org/project/decompyle3/)¿ Qué ?[xdis on PyPI](https://pypi.org/project/xdis/)¿ Qué ?[pycdc on GitHub](https://github.com/zrax/pycdc)¿ Qué ?[PyLingual on GitHub](https://github.com/syssec-utd/pylingual)
- [The Java Virtual Machine Specification, Java SE 25, Chapter 4: The class File Format](https://docs.oracle.com/javase/specs/jvms/se25/html/jvms-4.html)
- [The Java Virtual Machine Specification, Java SE 25, Chapter 6: The Java Virtual Machine Instruction Set](https://docs.oracle.com/javase/specs/jvms/se25/html/jvms-6.html)
- [javac manual, Java SE 25](https://docs.oracle.com/en/java/javase/25/docs/specs/man/javac.html)y [javap manual](https://docs.oracle.com/en/java/javase/25/docs/specs/man/javap.html)
- [CFR](https://www.benf.org/other/cfr/)¿ Qué ?[Procyon](https://github.com/mstrobel/procyon)¿ Qué ?[Vineflower](https://github.com/Vineflower/vineflower)¿ Qué ?[Fernflower in intellij-community](https://github.com/JetBrains/intellij-community/tree/master/plugins/java-decompiler/engine)¿ Qué ?[JADX](https://github.com/skylot/jadx)
- [Harrand, Soto-Valero, Monperrus, Baudry, "Java Decompiler Diversity and its Application to Meta-decompilation" (JSS 2020)](https://arxiv.org/abs/2005.11315)
- [ECMA-335, Common Language Infrastructure, 6th edition (June 2012)](https://ecma-international.org/publications-and-standards/standards/ecma-335/)
- [PE Format, Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)
- [System.Reflection.Metadata.Ecma335.TableIndex](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.metadata.ecma335.tableindex)
- [ILSpy on GitHub](https://github.com/icsharpcode/ILSpy)¿ Qué ?[ilspycmd on NuGet](https://www.nuget.org/packages/ilspycmd)¿ Qué ?[dnSpyEx on GitHub](https://github.com/dnSpyEx/dnSpy)
- [WebAssembly Core Specification: Binary Format, Modules](https://webassembly.github.io/spec/core/binary/modules.html)y [Values](https://webassembly.github.io/spec/core/binary/values.html)
- [WebAssembly Core Specification: Custom Sections (name section)](https://webassembly.github.io/spec/core/appendix/custom.html)
- [WebAssembly tool-conventions: Debugging](https://github.com/WebAssembly/tool-conventions/blob/main/Debugging.md)
- [wabt 1.0.42 release notes](https://github.com/WebAssembly/wabt/releases/tag/1.0.42)¿ Qué ?[wasm-tools](https://github.com/bytecodealliance/wasm-tools)¿ Qué ?[Binaryen](https://github.com/WebAssembly/binaryen)¿ Qué ?[ghidra-wasm-plugin](https://github.com/nneonneo/ghidra-wasm-plugin)
- [radare2 on GitHub](https://github.com/radareorg/radare2), versión 6.2.4 construido desde la fuente para esta lección
