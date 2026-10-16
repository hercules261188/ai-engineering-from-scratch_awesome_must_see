# Enlaces, cargas y binarios desnudos: PLT, GOT, importaciones, fijaciones e identificación de funciones

> Un programa vinculado llama a funciones que no contiene. Esta lección lee las tablas que el editor de enlaces escribió para esas llamadas en tres formatos, sigue una llamada a la ranura que llena el cargador y encuentra las funciones de una construcción después de que se hayan eliminado sus nombres.

**Type:** Build
**Languages:** Python, C, x86-64 and arm64 assembly
**Prerequisites:** Phase 13 · 32 (ELF from scratch), 33 (disassembly and control flow), 38 (calling conventions)
**Time:** ~160 minutes

## Objetivos de aprendizaje

- Decida de un registro de reubicación y de las tablas de símbolos lo que el editor de enlaces haga de él: una distancia fija, una entrada PLT con un espacio GOT o una reubicación dinámica.
- Lea la sección dinámica, el PLT y el GOT de un ejecutable ELF independiente de posición y siga una llamada a través de ellos con las fórmulas psABI.
- Lea un objeto COFF (20 bytes de encabezado de archivo, entradas de sección de 40 bytes, símbolos de 18 bytes, relocalizaciones de 10 bytes) y los directorios de importación, exportación y relocalización de base de una imagen PE32+.
- Descifrar una cadena de corrección en cadena Mach-O palabra por palabra y combinar cada palabra contra`dyld_info`¿ Qué ?
- ¿ Qué es lo que dice ?`strip`elimina y lo que sobrevive en cada formato, y ejecuta el programa despojado para demostrar que todavía funciona.
- Recupera la función inicia de una construcción de arm64 despojada con `LC_FUNCTION_STARTS`, explora el prólogo y llama a objetivos, marca cada método en la tabla de símbolos del gemelo sin cortar, y nombra las funciones con una firma en forma de FLIRT.

## El problema

Usted se conectó`main.c`contra`libledger.dylib`Y preguntó a un agente cuál es su función.`main`llamadas en`0x1000005a0`El agente desmontó la instrucción y leyó .`bl 0x100000694`Esa dirección contiene doce bytes:`adrp x16`¿ Qué ?`ldr x16, [x16, #0x38]`¿ Qué ?`br x16`El agente siguió la carga hasta ...`0x100004038`, lee los ocho bytes `00 00 00 00 00 00 10 80`, y reportó una llamada a través de un puntero corrupto en el espacio del núcleo.

Luego despojaste de la construcción estática del mismo programa y preguntaste cuántas funciones tiene.`nm`, vio un nombre,`__mh_execute_header`, y respondió uno.

Ambas respuestas leen el archivo como si fuera el proceso de ejecución.`0x100004038`Bit 63 dice "bind", 12 bits dicen dónde está la próxima fijación, y los bits bajos dicen qué importar para unir.`_puts`antes de`main`El archivo despojado perdió su tabla de símbolos y se mantuvo`LC_FUNCTION_STARTS`, que enumera las cinco funciones comienza en nueve bytes.

Esta lección construye cinco lectores en Python de biblioteca estándar. Un lector COFF para nuestros propios objetos y un lector PE32+ para los directorios de importación, exportación y reubicación de base. Un lector Mach-O para comandos de carga, estubes, la tabla de símbolos indirectos, la cadena de fijación en cadena y el trie de exportación, verificado palabra por palabra contra `dyld_info`. Un lector ELF para la sección dinámica, el PLT y el GOT, más un planificador que dice qué crearía un enlace PIE de nuestros objetos. Un buscador de funciones para una construcción despojada con cuatro métodos y una puntuación para cada uno. Un constructor de firma en forma de FLIRT, con bytes de reubicación con wildcard y un CRC16 de la cola.

Sólo Mach-O enlaces en esta máquina. los ejecutables arm64 y el dylib se conectaron, desnudos, y ejecutados. los objetos ELF y COFF compilado aquí con`clang -target`, y no se instala ningún editor de enlace ELF o PE, por lo que las dos imágenes vinculadas en esos formatos fueron escritas byte por byte a partir de las especificaciones por dos scripts Python en la carpeta de fijos. `llvm-objdump`No se han ejecutado, y el texto dice así donde importa.

```figure
re-40-call-through-three-formats
```

## Lo que hace el editor de enlaces con una reubicación

Lección 32 leer registros de reubicación y la lección 33 aplicada `S + A - P`a una `lea`Un registro nombra un lugar .`P`, un tipo, un símbolo `S`, y un añadido `A`. La tabla 4.9 de la x86-64 psABI da una fórmula por tipo, con `L`para el lugar de la entrada PLT del símbolo, `G`por su compensación de GOT,`GOT`para la dirección GOT, y `B`Tres resultados cubren nuestros objetos.

El editor de enlace escribe el valor en el campo y el registro se ha ido. `R_X86_64_PC32`En el caso de las imágenes, la distancia entre dos bytes de una imagen nunca cambia, sea cual sea la dirección de carga.

El editor de enlaces hace una entrada PLT y una ranura GOT. `R_X86_64_PLT32`En este caso, la llamada se hace`L + A - P`La entrada salta a través de la ranura.`R_X86_64_JUMP_SLOT`registro en `.rela.plt`dice al enlace dinámico que llene el espacio.

El editor de enlaces mantiene una reubicación para el cargador. `R_X86_64_64`es una dirección absoluta de ocho bytes, y un ejecutable independiente de posición no sabe su dirección hasta que se carga.`R_X86_64_RELATIVE`con fórmula `B + A`En el`.rela.dyn`¿ Qué ?

`main-x86_64-O1.o`tiene 18 registros: 14 en `.text`, 2 en .`.data.rel.ro`, y 2 en `.rodata`- ¿ Qué ?`plan_dynamic_link()`En el`code/main.py`aplica las tres reglas a ellos junto con `ledger-x86_64-O1.o`¿Qué es esto ?

| Record | Count | Decision | Why |
|---|---|---|---|
| `R_X86_64_PC32` to `.rodata`, `.L.str`, `fns`, the table in `.rodata` | 16 | in place | the symbol is defined in the inputs |
| `R_X86_64_PLT32` to `percent_change`, `grade`, `total`, `grade_label` | 4 | direct, no PLT | defined in `ledger-x86_64-O1.o`, so the link editor reduces PLT32 to PC32 |
| `R_X86_64_PLT32` to `puts` twice and `printf` twice | 4 | PLT entry and GOT slot | undefined, provided by a shared library |
| `R_X86_64_64` to `grade` in `fns[]` | 2 | `R_X86_64_RELATIVE` | an absolute field in a PIE |

Dos importaciones dan un `.plt`de 48 bytes (PLT0 más dos entradas de 16), un `.got.plt`de 40 bytes (tres ranuras reservadas más dos), y un `.rela.plt`Los dos registros relativos dan un`.rela.dyn`Estos números provienen de la especificación aplicada a nuestros registros.`tiny-dyn-x86_64`La Comisión ha adoptado el artículo 6 del Reglamento (CE) no 659/1999 de la Comisión, de conformidad con el artículo 6 del Reglamento (CE) no 659/1999.

El compilador escribe `PLT32`En cada llamada, definida o no definida, desde las binutils 2.31 (H.J. Lu, 2018-02-13) y LLVM 7.0.0 (D43383, 2018-02-20).`PLT32`En un objeto, por lo tanto, dice "una llamada", y no dice nada sobre si una entrada PLT existirá.`R_X86_64_GOTPCRELX`y `R_X86_64_REX_GOTPCRELX`(El artículo`G + GOT + A - P`) solicitar un espacio GOT sin PLT, que `-fno-plt`producen y que el editor de enlaces puede relajar a una direct`lea`o `call`cuando el símbolo sea local (psABI apéndice B). `R_X86_64_32`y `R_X86_64_32S`No puede contener una dirección en un PIE y el enlace falla.

## La sección dinámica, el PLT y el GOT

El capítulo 5 de gABI dice que un ejecutable que participa en la vinculación dinámica tiene un `PT_INTERP`programa, que debe preceder a cualquier segmento cargable, y una `PT_DYNAMIC`, que cubre el `.dynamic`La sección es una matriz de `Elf64_Dyn`pares, una etiqueta firmada y un valor, terminado por `DT_NULL`¿ Qué ?

`tiny-dyn-x86_64`es 5.184 bytes escritos por `build_tiny_dyn_elf.py`La sección dinámica de la misma contiene doce pares en la`0x1000`¿Qué es esto ?

