# Anatomía del programa: un lector ELF desde cero

> Un programa compilado es un conjunto de tablas que apuntan entre sí. Un lector que escribiste da salida que puedes verificar byte por byte.

**Type:** Build
**Languages:** Python (stdlib), C (fixture only)
**Prerequisites:** Phase 0 · 01 (Dev Environment) with clang installed, Python 3.10 or later, hexadecimal arithmetic
**Time:** ~120 minutes

## Objetivos de aprendizaje

- Explique la regla de autorización para cada objetivo de esta pista y los cuatro límites de 17 U.S.C. 1201 ((f).
- Lea el encabezado ELF, la tabla de encabezado de sección y ambas tablas de cuerdas con `struct.unpack`¿ Qué ?
- Descifrar`.symtab`las entradas y `.rela`registros con los nombres de la gABI y la x86-64 psABI.
- Regresa los bytes exactos de una función nombrada y mapa cada archivo desplazado a la estructura que la posee.
- Lea los encabezados del programa y mapee una dirección virtual a un archivo desplazado a través de `PT_LOAD`¿ Qué ?
- Lea los comandos de carga de un objeto Mach-O y nombre la idea ELF que cada uno coincide.

## Antes de comenzar: Autoridad y ley

Cada objetivo en las lecciones 32 a 36 es un programa que usted escribió y compila.`code/fixtures/ledger.c`No hay licencia para leer, no hay protección para pasar, y no hay terceros cuyo código se abre. La pista nunca cubre las protecciones de software, las verificaciones de licencias o los programas que no posee.

La regla existe porque la ley trata el desmontaje como copiar. En *Sega Enterprises Ltd. v. Accolade, Inc.*, 977 F.2d 1510 (9o Cir. 1992), el tribunal dictaminó que las copias intermedias hechas durante el desmontaje son copias en virtud de la Ley de Derechos de Autor. Luego sostuvo que "cuando el desmonte es la única manera de obtener acceso a las ideas y elementos funcionales incorporados en un programa informático protegido por derechos de autor y cuando hay una razón legítima para buscar dicho acceso, el desmonte es un uso justo de la obra protegida por derechos de autor, como cuestión legal". 

El segundo caso es *Sony Computer Entertainment, Inc. v. Connectix Corp. *, 203 F.3d 596 (9th Cir. 2000). Connectix copió el BIOS de PlayStation mientras construía un emulador, y el tribunal consideró que esas copias eran de uso justo. Ambos casos protegieron el estudio. Ninguno protegió el código copiado en el producto final.

Dos estatutos establecen los límites. 17 U.S.C. 1201 ((f) permite a una persona que "ha obtenido legalmente el derecho de utilizar una copia de un programa informático" eludir un control de acceso "con el único propósito de identificar y analizar aquellos elementos del programa que sean necesarios para lograr la interoperabilidad de un programa informático creado de forma independiente con otros programas". Cuatro niveles de limitación: una copia legal, el propósito de interoperabilidad, elementos que no estaban ya "disponibles fácilmente" y análisis que de otro modo no infringen. La sección 1201 ((f) es una excepción a la regla de elusión. La cuestión de la copia se mantiene con uso justo.

En la Unión Europea, la Directiva 2009/24/CE, artículo 5 ((3)), abarca el estudio. Una persona con derecho a usar una copia puede "observar, estudiar o probar el funcionamiento del programa con el fin de determinar las ideas y principios que subyacen a cualquier elemento del programa". El derecho se aplica al cargar, mostrar, ejecutar, transmitir o almacenar el programa.

El artículo 6 permite la descompilación sin la autorización del titular de derechos cuando "es indispensable obtener la información necesaria para lograr la interoperabilidad de un programa informático creado de forma independiente". Las condiciones coinciden con 1201 ((f). El titular de la licencia realiza los actos, la información no estaba disponible y los actos permanecen dentro de las partes necesarias. El artículo 6(2) prohíbe el uso del resultado para cualquier otro objetivo o para un programa que sea sustancialmente similar en expresión. El artículo 8 hace nulas las condiciones contractuales contrarias al artículo 6 o al artículo 5 ((3).

| Source | What it allows | Conditions |
|---|---|---|
| 17 U.S.C. 1201(f) | Circumventing an access control to analyze a program | Lawful copy, interoperability purpose, information not readily available, no infringement |
| Directive 2009/24/EC Art. 5(3) | Observing, studying, and testing a running program | A right to use the copy, acts within normal loading and running |
| Directive 2009/24/EC Art. 6 | Decompilation | Licensee, interoperability purpose, information not readily available, confined to the needed parts |
| *Sega v. Accolade* | Disassembly as fair use | The only way to reach the ideas and functional elements, a legitimate reason |
| *Sony v. Connectix* | Intermediate copies during reverse engineering | A non-infringing final product |

En el artículo 2, apartado 1, del manual de ingeniería inversa 101 se indican las citas completas y los casos del Tribunal de Justicia de la UE.

## El problema

Usted compilado`ledger.c`y corrió .`llvm-objdump -t`En el objeto. Las listas de salida `percent_change`en el valor `0000000000000000`con tamaño `0000000000000010`¿Qué 16 bytes del archivo de 1.768 bytes son esa función?`R_X86_64_PC32 .rodata+0x10`en compensación `0x36`, y los cuatro bytes en ese punto en el archivo son cero. ¿Qué se convertirán, y por qué es el adicion`0x10`Cuando la mesa comienza en `0x14`¿ Qué ?

rea responde a las preguntas de diseño con `inspect_binary_layout`, y esa herramienta sólo existe en Linux x64 con pwntools 4.15.0 detrás `REA_PWNTOOLS_PYTHON`En este Mac no está.`address_to_file_offset`necesita una sesión de Ghidra y una copia completa de los bytes de archivo original con un SHA-256 correspondiente. `inspect_artifact`En el libro de texto se incluye una lista de contenidos en un contenedor por ruta y hash de contenido. Nunca abre las tablas dentro de un archivo objeto. Así que usted mismo escribe el lector en aproximadamente 950 líneas de Python con`struct.unpack`Las pruebas comparan cada tabla con`llvm-objdump`texto capturado una vez en las lámparas.

```figure
elf-byte-map
```

## El archivo es una tabla que apunta a una tabla

El sistema V gABI, capítulo 4, figura 4-1, muestra dos puntos de vista de un archivo. La vista de enlace tiene un encabezado ELF, una tabla de encabezado de programa opcional, secciones y una tabla de encabezado de sección requerida. La vista de ejecución tiene un encabezado ELF, una tabla de encabezado de programa requerida, segmentos y una tabla de encabezado de sección opcional. El gABI añade una frase que rige a todo el lector: "Sólo el encabezado ELF tiene una posición fija en el archivo". Cada otra estructura se encuentra por un offset almacenado en otra estructura.

El gABI también fija los tipos escalares y dice que "ELF no utiliza campos de bits". `Elf64_Addr`y `Elf64_Off`son 8 bytes, `Elf64_Half`es 2, `Elf64_Word`es 4, y `Elf64_Xword`El psABI x86-64, sección 4.1.1, requiere `ELFCLASS64`¿ Qué ?`ELFDATA2LSB`¿ Qué ?`e_machine` `EM_X86_64`(62), y `e_flags`0. medias de pequeñas enedias con anchos fijos `struct.unpack`con un `<`El prefijo lee todos los registros en una sola llamada.

Los primeros 64 bytes de `ledger-x86_64-O1.o`¿Qué es esto ?

```text
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0100 3e00 0100 0000 0000 0000 0000 0000  ..>.............
00000020: 0000 0000 0000 0000 2804 0000 0000 0000  ........(.......
00000030: 0000 0000 4000 0000 0000 4000 0b00 0100  ....@.....@.....
```

Lea las páginas de "ELF Header" de la GABI. `e_ident`es los 16 bytes `7f 45 4c 46`(la magia), clase 2 (64 bits), datos 1 (pequeño-endian), versión 1, OS ABI 0 (Sistema V), y relleno.

Entonces ...`e_type` `0x0001`¿ Es verdad ?`ET_REL`, un objeto relocable.`e_machine` `0x003e`es 62, `EM_X86_64`- ¿ Qué ?`e_entry`es cero porque un objeto no tiene punto de entrada. `e_phoff`es cero porque un objeto no tiene cabeceras de programa. `e_shoff`¿ Es verdad ?`0x428`, el archivo de la tabla de encabezado de sección. La última fila se mantiene `e_ehsize` `0x40`¿ Qué ?`e_phentsize`0 , `e_phnum`0 , `e_shentsize` `0x40`¿ Qué ?`e_shnum` `0x0b`(11 cabeceras), y `e_shstrndx` `0x0001`¿ Qué ?

El lector convierte esa fila en una llamada:

```python
fields = struct.unpack_from("<HHIQQQIHHHHHH", data, 16)
```

La cadena de formato es la figura gABI que se lee de izquierda a derecha: dos semicombinas, una palabra, tres valores de 8 bytes, una palabra y seis semicombinas. El lector rechaza un archivo cuyo orden mágico, clase o byte difiere, y se niega.`e_shentsize`o `e_phentsize`Los valores que no son 64 y 56. un ancho equivocado haría que cada compensación posterior sea incorrecta, por lo que el cheque se encuentra en la parte superior.

## Título de sección y tablas de dos cuerdas

Cada encabezado de sección es un `Elf64_Shdr`de 64 bytes (figura 4-8 de la GABI): `sh_name`¿ Qué ?`sh_type`¿ Qué ?`sh_flags`¿ Qué ?`sh_addr`¿ Qué ?`sh_offset`¿ Qué ?`sh_size`¿ Qué ?`sh_link`¿ Qué ?`sh_info`¿ Qué ?`sh_addralign`, y `sh_entsize`La cadena de formato es `<IIQQQQIIQQ`- ¿ Qué ?`sh_name`es una compensación en la sección cuyo índice es `e_shstrndx`, y el nombre termina en el primer byte cero.`e_shstrndx`es 1, y el clang nombró esa sección `.strtab`, por lo que una tabla sirve tanto los nombres de la sección como los nombres de los símbolos.

El lector imprime los 11 encabezados del objeto -O1:

```text
Idx Name             Type         Flags   Offset     Size       Addr Link Info Align
  0                  NULL                    0x0      0x0        0x0    0    0     0
  1 .strtab          STRTAB                0x368     0xbe        0x0    0    0     1
  2 .text            PROGBITS     AX        0x40     0x9f        0x0    0    0    16
  3 .rela.text       RELA         I        0x2a8     0x48        0x0   10    2     8
  4 .rodata          PROGBITS     A         0xe0     0x50        0x0    0    0     8
  5 .rela.rodata     RELA         I        0x2f0     0x78        0x0   10    4     8
  6 .rodata.str1.1   PROGBITS     AMS      0x130      0xa        0x0    0    0     1
  7 .comment         PROGBITS     MS       0x13a     0x2f        0x0    0    0     1
  8 .note.GNU-stack  PROGBITS              0x169      0x0        0x0    0    0     1
  9 .llvm_addrsig    LLVM_ADDRSIG E        0x368      0x0        0x0   10    0     1
 10 .symtab          SYMTAB                0x170    0x138        0x0    1    9     8
```

El índice 0 es el encabezado nulo reservado. `.text`contiene 159 bytes de código en el archivo de compensación `0x40`, justo después del encabezado, con banderas `A`(allocado en memoria) y `X`(executable). `.rodata`contiene 80 bytes de datos sólo para lectura. `.rodata.str1.1`tiene las cinco cadenas de etiqueta `F\0D\0C\0B\0A\0`, 10 bytes, con banderas `M`(mergible) y `S`¿Qué es eso?

`.comment`tiene el sello de compilador de 47 bytes `\0Apple clang version 17.0.0 (clang-1700.6.3.2)\0`- ¿ Qué ?`.llvm_addrsig`tiene tipo `0x6fff4c03`, dentro del rango específico del sistema `SHT_LOOS`¿ Qué ?`SHT_HIOS`que las reservas de gABI y las de LLVM `ELF.h`Su bandera.`E`¿ Es verdad ?`SHF_EXCLUDE`, así que un enlace lo deja caer.

El `sh_link`y `sh_info`Las columnas son los punteros entre tablas. `.rela.text`¿ Qué ?`sh_info`2 y `sh_link`10: se corre la sección 2 y toma sus símbolos de la sección 10. `.symtab`¿ Qué ?`sh_link`1, la tabla de cuerdas para los nombres de símbolos, y `sh_info`9, el índice del primer símbolo no local. `sh_addr`es cero en cada fila porque un objeto no tiene direcciones todavía.

Dos reglas gABI permiten al lector dibujar un mapa completo del archivo. Las secciones "son contiguas y no se superponen", y un archivo "puede contener espacio inactivo". El lector clasifica cada estructura por offset y llena los agujeros:

```text
     0x0 +64    header   ELF header
    0x40 +159   section  .text
    0xdf +1     gap      inactive space
    0xe0 +80    section  .rodata
   0x130 +10    section  .rodata.str1.1
   0x13a +47    section  .comment
   0x169 +7     gap      inactive space
   0x170 +312   section  .symtab
   0x2a8 +72    section  .rela.text
   0x2f0 +120   section  .rela.rodata
   0x368 +190   section  .strtab
   0x426 +2     gap      inactive space
   0x428 +704   table    section header table
```

Las tres lagunas suman 10 bytes, y cada una es una alineación. `.rodata`¿ Qué ?`sh_addralign`8, así que comienza en `0xe0`en lugar de`0xdf`- ¿ Qué ?`.symtab`tiene alineamiento 8, así que siete bytes no se usan después `.comment`La tabla de encabezado de la sección en sí está alineada con 8.

Secciones de tipo `SHT_NOBITS`No ocupan ningún byte de archivo en absoluto, por lo que el mapa los omite. Nada en este objeto tiene ese tipo.`.bss`en un programa vinculado lo hace, y un lector que confía `sh_size`Se lee más allá del archivo.

## Símbolos: Enlace, Tipo, Sección, Valor, Tamaño

Un `Elf64_Sym`es de 24 bytes (GABI Figura 4-18): `st_name`¿ Qué ?`st_info`¿ Qué ?`st_other`¿ Qué ?`st_shndx`¿ Qué ?`st_value`¿ Qué ?`st_size`, formato `<IBBHQQ`- ¿ Qué ?`st_info`En el paquete se encuentran dos campos: la unión en los cuatro bits superiores y el tipo en los cuatro inferiores.`STB_LOCAL`0 , `STB_GLOBAL`1, `STB_WEAK`2, y los tipos `STT_NOTYPE`0 , `STT_OBJECT`1, `STT_FUNC`2, `STT_SECTION`3, `STT_FILE`Los dos bits más bajos de `st_other`mantener la visibilidad,`STV_DEFAULT`0 aquí por todo.

```text
Idx            Value   Size Bind   Type    Section          Name
  0              0x0      0 LOCAL  NOTYPE  *UND*
  1              0x0      0 LOCAL  FILE    *ABS*            ledger.c
  2              0x0     20 LOCAL  OBJECT  .rodata          labels.rel
  3              0x0      0 LOCAL  SECTION .rodata          .rodata
  4              0x2      2 LOCAL  OBJECT  .rodata.str1.1   .L.str.1
  5              0x4      2 LOCAL  OBJECT  .rodata.str1.1   .L.str.2
  6              0x6      2 LOCAL  OBJECT  .rodata.str1.1   .L.str.3
  7              0x8      2 LOCAL  OBJECT  .rodata.str1.1   .L.str.4
  8              0x0      0 LOCAL  SECTION .rodata.str1.1   .rodata.str1.1
  9              0x0     16 GLOBAL FUNC    .text            percent_change
 10             0x10     46 GLOBAL FUNC    .text            grade
 11             0x40     31 GLOBAL FUNC    .text            total
 12             0x60     63 GLOBAL FUNC    .text            grade_label
```

El índice 0 es el símbolo indefinido reservado. `st_shndx`Nombrará la sección que define el símbolo, con tres valores reservados: `SHN_UNDEF`0 , `SHN_ABS` `0xfff1`para el nombre del archivo, y `SHN_COMMON` `0xfff2`Los símbolos de la sección tienen un nombre vacío, así que `llvm-objdump`Imprime el nombre de la sección para ellos, y el lector hace lo mismo. La regla gABI de que "todos los símbolos con STB_LOCAL prevalecen antes que los símbolos débiles y globales" aparece como las primeras nueve filas, y `.symtab`registrado 9 en `sh_info`Por eso.

En un objeto relocable`st_value`es un desplazamiento dentro de la sección del símbolo, porque `sh_addr`es cero. El archivo de compensación de una función es por lo tanto `sh_offset + st_value - sh_addr`, y su longitud es `st_size`¿Qué es esto ?

```python
section = elf.sections[symbol.st_shndx]
file_offset = section.sh_offset + (symbol.st_value - section.sh_addr)
code = elf.data[file_offset : file_offset + symbol.st_size]
```

Para`percent_change`Eso es .`0x40 + 0`, y los 16 bytes son `85 ff 74 09 29 fe 6b c6 64 99 f7 ff c3 31 c0 c3`Los tamaños importan tanto como los comienzos.`grade`termina en `.text`compensación `0x3e`Los dos bytes .`66 90`que siguen son un `nop`el compilador se añadió para que `total`comienza en`0x40`En la lección 33 se usa un límite de 16 bytes.`st_size`como la condición de parada para un desmonteador, y la lección 36 los enmascara antes de comparar una reconstrucción.

## Relocalizaciones: lo que significan los bytes cero

Un `Elf64_Rela`es de 24 bytes (GABI Figura 4-23): `r_offset`¿ Qué ?`r_info`, y un firmado `r_addend`, formato `<QQq`El GABI se divide .`r_info`con dos macros: `ELF64_R_SYM(i)`¿ Es verdad ?`i >> 32`y `ELF64_R_TYPE(i)`¿ Es verdad ?`i & 0xffffffff`En un archivo relocable`r_offset`es una compensación dentro de la sección nombrada por la tabla `sh_info`Los nombres y fórmulas de los tipos provienen de la x86-64 psABI, Tabla 4.9.`S`es el valor del símbolo, `A`el añadido, `P`el lugar en el que se haga un parche, y `L`la entrada del PLT. `R_X86_64_64`¿ Es verdad ?`S + A`¿ Qué ?`R_X86_64_PC32`¿ Es verdad ?`S + A - P`, y `R_X86_64_PLT32`¿ Es verdad ?`L + A - P`¿ Qué ?

El objeto -O1 tiene ocho registros:

```text
  .rela.text       0x36 R_X86_64_PC32          .rodata+0x10
  .rela.text       0x84 R_X86_64_PC32          .rodata+0x24
  .rela.text       0x93 R_X86_64_PC32          .rodata-0x4
  .rela.rodata      0x0 R_X86_64_PC32          .rodata.str1.1
  .rela.rodata      0x4 R_X86_64_PC32          .L.str.1+0x4
  .rela.rodata      0x8 R_X86_64_PC32          .L.str.2+0x8
  .rela.rodata      0xc R_X86_64_PC32          .L.str.3+0xc
  .rela.rodata     0x10 R_X86_64_PC32          .L.str.4+0x10
```

Despeje el primero.`0x36`Está dentro .`grade`, que se extiende `0x10`¿ Qué ?`0x3e`La instrucción en`0x33`¿ Es verdad ?`48 8d 0d 00 00 00 00`, una `lea`de una dirección relativa al RIP, y su desplazamiento de cuatro bytes se encuentra en `0x36`. El procesador agrega el desplazamiento a la dirección de la siguiente instrucción, `0x3a`¿ Qué ?

El compilador quiere que el resultado sea `.rodata + 0x14`, donde una tabla de cinco entradas de 32 bits `1, 2, 3, 4, 4`Así que el valor almacenado debe ser`(.rodata + 0x14) - 0x3a`La fórmula`S + A - P`con`P = 0x36`da .`.rodata + A - 0x36`. Establecer los dos resultados iguales .`A = 0x14 - 4 = 0x10`El complemento contiene el desplazamiento objetivo menos los cuatro bytes entre el desplazamiento y el final de la instrucción.

Esa tabla es cómo Clang compilado el `switch`En el`grade`En -O1. El código calcula`score / 10`con un multiplicador por `0x66666667`y dos cambios, restar 6, compara con 4, e indexa la tabla. El objeto -O1 ELF no conserva ningún símbolo para la tabla. La referencia es el símbolo de sección `.rodata`Además de un aditivo, y sólo la construcción de arm64 de la misma fuente lo nombra `l_switch.table.grade`El segundo disco, en`0x84`En el`grade_label`, puntos en una segunda tabla de valores de 8 bytes en `.rodata + 0x28`El tercero, en`0x93`, puntos en `.rodata + 0x0`, el comienzo de `labels.rel`¿ Qué ?

Los cinco .`.rela.rodata`llenar los registros `labels.rel`Cada entrada está en el descenso .`4i`y obtiene el complemento.`4i`, así que`S + A - P`se reduce a la dirección de la cadena menos la base de la tabla. `grade_label`se lee una entrada con `movslq`, añade la base de la tabla de nuevo, y tiene el puntero. El primer registro nombra el símbolo de la sección`.rodata.str1.1`con aditivo 0 porque la cadena `F`Es el primer byte de esa sección.`.L.str.1`¿ Qué ?`.L.str.4`, las etiquetas locales del compilador para `D`¿ Qué ?`C`¿ Qué ?`B`, y `A`¿ Qué ?

La misma fuente compiló a tres tablas diferentes en tres niveles de optimización. El lector muestra la diferencia sin un desmonteador:

| Build | Where `labels` lives | How `grade` selects | How `grade_label` reaches `grade` |
|---|---|---|---|
| -O0 | `.data`, five 8-byte pointers, five `R_X86_64_64` records | A jump table in `.rodata`: five `R_X86_64_PC32` records pointing into `.text` | A call with `R_X86_64_PLT32 grade-0x4` |
| -O1 | `.rodata` as `labels.rel`, five 32-bit relative entries | A value table at `.rodata+0x14`, one `R_X86_64_PC32` | Inlined, no call |
| -O2 | Same as -O1 | Same as -O1 | Inlined, `total` grows to 140 bytes |

En -0 el `switch`Es una verdadera mesa de saltos cuyas cinco entradas apuntan hacia atrás en`.text`En el`+0x91`¿ Qué ?`+0x8c`¿ Qué ?`+0x87`¿ Qué ?`+0x82`, y `+0x86`La llamada a`grade`lleva el mismo `-0x4`Addición por el mismo motivo que la `lea`En -O1 el optimizador vio que cada caso devuelve una constante y reemplazó los saltos con una tabla de resultados.

## Los encabezados del programa: el punto de vista del cargador

Un objeto relocable no tiene cabeceras de programa, así que `e_phnum`Es un ejecutable o un objeto compartido los tiene, y el cargador no lee nada más.`Elf64_Phdr`es 56 bytes (figura 5-1 de la ABGG): `p_type`¿ Qué ?`p_flags`¿ Qué ?`p_offset`¿ Qué ?`p_vaddr`¿ Qué ?`p_paddr`¿ Qué ?`p_filesz`¿ Qué ?`p_memsz`¿ Qué ?`p_align`, formato `<IIQQQQQQ`El GABI fija tres reglas para`PT_LOAD`Aparecen en ascendente.`p_vaddr`orden, y un `p_memsz`más grande que `p_filesz`significa cero llenado.`p_vaddr`es congruente con `p_offset`modulo `p_align`¿ Qué ?

Cada objetivo en esta pista es un programa que has construido, y ningún enlace en este Mac produce un ejecutable de Linux desde `ledger.o`Así que el dispositivo`tiny-x86_64`proviene de`code/fixtures/build_tiny_elf.py`, que escribe 736 bytes con `struct.pack`El archivo contiene un encabezado ELF, dos cabeceras de programa, un 36 bytes.`_start`, un 3 bytes `.rodata`, una tabla de símbolos y seis encabezados de sección. `_start`escribe `ok\n`En Linux se ejecuta x86-64. Aquí es un fijo para el camino de encabezado del programa.

```text
Program headers:
  PT_LOAD          r-x offset=0x0 vaddr=0x400000 filesz=0x2e0 memsz=0x2e0 align=0x1000
  PT_GNU_STACK     rw- offset=0x0 vaddr=0x0 filesz=0x0 memsz=0x0 align=0x10

entry 0x4000b0 is file offset 0xb0 through PT_LOAD
```

El único .`PT_LOAD`Mapa todo el archivo de 736 bytes en `0x400000`El lector asigna una dirección a un archivo desplazado encontrando el `PT_LOAD`que lo contiene y la computación `p_offset + (address - p_vaddr)`. para el punto de entrada que es `0 + (0x4000b0 - 0x400000) = 0xb0`, el comienzo de `.text`¿ Qué ?

Este es el cálculo detrás de rea.`address_to_file_offset`Para un objetivo de Ghidra, esa herramienta regresa.`{address, file_offset}`El proveedor de Hopper también regresa .`provider_file_offset`y `image_base_file_offset`La herramienta necesita una copia original completa del archivo, y el SHA-256 de esa copia debe coincidir con el objetivo.`inspect_binary_layout`añade una línea para mantener: "Una dirección vinculada no es una dirección de tiempo de ejecución". Una carga ejecutable independiente de posición en una base que el kernel elija, y el desvio de archivo es la única coordenada estable.

Los encabezados del programa también explican por qué un ejecutable despojado sigue funcionando. Un enlace puede eliminar la tabla de encabezado de sección por completo, y `e_shoff`Una herramienta que camina por secciones no ve nada.

## Las mismas ideas en Mach-O

Apple no publica ninguna especificación de prosa para Mach-O. Los encabezados en la fuente XNU son las especificaciones: `EXTERNAL_HEADERS/mach-o/loader.h`¿ Qué ?`nlist.h`, y `reloc.h`- El dispositivo .`ledger-arm64-O1.o`Es lo mismo.`ledger.c`construido con `clang -target arm64-apple-macos -O1 -fno-asynchronous-unwind-tables -c`¿ Qué ?

¿ Qué es esto ?`mach_header_64`es 32 bytes: `magic`¿ Qué ?`cputype`¿ Qué ?`cpusubtype`¿ Qué ?`filetype`¿ Qué ?`ncmds`¿ Qué ?`sizeofcmds`¿ Qué ?`flags`¿ Qué ?`reserved`La magia es`0xfeedfacf`¿ Qué ?`MH_MAGIC_64`- ¿ Qué ?`cputype`¿ Es verdad ?`0x0100000c`¿ Qué ?`CPU_TYPE_ARM64`- ¿ Qué ?`filetype`es 1, `MH_OBJECT`- ¿ Qué ?`ncmds`es 5 y `sizeofcmds`es 616: los comandos de carga ocupan los 616 bytes después del encabezado, y cada uno comienza con `{cmd, cmdsize}`La bandera .`0x2000`¿ Es verdad ?`MH_SUBSECTIONS_VIA_SYMBOLS`¿ Qué ?

```text
Load commands:
  0 LC_SEGMENT_64                cmdsize=472 at 0x20
  1 LC_BUILD_VERSION             cmdsize=24 at 0x1f8
  2 LC_LINKER_OPTIMIZATION_HINT  cmdsize=16 at 0x210
  3 LC_SYMTAB                    cmdsize=24 at 0x220
  4 LC_DYSYMTAB                  cmdsize=80 at 0x238
```

`LC_SEGMENT_64`(0x19) es la respuesta Mach-O a ambas tablas ELF.`segment_command_64`¿ Qué ?`segname`¿ Qué ?`vmaddr`¿ Qué ?`vmsize`¿ Qué ?`fileoff`¿ Qué ?`filesize`¿ Qué ?`maxprot`¿ Qué ?`initprot`¿ Qué ?`nsects`, y `flags`Entonces ...`nsects`registros de tipo `section_64`, 80 bytes cada uno, seguir en el mismo comando.`loader.h`comentar sobre `MH_OBJECT`dice que las secciones viven "en un segmento sin nombre sin relleno de segmento".`fileoff`648, `filesize`456, y cinco secciones.

```text
  Idx Section                    Addr   Size Offset Reloff Nreloc Type
    1 __TEXT,__text               0x0   0xd0    648   1104      6 S_REGULAR
    2 __DATA,__const             0xd0   0x28    856   1152      5 S_REGULAR
    3 __TEXT,__cstring           0xf8    0xa    896      0      0 S_CSTRING_LITERALS
    4 __TEXT,__const            0x108   0x40    912      0      0 S_REGULAR
    5 __LD,__compact_unwind     0x148   0x80    976   1192      4 S_REGULAR
```

Cada sección nombra su segmento principal en `segname`, así que dos secciones llamadas`__const`son distintos porque uno es`__DATA,__const`y el otro .`__TEXT,__const`Las mudanzas son por sección hasta la`reloff`y `nreloc`, donde ELF tiene un separado `.rela.X`sección por objetivo.

Cada uno .`relocation_info`es 8 bytes. Un firmado `r_address`Se sigue un campo de 32 bits, lleno como`r_symbolnum`(de 24 bits), `r_pcrel`(1) `r_length`(2) `r_extern`(1) y `r_type`(4) Los seis `__text`Los registros son tres pares de `ARM64_RELOC_PAGE21`y `ARM64_RELOC_PAGEOFF12`, un par por mesa de referencia.`adrp`Además`add`¿Qué hace uno x86-64 `lea`Si, entonces cada referencia necesita dos registros.`__DATA,__const`los registros son `ARM64_RELOC_UNSIGNED`con`r_length`3, así que cada uno corre un puntero absoluto de 8 bytes.

En -O1 la construcción del arm64 se mantuvo `labels`como punteros simples donde la construcción x86-64 hizo una tabla relativa.

`LC_SYMTAB`(0x2) se mantiene `symoff`1240 , `nsyms`¿ Qué pasa?`stroff`1512, `strsize`176 y 1512 más 176 es 1688, el tamaño del archivo.`nlist_64`es de 16 bytes: `n_strx`¿ Qué ?`n_type`¿ Qué ?`n_sect`¿ Qué ?`n_desc`¿ Qué ?`n_value`- ¿ Qué ?`n_type`Es un poco de campo.`N_TYPE`(máscara)`0x0e`) igual a `N_SECT`(El artículo`0xe`) se entiende por el definido en la sección cuya ordinal de base 1 es `n_sect`- ¿ Qué ?`N_EXT`(El artículo`0x01`) marca un símbolo externo.

`nlist_64`No tiene campo de tamaño. para obtener los 24 bytes de`_percent_change`el lector toma la siguiente dirección de símbolo en la misma sección, `_grade`En el`0x18`Los símbolos C llevan un liderazgo`_`aquí, así que la función es `_percent_change`¿ Qué ?

`LC_DYSYMTAB`(0xb) registra la misma regla local-primera que la ELF `sh_info`¿ Qué es esto ?`ilocalsym`0 , `nlocalsym`13, `iextdefsym`13, `nextdefsym`4 .`LC_BUILD_VERSION`(0x32) registros `platform`1 (macOS), `minos`26.0 y `sdk`26.2 como empaquetado `XXXX.YY.ZZ`Es por eso que el arm64 fijo se compromete como bytes. Una construcción en un Mac con otro SDK produce un archivo diferente.

El lector es `compare_formats`pone los dos archivos uno al lado del otro:

| Idea | ELF, `ledger-x86_64-O1.o` | Mach-O, `ledger-arm64-O1.o` |
|---|---|---|
| File identity | `e_ident` `7f 45 4c 46`, class 64, data LSB | `magic` `0xfeedfacf` |
| Machine | `e_machine` 62, `EM_X86_64` | `cputype` `0x100000c`, `CPU_TYPE_ARM64` |
| File kind | `e_type` 1, `ET_REL` | `filetype` 1, `MH_OBJECT` |
| Where the tables are | `e_shoff` `0x428`, 11 section headers | 5 load commands, 616 bytes after the header |
| The code | `.text` at `0x40`, size `0x9f`, flags `AX` | `__TEXT,__text` at 648, size `0xd0`, in the unnamed segment |
| Symbol names | `.symtab` (section 10), names in `.strtab` through `sh_link` 1 | `LC_SYMTAB` `symoff` 1240, `nsyms` 17, names at `stroff` 1512 |
| Symbol size | `st_size`: `percent_change` is 16 bytes | No field: next symbol minus this one, 24 bytes |
| Relocations | `.rela.text`, 3 records, `R_X86_64_PC32` | `__text` `reloff` 1104, 6 records, `PAGE21` and `PAGEOFF12` |
| Function bytes | `percent_change`: 16 bytes at `0x40` | `_percent_change`: 24 bytes at `0x288` |

Rea lee Mach-O a través de `inspect_macho`, que se ejecuta `otool -h -l`¿ Qué ?`nm -gjU`¿ Qué ?`dyld_info -imports -exports`, y `dwarfdump --uuid`por el`xcrun`Su lector no necesita ninguno de ellos, y lee lo mismo.`symoff`y `stroff`Eso es .`otool -l`Las huellas.`open_binary`La herramienta de rea detecta ELF, MZ, PE y Mach-O delgado o universal de los bytes mágicos.`e_machine`Es el primer 20 bytes de lo que construyó aquí. para un archivo sin encabezado, usted da`format: raw-binary`El proveedor de Ghidra rechaza un binario universal y pide una rebanada.`0xcafebabe`¿ Qué ?

## Construye el mismo

Ejecutar el lector de biblioteca estándar:

```bash
cd phases/13-tools-and-protocols/32-program-anatomy-elf-from-scratch
python3 code/main.py
```

La demostración dice`ledger-x86_64-O1.o`y imprime su encabezado, secciones, símbolos, reubicaciones y mapa de desplazamiento de archivos. Luego imprime las cuatro cadenas de byte de función, lee `ledger-arm64-O1.o`, y imprime la tabla de comparación.`tiny-x86_64`- Las líneas esperadas:

```text
percent_change: 16 bytes at file offset 0x40: 85 ff 74 09 29 fe 6b c6 64 99 f7 ff c3 31 c0 c3
grade: 46 bytes at file offset 0x50: 48 63 c7 48 69 c0 67 66 66 66 ...
entry 0x4000b0 is file offset 0xb0 through PT_LOAD
```

Pasar un camino para leer un archivo: `python3 code/main.py code/fixtures/ledger-x86_64-O0.o`El lector elige ELF o Mach-O de los primeros cuatro bytes.

Leer .`code/main.py`en este orden:

1. `ElfFile._read_header()`verifica la magia, clase, orden de byte y tamaños de entrada, luego desempaca el encabezado.
2. `ElfFile._read_sections()`y `read_cstring()`Lea la tabla de encabezado de sección y resuelva los nombres a través de `e_shstrndx`¿ Qué ?
3. `ElfFile._read_symbols()`Se divide`st_info`, resuelve nombres a través de `sh_link`, y mantiene legibles los índices de las secciones reservadas.
4. `ElfFile._read_relocations()`camina cada vez `SHT_RELA`sección, división `r_info`, y los tipos de nombres de la tabla 4.9.
5. `ElfFile.function_span()`y `function_bytes()`convertir un símbolo en un rango de archivos y los bytes en él.
6. `ElfFile.file_offset_map()`clasifica cada estructura por compensación y llena el espacio inactivo.
7. `ElfFile._read_program_headers()`y `address_to_file_offset()`manejar ejecutables.
8. `MachOFile`lee el encabezado, las órdenes de carga,`LC_SEGMENT_64`con sus secciones, `LC_SYMTAB`¿ Qué ?`LC_BUILD_VERSION`, y las reubicaciones por sección.
9. `compare_formats()`nombres de la misma idea en ambos archivos con los valores leídos de cada uno.

## Usalo

Importa el lector de las lecciones posteriores con `from main import ElfFile`Después de añadir la lección `code`directorio de `sys.path`La lección 33 lo pide .`function_bytes`y `relocations_for(".text")`La lección 35 registra su salida como evidencia observada. La lección 36 le pide que los desplazamientos de desplazamiento se masquen antes de una comparación en byte.

Compile su propio C en un objeto con el mismo `clang -target x86_64-unknown-linux-gnu -c`Cuando el lector encuentra un valor que no tiene nombre, imprime el número en hexadecimal en lugar de adivinar.

## Laboratorio interactivo

### Laboratorio A: el encabezado a mano

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/32-program-anatomy-elf-from-scratch/code
python3 -q
```

- ¿Qué quieres decir ?

```python
import struct
from main import ElfFile, FIXTURES
data = (FIXTURES / "ledger-x86_64-O1.o").read_bytes()
data[:4], data[4], data[5]
struct.unpack_from("<HHIQQQIHHHHHH", data, 16)
elf = ElfFile(data)
elf.header.e_shoff, elf.header.e_shnum, elf.sections[elf.header.e_shstrndx].name
```

Los tuplos muestran`e_type`1, `e_machine`62, y `e_shoff`1064, que es `0x428`La última línea muestra que la sección 1,`.strtab`, tiene los nombres de las secciones.

### Laboratorio B: la tabla de valores detrás del interruptor

```python
rodata = elf.section_bytes(elf.section(".rodata"))
[int.from_bytes(rodata[i : i + 4], "little") for i in range(0x14, 0x28, 4)]
[int.from_bytes(rodata[i : i + 8], "little") for i in range(0x28, 0x50, 8)]
[r for r in elf.relocations_for(".text") if r.symbol == ".rodata"]
```

Las dos tablas se leen `1, 2, 3, 4, 4`- Controla con ellos .`ledger.c`¿ Qué es esto ?`grade(65)`computaciones`65 / 10 - 6 = 0`y se lee la entrada 0, que es 1, el valor de `case 6`- ¿ Qué ?`grade(100)`se lee la entrada 4, que es 4, el valor compartido por `case 10`y `case 9`¿ Qué ?

### Laboratorio C: tres construcciones de una fuente

```python
for level in ("O0", "O1", "O2"):
    obj = ElfFile.from_path(FIXTURES / f"ledger-x86_64-{level}.o")
    print(level, [s.name for s in obj.sections if s.name in (".data", ".rodata")])
    print(level, sorted({r.type_name for r in obj.relocations}))
    print(level, {s.name: s.st_size for s in obj.symbols if s.type == "FUNC"})
```

El objeto -O0 tiene un `.data`Los objetos -O1 y -O2 no tienen`.data`y un tipo.`total`es 66 bytes en -O0, 31 en -O1, y 140 en -O2.

### Laboratorio D: la misma función en Mach-O

```python
from main import MachOFile
macho = MachOFile.from_path(FIXTURES / "ledger-arm64-O1.o")
[(c.name, c.cmdsize) for c in macho.load_commands]
macho.function_span("_percent_change")
macho.function_bytes("_percent_change")[:4].hex()
```

El espacio es de 24 bytes en el archivo defset 648. Los primeros cuatro bytes son `a0000034`, la forma de instrucción de pequeño endio .`0x340000a0`, que `llvm-objdump -d`Imágenes como `cbz w0, 0x14`Eso es lo que .`before == 0`guard, la misma prueba que la construcción x86-64 codifica como `85 ff 74 09`¿ Qué ?

## Laboratorio de práctica

Extenda el lector a un programa vinculado. En una máquina x86-64 de Linux, construye `clang -O1 -shared -fPIC code/fixtures/ledger.c -o libledger.so`y `clang -O1 code/fixtures/ledger.c -o ledger_main`con un `main`añade, luego ejecuta el lector en ambos.

Requisitos:

- Leer .`.dynsym`con el mismo `Elf64_Sym`el diseño, desde `SHT_DYNSYM`es 11 y el registro es inalterado.
- Leer .`.dynamic`¿ Cómo ?`Elf64_Dyn`registros de `d_tag`y `d_val`, y nombre `DT_NEEDED`¿ Qué ?`DT_STRTAB`¿ Qué ?`DT_SYMTAB`¿ Qué ?`DT_RELA`, y `DT_JMPREL`de la página de la sección "Dynamic Section" de la gABI.
- Enumera los símbolos indefinidos con `st_shndx` `SHN_UNDEF`En el caso de las importaciones, el`R_X86_64_JUMP_SLOT`registros en `.rela.plt`como los sitios de llamadas que los alcanzan.
- Mapa de cada uno `PT_LOAD`segmento a las secciones que contiene, utilizando la regla gABI que "un segmento de archivo objeto contiene una o más secciones".
- Prueba con `address_to_file_offset`que los bytes de cada función definida en el objeto compartido son los bytes `function_bytes`regresa.
- Captura .`llvm-objdump -h -t -r -p`y `llvm-objdump -T`texto para ambos archivos en `code/fixtures/`y comparar al lector con ellos en pruebas.

El éxito significa que la lista de importación, el mapa de segmento y cada rango de funciones coinciden con la salida de la herramienta capturada, sin que la herramienta se ejecute durante las pruebas.

## Laboratorio de calificación

Cinco preguntas sobre los accesorios de esta lección.`code/lab_answers.json`Escribir números como números o como cuerdas hexáticas como `"0x40"`, y listas como listas JSON.

1. `t1`: e_shoff del libro mayor-x86_64-O1.o, como número.
2. `t2`: st_size de porcentaje_cambio en el libro mayor-x86_64-O1.o.
3. `t3`: el archivo de .text en el libro mayor-x86_64-O1.o, como hex.
4. `t4`: nombres de tipos de reubicación en el libro mayor-x86_64-O0.o, ordenados, como una lista.
5. `t5`: los cinco valores int en .rodata+0x14 en el libro mayor-x86_64-O1.o, como una lista.

Compruebe sus respuestas:

```bash
cd phases/13-tools-and-protocols/32-program-anatomy-elf-from-scratch
python3 code/lab.py
```

El verificador calcula cada valor esperado desde `code/main.py`En los accesorios comprometidos.`PASS`¿ Qué ?`FAIL`o `TODO`para cada tarea y una puntuación, y sale de 0 sólo cuando pasan los cinco. `python3 code/lab.py my-answers.json`- ¿Qué? - Vea otro archivo.`code/tests/test_lab.py`prueba que el comprobador acepta las respuestas correctas y rechaza las incorrectas.

### Ve cómo se ejecuta

El [Inside a Decompiler](https://aiengineeringfromscratch.com/inside-a-decompiler)La máquina ejecuta esta lección como sus dos primeras estaciones. El carretillo lee el encabezado del archivo objeto que contiene `percent_change`, y el gabinete se abre .`.text`Desactivar los símbolos para ver qué datos lleva la tabla de símbolos.

## Lección Artefacto

Esta lección produce`outputs/skill-program-anatomy-elf-from-scratch.md`. Use con Claude, Cursor, Codex, OpenClaw, Hermes o cualquier agente que lea SKILL.md. La habilidad convierte un objeto compilado o ejecutable que el usuario construyó en un informe de diseño. El informe da el rango de archivos de cada estructura, símbolo y reubicación, verificado contra la salida de la herramienta capturada.

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/32-program-anatomy-elf-from-scratch
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

La suite prueba:

- Cada sección de índice, nombre, tamaño, dirección y tipo coincide con la capturada `llvm-objdump -h`texto para los tres objetos x86-64 y el ejecutable pequeño
- Cada valor de símbolo, vinculación, tipo, sección, tamaño y nombre coincide con lo capturado `llvm-objdump -t`texto
- Cada desplazamiento de compensación, nombre de tipo, símbolo y adendo coincide con lo capturado.`llvm-objdump -r`texto, incluido `R_X86_64_PLT32`y `R_X86_64_64`en -O0
- Los dos directores de programas de `tiny-x86_64`Encuentra a los capturados .`llvm-objdump -p`texto, y los objetos no tienen ninguna
- `percent_change`es 16 bytes en el archivo defset `0x40`con los bytes esperados, y cada función se encuentra dentro `.text`
- El mapa de compensación de archivos cubre cada byte de cada fijación una vez, con los tres huecos de alineación del objeto -O1 en sus lugares
- el punto de entrada de `tiny-x86_64`mapas para el archivo de compensación `0xb0`por el`PT_LOAD`, y un objeto relocable rechaza el mapeo de direcciones
- El encabezado Mach-O, cada comando de carga, el segmento, las cinco secciones,`LC_SYMTAB`, y `LC_BUILD_VERSION`Encuentra a los capturados .`--private-headers`texto
- Los símbolos de Mach-O y las 15 reubicaciones coinciden con los capturados.`llvm-objdump`el texto, y `_percent_change`es 24 bytes en el desplazamiento 648
- una magia equivocada, una clase de 32 bits, un archivo de grandes endianos, una tabla truncada, un binario universal y un archivo fuente C todos aumentan.`FormatError`
- `build_tiny_elf.py`reproducir los compromisos `tiny-x86_64`byte por byte
- el módulo de figuras `site/figures-re-32.js`lleva los bytes exactos de `ledger-x86_64-O1.o`

## Modos de falla de producción

| Failure | What the weak reading reports | What the reader must prove |
|---|---|---|
| `st_value` used as a file offset | "`percent_change` is at 0" | Offset is `sh_offset + st_value - sh_addr`, here `0x40` |
| Padding counted as code | "`grade` is 48 bytes" | `st_size` is 46 and the `66 90` after it belongs to no symbol |
| Section names read from the wrong table | Garbage names or an exception | `e_shstrndx` names the table, here section 1 |
| `sh_size` trusted for `SHT_NOBITS` | A read past the end of the file | `.bss` has no file bytes and the map skips it |
| Zero bytes read as the operand | "`lea 0(%rip)`" | The `.rela.text` record at `0x36` names `.rodata+0x10` |
| Addend read as the target offset | "the table starts at `0x10`" | `S + A - P` with the instruction end gives `0x14` |
| Sections walked on a stripped executable | "the file has no code" | `PT_LOAD` maps every address the loader uses |
| A linked address treated as a runtime address | A wrong offset under a random base | The file offset from `PT_LOAD` is the stable coordinate |
| A Mach-O section looked up by `sectname` alone | Two `__const` sections collapse into one | `segname,sectname` is the name |
| A size field expected in `nlist_64` | An exception or zero bytes | The next symbol in the same section ends the function |
| A universal binary read as one Mach-O | "magic `0xcafebabe` is unknown" | Refuse it and ask for a thinned slice |
| The arm64 object rebuilt on another Mac | "the fixture changed" | `LC_BUILD_VERSION` records the SDK, so the committed bytes are the fixture |

## Regla de funcionamiento

Lea el encabezado en el que puede confiar, siga los desreglados que contiene y nombre cada campo de la especificación que lo define. Mantenga la salida de la herramienta que comparó como un fijo comprometido, por lo que la comparación se ejecuta sin la herramienta. Cuando un valor no tiene nombre en la especificación, imprima el número. El lector que puede comprobar byte por byte es el que cada lección posterior se basa en.

## Leer más

- [System V gABI, chapter 4: Object Files](https://www.sco.com/developers/gabi/latest/ch4.intro.html)y el [ELF Header](https://www.sco.com/developers/gabi/latest/ch4.eheader.html)¿ Qué ?[Sections](https://www.sco.com/developers/gabi/latest/ch4.sheader.html)¿ Qué ?[Symbol Table](https://www.sco.com/developers/gabi/latest/ch4.symtab.html), y [Relocation](https://www.sco.com/developers/gabi/latest/ch4.reloc.html)páginas
- [System V gABI, chapter 5: Program Header](https://www.sco.com/developers/gabi/latest/ch5.pheader.html)y [Dynamic Linking](https://www.sco.com/developers/gabi/latest/ch5.dynamic.html)
- [System V ABI, AMD64 Architecture Processor Supplement](https://gitlab.com/x86-psABIs/x86-64-ABI), Cuadro 4.9 Tipos de traslado
- [Apple XNU `loader.h`](https://github.com/apple-oss-distributions/xnu/blob/main/EXTERNAL_HEADERS/mach-o/loader.h)¿ Qué ?[`nlist.h`](https://github.com/apple-oss-distributions/xnu/blob/main/EXTERNAL_HEADERS/mach-o/nlist.h), y [`reloc.h`](https://github.com/apple-oss-distributions/xnu/blob/main/EXTERNAL_HEADERS/mach-o/reloc.h)
- [17 U.S.C. 1201](https://uscode.house.gov/view.xhtml?req=granuleid:USC-prelim-title17-section1201&num=0&edition=prelim)y [Directive 2009/24/EC](https://eur-lex.europa.eu/eli/dir/2009/24/oj/eng)
- [Sega v. Accolade, 977 F.2d 1510](https://law.resource.org/pub/us/case/reporter/F2/977/977.F2d.1510.92-15655.html)y [Sony v. Connectix, 203 F.3d 596](https://law.resource.org/pub/us/case/reporter/F3/203/203.F3d.596.99-15852.html)
- [rea 6.5.0 `docs/binary-diagnostics.md`](https://github.com/morluto/rea/blob/rea-agents-6.5.0/docs/binary-diagnostics.md)y [`docs/ghidra-provider.md`](https://github.com/morluto/rea/blob/rea-agents-6.5.0/docs/ghidra-provider.md)