| Tag | Number | Value | Meaning |
|---|---|---|---|
| `DT_NEEDED` | 1 | `libc.so.6` | a string table offset, `0xb`, the library to load |
| `DT_HASH` | 4 | `0x1a0` | the SysV hash table |
| `DT_STRTAB` | 5 | `0x188` | `.dynstr` |
| `DT_SYMTAB` | 6 | `0x140` | `.dynsym`, 24 bytes per entry |
| `DT_STRSZ` | 10 | `0x15` | 21 bytes of strings |
| `DT_SYMENT` | 11 | `0x18` | 24 |
| `DT_PLTGOT` | 3 | `0x10c0` | `.got.plt` |
| `DT_PLTRELSZ` | 2 | `0x30` | two 24-byte records |
| `DT_PLTREL` | 20 | 7 | the records are `DT_RELA` |
| `DT_JMPREL` | 23 | `0x1b8` | `.rela.plt` |
| `DT_FLAGS_1` | `0x6ffffffb` | `0x08000000` | `DF_1_PIE`, a GNU tag |
| `DT_NULL` | 0 | 0 | end |

El GABI dice por qué .`DT_JMPREL`Separar estas entradas de reubicación permite que el enlace dinámico las ignore durante la inicialización del proceso, si se habilita la vinculación perezosa.`DF_BIND_NOW`En el`DT_FLAGS`¿ Qué ?`DF_1_NOW`En el`DT_FLAGS_1`, o`LD_BIND_NOW`en el entorno se apagan las fijaciones perezosas y el enlace dinámico las procesa en el inicio.

### PLT0 y la entrada para los putos

psABI sección 5.2 da las dos formas en la figura 5.2. Aquí están los bytes de nuestro archivo:

```text
     1f0: ff 35 d2 0e 00 00    pushq  0xed2(%rip)      # 0x10c8 = GOT+8
     1f6: ff 25 d4 0e 00 00    jmpq   *0xed4(%rip)     # 0x10d0 = GOT+16
     1fc: 0f 1f 40 00          nopl   (%rax)

     200: ff 25 d2 0e 00 00    jmpq   *0xed2(%rip)     # 0x10d8 = GOT+24, the slot for puts
     206: 68 00 00 00 00       pushq  $0x0             # index 0 into DT_JMPREL
     20b: e9 e0 ff ff ff       jmp    0x1f0            # PLT0
```

Haga la aritmética una vez. El desplazamiento relativo al RIP es relativo a la siguiente instrucción. `0x1f6 + 0xed2 = 0x10c8`, que es `.got.plt + 8`- ¿ Qué ?`0x1fc + 0xed4 = 0x10d0`, que es `.got.plt + 16`- ¿ Qué ?`0x206 + 0xed2 = 0x10d8`, la cuarta ranura.`0x210 - 0x20 = 0x1f0`Cada entrada es de 16 bytes, así que la entrada para `exit`comienza en`0x210`, empuja`$0x1`, y lee`0x10e0`¿ Qué ?

El GOT reserva tres ranuras. psABI 5.2: la entrada cero tiene la dirección de `_DYNAMIC`, y "las entradas uno y dos en la tabla global de compensación también están reservadas".`.got.plt`Leía`0x1000, 0, 0, 0x206, 0x216`. glibc llena el espacio uno con su `struct link_map`y la ranura dos con `_dl_runtime_resolve`Cuando carga el archivo.`0x206`en el expediente, que es el `pushq`En el interior .`puts@plt`. psABI: la ranura "initialemente ... contiene la dirección de la siguiente instrucción pushq".

Así que la primera llamada se ejecuta`call 0x200`¿ Qué ?`jmpq *0x10d8`las tierras en`pushq $0`¿ Qué ?`jmp 0x1f0`empuja GOT+8 y pasa a través de GOT+16 en el resolver, que lee el registro 0 de `DT_JMPREL`¿ Qué ?`R_X86_64_JUMP_SLOT puts`, parece`puts`arriba en`libc.so.6`, escribe su dirección en `0x10d8`Y salta allí. La segunda llamada se ejecuta.`call`¿ Qué ?`jmpq *0x10d8`Drepper escribe que "el coste general para todos excepto la primera llamada es por lo tanto 'sólo' un salto indirecto".`%r11`explica una opción de registro en la lección 38: es un registro de rasguño por lo que "el código en el PLT no necesita verter ningún registro".

El sitio de llamadas en `.text`Leía`e8 d4 ff ff ff`En el`0x227`, y `0x22c - 0x2c = 0x200`- ¿ Qué ?`llvm-objdump`huellas`callq 0x200 <puts@plt>`para el mismo y nombra la entrada de la `.rela.plt`registro, que es cómo `plt_entries()`también lo nombrará.

### Lo que el archivo no tiene

El archivo no tiene `PT_GNU_RELRO`. Jakub Jelinek propuso a RELRO en la lista de servicios binarios el 1 de enero de 2004: mover las secciones escritas sólo durante la reubicación al inicio del segmento de escritura, para que el enlace dinámico pueda `mprotect`El script predeterminado de GNU ID pone`.init_array`¿ Qué ?`.fini_array`¿ Qué ?`.data.rel.ro`¿ Qué ?`.dynamic`, y `.got`dentro, y sólo mantiene las tres ranuras reservadas de `.got.plt`dentro, 24 bytes, porque las otras ranuras están escritas con pereza.`-z now`conjuntos `DF_BIND_NOW`y mueve todo.`.got.plt`dentro, RELRO completo, porque nada se escribe después de la puesta en marcha.`_dl_protect_relro`llamadas`mprotect(start, end - start, PROT_READ)`en el rango alineado con las páginas. `-fno-plt`va un paso más allá en el lado del compilador: las llamadas pasan `jmpq *name@GOTPCREL(%rip)`, lazosos es imposible, y el manual del CCG dice que "todos los símbolos externos se resuelven en el momento de carga".

El archivo tampoco tiene secciones de versiones.`libc.so.6`añade `.gnu.version`, uno `Elfxx_Half`por `.dynsym`la entrada, y `.gnu.version_r`, una `Verneed`para cada biblioteca con `Vernaux`registros para cada nombre de versión. `printf@GLIBC_2.2.5`medios: la entrada de `printf`En el`.gnu.version`tiene un índice, el `Vernaux`cuyo `vna_other`es igual a que el índice tiene `vna_name`"GLIBC_2.2.5", en el apartado `Verneed`cuyo `vn_file`El índice 0 es local, 1 es global y el bit 15 (`VERSYM_HIDDEN`¿ Qué ?`0x8000`) oculta una versión del editor de enlaces.

La tabla de hash es un SysV `DT_HASH`con un cubo y una cadena de tres entradas, 24 bytes.`DT_HASH`o `DT_GNU_HASH`Este archivo no exporta nada, y no se ejecuta aquí, por lo que si Glibc acepta una tabla de un balde no se verifica.

### .dynsym, .symtab y strip

El GABI hace`.dynsym`y `.dynstr`secciones asignadas y `.symtab`y `.strtab`El enlace dinámico lee el primer par y nunca el segundo. Drepper: "La tabla de símbolos normal no se utiliza en el tiempo de ejecución y, por lo tanto, no es necesario deshacerse de un binario. No tiene impacto en el rendimiento".

GNU `strip --strip-all`se elimina `.symtab`¿ Qué ?`.strtab`¿ Qué ?`.debug_*`, y `.comment`Cada sección asignada se mantiene:`.interp`¿ Qué ?`.dynsym`¿ Qué ?`.dynstr`¿ Qué ?`.gnu.hash`¿ Qué ?`.gnu.version*`¿ Qué ?`.rela.dyn`¿ Qué ?`.rela.plt`¿ Qué ?`.plt`¿ Qué ?`.text`¿ Qué ?`.rodata`¿ Qué ?`.eh_frame`¿ Qué ?`.init_array`¿ Qué ?`.dynamic`¿ Qué ?`.got`¿ Qué ?`.got.plt`¿ Qué ?`.data`¿ Qué ?`.bss`Los cabezas de sección se quedan a menos que agreguen`--strip-section-headers`(artículos 2.41) o ejecutar `llvm-strip --strip-sections`- ¿ Qué ?`--strip-debug`conserva los símbolos.`--only-keep-debug`escribe el archivo de depuración que `--add-gnu-debuglink`nombres de la desnudada, con un CRC32 en `.gnu_debuglink`, y `ld --build-id`escribe `.note.gnu.build-id`Así que un depurador puede coincidir con los dos.

## El objeto de la COFF

`clang -target x86_64-pc-windows-msvc -O1 -fno-asynchronous-unwind-tables -c`producido `main-x86_64.obj`y `ledger-x86_64.obj`. La página de formato PE define el objeto: un encabezado de archivo de 20 bytes, sin encabezado opcional, una tabla de sección de entradas de 40 bytes, una tabla de símbolos de registros de 18 bytes y registros de reubicación de 10 bytes por sección.

`CoffObject`En el`code/main.py`El encabezado del archivo dice máquina.`0x8664`, 10 secciones, 37 registros de símbolos,`SizeOfOptionalHeader`0. Los nombres de las tablas de las secciones `.text`¿ Qué ?`.data`¿ Qué ?`.bss`, seis secciones llamadas`.rdata`, y `/16`Un nombre que comienza con`/`es una desviación decimal en la tabla de cuerdas, que comienza con su propio tamaño de cuatro bytes; `/16`se resuelve a `.llvm_addrsig`, que es el nombre `llvm-objdump -h`Las huellas.

Los 37 registros incluyen registros auxiliares. Un símbolo de sección (clase de almacenamiento 3) contiene uno que contiene la longitud de la sección, el recuento de reubicación, una suma de comprobación y una selección COMDAT.`.rdata`las secciones tienen la selección 2, "cualquier", y contienen una cadena literal cada uno bajo un nombre MSVC-malgradado: `??_C@_04JDDJEAHN@even?$AA@`El editor de enlaces guarda una copia de cada uno.`.file`con el registro auxiliar "main.c". `llvm-objdump -t`Imprima esto como `[ 0]`¿ Qué ?`AUX scnlen ...`¿ Qué ?`[ 2]`, y así sucesivamente, saltando los índices auxiliares, y la prueba compara nuestro índice y pares de nombres con ese texto.

Las reubicaciones: 14 `IMAGE_REL_AMD64_REL32`En el`.text`y 4 `IMAGE_REL_AMD64_ADDR64`En el`.rdata`- ¿ Qué ?`REL32`es "la dirección relativa de 32 bits del byte después de la reubicación".`R_X86_64_PC32`con el `-4`incorporado: el registro ELF contiene un complemento `-4`porque el registro tiene un campo de adjunto y el registro COFF no tiene ninguno. `ledger-x86_64.obj`tiene tres `REL32`En el`0x36`¿ Qué ?`0x84`, y `0x93`, y la lección 32's `ledger-x86_64-O1.o`Tiene sus tres .`R_X86_64_PC32`El código es el mismo y los formatos difieren sólo en el lugar donde se escriben los cuatro. `ADDR64`es una dirección absoluta de ocho bytes, el `labels`tabla de los punteros de cuerdas, y se convierte en una reubicación de base en la imagen, como `R_X86_64_64`Se convierte en`R_X86_64_RELATIVE`¿ Qué ?

| | ELF `Elf64_Rela` | COFF relocation | Mach-O `relocation_info` |
|---|---|---|---|
| record size | 24 bytes | 10 bytes | 8 bytes |
| fields | offset, type and symbol in `r_info`, addend | offset, symbol index, type | address, then a 32-bit word with symbolnum:24, pcrel:1, length:2, extern:1, type:4 |
| addend | explicit | none, lives in the field | none, or an `ARM64_RELOC_ADDEND` record before the real one |
| the call to `puts` | `R_X86_64_PLT32 puts-0x4` | `IMAGE_REL_AMD64_REL32 puts` | `ARM64_RELOC_BRANCH26 _puts` |

## La imagen del PE: importaciones, exportaciones y traslados de base

`tiny-x86_64.exe`es 2,560 bytes escritos por `build_tiny_pe.py`Tiene el mismo trabajo que el pequeño ELF:`puts`y `exit`de la`msvcrt.dll`También exporta un nombre y lleva una reubicación de base, por lo que los tres directorios que necesita esta lección están todos presentes. `pefile`2024.8.26 y `llvm-objdump -p`Los dos lo leen, y ambas capturas son accesorios.

El tema de MS-DOS comienza con`MZ`y en compensación `0x3c`Tiene`e_lfanew`¿ Qué ?`0x40`- En el`0x40`La firma `PE\0\0`Luego el encabezado del archivo COFF: máquina `0x8664`, 4 secciones, `SizeOfOptionalHeader`240, características `0x22`El encabezado opcional comienza con el `Magic`campo, `0x20b`, lo que significa PE32+. `AddressOfEntryPoint`¿ Es verdad ?`0x1000`¿ Qué ?`ImageBase`¿ Es verdad ?`0x140000000`¿ Qué ?`SectionAlignment` `0x1000`¿ Qué ?`FileAlignment` `0x200`¿ Qué ?`SizeOfImage` `0x5000`¿ Qué ?`Subsystem`3 (consola). `DllCharacteristics` `0x8160`decodificaciones de `HIGH_ENTROPY_VA 0x0020`¿ Qué ?`DYNAMIC_BASE 0x0040`¿ Qué ?`NX_COMPAT 0x0100`, y `TERMINAL_SERVER_AWARE 0x8000`. Luego 16 directorios de datos de RVA y tamaño.`0x2080`tamaño `0x4d`, Importar en`0x2000`tamaño `0x28`, Traslado de la Base en `0x4000`tamaño `0xc`, IAT en`0x2040`tamaño `0x18`¿ Qué ?

Las direcciones en una imagen son RVAs, compensaciones desde la base de carga. La tabla de sección las mapea a los desplazamientos de archivo: `.text`RVA `0x1000`en el archivo `0x200`¿ Qué ?`.rdata`RVA `0x2000`En el`0x400`¿ Qué ?`.data`RVA `0x3000`En el`0x600`¿ Qué ?`.reloc`RVA `0x4000`En el`0x800`- ¿ Qué ?`PeImage.rva_to_offset()`Es esa mesa.

### El directorio de importaciones

En RVA `0x2000`Se encuentra una matriz de descriptores de importación de 20 bytes terminados por un descriptor cero.`0x2028`, timestamp 0, cadena del remitente 0, nombre `0x2068`, Tabla de direcciones de importación `0x2040`. El nombre RVA apunta a "msvcrt.dll". La tabla de búsqueda contiene entradas de 64 bits terminadas con cero.`0x2058`(indicación 0, "puts") y `0x2060`(indicación 0, "salida").

La especificación dice que el IAT es "identico al contenido de la tabla de búsqueda de importación hasta que la imagen está ligada".`0x2040`tiene las mismas dos RVAs, y la prueba verifica que los 24 bytes en `0x2028`igual a los 24 bytes en `0x2040`. "Durante la unión, las entradas en la tabla de direcciones de importación se sobrescriben con las direcciones de 32 bits (para PE32) o 64 bits (para PE32+) de los símbolos que se están importando". El cargador de Windows lo hace antes de que el punto de entrada se ejecute. No hay unión perezosa y no hay tritón. El código llega directamente a la ranura:

```text
    100b: ff 15 2f 10 00 00    call qword ptr [rip + 0x102f]   # 0x1011 + 0x102f = 0x2040, the IAT slot of puts
    1013: ff 15 2f 10 00 00    call qword ptr [rip + 0x102f]   # 0x1019 + 0x102f = 0x2048, the IAT slot of exit
```

`FF /2`Las dos instrucciones tienen los mismos bytes porque cada una se sienta ocho bytes después de la otra y lee una ranura ocho bytes después de la otra. `iat_calls()`escanear la sección de código para `ff 15`, añade seis y el desplazamiento, y nombra la ranura del directorio de importación.

### El directorio de exportación

En RVA `0x2080`la tabla de directorio de exportación de 40 bytes: banderas, timestamp, versión, un nombre RVA que apunta a "tiny-x86_64.exe", base ordinal 1, entrada de la tabla de direcciones 1, indicador de nombre.`0x20a8`tiene una RVA, `0x1000`. La tabla de indicadores de nombres de exportación en `0x20ac`El número de entradas de exportación ordinaria se encuentra en la tabla de la cadena "tiny_entry".`0x20b0`tiene un índice imparcial de 16 bits, 0. Nombre `k`mapas para la dirección de la entrada `ordinals[k]`, y el ordenal exportado es `ordinals[k] + ordinal base`. La especificación añade una regla que no utilizamos: una entrada de dirección que apunta dentro de la sección de exportación es una cadena de remitente como `MYDLL.expfunc`, y la tabla de indicadores de nombre está "ordenada léxicamente para permitir búsquedas binarias".

### Traslados de bases

`.reloc`Cada bloque comienza con una página RVA y un tamaño de bloque, luego entradas de 16 bits con el tipo en los cuatro bits superiores y una página compensada en la doce inferior. Nuestro bloque: página `0x3000`, tamaño 12, entrada `0xa000`, tipo 10, `IMAGE_REL_BASED_DIR64`, compensación 0, y una entrada de relleno de tipo 0, `IMAGE_REL_BASED_ABSOLUTE`Los ocho bytes en RVA`0x3000`¿ Qué ?`0x1400020d0`, la base de imagen más el RVA de la cadena "enlazada a mano". Si el cargador coloca la imagen en otro lugar, añade la diferencia a ese campo. La especificación: "Los archivos de imagen no contienen relocaciones COFF ... Una imagen contiene información de reubicación en forma de relocaciones de base en la sección .reloc (a menos que la imagen tenga el atributo IMAGE_FILE_RELOCS_STRIPPED). " `/DYNAMICBASE`está activado por defecto en el link.exe actual, por lo que las imágenes llevan `.reloc`y `DYNAMIC_BASE`¿ Qué ?

Una imagen PE no tiene tabla de símbolos para desprender.`PointerToSymbolTable`"debería ser cero para una imagen porque la información de depuración de COFF está obsoleta", y los nombres viven en un PDB separado. Lo que sobrevive en cada imagen es el directorio de importación, el directorio de exportación y, para el código de 64 bits,`.pdata`, cuyo `RUNTIME_FUNCTION`Las entradas nombran el comienzo y el final de cada función que asigna una pila o llama a otra función.

## La imagen Mach-O: comandos de carga, estampas y la tabla de símbolos indirectos

`ledger_main`es el arm64 ejecutable vinculado. `otool -h`Imprima 19 comandos de carga y banderas `0x00200085`- ¿ Qué ?`MH_NOUNDEFS 0x1`¿ Qué ?`MH_DYLDLINK 0x4`¿ Qué ?`MH_TWOLEVEL 0x80`, y `MH_PIE 0x200000`Los 19 comandos son cuatro.`LC_SEGMENT_64`(El artículo`__PAGEZERO`¿ Qué ?`__TEXT`¿ Qué ?`__DATA_CONST`¿ Qué ?`__LINKEDIT`), dos `LC_LOAD_DYLIB`(El artículo`@rpath/libledger.dylib`Entonces ...`/usr/lib/libSystem.B.dylib`), y uno de cada uno `LC_DYLD_CHAINED_FIXUPS`¿ Qué ?`LC_DYLD_EXPORTS_TRIE`¿ Qué ?`LC_SYMTAB`¿ Qué ?`LC_DYSYMTAB`¿ Qué ?`LC_LOAD_DYLINKER`¿ Qué ?`LC_UUID`¿ Qué ?`LC_BUILD_VERSION`¿ Qué ?`LC_SOURCE_VERSION`¿ Qué ?`LC_MAIN`¿ Qué ?`LC_RPATH`¿ Qué ?`LC_FUNCTION_STARTS`¿ Qué ?`LC_DATA_IN_CODE`, y `LC_CODE_SIGNATURE`- ¿ Qué ?`MachoFile.command_names()`devuelve la misma lista en el mismo orden que `otool -l`¿ Qué ?

`__TEXT`comienza en`0x100000000`- ¿ Qué ?`__text`es 256 bytes en `0x100000558`¿ Qué ?`main`y nada más, porque las cuatro funciones del libro mayor viven en el Dylib. `__stubs`es 84 bytes en `0x100000658`, siete estubes de doce bytes.`__DATA_CONST`comienza en`0x100004000`con`__got`, 64 bytes, ocho ranuras de punteros de tipo sección `S_NON_LAZY_SYMBOL_POINTERS`, y `__const`, 32 bytes, el `fns[]`y `names[]`tablas de `main.c`¿ Qué ?

### Un palo, a mano

El estubrio para `_puts`¿ Qué es ?`0x100000694`¿Qué es esto ?

```text
    30 00 00 90    adrp x16, 0x100004000
    10 1e 40 f9    ldr  x16, [x16, #0x38]
    00 02 1f d6    br   x16
```

`0x90000030`¿ Es verdad ?`adrp`El registro es el bajo de cinco bits, 16.`immlo`es bits 30 a 29, 0. `immhi`es 23 bits a 5, 1. El valor es `immhi << 2 | immlo = 4`, en páginas de 4,096 bytes, por lo que la página de destino es la página de la instrucción, `0x100000000`, más `0x4000`- ¿ Qué ?`0xf9401e10`¿ Es verdad ?`ldr`con un desafío sin firmar de 12 bits escalado por 8: bits 21 a 10 son 7, veces 8 es `0x38`, así que la carga se lee`0x100004038`- ¿ Qué ?`0xd61f0200`¿ Es verdad ?`br x16`El arm64 en la fuente de ID64 es exactamente esta instrucción de tres.`StubAtom`¿ Qué ?

El nombre que pertenece a la ranura proviene de la tabla de símbolos indirectos. `LC_DYSYMTAB`Los puntos de 15 índices en la tabla de símbolos.`reserved1`donde comienza su ciclo de índices y en`reserved2`¿Cuánto mide cada palo?`reserved1`y ocho bytes por ranura. `__stubs`comienza en el índice 0 y `__got`en el índice 7, por lo que rebote 5 mapas a la entrada de la tabla de símbolos 10, `_puts`, y la ranura 7 de `__got`mapas de la misma entrada. `otool -Iv`Imprenta ambas corrientes, y `indirect_table()`reproducir cada fila.

### Espacio de nombres de dos niveles

`nm -m`huellas`(undefined) external _puts (from libSystem)`y `(undefined) external _grade (from libledger)`La biblioteca proviene del registro de símbolos.`nlist.h`define `GET_LIBRARY_ORDINAL(n_desc)`como bits de 15 a 8 de `n_desc`, 1 por la primera `LC_LOAD_DYLIB`, 2 por el segundo, `0xfe`para la búsqueda dinámica, `0xff`para el ejecutable. `ld(1)`: "Con -two_levelnamespace (el predeterminado), el enlace solo busca los dilibes en la línea de comandos para los símbolos y registros en los que se encontraron dilibes". `nm_lines()`Render los doce discos de `n_type`¿ Qué ?`n_sect`¿ Qué ?`n_desc`, y `n_value`, y el ensayo compara su línea de salida por línea con la capturada `nm -m`. Reconectarse con `-Wl,-flat_namespace`y leer .`otool -h`para ver el `MH_TWOLEVEL`Un poco desaparecer.

## Correcciones en cadena, byte por byte

`LC_DYLD_CHAINED_FIXUPS`es un `linkedit_data_command`¿ Qué es esto ?`dataoff 0x8000`¿ Qué ?`datasize 208`Los 208 bytes siguen .`fixup-chains.h`¿ Qué ?

El encabezado es de siete campos de 32 bits: `fixups_version 0`¿ Qué ?`starts_offset 32`¿ Qué ?`imports_offset 80`¿ Qué ?`symbols_offset 112`¿ Qué ?`imports_count 8`¿ Qué ?`imports_format 1`(El artículo`DYLD_CHAINED_IMPORT`), `symbols_format 0`(no comprimido). `llvm-objdump --macho --chained-fixups`Imprime estos siete valores y la prueba los compara.

En el offset 32, `dyld_chained_starts_in_image`¿ Qué es esto ?`seg_count 4`y cuatro compensaciones, una por segmento, cero para segmentos sin correcciones. Sólo `__DATA_CONST`Tiene uno.`dyld_chained_starts_in_segment`Leía`size 24`¿ Qué ?`page_size 0x4000`¿ Qué ?`pointer_format 6`(El artículo`DYLD_CHAINED_PTR_64_OFFSET`, "el objetivo es vm compensado"), `segment_offset 0x4000`¿ Qué ?`max_valid_pointer 0`¿ Qué ?`page_count 1`, y uno `page_start`de 0. `DYLD_CHAINED_PTR_START_NONE`¿ Qué ?`0xffff`, significaría una página sin problemas.

La cadena comienza en el archivo de compensación `0x4000 + 0`Cada palabra de 64 bits lleva dos cosas: qué escribir allí y dónde está la siguiente palabra.

```text
  dyld_chained_ptr_64_bind    ordinal:24  addend:8  reserved:19  next:12  bind:1 = 1
  dyld_chained_ptr_64_rebase  target:36   high8:8   reserved:7   next:12  bind:1 = 0
```

Lea la primera palabra,`0x8010000000000000`- Bit 63 está listo, un enlace.`next`es bits 62 a 51, `0x002`, en unidades de 4 bytes, así que la siguiente corrección es 8 bytes en.`0x4058`¿ Qué ?`0x00000000000006db`- 63 puntos libres, una rebasada.`next`Es 0, el final de la cadena.`0x6db`, un vm de compensación, y la base de imagen `0x100000000`¿ Qué hace ?`0x1000006db`, la cadena "ocio" en `__cstring`La palabra antes de ella,`0x00100000000006d6`, rebajas `names[0]`a "incluso".

El paseo visita doce palabras: ocho se une en `__got`con ordenal 0 a 7, dos enlaces en `__const`con ordenal 0 (ambas entradas de `fns[]`punto en `_grade`), y las dos rebases de `names[]`- ¿ Qué ?`chained_fixups()`los devuelve con sus ficheros de compensación y palabras crudas, y la prueba compara los doce `(offset, raw)`parejas con `dyld_info -fixup_chain_details`, que imprime `raw: 0x8010000000000000 bind: (next: 002, bindOrdinal: 0x000000)`Por primera vez.

La tabla de importaciones en el offset 80 tiene ocho entradas de 32 bits: `lib_ordinal:8`¿ Qué ?`weak_import:1`¿ Qué ?`name_offset:23`, con nombres en el descalzo 112.`_grade`¿ Qué ?`_grade_label`¿ Qué ?`_percent_change`¿ Qué ?`_total`de la biblioteca 1, y de 4 a 7 son `___stack_chk_fail`¿ Qué ?`___stack_chk_guard`¿ Qué ?`_printf`¿ Qué ?`_puts`de la biblioteca 2. `fixup_rows()`se une las dos tablas en las líneas `dyld_info -fixups`huellas, como `__DATA_CONST __got 0x100004038 bind libSystem/_puts`, y la prueba compara a los doce.

Así que el "puntero corrupto" en `0x100004038`es importar 7. dyld camina esta cadena una vez, en carga, y escribe la dirección de `_puts`En el ranura. La sesión WWDC22 de Apple lo expresó de esta manera: "en la ubicación del puntero de 64 bits en DATA, algunos de los bits contienen el desplazamiento a la próxima ubicación de fijación. También está un bit que dice si la fijación es un bind o una rebase". Xcode 13 hizo que el formato fuera el predeterminado para un objetivo de implementación de macOS 12 o iOS 15, y Xcode 13.3 lo extendió a macOS 11 e iOS 13.4. No hay un enlace perezoso en este formato. Cada ranura está ligada antes .`main`- ¿Qué?

### El diseño que reemplazó

`ledger_main_nochain`es el mismo vínculo con `-Wl,-no_fixup_chains`- Sí , lo ha hecho .`LC_DYLD_INFO_ONLY`En cambio, con cuatro tablas codificadas en byte: rebase 8 bytes, bind 72, lazy bind 120, export 64. También tiene tres secciones que faltan al archivo encadenado: `__TEXT,__stub_helper`(108 bytes), un `__got`de sólo 16 bytes de capacidad `___stack_chk_guard`y `dyld_stub_binder`, y `__DATA,__la_symbol_ptr`(56 bytes, siete ranuras). `dyld_info -fixups`listas siete `lazy-bind`filas y siete rebases que apuntan a cada ranura perezosa en su propio trampolín en`__stub_helper`El trampolín empuja un índice y salta a`dyld_stub_binder`, que resuelve el nombre y escribe la ranura. Es el PLT y GOT de la sección anterior en formas Mach-O: `__la_symbol_ptr`¿ Es verdad ?`.got.plt`¿ Qué ?`__stub_helper`es PLT0 y el `pushq $index`, y `dyld_stub_binder`¿ Es verdad ?`_dl_runtime_resolve`- ¿ Qué ?`code/main.py`El laboratorio de práctica solicita que se den las compensaciones y secciones de este archivo y no se decodifique el código de enlace.

### El trie de las exportaciones

`LC_DYLD_EXPORTS_TRIE`Cada nodo es un tamaño terminal uleb128, entonces, si no es cero, uleb128 banderas y un uleb128 offset, entonces un byte de conteo de hijos y, para cada niño, un borde de cadena y un uleb128 nodo offset. `exports_trie()`- Caminando.`libledger.dylib`Se vuelve .`_percent_change 0x410`¿ Qué ?`_grade 0x428`¿ Qué ?`_total 0x464`¿ Qué ?`_grade_label 0x498`, los cuatro offsets `dyld_info -exports`En el ejecutable regresa.`__mh_execute_header`¿ Qué ?`_fns`, y `_main`¿ Qué ?

## ¿Qué tira de las hojas

La de Apple `strip(1)`dice que sin opciones en un ejecutable dinámicamente vinculado se comporta como `-u -r`: salvar símbolos indefinidos y símbolos referidos dinámicamente. `-x`elimina los símbolos locales. `-N`elimina toda la tabla de símbolos y la tabla de cadenas. "Para las bibliotecas compartidas dinámicas, el nivel máximo de desprendimiento es generalmente -x", porque "Sólo se utilizan símbolos globales por el proceso de enlace dinámico".

Observado en nuestros accesorios.`strip ledger_main`¿ Qué ?`LC_SYMTAB`de 12 a 10 registros y eliminados `_main`¿ Qué ?`_fns`, y `_names`. las ocho importaciones no definidas permanecen, porque se necesita que sus nombres se vinculen, y `__mh_execute_header`se queda porque es `REFERENCED_DYNAMICALLY`El décimo registro es un cuchillo de tipo .`N_OPT`nombre`radr://5614542`, que las herramientas de Apple escriben para que la mesa nunca esté vacía; `nm`El programa desnudado imprime las mismas tres líneas que el original y sale de 0.`LC_FUNCTION_STARTS`Aún tiene un comienzo,`main`¿ Qué ?

`strip libledger.dylib`Sin opciones, fue más allá de lo habitual.`-x`¿ Qué es esto ?`LC_SYMTAB`tiene 0 registros y `nm`El trie de exportación sigue listando las cuatro funciones en los mismos desreglados, porque Dyld se une a través del trie y nunca a través.`LC_SYMTAB`- ¿ Qué ?`LC_FUNCTION_STARTS`El laboratorio C vincula un nuevo`main.c`contra el dillib despojado para demostrar que el editor de enlaces también es servido por el trie.

| What | ELF after `--strip-all` | PE image | Mach-O after `strip` |
|---|---|---|---|
| static symbol table | gone | never present in an image | local and defined names gone, undefined imports stay |
| names the loader binds | `.dynsym` and `.dynstr` stay | import directory stays | imports stay in `LC_SYMTAB` and in the chained imports table |
| exported names | `.dynsym` stays | export directory stays | exports trie stays |
| function starts | `.eh_frame` FDEs, unless built without unwind tables | `.pdata` for non-leaf functions | `LC_FUNCTION_STARTS` stays |
| debug information | `.debug_*` gone | in the PDB, never in the image | in the dSYM, never in the image |
| section or load command headers | stay unless `--strip-section-headers` | stay | stay |

## Encontrar funciones sin símbolos

`static-arm64-O2`enlaces`main-arm64-O2.o`y `ledger-arm64-O2.o`en un ejecutable con cinco funciones en `__text`- ¿ Qué ?`nm -n`de la historia sin extraer es la verdad de fondo: `_main`En el`0x100000500`(244 bytes), `_percent_change`En el`0x1000005f4`(24), `_grade`En el`0x10000060c`60), `_total`En el`0x100000648`(248), `_grade_label`En el`0x100000740`(72) `static-arm64-O2.stripped`No tiene`N_SECT`símbolo dentro `__text`- ¿ Qué ?`FunctionFinder`ejecuta cuatro métodos en ella y `score()`compara cada lista con la verdad.

### LC_FUNCTION_STARTS

Los puntos de mando en nueve bytes en `__LINKEDIT`¿ Qué es esto ?`80 0a f4 01 18 3c f8 01 00`El formato es uleb128 deltas, el primero de la serie `__TEXT`dirección.`80 0a`¿ Es verdad ?`0x500`, así que el primer comienzo es`0x100000500`- ¿ Qué ?`f4 01`¿ Es verdad ?`0xf4`, así que`0x1000005f4`- ¿ Qué ?`18`da .`0x10000060c`¿ Qué ?`3c`da .`0x100000648`¿ Qué ?`f8 01`da .`0x100000740`, y `00`Cinco de cinco, sin falsos positivos, en`-O0`y en`-O2`El editor de enlaces escribe esta tabla para el desmantelador y el reportero de choque, y`strip`Ghidra lo lee en su analizador "Mach-O Function Starts",`dyld_info -function_starts`Imprima, y LIEF lo expone como `function_starts`¿ Qué ?

### Las exploraciones del prólogo

Los patrones de inicio de Ghidra AARCH64 comienzan con `fd 7b bf a9`, que es `stp x29, x30, [sp, #-0x10]!`En nuestros archivos encuentra 0 de 5 en ambos niveles.`sub sp, sp, #N`Primero y después .`stp x29, x30, [sp, #M]`con una compensación positiva, por lo que la forma preindexación nunca ocurre.`sub sp, sp, #imm`¿ Qué ?`(word & 0xff0003ff) == 0xd10003ff`, encuentra 5 de 5 en `-O0`, donde cada función tiene un marco, y 1 de 5 en `-O2`, donde `percent_change`¿ Qué ?`grade`¿ Qué ?`total`, y `grade_label`son funciones de hoja sin marco en absoluto. `percent_change`comienza con `cbz w0, ...`Un análisis de prólogo informa lo que el compilador eligió emitir, y la lección 38 mostró cuán a menudo eso no es nada.

Ninguno de los escaneadores produjo un falso positivo aquí, porque el programa es pequeño.`55`de `push rbp`Entra en otras instrucciones, por lo que los archivos de patrón x86-64 de Ghidra emparejan un patrón posterior como `55 48 89 e5`con un modelo previo como `ret`o `nop`que debe terminar con la función anterior.

### Los objetivos de llamadas

Cada uno .`bl`En el`__text`se llama un inicio de función, porque un `bl`en el centro de una función es algo que un compilador no emite para C. La codificación es `0x94000000`con un contraste de palabras de 26 bits firmado.`bl`En el`0x1000005a0`Leía`3d 00 00 94`¿ Qué es esto ?`0x3d`es 61, veces 4 es `0xf4`, así que el objetivo es`0x100000694`En el .`ledger_main`Ese es el estómago para `_puts`En la construcción estática,`call_targets()`Recolecta los objetivos dentro de la`__text`, deja caer a los que aterrizan en`__stubs`, y añade el `LC_MAIN`Entradas, porque nada llama`main`Nucleus (Andriesse, Slowinska, Bos, 2017) construye su detección de funciones sobre esta idea de código generado por el compilador, y Andriesse y otros midieron en 2016 que los límites de funciones son la parte más difícil del desmonte, más difícil que los límites de instrucciones.

## Una firma en forma de flauta

Hex-Rays describe FLIRT en su documentación: un patrón es "los primeros 32 bytes de una función donde todos los bytes variantes están marcados", los bytes variantes provienen de fijos y referencias externas, los patrones viven en un árbol, y más allá del patrón "calculamos el CRC16 de los bytes a partir de la posición 33 hasta el primer byte variante" y almacenamos esa longitud. Dos funciones con el mismo patrón y CRC se separan por un byte de cola en una posición donde difieren, luego por los nombres a los que se refieren. Una colisión que nada separa se "soluciona por inteligencia natural". Las funciones de menos de cuatro bytes sin referencias externas se excluyen, y "el archivo de firma no contiene ningún byte de las bibliotecas originales, excepto los nombres de las funciones".

`signatures_from_object()`construye eso desde `ledger-arm64-O2.o`y `main-arm64-O2.o`, la forma en que FLAIR lo construye a partir de los objetos de una biblioteca.`relocation_info`El registro cubre una palabra de cuatro bytes, y esos bytes se convierten en variantes.`.pat`el diseño de línea, documentado en el parser de rizin, es el patrón, la longitud CRC, la CRC16, la longitud de la función, `:0000`y el nombre, y `^offset name`para cada nombre de referencia:

```text
A00000342800004B890C8052087D091B000DC01AC0035FD6 00 0000 0018 :0000 _percent_change
E8CC8C52C8CCAC72087C289B09FD7FD308FD62930801090B081900511F110071 04 9AAF 003C :0000 _grade ^0000 l_switch.table.grade ^0001 l_switch.table.grade
E8CC8C52C8CCAC72087C289B09FD7FD308FD62930801090B081900511F110071 04 9AAF 0048 :0000 _grade_label ^0000 l_switch.table.grade_label ^0001 l_switch.table.grade_label ^0002 _labels ^0003 _labels
```

`_percent_change`es 24 bytes, así que su patrón es toda la función y la longitud CRC es 0. `_grade`es de 60 bytes con su primera reubicación en el byte 36, por lo que el CRC cubre cuatro bytes.`.pat`uso de parseres: polinomio `0x8408`, valor inicial `0xffff`Su valor para la cadena ASCII "123456789" es `0x6e90`, y los pines de prueba que.

`match_signatures()`Prueba cada firma en cada inicio de función de la construcción despojada, patrón primero, luego CRC.`-O0`Las cinco funciones obtienen sus nombres y nada choca.`-O2`Las dos líneas de arriba chocan.`grade_label`en línea`grade`, por lo que las dos funciones comparten sus primeros 36 bytes y los cuatro bytes CRC. Ambas firmas coinciden en`0x10000060c`y en`0x100000740`La cola los divide: en 44 byte`_grade`¿ Qué ?`20`y `_grade_label`¿ Qué ?`28`, que es `ldr w0, [x9, w8, uxtw #2]`contra`ldr x8, [x9, w8, uxtw #3]`El informe registra las colisiones y el byte que las decidió.

El examen`test_object_bytes_equal_linked_bytes_outside_relocated_words`verifica la premisa: cada byte no variante de cada función de objeto es igual al byte en el mismo desplazamiento en el ejecutable vinculado, por lo que el editor de enlace cambió sólo las palabras reubicadas. `_total`No tiene ninguna reubicación y ninguna variante de byte en absoluto.

### Lo que las herramientas hacen con la misma idea

IDA Pro 9.4 envía firmas FLIRT por procesador y el `sigmake`herramienta para construirlos; IDA Free cubre sólo x86 y x86-64 y es para uso no comercial. Lumina es un servidor que clave metadatos de función por md5 de la función y omite funciones menores a 32 bytes. Ghidra 12.1.4 (Apache-2.0) tiene ID de Función: un hash completo sobre mnemónicos y modos de dirección que es "robusto contra los cambios debido a la vinculación" y un hash específico que agrega operandos constantes, anotados por instrucción coincidente, con funciones de padre e hijo como corta-corta; sus bases de datos enviadas cubren solo las bibliotecas de Visual Studio, y usted llena la suya de sus propios binarios. Binary Ninja 6.0 Krypton (comercial, con una edición gratuita) utiliza WARP (Apache-2.0): la función GUID es una UUIDv5 sobre los GUID de bloque básicos, cada uno una UUIDv5 sobre los bytes de bloque después de ceroar "todas las instrucciones relocables" y dejar caer NOPs, con llamadas, llamador y funciones adyacentes como restricciones. BinDiff 8 (Apache-2.0) combina funciones entre dos binarios con un orden fijo de pasos, desde la coincidencia hash a través de índices de MD de gráfico de flujo hasta llamar coincidencia de secuencias, y pesa la similitud de bordes, bloques, instrucciones y el índice MD de gráfico de flujo. Diafora 3.4.3 (AGPL-3.0, IDA solamente) añade el producto de los primos pequeños, donde cada mapa mnemónico se asigna a un primo y una función es el producto, una huella digital multiset libre de orden. El radar2 6.2.4 (LGPL-3.0) tiene firmas (`zg`¿ Qué ?`z/`¿ Qué ?`zb`) y se lee FLIRT `.sig`archivos con `zfs`La rizina 0.9.1 (LGPL-3.0) crea `.pat`o `.sig`de un binario abierto con `Fc`y aplica uno con `Fs`¿ Qué ?

IDA y Binary Ninja están cerrados y se describen aquí a partir de su documentación. radar2 y rizin no están instalados en esta máquina, y sus comandos se citan de sus libros y fuentes.`clang`¿ Qué ?`ld`¿ Qué ?`llvm-objdump`¿ Qué ?`otool`¿ Qué ?`nm`¿ Qué ?`strip`, y `dyld_info`LIEF lee todos los formatos aquí, incluyendo los objetos COFF, que ha soportado desde 0.17.0, y informó diez ataduras y dos bases en `ledger_main`, las mismas doce correcciones . las pieles de trabajo sólo leen los archivos ELF .`main-x86_64.obj`con`PEFormatError: 'DOS Header magic not found.'`y leer la pequeña imagen.

## Construye el mismo

Ejecutar la demostración:

```bash
cd phases/13-tools-and-protocols/40-linking-loading-and-stripped-binaries
python3 code/main.py
```

La salida es un informe JSON con cinco partes. `elf`tiene los recuentos de reubicación de los dos objetos, el plan de enlace y la sección dinámica, PLT y GOT de `tiny-dyn-x86_64`- ¿ Qué ?`coff`contiene las secciones, las reubicaciones, las funciones y los nombres indefinidos de los dos objetos. `pe`En el caso de las empresas de transporte, el número de pasajeros en el mercado de transporte de mercancías es el número de pasajeros en el mercado de transporte.`tiny-x86_64.exe`- ¿ Qué ?`macho`Tiene`ledger_main`con sus estumbres, arreglos encadenados, filas de arreglos y exportaciones, el diseño heredado de `ledger_main_nochain`, el Dyllib, y los dos informes de strip.`identification`Tiene la verdad, los cuatro métodos con puntajes, y las firmas con sus colisiones, en `-O0`y `-O2`¿ Qué ?

Forma esperada:

```json
{
  "elf": {"linkPlan": {"pltSymbols": ["puts", "printf"], "sizes": {".plt": 48, ".got.plt": 40, ".rela.plt": 48, ".rela.dyn": 48}, "...": "..."}, "tiny-dyn-x86_64": {"...": "..."}},
  "coff": {"main-x86_64.obj": {"...": "..."}},
  "pe": {"imports": [{"dll": "msvcrt.dll", "iat": "0x2040", "...": "..."}], "...": "..."},
  "macho": {"ledger_main": {"flags": "0x200085", "fixupRows": ["__DATA_CONST __got 0x100004000 bind libledger/_grade", "..."], "...": "..."}, "strip": {"...": "..."}},
  "identification": {"O2": {"methods": {"functionStarts": {"recall": "5/5"}, "prologueSub": {"recall": "1/5"}, "...": "..."}, "correctNames": 5}}
}
```

Leer .`code/main.py`en este orden:

1. `ElfFile`se leen los encabezados, las secciones, las tablas de símbolos y las reubicaciones, y `dynamic_entries()`¿ Qué ?`jump_slots()`¿ Qué ?`plt_entries()`, y `plt0()`Leer la imagen enlazada.
2. `plan_dynamic_link()`se aplicará el cuadro 4.9 y la sección 5.2 a los registros de reubicación de los objetos reubicables.
3. `CoffObject`lee el encabezado del archivo, la tabla de secciones, los símbolos de 18 bytes con sus registros auxiliares y las reubicaciones de 10 bytes.
4. `PeImage`se lee el documento de MS-DOS, la firma PE, el encabezado opcional, los directorios de datos, las secciones y los directorios de importación, exportación y reubicación de base, y `iat_calls()`decodificación`ff 15`¿ Qué ?
5. `MachoFile`lee comandos de carga, segmentos,`nlist`registros, la tabla de símbolos indirectos, los estubes, las correcciones en cadena, el trie de exportación y las reubicaciones de objetos.
6. `FunctionFinder`ejecuta los cuatro buscadores de inicio y `score()`comparar cada uno con la verdad.
7. `signatures_from_object()`y `match_signatures()`construir y aplicar las firmas en forma de FLIRT.
8. El `parse_*`las funciones de lectura del texto de la herramienta capturada que las pruebas comparan con.

Más órdenes:`python3 code/main.py elf FILE`¿ Qué ?`coff FILE`¿ Qué ?`pe FILE`, y `macho FILE`imprimir un informe. `python3 code/main.py plan OBJECT...`Imprime el plan de enlace para cualquier conjunto de objetos ELF. `python3 code/main.py functions STRIPPED UNSTRIPPED OBJECT...`Pone los marcadores y las firmas en otro par de sus propias construcciones.

## Usalo

Utilice los lectores en este orden en un programa que construyó.

1. Leer los encabezados y nombrar el formato y el tipo: objeto trasladable, ejecutable o biblioteca compartida.
2. Enumera las importaciones con sus proveedores: `DT_NEEDED`y `.dynsym`, el directorio de importaciones o la tabla de importaciones en cadena y `LC_LOAD_DYLIB`Cada llamada a un nombre en esa lista deja la imagen.
3. Sigue una llamada a mano a su ranura. Escriba los bytes de llamada, los bytes de la barra o PLT, la dirección de la ranura, la palabra en el archivo y lo que escribe el cargador allí. Si la palabra en el archivo se parece a basura, decodifique como una corrección antes de llamarlo basura.
4. Comparar la tabla de símbolos con las exportaciones y la función se inicia.`strip`No se necesita un desembolso, pero se necesita un desembolso.
5. En una construcción despojada, ejecuta los buscadores que puedes ejecutar y puntuarlos contra el gemelo sin despojar mientras aún tienes uno. Prefiere las tablas que el editor de enlaces escribió, `LC_FUNCTION_STARTS`¿ Qué ?`.pdata`, o`.eh_frame`, sobre patrones de byte, y utilizar objetivos de llamada para llenar las lagunas.
6. Construye firmas de sus propios objetos y apliquelas. Graba cada colisión y el byte que la divide. Un nombre que dos firmas reclaman es un desconocido hasta que la cola o un nombre de referencia lo decida.

Mantenga el texto de la herramienta capturada con los archivos. Cuando la cadena de herramientas cambia, recapturar, porque una lista cambiada es un fijo cambiado.

## Laboratorio interactivo

### Laboratorio A: el PLT a mano

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/40-linking-loading-and-stripped-binaries/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
elf = load_elf("tiny-dyn-x86_64")
[(e.tag_name, e.text) for e in elf.dynamic_entries()]
elf.plt0()
[(e.symbol, hex(e.address), hex(e.slot), hex(e.slot_initial)) for e in elf.plt_entries()]
[hex(w) for w in elf.got_plt_words()]
```

Antes de cada llamada, calcular la respuesta a partir de los bytes en el texto anterior.`IMPORTS`En el`build_tiny_dyn_elf.py`a tres nombres, reconstruir y predecir los nuevos tamaños de `.plt`¿ Qué ?`.got.plt`, y `.rela.plt`antes de leerlas.

### Laboratorio B: un arreglo encadenado

```python
exe = load_macho("ledger_main")
stub = next(s for s in exe.stubs() if s.symbol == "_puts")
stub.lines()
raw = int.from_bytes(exe.read(stub.slot, 8), "little")
hex(raw), raw >> 63, (raw >> 51) & 0xfff, raw & 0xffffff
[(i.index, i.name, i.library) for i in exe.chained_fixups()["imports"]]
```

Decodifica la palabra de enlace en papel primero: el bit de enlace, el `next`En el campo en unidades de cuatro bytes, la ordenal. Luego lee la palabra ocho bytes más adelante y confirmar que es la siguiente fijación.`0x100004058`y explicar por qué es `next`es cero.

### Laboratorio C: desprestigio y reencadenamiento

En el directorio de accesorios, enlazar un programa nuevo con el dilib completamente despojado:

```bash
cd phases/13-tools-and-protocols/40-linking-loading-and-stripped-binaries/code/fixtures
clang -target arm64-apple-macos -O1 main.c -L. -lledger.stripped -Wl,-rpath,@executable_path -o /tmp/relinked
nm /tmp/relinked | grep ' U '
dyld_info -exports libledger.stripped.dylib
```

El enlace tiene éxito con una biblioteca cuyo `LC_SYMTAB`Es vacío, porque el editor de enlaces lee el trie de exportaciones.`python3 ../main.py macho libledger.stripped.dylib`y comparar`symtab`¿ Qué ?`exports`, y `functionStarts`Luego deshace la estructura estática con`-N`En cambio , veamos qué .`dyld_info -function_starts`todavía huellas.

### Laboratorio D: la colisión

```python
from main import *
stripped = load_macho("static-arm64-O2.stripped")
sigs = signatures_from_object(load_macho("ledger-arm64-O2.o"))
[s.pat_line() for s in sigs]
starts = FunctionFinder(stripped).function_starts()
match_signatures(sigs, stripped, starts)
```

Lea los dos registros de colisión y luego construye las firmas de`ledger-arm64-O0.o`, emparejarlos contra`static-arm64-O0.stripped`, y explicar desde el `.pat`líneas por las que no se produce ninguna colisión en `-O0`Entonces acortad .`PATTERN_LENGTH`a 16 y contar cuántas funciones chocan.

## Laboratorio de práctica

Descifrar las tablas de vinculación heredadas y agregar el buscador de PE.

Requisitos:

- Descifrar el `LC_DYLD_INFO_ONLY`rebase, binda y perezoso binda flujos de código op de `ledger_main_nochain`en filas con segmento, sección, dirección, tipo y objetivo, y hacer que sean iguales a las 21 filas en la capturada `dyld_info -fixups`¿Qué es eso?
- Añadir un`PT_GNU_RELRO`programa de título y un `DT_FLAGS`con`DF_BIND_NOW`¿ Qué ?`build_tiny_dyn_elf.py`, moverse`.got.plt`dentro del rango cubierto, y hacer `llvm-objdump -p`En las fichas README, diga qué parte de RELRO completo reproduce y qué parte sólo puede realizar Glibc.
- Escriba un`.pdata`lector para una imagen PE32+ y un buscador de funciones que utiliza `RUNTIME_FUNCTION`Comience con direcciones.`clang -target x86_64-pc-windows-msvc -c`Sin`-fno-asynchronous-unwind-tables`, lee su `.pdata`las transferencias y indicar cuáles de sus funciones son las funciones de hoja que la tabla omite.
- Escriba una prueba para cada lector que comparar con el texto capturado de la herramienta.

El éxito significa`python3 code/main.py macho ledger_main_nochain`Imprime las mismas 21 filas de fijación que `dyld_info`Las huellas, el pequeño ELF lleva un`PT_GNU_RELRO`Eso es .`llvm-objdump`Las listas y los`.pdata`el buscador nombra las funciones que puede y enumera las funciones de hoja que no puede.

## Lección Artefacto

Esta lección incluye `outputs/skill-linking-loading-and-stripped-binaries.md`.Utilizarlo para convertir un programa que construye en una lista de sus importaciones con sus proveedores, una llamada rastreada a la ranura que llena el cargador, una declaración de lo que `strip`Se requiere el gemelo sin cortar como verdadera, una lista de herramientas capturadas para cada reclamación y un registro de cada colisión.

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/40-linking-loading-and-stripped-binaries
python3 code/main.py
python3 code/tests/test_main.py
```

La verificación deberá demostrar:

- la sección dinámica de `tiny-dyn-x86_64`las partidas`llvm-objdump -p`etiqueta por etiqueta, y `DT_FLAGS_1`que lleva`DF_1_PIE`
- Los dos .`R_X86_64_JUMP_SLOT`los registros coinciden .`llvm-objdump -R`y nombre`puts`y `exit`
- PLT0 empuja GOT+8 y salta a través de GOT+16, cada entrada lee su ranura y empuja su índice, y cada ranura tiene la dirección de su propia `pushq`
- las secciones y los encabezados del programa del pequeño partido ELF `llvm-objdump -h -p`
- Los registros de reubicación de los dos objetos de ELF coinciden `llvm-objdump -r`
- El plan de enlace encuentra dos símbolos PLT, cuatro llamadas directas, dos reubicaciones relativas, y los tamaños de la pequeña imagen
- la tabla de la sección COFF, 37 registros de símbolos con registros auxiliares y coincidencias de traslados `llvm-objdump -h -t -r`, y `/16`se resuelve a `.llvm_addrsig`
- los tres COFF `REL32`registros de `ledger`Siéntate en el ELF tres`PC32`compensaciones
- los encabezados de la PE, `DllCharacteristics`, y los directorios de datos coinciden `llvm-objdump -p`
- el directorio de importaciones, las entradas de indicios/nombres y la correspondencia de IAT `llvm-objdump`y `pefile`, y el IAT es igual a la tabla de búsqueda antes de la unión
- el directorio de exportaciones y el `DIR64`el código de reubicación de base, y el puntero reubicado nombra la cadena "enlazada a mano"
- `ff 15`en RVA `0x100b`se resuelve en la ranura de IAT de `puts`
- Banderas Mach-O, los 19 nombres de comandos de carga, y las dos bibliotecas coinciden.`otool`
- Los siete estubes se decodan a los ocho `__got`ranuras y coincidencias de la tabla de símbolos indirectos `otool -Iv`
- Los encabezados en cadena coinciden .`llvm-objdump --macho --chained-fixups`
- La cadena de paseo reproduce las doce palabras crudas de `dyld_info -fixup_chain_details`
- las filas de fijación y la tabla de importaciones coinciden `dyld_info -fixups`y `-imports`
- las exportaciones se ajustan `dyld_info -exports`para el dilib y para su copia despojada
- `nm_lines()`igual `nm -m`para el ejecutable y su copia despojada
- `strip`Se retiró .`_main`¿ Qué ?`_fns`, y `_names`, mantuvo nueve nombres y uno `radr://5614542`puñala, y el programa despojado imprime la misma salida
- el diseño heredado tiene `LC_DYLD_INFO_ONLY`, siete punteros perezosos, y rebases en `__stub_helper`
- arm64 reubicaciones de objetos coinciden `llvm-objdump -r`
- `LC_FUNCTION_STARTS`recupera cinco de cinco en ambos niveles y es igual `nm`y `dyld_info -function_starts`
- el `sub sp`La exploración encuentra cinco de cinco en `-O0`y uno de cinco en `-O2`, y el `stp`El escaneo de marco no encuentra ninguna
- `bl`objetivos más el `LC_MAIN`entrada recuperada cinco de cinco
- Las firmas nombran las cinco funciones en ambos niveles y registran las `grade`colisión dividida por byte 44
- La CRC16 de "123456789" es `0x6e90`, y los ocho bytes bajo el par de carga GOT de `main`son variantes
- cada byte de objeto no variante es igual al byte vinculado
- la demostración y los datos de las cifras informan las mismas direcciones, y el módulo de sitio incorpora los mismos datos en ES5

## Modos de falla de producción

| Failure | What the weak tool reports | What the analysis must prove |
|---|---|---|
| A `bl` or `call` lands in a stub | "calls an unnamed function at 0x694" | the stub bytes, the slot it loads, and the import the slot binds |
| A GOT word read from the file | "corrupt pointer 0x8010000000000007" | bit 63 is bind, `next` is 2, ordinal 7 is `_puts` from library 2 |
| `PLT32` in an object | "every call goes through the PLT" | defined symbols are reduced to a direct distance, only undefined ones get an entry |
| Partial RELRO read as full | "GOT is read-only" | `.got.plt` slots after the three reserved ones sit outside `PT_GNU_RELRO` without `-z now` |
| COFF `REL32` compared with ELF `PC32` | "the Windows build has different relocations" | same offsets, the `-4` lives in the ELF addend and in the COFF field |
| IAT read before load | "imports point at strings" | the IAT equals the lookup table until the loader binds, and the strings are hint/name entries |
| A stripped dylib | "no symbols, nothing exported" | the exports trie and `LC_FUNCTION_STARTS` list every function |
| `nm` on a stripped executable | "one function" | `LC_FUNCTION_STARTS` lists five starts, nine bytes long |
| A prologue scan at `-O2` | "one function" | four leaf functions have no frame, call targets and the starts table find them |
| Two functions share 36 bytes | "grade_label is grade" | the tail byte at 44 and the lengths 60 and 72 split the collision |
| A tool run on another machine's build | the numbers in this text | recapture every listing from your own fixtures |

## Regla de funcionamiento

Siga cada llamada externa desde el sitio de llamada hasta la ranura que llena el cargador, y decodifique la palabra en esa ranura como la configuración la define antes de llamarla una dirección. Solo se desprenda después de haber grabado lo que el cargador todavía necesita. La función de recuperación comienza a partir de las tablas que el editor de enlaces escribió antes de escanear para bytes, marcar cada buscador contra un gemelo con símbolos, y grabar cada colisión de firma con el byte que lo divide.

## Leer más

- [System V gABI, chapter 5: Program Loading and Dynamic Linking](https://www.sco.com/developers/gabi/latest/ch5.dynamic.html)
- [System V Application Binary Interface, AMD64 Architecture Processor Supplement: section 5.2 and Table 4.9](https://gitlab.com/x86-psABIs/x86-64-ABI)
- [Drepper, "How To Write Shared Libraries", version 4.1.2](https://akkadia.org/drepper/dsohowto.pdf)
- [Drepper, "ELF Symbol Versioning"](https://akkadia.org/drepper/symbol-versioning)
- [Jelinek, "Little hardening DSOs/executables against exploits", binutils list, 2004-01-06](https://www.sourceware.org/ml/binutils/2004-01/msg00070.html)
- [Taylor, "Linker relro", 2008](https://www.airs.com/blog/archives/189)
- [binutils commit bd7ab16b: PLT32 for calls, 2018-02-13](https://gnu.googlesource.com/binutils-gdb/+/bd7ab16b4537788ad53521c45469a1bdae84ad4a)y [LLVM D43383](https://reviews.llvm.org/D43383)
- [GNU Binutils: strip](https://sourceware.org/binutils/docs/binutils/strip.html)y [llvm-objcopy](https://llvm.org/docs/CommandGuide/llvm-objcopy.html)
- [Microsoft, PE Format](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)
- [CALL instruction reference, FF /2](https://www.felixcloutier.com/x86/call)
- [Apple, mach-o/loader.h](https://github.com/apple-oss-distributions/xnu/blob/main/EXTERNAL_HEADERS/mach-o/loader.h)y [mach-o/nlist.h](https://github.com/apple-oss-distributions/xnu/blob/main/EXTERNAL_HEADERS/mach-o/nlist.h)
- [Apple dyld, mach-o/fixup-chains.h](https://github.com/apple-oss-distributions/dyld/blob/main/include/mach-o/fixup-chains.h)
- [Apple ld64, stub_arm64.hpp](https://github.com/apple-oss-distributions/ld64/blob/main/src/ld/passes/stubs/stub_arm64.hpp)
- [Xcode 13 release notes: chained fixups](https://developer.apple.com/documentation/xcode-release-notes/xcode-13-release-notes)y [WWDC22 session 110362, "Link fast: Improve build and launch times"](https://developer.apple.com/videos/play/wwdc2022/110362/)
- [strip(1)](https://keith.github.io/xcode-man-pages/strip.1.html)¿ Qué ?[ld(1)](https://keith.github.io/xcode-man-pages/ld.1.html), y [dyld(1)](https://keith.github.io/xcode-man-pages/dyld.1.html)
- [Hex-Rays, IDA F.L.I.R.T. Technology: In-Depth](https://docs.hex-rays.com/core/flirt/concepts/ida-f.l.i.r.t.-technology-in-depth)y [Lumina overview](https://docs.hex-rays.com/add-ons/lumina/overview)
- [Ghidra Function ID help](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/FunctionID/src/main/help/help/topics/FunctionID/FunctionID.html)y [AARCH64 function start patterns](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Processors/AARCH64/data/patterns/AARCH64_LE_patterns.xml)
- [Vector 35, WARP](https://github.com/Vector35/warp)y [Binary Ninja WARP guide](https://docs.binary.ninja/guide/warp.html)
- [BinDiff manual](https://www.zynamics.com/bindiff/manual/)y [BinDiff source](https://github.com/google/bindiff)
- [Diaphora](https://github.com/joxeankoret/diaphora)
- [rizin pat.c, the .pat parser](https://github.com/rizinorg/rizin/blob/dev/librz/sign/pat.c)¿ Qué ?[radare2 flirt.c](https://github.com/radareorg/radare2/blob/master/libr/anal/flirt.c), y [radare2 book: Zignatures](https://book.rada.re/signatures/zignatures.html)
- [Andriesse et al., "An In-Depth Analysis of Disassembly on Full-Scale x86/x64 Binaries" (USENIX Security 2016)](https://www.usenix.org/conference/usenixsecurity16/technical-sessions/presentation/andriesse)y [Andriesse, Slowinska, Bos, "Compiler-Agnostic Function Detection in Binaries" (EuroS&P 2017)](https://danielandriesse.com/papers/eurosp-2017.pdf)
- [Bao et al., ByteWeight (USENIX Security 2014)](https://www.usenix.org/conference/usenixsecurity14/technical-sessions/presentation/bao)y [Shin, Song, Moazzezi, "Recognizing Functions in Binaries with Neural Networks" (USENIX Security 2015)](https://www.usenix.org/conference/usenixsecurity15/technical-sessions/presentation/shin)
- [LIEF changelog](https://lief.re/doc/latest/changelog.html)¿ Qué ?[pyelftools](https://github.com/eliben/pyelftools), y [pefile](https://github.com/erocarrera/pefile)
