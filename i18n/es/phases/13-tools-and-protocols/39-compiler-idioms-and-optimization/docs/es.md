# Idiomas de compilador: qué hace la optimización de su código y cómo leerlo de nuevo

> Un compilador optimizador reemplaza las construcciones de origen con formas de instrucción más baratas. Cada forma tiene una derivación, y esta lección construye el reconocedor que ejecuta la derivación hacia atrás y la comproba contra la fuente.

**Type:** Build
**Languages:** Python, C, x86-64 assembly, AArch64 assembly
**Prerequisites:** Phase 13 · 33 (disassembly and control flow), Phase 13 · 34 (lifting to pseudocode), Phase 13 · 38 (calling conventions)
**Time:** ~150 minutes

## Objetivos de aprendizaje

- Derivar el número mágico y cambiar para la división por cualquier constante con el Granlund-Montgomery ligado, y recuperar el divisor de las constantes emitido clang.
- Nombre de las cinco formas en que LLVM reduce un `switch`y leer el recuento de entradas, la ubicación de la tabla y las entradas del objeto.
- Lea un bucle vectorizado como carriles, acumuladores, elementos por iteración, máscara de recuento triples, reducción y bucle restante.
- Reconocer llamadas de cola, callees en línea, selecciones sin ramas, y los bucles que se convirtieron en`memset`o `memcpy`¿ Qué ?
- Informar cada idioma con sus instrucciones de prueba y su formulario fuente decodificado, en x86-64 y en AArch64, en `-O0`por el`-O3`¿ Qué ?
- Comparar el resultado con lo que Ghidra, Angr, radar2, rizin, y lldb muestran para los mismos bytes.

## El problema

Usted compilado`idioms.c`En el`-O2`y preguntó a un agente qué.`div7u`El agente leyó.`imulq $0x24924925, %rax, %rax`y respondió que la función se multiplica por 613566757, cambia y se suma.`x / 7`¿ Qué ?

El mismo objeto tiene un`switch`Con siete casos que no tienen salto en absoluto, un `switch`con cinco casos que es una mesa de salto, un `switch`con dos casos que es un `sete`y un `cmovne`, y un bucle hacia arriba `n`números enteros que ejecuta ocho números enteros por iteración en dos `xmm`El sistema de registro de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de cuyo de cuyo sitio de cuyo sitio de cuyo sitio de cuyo sitio de cuenta de cuenta de cuenta de cuenta de cuenta de cuenta de cuenta de cuenta de cuenta de cuenta de cuenta de cuenta de cuenta de cuenta de cuyo sitio de cuenta de cuenta de cuyo sitio de cuenta de cuenta de cuyo sitio de`-O0`la misma fuente producida `divl`, siete entradas de salto de mesas, y un bucle escalar.`-O1`el interruptor de siete casos se convirtió en una tabla de valores y el bucle vectorial aún no existía.

Estas formas son idiomas. Un compilador emite cada una de ellas de una regla que se puede leer en su fuente, y cada una puede ser decodificada de vuelta a la construcción de la que proviene. La decodificación es aritmética, y la aritmética se puede comprobar.`0x66666667`Esta lección construye el reconocedor para todos ellos, lo ejecuta en 18 versiones de nuestro propio código, y verifica todos los 373 informes contra la fuente C.

```figure
re-39-idioms-by-level
```

## Un idioma es una forma con una derivación

Un idioma es una secuencia de instrucciones que el compilador emite para una construcción de fuente. Tres cosas hacen que sea reconocible. Tiene una forma fija, los operandos llevan constantes que la derivación predice, y la forma difiere de lo que un programador escribiría a mano.

El reconocedor en `code/main.py`trabajos sobre el texto de `llvm-objdump -d -r`, que la lección 33 estableció como el oráculo para los límites de instrucciones. analiza cada función en instrucciones con dirección, bytes, mnemónica, operandos y la línea de reubicación que la sigue.`.rodata`, donde se encuentran las tablas de valores, y el `.rela.rodata`entradas, donde los objetivos de la mesa de salto viven antes de vincularse.

Cada detección es un`Idiom`registro con un `kind`, un decodificado .`source`formulario, una lista de `evidence`Direcciones, a `detail`Diccionario y un`verified`La bandera proviene de la fuente C.`parse_source()`Leía`idioms.c`, encuentra cada función, y extrae los divisores después `/`y `%`, los multiplicadores después de `*`, el `switch`Los casos y sus valores, la`? :`Los operadores, los`for`Un idioma de división se verifica cuando su divisor está en la lista de división de la función. Una tabla de búsqueda se verifica cuando sus entradas coinciden con los valores del caso, hasta un factor constante. Una llamada de cola se verifica cuando se llama a la llamada en la fuente. El informe demo termina con un `unverified`lista, y la suite de pruebas afirma que la lista está vacía para todas las 18 construcciones.

El reconocedor ejecuta un pequeño evaluador simbólico sobre los bloques básicos de cada función, en orden de lista, con los nombres de parámetros de la fuente en los registros de argumentos de las convenciones del Sistema V y AAPCS64 de la lección 38.`-O0`En cada entrada de bloque los estados de los predecesores se fusionan, y donde difieren un nombre que un idioma anterior asignado gana sobre una constante.`cmovll %edx, %eax`En el`digest`Se lee como `sum % 10 < 5 ? sum % 10 : 5`: el lenguaje de bucle llamado acumulador `sum`, el lenguaje de división nombró su cuotiente `sum / 10`, y el modulo idiom nombró su resultado `sum % 10`La fuente dice:`rem > 5 ? 5 : rem`Ambas expresiones dan el menor de `rem`y 5, y están de acuerdo en `rem == 5`¿ Qué ?

## División por una constante

Granlund y Montgomery, "División por integros invariantes mediante multiplicación" (PLDI 1994), sustituyen `n / d`con un multiplicador y un turno.`s`y un multiplicador `m = ceil(2^s / d)`Entonces ...`floor(n * m / 2^s)`igual `floor(n / d)`por cada uno `n`en el rango cuando el error `e = m * d - 2^s`Es bastante pequeño. Para sin firmar de 32 bits.`n`La línea es `e <= 2^(s - 32)`. Para firmar`n`con`|n| < 2^31`La línea es `e <= 2^(s - 31)`, y una fijación de señal añade 1 cuando `n`es negativo, porque el piso rodea en el sentido equivocado para los cuotientes negativos.

`magic_numbers(d, bits, signed)`En el`main.py`búsquedas `s`La búsqueda reproduce cada constante de sonido emitido en los accesorios.

### 10 sin firmar a mano

`2^35 / 10 = 3435973836.8`El techo es`3435973837 = 0xCCCCCCCD`El error es`0xCCCCCCCD * 10 - 2^35 = 2`, y el límite en `s = 35`¿ Es verdad ?`2^3 = 8`- En el`s = 34`El error sería 6 contra un límite de 4, por lo que 35 es el cambio más pequeño.`n = 123`¿ Qué es esto ?`123 * 3435973837 = 422624781951`, y `422624781951 >> 35 = 12`¿ Qué ?

Clang emite exactamente esto para`div10u`En el`-O2`en el punto x86-64:

```text
       0: 89 f9                        	movl	%edi, %ecx
       2: b8 cd cc cc cc               	movl	$0xcccccccd, %eax
       7: 48 0f af c1                  	imulq	%rcx, %rax
       b: 48 c1 e8 23                  	shrq	$0x23, %rax
```

`0x23`Es 35.`movl %edi, %ecx`El reconocedor lee el multiplicador de la letra `movl $imm`que alimenta a los dos operandos.`imulq`, lee el turno y llama`divisor_from_magic(0xCCCCCCCD, 35)`Esa función intenta`2^35 // m = 10`y 11, y acepta 10 porque `magic_numbers(10)`da el mismo multiplicador y se desplaza hacia atrás.

En AArch64 la misma función es `mov w8, #0xcccd; movk w8, #0xcccc, lsl #16; umull x8, w0, w8; lsr x0, x8, #35`La constante llega en dos mitades de 16 bits, el multiplicador es `umull`El reconocedor reúne las mitades y informa el mismo divisor.

### 7 sin firmar a mano, con la corrección adicional

`2^35 / 7 = 4908534052.57`El techo es`4908534053 = 0x124924925`, que necesita 33 bits. El error es 3 contra un límite de 8, por lo que el cambio es 35, y ningún cambio menor funciona. Un multiplicador de 32 bits no puede sostener el multiplicador, por lo que el compilador se multiplica por los 32 bits bajos`0x24924925`y suma el dividendo de nuevo en una secuencia fija:

```text
      32: 48 69 c0 25 49 92 24         	imulq	$0x24924925, %rax, %rax
      39: 48 c1 e8 20                  	shrq	$0x20, %rax
      3d: 29 c7                        	subl	%eax, %edi
      3f: d1 ef                        	shrl	%edi
      41: 01 f8                        	addl	%edi, %eax
      43: c1 e8 02                     	shrl	$0x2, %eax
```

Leerlo como `t = (n * 0x24924925) >> 32`, entonces`q = (((n - t) >> 1) + t) >> 2`El cambio total es`32 + 1 + 2 = 35`- Con`n = 100`¿ Qué es esto ?`t = 14`¿ Qué ?`(100 - 14) >> 1 = 43`¿ Qué ?`43 + 14 = 57`¿ Qué ?`57 >> 2 = 14`El reconocedor detecta el`sub`¿ Qué ?`shr`¿ Qué ?`add`¿ Qué ?`shr`cola después de un cambio de 32 bits, agrega `2^32`regresar al multiplicador, y se recupera 7. `evaluate_division()`corre esta secuencia exacta y el test lo verifica contra`n // 7`En el formulario AArch64 se encuentra `sub w9, w0, w8; add w8, w8, w9, lsr #1; lsr w0, w8, #2`, una instrucción más corta porque la suma desplazada es una instrucción.

### Firmado 3 a mano

`2^32 / 3 = 1431655765.33`, techo`1431655766 = 0x55555556`, error 2, y el límite firmado en `s = 32`¿ Es verdad ?`2^1 = 2`El límite se mantiene sin espacio libre, así que el turno es 32 y el clang utiliza`shrq $0x20`- Con`n = -7`: el producto es `-10021590362`, el cambio aritmético por 32 da `-3`, el bit de signo está establecido, por lo que la fijación agrega 1 y el resultado es `-2`, que es `-7 / 3`en C. Los accesorios firmados `div10s`y `div3s`Los dos llevan el arreglo como`shrq $0x3f`en una copia del producto seguida de `addl`¿ Qué ?

### Lo que las constantes no son

El multiplicador solo no nombra divisor.`0x66666667`El multiplicador con el cambio 33 es 5, como se mostró en la lección 34.`clangMatchesAlgorithm`Los registros de campo que el par es el que la búsqueda produce.`magic_numbers()`los rechaza, porque el compilador emite un cambio con un sesgo de signo en su lugar, y `mod8s`en la siguiente sección muestra esa forma.

### El idioma pertenece al selector de instrucciones

En el`-O0`¿ Qué ?`div10u`¿ Es verdad ?`movl $0xa, %ecx; xorl %edx, %edx; divl %ecx`- ¿ Qué ?`div10s`¿ Es verdad ?`cltd; idivl %ecx`En el mismo objeto,`digest`divide su suma de bucle por 10 con `imulq $0x66666667`Una bandera explica la diferencia.`idioms-x86_64-nofastisel-O0.o`Fue construido con `-O0 -mllvm -fast-isel=false`, y en ella cada división es el multiplicador, incluyendo `div10u`con`movl $0xcccccccd, %ecx; imulq %rcx, %rax; shrq $0x23`La división de hardware es la elección del selector de instrucciones rápidas en `-O0`. El multiplicador es la disminución de SelectionDAG, que es lo que LLVM `UnsignedDivisionByConstantInfo`y `SignedDivisionByConstantInfo`La producción de productos.`digest`El bloque que sostiene la división también sostiene un`switch`El reconocedor informa de ambos formularios,`hardware idivl`y `magic multiply`, y la figura muestra la mezcla`-O0`La función.

## Modulo sigue el cuotiente

`x % d`¿ Es verdad ?`x - (x / d) * d`El compilador calcula el cuotiente con el idioma anterior, lo multiplica y restrae.`addl %ecx, %ecx; leal (%rcx,%rcx,4), %ecx`, que es `2 * q * 5`En AArch64 es una instrucción.`msub w0, w8, w9, w0`, que calcula .`w0 - w8 * w9`con`w9 = 10`El reconocedor sigue el registro de los cuotientes a través de copias, sigue el factor acumulado a través de `add`¿ Qué ?`lea`¿ Qué ?`shl`, y `imul`, y acepta una`sub`cuando el factor es igual al divisor.

Cuando el mismo coeficiente también alimenta otra expresión, ambos idiomas permanecen en el informe.`-O1`el compilador computa `sum / 10`Una vez en .`digest`y lo utiliza para el `switch`En x86-64 incluso combina el cuotiente dos veces,`leal (%rax,%rsi), %edi`para el modulo y `addl %esi, %eax`El reconocedor informa una división y un módulo y marca el cuotiente como compartido.`switch`el índice no se puede leer.

El modulo firmado por una potencia de dos tiene su propia forma, porque el cuotiente nunca se calcula:

```text
      92: 8d 48 07                     	leal	0x7(%rax), %ecx
      95: 85 ff                        	testl	%edi, %edi
      97: 0f 49 cf                     	cmovnsl	%edi, %ecx
      9a: 83 e1 f8                     	andl	$-0x8, %ecx
      9d: 29 c8                        	subl	%ecx, %eax
```

Para un negativo `x`El código se desvía por 7 antes de la mascarada, de modo que la truncation gira hacia cero como C requiere.`x`redondeado a un múltiplo de 8, y la restancia deja `x % 8`. AArch64 lo hace con `negs w8, w0; and w9, w0, #0x7; and w8, w8, #0x7; csneg w0, w9, w8, mi`: mascarar a ambos `x`y `-x`Ghidra llama su regla para esto.`RuleSignMod2nOpt`, con el resumen "Convertir los formularios INT_SREM: `(V + (sign >> (64-n)) & (2^n-1)) - (sign >> (64-n)) => V s% 2^n`" . " .

En el`-O0`AArch64 escribe modulo como `sdiv w9, w8, w10; mul w9, w9, w10; subs w0, w8, w9`El reconocedor también acepta ese formulario y informa.`hardware sdiv + mul + subs`¿ Qué ?

## Cinco maneras de bajar el interruptor

La LLVM decide la forma de un`switch`En dos lugares.`SimplifyCFG::simplifySwitchLookup`se ejecuta en`-O1`y arriba y convierte un interruptor cuyos casos producen constantes en una tabla de valores, cuando hay al menos tres casos y el objetivo puede construir tablas de búsqueda.`SingleValueKind`¿ Qué ?`LinearMapKind`¿ Qué ?`BitMapKind`, y `ArrayKind`- El .`-O0`El oleoducto nunca pasa por este paso.`SwitchLowering::findJumpTables`en la selección de instrucciones, que construye una mesa de salto cuando el grupo tiene al menos `min-jump-table-entries`entradas, 4 por defecto, y es lo suficientemente denso: `NumCases * 100 >= Range * MinDensity`con`jump-table-density`10 para funciones normales y 40 para funciones inferiores a `-Os`Lo que queda se convierte en una búsqueda binaria sobre las constantes de caso, con `splitWorkItem`Las instalaciones tienen una función por resultado.

| Function | Source | `-O0` x86-64 | `-O2` x86-64 | `-O2` AArch64 |
|---|---|---|---|---|
| `linear_switch` | cases 0 to 6 return 10 to 70 | jump table, 7 entries | linear map `10*k + 10`, default -1 | `madd` and `csinv` |
| `table_switch` | cases 0 to 6 return 10 to 60 then 71 | jump table, 7 entries | lookup table at `.rodata+0x28` | lookup table at `.rodata+0x0` |
| `call_switch` | cases 0 to 4 call one of five pointers | jump table, 5 entries | jump table, 5 entries | comparison tree, pivot 1 |
| `sparse_switch` | cases 1, 100, 10000, 1000000, 100000000 | compare chain | comparison tree, pivot 9999 | comparison tree, pivot 9999 |
| `two_cases` | cases 3 and 4 return 7 and 9 | compare chain | select chain `k != 3 ? (k == 4)*9 : 7` | two `csel` |

### La tabla de búsqueda, leída desde el objeto

`table_switch`En el`-O2`¿Qué es esto ?

```text
      d0: b8 ff ff ff ff               	movl	$0xffffffff, %eax
      d5: 83 ff 06                     	cmpl	$0x6, %edi
      d8: 77 0c                        	ja	0xe6 <table_switch+0x16>
      da: 89 f8                        	movl	%edi, %eax
      dc: 48 8d 0d 00 00 00 00         	leaq	(%rip), %rcx
		00000000000000df:  R_X86_64_PC32	.rodata+0x24
      e3: 8b 04 81                     	movl	(%rcx,%rax,4), %eax
      e6: c3                           	retq
```

Los límites de control`cmpl $6; ja`El aditivo de reubicación `0x24`más los 4 bytes del campo relativo al RIP pone la tabla en `.rodata+0x28`, la lección de la regla 33 derivado.`0a 00 00 00 14 00 00 00 ...`, así que las entradas son `[10, 20, 30, 40, 50, 60, 71]`, y son iguales a los valores de la fuente de caso.`.Lswitch.table.table_switch`en el `.s`archivo, con un comentario `@switch.table.table_switch`En el objeto sólo quedan la reubicación y los bytes. No hay salto, por lo que el gráfico de flujo de control tiene tres bloques.

El formulario AArch64 pone la dirección de la tabla junto con `adrp x8`y `add x8, x8, #0x0`, cada uno llevando su propia reubicación a `.rodata`, y cargas con `ldr w0, [x8, w0, uxtw #2]`El reconocedor lee el adjunto de la`add`el traslado.

### El mapa lineal, sin una tabla en absoluto

`linear_switch`retorno `10 * (k + 1)`por`k`en 0 a 6. `simplifySwitchLookup`detecta la relación afina y emite aritmética:

```text
      b0: 83 ff 07                     	cmpl	$0x7, %edi
      b3: 8d 04 bf                     	leal	(%rdi,%rdi,4), %eax
      b6: 8d 4c 00 0a                  	leal	0xa(%rax,%rax), %ecx
      ba: b8 ff ff ff ff               	movl	$0xffffffff, %eax
      bf: 0f 42 c1                     	cmovbl	%ecx, %eax
```

El evaluador lee `leal (%rdi,%rdi,4)`¿ Cómo ?`5*k`¿ Qué ?`leal 0xa(%rax,%rax)`¿ Cómo ?`10*k + 10`, y el `cmovb`después de`cmpl $7`¿ Cómo ?`k < 7 ? 10*k + 10 : -1`El reconocedor verifica la fórmula con cada caso de origen y marca el idioma verificado.`madd w8, w0, w8, w9`y `csinv w0, w8, wzr, lo`, donde `csinv`con`wzr`rendimientos `-1`en el camino predeterminado.

### La mesa de saltos, leída de las mudanzas

`call_switch`En el`-O2`¿Es la lección de forma 33 recuperada en `grade`En el`-O0`¿ Qué es esto ?`cmpl $4; ja`, una `leaq`¿ Qué ?`.rodata-0x4`, una `movslq`de una entrada de 4 bytes, `addq`, y `jmpq *%rax`Las cinco entradas son reubicaciones en`.rela.rodata`, y `target = addend - 4k`da .`0x10c, 0x107, 0x10e, 0x110, 0x109`, cinco bloques distintos que cada cola salta a través de un puntero de función.`dispatch`fixture escala esto a 16 casos: 16 entradas de reubicación en x86-64, y en AArch64 una tabla comprimida de 16 bytes únicos leídos con `ldrb w11, [x9, x8]`, escalado por 4 con `add x10, x10, x11, lsl #2`, y basado en un`adr`El reconocedor decodifica`[0x24, 0x74, 0x54, ...]`de la tabla de byte y los informes `compressed: true`¿ Qué ?

### El mismo interruptor en cuatro niveles

`digest`tiene un interruptor encendido `sum / 10`con casos 6 a 10 y valores 1, 2, 4, 8, 8, entonces multiplica el resultado por 16. en x86-64 es una tabla de salto en `-O0`, una mesa de búsqueda en `-O1`con entradas `[16, 32, 64, 128, 128]`, y una mesa de salto de nuevo en `-O2`y `-O3`cuyos cuatro bloques de caja se almacenan `0x10`¿ Qué ?`0x20`¿ Qué ?`0x40`¿ Qué ?`0x80`- Desde`-O1`En el caso de los casos, la multiplicación por 16 ha aumentado.`-O0`Los casos almacenan 1, 2, 4 y 8, y la multiplicación permanece después del cambio.`-O1`tabla de búsqueda el reconocedor compara las entradas con los valores de origen, encuentra una relación constante, y informa `foldedFactor: 16`En el AArch64 el`-O2`forma es un árbol de comparación con un pivote en 8, pruebas de igualdad para 6, 7 y 8, y una prueba de rango `sub w9, w9, #9; cmp w9, #2; b.hs`El reconocedor informa que los casos 9 y 10 se cubren en dos instrucciones.`rangeTests: [[9, 10]]`¿ Qué ?

Dos banderas mueven estos límites.`-fno-jump-tables`- ¿ Qué ?`table_switch`¿ Qué ?`call_switch`, y el `digest`en el arco de comparación en x86-64, con los pivotes 2 y 4 para la función de siete casos.`-mllvm -min-jump-table-entries=4`Las hojas`call_switch`En el caso de los grupos de referencia, el número de entradas no era el factor decisivo para un cambio de cinco casos de llamadas indirectas.`idioms-arm64-jt4-O2.o`registros que funcionan.

El CCG aplica ambas decisiones .`tree-switch-conversion.cc`, con las clases `jump_table_cluster`¿ Qué ?`bit_test_cluster`¿ Qué ?`switch_decision_tree`, y `switch_conversion`- Es un .`-ftree-switch-conversion`está en `-O2`¿ Qué ?`-fjump-tables`y `-fbit-tests`En el`-O1`, y el umbral de la mesa de salto proviene de`case-values-threshold`No hay GCC instalado en esta máquina, por lo que esos nombres provienen del código fuente y manual del GCC, y el Compiller Explorer es donde los comprueba.

## Los bucles: Vectorizados, Desarrollados o reemplazados

El vectorizador de bucle de LLVM está encendido en `-O2`¿ Qué ?`-O3`, y `-Os`, y fuera en `-O0`y `-O1`, una regla que data de LLVM 3.4 en enero de 2014. `sum_n`En el`-O2`muestra cada parte de la forma:

```text
     1f6: 83 fe 07                     	cmpl	$0x7, %esi
     1f9: 77 09                        	ja	0x204 <sum_n+0x14>
     206: 81 e2 f8 ff ff 7f            	andl	$0x7ffffff8, %edx
     230: f3 0f 6f 14 37               	movdqu	(%rdi,%rsi), %xmm2
     235: 66 0f fe c2                  	paddd	%xmm2, %xmm0
     239: f3 0f 6f 54 37 10            	movdqu	0x10(%rdi,%rsi), %xmm2
     23f: 66 0f fe ca                  	paddd	%xmm2, %xmm1
     243: 48 83 c6 20                  	addq	$0x20, %rsi
     24a: 75 e4                        	jne	0x230 <sum_n+0x40>
     24c: 66 0f fe c8                  	paddd	%xmm0, %xmm1
     250: 66 0f 70 c1 ee               	pshufd	$0xee, %xmm1, %xmm0
     262: 66 0f 7e c8                  	movd	%xmm1, %eax
     270: 03 04 97                     	addl	(%rdi,%rdx,4), %eax
     279: 75 f5                        	jne	0x270 <sum_n+0x80>
```

El guardia .`cmpl $7; ja`Envía el recuento de viaje por debajo de 8 al bucle escalar.`andl $0x7ffffff8`Las rondas del viaje cuentan hasta un múltiplo de 8. El cuerpo carga 32 bytes por iteración en dos `xmm`acumuladores, cuatro carriles de 32 bits cada, por lo que el factor de vectorización es 4 y el factor de interleave es 2. Después del bucle, `paddd`, dos .`pshufd`, dos más .`paddd`, y `movd`Reducir los ocho carriles a un número entero.`0x270`El reconocedor encuentra el bucle desde su borde trasero, ve los registros SIMD, lee el recuento de carril desde `paddd`, cuenta acumuladores distintos, divide el paso `0x20`por el 4 y por los informes `4 lanes x 2 accumulators = 8 elements per iteration`, con la máscara, el guard, la reducción y el ciclo restante como campos.`ldp q2, q3`, dos .`add v.4s`¿ Qué ?`addv s0, v0.4s`, y `fmov w8, s0`¿ Qué ?

Un recuento fijo de viajes elimina el bucle. `sum4`Es uno .`movdqu`, la reducción y `movd`, sin borde trasero.`poly8`no puede vectorizar, porque cada iteración depende del acumulador anterior, por lo que el compilador lo despliegue ocho veces: ocho cargas en compensaciones de 0 a `0x1c`de la`%rdi`, cada uno seguido de un cambio por 5 y una restancia que calcula `acc * 31`El reconocedor cuenta las cargas en compensaciones consecutivas desde una base y informa `loop unrolled by 8, no back edge`En el AArch64 las mismas cargas llegan como cuatro .`ldp`- En parejas.

Apague el vectorizador con `-fno-vectorize -fno-slp-vectorize`y `sum_n`se convierte en un bucle escalar desarrollado por 8, con ocho `addl k(%rdi,%rdx,4), %eax`y `addq $8`, más un bucle restante.`sum4`El reconocedor informa`unrolled loop`con`copies: 8`y `indexStep: 8`para la primera y `copies: 4`Por el segundo.

Lección 33 encontró dos bordes traseros en `total`En el`-O2`En esta lección se les da el nombre de un cuerpo vectorial y un resto escalar del mismo bucle fuente.`is_loop()`¿Qué es eso?`digest`en AArch64 contiene ramas retrocedientes a un bloque predeterminado compartido que no son bucles, y la primera versión del reconocedor las contó como bucles vectoriales.

### Los bucles que se convierten en llamadas de la biblioteca

`LoopIdiomRecognize`transforma un bucle que almacena cero en cada byte en una llamada a `memset`, desde `-O1`hacia arriba.`fill`En el`-O2`¿ Es verdad ?`testl %esi, %esi; jle; movl %esi, %edx; xorl %esi, %esi; jmp memset@PLT`La llamada es un salto de cola con la reubicación`R_X86_64_PLT32 memset-0x4`, y la función es `retq`El reconocedor informa que el sistema de detección de los virus está en el camino vacío.`store loop recognized as memset(...)`y lo verifica contra el bucle fuente, porque la fuente nunca nombra `memset`¿ Qué ?

`copy16`llamadas`memcpy`con una constante 16, y el backend x86 lo expande en línea como `movups (%rsi), %xmm0; movups %xmm0, (%rdi)`, dentro de su límite de 8 tiendas por`memcpy`- En el`-O0`La misma llamada es dos 8 bytes `movq`- En parejas.`-fno-builtin`La llamada sigue siendo una llamada,`jmp memcpy@PLT`con`movl $0x10, %edx`, y `fill`mantiene su bucle, que el vectorizador luego convierte en un bucle de dos 16 bytes `movups`Los registros de reconocimiento de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de datos de la red de datos de datos de datos de la red de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de la red de datos de de de de datos de datos de de de de datos de de de de de de de de de de de de de de de de de de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de un de de de un de un de un de de un de un de un de de de un de un`vectorized store loop: 2 x 16-byte stores = 32 bytes per iteration`para esa construcción.

## Llamadas que desaparecen o se convierten en saltos

`tail_caller`retorno `tail_target(x + 1)`- En el`-O0`Es verdad.`addl $0x1, %edi; callq tail_target@PLT; addq $0x10, %rsp; popq %rbp; retq`- En el`-O2`Es verdad.`incl %edi; jmp tail_target@PLT`El marco del llamador se ha ido y el llamador vuelve al llamador.`b tail_target`con`R_AARCH64_JUMP26`El reconocedor trata a un`jmp`o `b`La descesión recursiva de la lección 33 debe tratarla como una salida de función, porque el objetivo está fuera de la función.

`twice_sum`llamadas`leaf_add`- En el`-O0`La llamada está ahí,`callq leaf_add@PLT`- Desde`-O1`el cuerpo de `leaf_add`¿ Qué ?`leal (%rdi,%rsi), %eax`, aparece dentro .`twice_sum`Y no queda ninguna llamada.`-O0`Clang marca cada función `optnone`y `noinline`El reconocedor sabe por la fuente que`twice_sum`llamadas`leaf_add`, no encuentra ninguna llamada o salto de cola a él, y luego busca la forma del cuerpo de la llamada en el llamador: la mnemónica y la fuente operands con el destino eliminado, por lo que `add w8, w1, w0`en los pares de llamadas de AArch64 `add w0, w1, w0`Una coincidencia es la evidencia.

`dispatch`combina ambas cosas: una mesa de salto de 16 entradas cuyos casos son`movl %esi, %edi; jmp fN@PLT`El reconocedor informa de un interruptor y 16 llamadas de cola, todas verificadas en comparación con las 16 llamadas en la fuente.

## Las ramas que se convirtieron en datos se mueven

`clamp_max`¿ Es verdad ?`movl %esi, %eax; cmpl %esi, %edi; cmovgl %edi, %eax`Los nombres de los evaluadores`%edi`¿ Cómo ?`a`y `%esi`¿ Cómo ?`b`, registra la comparación y lee la`cmovg`¿ Cómo ?`a > b ? a : b`, que es el texto fuente. `abs_int`¿ Es verdad ?`movl %edi, %eax; negl %eax; cmovsl %edi, %eax`y se lee como `-x < 0 ? x : -x`; AArch64 lo escribe como `cmp w0, #0; cneg w0, w0, mi`y se lee como `x < 0 ? -x : x`- ¿ Qué ?`is_pos`¿ Es verdad ?`xorl %eax, %eax; testl %edi, %edi; setg %al`, que se lee como `(x > 0)`- Estos son ...`-fif-conversion`En palabras del CCG y `cmov`La selección en los LLVM.`X86CmovConversion`El paso puede dar una vuelta.`cmov`de nuevo en ramas cuando juzga que rentable, por lo que la misma fuente puede mostrar cualquiera de las formas en un bucle.

En el`-O0`Ninguno de ellos existe.`clamp_max`es una comparación, una `jle`El reconocedor no informa nada por ello en el`-O0`y `branchless select`de la`-O1`, que es uno de los hechos que la figura hace visibles.

`mul9`Es uno .`leal (%rdi,%rdi,8), %eax`, y los informes del reconocedor `strength reduction`con`x * 9`, porque la fuente se multiplica por 9. Sólo lo hace cuando la fuente tiene el factor; a `lea`que escalas un índice de tabla no es una multiplicación.

## Dos características que hacen que el corazón se endurezca

El protector de pila es una característica de compilador, y deja una forma que un agente leerá de otra manera como lógica del programa. `canary.c`Compilado con `-fstack-protector-strong`para x86-64 Linux da:

```text
       7: 64 48 8b 04 25 28 00 00 00   	movq	%fs:0x28, %rax
      10: 48 89 44 24 40               	movq	%rax, 0x40(%rsp)
      26: 64 48 8b 0c 25 28 00 00 00   	movq	%fs:0x28, %rcx
      2f: 48 3b 4c 24 40               	cmpq	0x40(%rsp), %rcx
      34: 75 05                        	jne	0x3b <with_canary+0x3b>
      3b: e8 00 00 00 00               	callq	0x40 <plain_leaf>
		000000000000003c:  R_X86_64_PLT32	__stack_chk_fail-0x4
```

`%fs:0x28`¿ Es verdad ?`stack_guard`en Glibc's `tcbhead_t`, en byte offset 40, y LLVM `getIRStackGuard`El prólogo copia la guardia sobre el amortiguador, el epílogo compara y el camino de fracaso llama`__stack_chk_fail`En AArch64 Linux el guardián proviene del global .`__stack_chk_guard`a través de la GOT con `adrp`y `ldr`, y el fracaso es`bl __stack_chk_fail`El reconocedor informa la fuente de la guardia, la ranura, la comparación y la llamada de fallo.`plain_leaf`, que no tiene array, no tiene canario bajo `strong`, que protege las funciones con matrices,`alloca`Las notas de investigación también registran el formulario MSVC en el apartado`/GS`, una galleta mezclada con `%rsp`y verificado por `__security_check_cookie`; no existe en esta máquina un enlace MSVC, por lo que la forma se describe únicamente a partir de la salida del montaje del compilador.

`-fcf-protection=full`Pone en el`endbr64`, bytes `f3 0f 1e fa`Es una plataforma de aterrizaje para ramas indirectas en procesadores con control de flujo de ejecución, y se ejecuta como un`nop`Los patrones de inicio de la función x86-64 de Ghidra incluyen el`endbr64`Variantes prefijadas de `push rbp; mov rbp, rsp`El reconocedor lo informa como:`cet landing pad`, así que una función que comienza con cuatro bytes inusuales no se confunde con datos.

## Lo que los decomponedores doblan

Cada decompiler en la lección 37 ejecuta algunas de estas derivaciones como reglas de reescritura.`ruleaction.cc`- ¿ Qué ?`RuleDivOpt`es "Convertir INT_MULT y cambiar los formularios a INT_DIV o INT_SDIV", con los formularios `sub( (zext(V)*c), d) >> e => V / (2^n/(c-1)) where n = d*8 + e`y la variante firmada con `sext`y `s>>`- ¿ Qué ?`RuleModOpt`es "Simplificar expresiones que optimizan INT_REM e INT_SREM". `RuleDivTermAdd`maneja la corrección adicional, `RuleSignDiv2`- ¿ Qué ?`(V + -1*(V s>> 31)) s>> 1`en el`V s/ 2`, y `RulePositiveDiv`La lección 37 capturó Ghidra 12.1.4 en la lección 33's `grade`En el`-O1`, y la salida se lee `uVar2 = param_1 / 10 - 6;`La división está doblada.

La entrada a esas reglas es visible sin Java. el pypcode de venv 4.0.1 se eleva con la misma especificación SLEIGH que utiliza Ghidra, y `code/fixtures/idioms-x86_64-O2.lifts.json`tiene el código P bruto para `div10s`: 112 entradas para siete instrucciones, de las cuales 7 son las instrucciones`IMARK`marcadores que abren cada instrucción, entre ellos `RAX = RAX * unique[95a00:8]`después de`unique[95a00:8] = sext(0x66666667)`¿ Qué ?`RCX = RCX >> unique[d1a00:4]`, y `RAX = RAX s>> unique[c5c00:4]`- El .`shrq`La Comisión de Agricultura y Desarrollo de la Agricultura de la Comunidad Europea (CE)`sarq`lleva 40, y la mayoría de ellos cuentan las banderas.`RuleDivOpt`coincide con el multiplicador y los dos cambios en esta corriente, y la eliminación de código muerto elimina las banderas que nada lee.

Angr 10.0.3 se ejecutó en nuestro `-O2`objeto, y `idioms-x86_64-O2.angr.json`Tiene su salida de decompiler para 20 funciones.`div10u`¿ Es verdad ?`return a0 / 10;`No dobla el formulario firmado: `div10s`¿ Es verdad ?`v1 = a0 * 1717986919; return ((long long)(v1) >> 0x22) + (v1 >> 63);`y `mod10s`¿ Es verdad ?`a0 - (...) * 2 * 5`- Para .`div7u`Imprima .`(a0 / 7 + ((a0 * 6 / 7 & 0xffffffff) >> 1 & 0xffffffff) & 0xffffffff) >> 2`, donde los términos de fijación sobreviven como aritmética.`mul9`¿ Qué ?`a0 * 9`¿ Qué ?`linear_switch`¿ Qué ?`(a0 < 7 ? a0 * 10 + 10 : 0xffffffff)`¿ Qué ?`clamp_max`¿ Qué ?`((int)a1 < (int)a0 ? a0 : a1)`¿ Qué ?`is_pos`¿ Qué ?`a0 > 0`, y `fill`¿ Qué ?`return memset(a0, 0, a1);`Imprime .`table_switch`como carga .`*(int *)(4 * a0 + 4195400)`sin los valores, `sum_n`con`_mm_add_epi32`y `_mm_shuffle_epi32`las características intrínsecas y `tail_caller`¿ Cómo ?`tail_target(); return;`Con el argumento perdido.`digest`La mesa de salto se convierte en`goto *((void *)(...))`con la aritmética de la tabla escrita.

Radar2 6.2.4 y rizin 0.9.1, construidos desde la fuente en el caché de investigación, se ejecutaron en el mismo objeto.`pdc`huellas`div10s`¿ Cómo ?`rax = rax * 0x66666667; rcx >>>= 0x3f; rax >>= 0x22; eax += ecx`, pseudo-C de nivel de registro sin plegamiento, y `table_switch`¿ Cómo ?`eax = dword [rcx + rax*4]`con un `RELOC 32 .rodata`comentario sobre el `lea`. rizin no tiene un descompilador incorporado; su `aoi`Imprima el RzIL de la `imul`como un producto de 128 bits. lldb-1703.0.236.21 carga el objeto ELF y su `disassemble -n div10s`Imprime las mismas siete instrucciones que `llvm-objdump`, porque ambos utilizan el desmonteador de LLVM. Capstone 5.0.7 decodifica los mismos bytes y informa grupos como `ret`y `mode64`La documentación de IDA Pro y Binary Ninja no se ejecuta aquí. Su documentación describe las etapas en las que se produce dicha reescritura, la optimización del microcodo y el MLIL a HLIL pasa, y esta lección no hace ninguna afirmación sobre qué idiomas doblan.

El compilador Explorer es la herramienta para la dirección hacia adelante.`idioms.c`, elige una entrada de compilador como "x86-64 clang 18.1.0" o "ARM64 GCC 14.2.0", y establece las banderas. El servicio enumera alrededor de 1.050 entradas de compilador C y tiene una API: `GET /api/compilers/c`devuelve los identificadores y `POST /api/compiler/<id>/compile`Retorna la asamblea como JSON. El artículo ACM Queue de Matt Godbolt de 2019 recorre la división, el conteo de población, los idiomas condicionales en cadena y la sumación con esta herramienta.`-O2`vectorization defectuosos de GCC 12 con la salida de clang en las fichas.

## Construye el mismo

Ejecutar la demostración:

```bash
cd phases/13-tools-and-protocols/39-compiler-idioms-and-optimization
python3 code/main.py
```

La salida es un informe JSON. Para cada una de las 18 ediciones enumera todas las funciones y sus idiomas, con tipo, forma de fuente, direcciones de evidencia, verificación y los campos de detalles.`-O2`En comparación`-O3`La comparación de ambas arquitecturas y la`unverified`- ¿Qué es eso?

Forma esperada:

```json
{
  "builds": {
    "idioms-x86_64-O2": {
      "arch": "x86_64", "level": "O2", "flags": "x86_64",
      "functions": {
        "div7u": [{"kind": "division", "source": "x / 7", "evidence": ["0x30", "0x32", "0x39", "0x3d", "0x3f", "0x41", "0x43"], "verified": true, "form": "magic multiply with add fix-up", "magic": "0x24924925", "effectiveMagic": "0x124924925", "shift": 35, "...": "..."}],
        "table_switch": [{"kind": "switch", "source": "switch (k) as lookup table", "table": ".rodata+0x28", "entries": 7, "values": [10, 20, 30, 40, 50, 60, 71], "...": "..."}],
        "sum_n": [{"kind": "vector loop", "lanes": 4, "accumulators": 2, "elementsPerIteration": 8, "...": "..."}]
      }
    }
  },
  "kindMatrix": {"idioms-x86_64-O2": {"division": 5, "modulo": 3, "switch": 6, "vector loop": 3, "...": "..."}},
  "sameCodeAtO2AndO3": {"x86_64": {"identical": true}, "arm64": {"identical": false, "differences": ["..."]}},
  "unverified": []
}
```

Leer .`code/main.py`en este orden:

1. `parse_listing()`vuelve el `llvm-objdump -d -r`el texto en funciones, instrucciones y reubicaciones adjuntas para ambas arquitecturas.
2. `ElfFile`Leer secciones, símbolos y `.rela.*`las entradas y `words()`Leer entradas de la mesa.
3. `magic_numbers()`¿ Qué ?`divisor_from_magic()`, y `evaluate_division()`son la derivación Granlund-Montgomery hacia adelante, hacia atrás, y como la secuencia de instrucciones.
4. `parse_source()`extrae las construcciones de las necesidades de verificación del archivo C.
5. `Evaluator`es el pase simbólico bloque por bloque que registra nombres y espacios de pila.
6. `find_divisions()`y `find_modulos()`coincide con el multiplicador, el desplazamiento, el arreglado, el multiplicador y el poder de dos formas.
7. `find_switches()`clasifica tablas de saltos, tablas de búsqueda, mapas lineales, seleccionar cadenas, árboles de comparación y comparar cadenas.
8. `find_vector_loops()`¿ Qué ?`find_unrolled_loops()`¿ Qué ?`find_tail_calls()`¿ Qué ?`find_inlined_calls()`¿ Qué ?`find_selects()`¿ Qué ?`find_memory_builtins()`¿ Qué ?`find_stack_protector()`¿ Qué ?`find_cet()`, y `find_strength_reduction()`cubre el resto.
9. `analyze_function()`los corre en orden, alimentando el nombre del resultado de cada idioma en el siguiente pase de evaluador.

Hay tres órdenes más.`python3 code/main.py scan FILE.objdump.txt [FUNCTION]`ejecuta el reconocedor en una lista, con el objeto y el archivo C correspondiente que se encuentra al lado. `python3 code/main.py magic DIVISOR [signed]`Imprime el multiplicador, el cambio, el error y si la identidad se mantiene en los valores de borde. `python3 code/main.py figure-data`Imprime los datos que la figura incorpora.

## Usalo

Utilice el reconocedor en este orden en un nuevo objeto que construyó.

1. Captura .`llvm-objdump -d -r`El reconocedor lee la arquitectura desde el encabezado de la lista.
2. - ¿ Qué ?`scan`en la lista y leer el `unverified`Cada uno es una construcción que el analizador de origen no vio o una forma que el reconocedor ha interpretado mal, y ambos merecen una mirada a las direcciones de la evidencia.
3. Para cada división, lee el`algorithm`campo. ¿Cuándo?`clangMatchesAlgorithm`Si es falso, el compilador tenía una razón que aún no sabes; escribe las constantes.
4. Para cada interruptor, comparar los`lowering`Una tabla que se convierte en un árbol, o un árbol que se convierte en una tabla, marca un umbral que se puede nombrar a partir de la fuente LLVM.
5. Para cada bucle, lee.`elementsPerIteration`y `scalarRemainderLoop`, entonces confirme que los dos bordes de atrás de la lección 33 informes son un bucle de fuente.
6. ejecuta un decompiler en el mismo objeto, cita su texto junto a cada idioma, y graba cuáles doblaron. Ese registro es la línea de base para la lección 45.

## Laboratorio interactivo

### Laboratorio A: derivar un número mágico, y luego encontrarlo

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/39-compiler-idioms-and-optimization/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
magic_numbers(10), magic_numbers(10, signed=True)
magic_numbers(7), magic_numbers(7, signed=True)
divisor_from_magic(0x66666667, 34, signed=True), divisor_from_magic(0x66666667, 33, signed=True)
```

Computación`ceil(2^35 / 10)`en papel antes de la primera llamada. Luego añadir una función `unsigned div13(unsigned x) { return x / 13; }`a una copia de `idioms.c`, construye en `-O2`con el mando en `build_fixtures.sh`, y comprobar la constante en la lista contra `magic_numbers(13)`¿ Qué ?

### Laboratorio B: la mesa y el árbol

```python
listing = load_listing("idioms-x86_64-O2.objdump.txt")
elf = load_elf("idioms-x86_64-O2.o")
sources = load_source("idioms.c")
[i.to_dict() for i in analyze_function(listing, "table_switch", elf, sources)]
[i.to_dict() for i in analyze_function(listing, "sparse_switch", elf, sources)]
elf.words(".rodata", 0x28, 7)
```

Cambiar`71`De vuelta a`70`en una copia de `table_switch`, reconstruir en`-O2`, y ver la tabla de búsqueda se convierte en un mapa lineal. Luego cambiar un valor de caso a 1000 y ver que se mantiene un`ArrayKind`¿Qué es?

### Laboratorio C: el bucle en tres configuraciones

```python
for name in ("idioms-x86_64-O1", "idioms-x86_64-O2", "idioms-x86_64-novec-O2"):
    print(name, [i.source for i in analyze_function(load_listing(name + ".objdump.txt"), "sum_n", load_elf(name + ".o"), sources)])
```

Dibujar el gráfico de flujo de control de `sum_n`En el`-O2`de mano, marque el guard, la máscara, el cuerpo vectorial, la reducción y el ciclo restante, y compare con el `detail`los campos.

### Laboratorio D: el cuotiente compartido

```python
for name in ("idioms-x86_64-O1", "idioms-arm64-O1"):
    print([(i.kind, i.source, i.detail.get("sharedQuotient")) for i in analyze_function(load_listing(name + ".objdump.txt"), "digest", load_elf(name + ".o"), sources)])
```

Lea el `-O1`lista de `digest`y encontrar las dos instrucciones que combinan el cuotiente con su señal de fijación en x86-64.

## Laboratorio de práctica

Añade trucos a los accesorios y amplía el reconocedor.

Requisitos:

- Escriba .`popcount`¿ Qué ?`ctz`, y `rotl`Las funciones de la construcción de los CCG y un cambio o rotación, construirlos en `-O2`con y sin`-mpopcnt -mbmi`, y comprometer los objetos con sus listas.
- Reconozca el `0x55555555`¿ Qué ?`0x33333333`¿ Qué ?`0x0f0f0f0f`¿ Qué ?`0x01010101`Secuencia de recuento de población que `TargetLowering::expandCTPOP`emitir, y reportarlo como `popcount`con las cuatro constantes como evidencia.
- Reconocerlo`rep bsfl`como el `tzcnt`- O - - ¿Qué es eso?`bsf`codificación y `roll %cl`como el girar, y descifrar ambos a la construcción.
- Añadir los nuevos tipos a `parse_source()`la verificación, por lo que una `popcount`El idioma sólo se verifica cuando la fuente llama `__builtin_popcount`¿ Qué ?
- Añadir una prueba por idioma que lee las constantes de la lista de fijación.

El éxito significa`python3 code/main.py`Las nuevas especies se han introducido en la lista de los Estados miembros.`verified: true`En ambas construcciones, y la suite todavía informa de un vacío`unverified`- ¿Qué es eso?

## Lección Artefacto

Esta lección incluye `outputs/skill-compiler-idioms-and-optimization.md`. Utilice para convertir una lista, un objeto y su fuente en un informe de idioma verificado con una comparación de descompilador. Requiere direcciones de evidencia para cada afirmación, la derivación de número mágico para cada división, el tipo de bajada y la fuente de entrada para cada interruptor, y una línea de descompilador citada para cada idioma.

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/39-compiler-idioms-and-optimization
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

La verificación deberá demostrar:

- el parser de lista lee 25 funciones de la `-O2`Objeto, mantiene los bytes, y se adhiere `R_X86_64_PC32 .rodata+0x24`a la `leaq`En el`table_switch`
- El lector de ELF devuelve `[10, 20, 30, 40, 50, 60, 71]`de la`.rodata+0x28`y diez reubicaciones en `.rela.rodata`
- `magic_numbers`retorno `0xcccccccd`con el turno 35, `0x66666667`con 34, `0x124924925`con 35 y el formulario adicional, `0x55555556`con 32, y `0xaaaaaaab`con 33
- la identidad mantiene los valores de borde de 32 bits para cada divisor de 2 a 64 que no sea un poder de dos, firmado y no firmado
- `divisor_from_magic`recupera 10, 5, 7 y 3 de sus pares y rechaza un par equivocado
- el computo de la secuencia de fijación de la adición `n // 7`exactamente
- Las cuatro divisiones x86-64 informan el multiplicador derecho, el cambio, el divisor y la forma, y las cuatro divisiones AArch64 decodifican `mov`y `movk`
- `-O0`usos `divl`y `idivl`¿ Qué ?`-O0 -fast-isel=false`utiliza el multiplicador, y `digest`En el`-O0`mezcla las dos
- el modulo se reconoce como el multiplicador de la vuelta, `msub`, potencia de dos, `sdiv`Además`mul`Además`subs`, y el hardware `idivl`
- un cuotiente compartido mantiene la división y el modulo en `-O1`
- Las cinco reducciones se clasifican en `-O2`En ambas arquitecturas, con la fórmula, los valores, los objetivos, los pivotes y la expresión seleccionada
- `-O0`los interruptores son tablas de salto en x86-64 y comparar cadenas en AArch64
- el `digest`el switch es una tabla de saltos, luego una tabla de búsqueda con factor 16 doblado, luego una tabla de saltos con cuatro objetivos distintos en x86-64, y un árbol de comparación con el test de rango 9 a 10 en AArch64
- `-fno-jump-tables`produce árboles de comparación, y el caso 16 `dispatch`lee sus objetivos de las reubicaciones en x86-64 y de una tabla de byte comprimida en AArch64
- El bucle vectorial informa 4 carriles, 2 acumuladores, 8 elementos por iteración, la máscara, la guardia, la reducción y el bucle restante
- Los bucles de trayecto fijo están totalmente vectorizados o completamente desarrollados, y `-fno-vectorize`desdoblaciones a partir de 8
- `-O1`no tiene vectorización, `-O2`y `-O3`son idénticos en x86-64, y en AArch64 sólo difieren en `call_switch`, donde `-O3`Tiene 17 instrucciones y `-O2`tiene 18
- `-fno-builtin`transforma el bucle de la tienda en un bucle de la tienda vectorizado y mantiene `memcpy`una llamada
- Las llamadas de cola, el `memset`la copia en línea de 16 bytes, y el `-O0`se reconocen las copias de dos movimientos
- La calle en línea se encuentra en `-O1`por su forma de cuerpo
- selecciona el decodificación a `a > b ? a : b`¿ Qué ?`-x < 0 ? x : -x`¿ Qué ?`(x > 0)`, y `sum % 10 < 5 ? sum % 10 : 5`
- los informes de protección de la pila `%fs:0x28`, la ranura, y `__stack_chk_fail`, y `endbr64`se informa sobre cada función de la estructura de la CET
- el parser de origen lee los parámetros de función-pointer, los casos de interruptores, las llamadas y los nombres del acumulador
- Cada idioma en cada construcción se verifica y las matrices de tipo coinciden
- La figura interactiva incorpora los mismos datos que el análisis produce

## Modos de falla de producción

| Failure | What the weak tool reports | What the analysis must prove |
|---|---|---|
| Multiply by a large odd constant | "multiplies by 1717986919" | The constant is `ceil(2^s / d)` for one `d`, the error satisfies the bound, and the shift is the smallest valid one |
| Add fix-up after a 32-bit shift | Five arithmetic instructions | The multiplier is 33 bits wide and the tail is the add form of the same method |
| Same multiplier, different shift | The same divisor | The shift selects the divisor: 34 is 10, 33 is 5 |
| Power-of-two signed modulo | "adds 7, masks, subtracts" | The bias rounds toward zero and the result is `x % 8` |
| Indexed load from `.rodata` | "a jump table" | A lookup table of values, with entries read from the bytes at addend plus 4 |
| Affine arithmetic and a `cmov` | "computes 10k + 10" | A switch lowered as a linear map, verified against every source case |
| Compare chain with ordered splits | "a sequence of ifs" | A binary search over the case constants with the pivots named |
| Backward branch to a default block | "a loop" | The target leaves the region with an unconditional jump; it is a shared tail |
| Two back edges in one function | "two loops" | One source loop: the vector body and its scalar remainder |
| `jmp memset@PLT` with no `ret` | "the function has no return" | A tail call that returns through the callee, replacing a store loop |
| Callee missing from the call graph | "never called" | Inlined: the callee's body shape appears in the caller |
| `%fs:0x28` load and compare | "reads thread-local data" | The stack protector canary, with the fail path to `__stack_chk_fail` |
| Four odd bytes at function start | "data before the function" | `endbr64`, the CET landing pad |
| Decompiler shows raw arithmetic | "the tool is wrong" | The tool did not apply a fold; the derivation gives the source form and the quoted output records the gap |

## Regla de funcionamiento

Compare la forma, ejecuta la derivación hacia atrás y escribe las constantes que lo prueban. Informar cada idioma con sus direcciones de evidencia y su forma de fuente decodificada, verificar cada uno contra la fuente que compila, y citar lo que cada decompiler impreso para los mismos bytes.

## Leer más

- [Granlund, Montgomery, "Division by Invariant Integers using Multiplication" (PLDI 1994)](https://gmplib.org/~tege/divcnst-pldi94.pdf)
- [Warren, Hacker's Delight, 2nd edition, Chapter 10: Integer Division By Constants](https://www.informit.com/store/hackers-delight-9780321842688)
- [Lemire, Kaser, Kurz, "Faster Remainder by Direct Computation" (2019)](https://arxiv.org/abs/1902.01961)
- [libdivide, header-only division by runtime constants](https://github.com/ridiculousfish/libdivide)
- [LLVM `DivisionByConstantInfo.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Support/DivisionByConstantInfo.cpp)
- [LLVM `SwitchLoweringUtils.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/CodeGen/SwitchLoweringUtils.cpp)y [`TargetLoweringBase.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/CodeGen/TargetLoweringBase.cpp)
- [LLVM `SimplifyCFG.cpp`, `simplifySwitchLookup`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SimplifyCFG.cpp)
- [LLVM Auto-Vectorization documentation](https://llvm.org/docs/Vectorizers.html)y [LLVM 3.4 release notes](https://releases.llvm.org/3.4/docs/ReleaseNotes.html)
- [LLVM `LoopIdiomRecognize.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopIdiomRecognize.cpp)
- [LLVM `X86CmovConversion.cpp`](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/lib/Target/X86/X86CmovConversion.cpp)
- [LLVM `StackProtector.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/CodeGen/StackProtector.cpp)y [`X86ISelLoweringCall.cpp`, `getIRStackGuard`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Target/X86/X86ISelLoweringCall.cpp)
- [glibc `sysdeps/x86_64/nptl/tls.h`, `tcbhead_t`](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/nptl/tls.h)
- [GCC `tree-switch-conversion.cc`](https://github.com/gcc-mirror/gcc/blob/master/gcc/tree-switch-conversion.cc)¿ Qué ?[GCC Optimize Options](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html), y [GCC 12 release notes](https://gcc.gnu.org/gcc-12/changes.html)
- [Ghidra decompiler `ruleaction.cc`](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/Decompiler/src/decompile/cpp/ruleaction.cc)
- [angr decompiler documentation](https://docs.angr.io/en/latest/analyses/decompiler.html)
- [radare2 book: decompilers](https://book.rada.re/arch/decompile.html)
- [Compiler Explorer API](https://github.com/compiler-explorer/compiler-explorer/blob/main/docs/API.md)
- [Godbolt, "Optimizations in C++ Compilers: A practical journey", ACM Queue 17(5), 2019](https://queue.acm.org/detail.cfm?id=3372264)
- [Microsoft, `/GS` (Buffer Security Check)](https://learn.microsoft.com/en-us/cpp/build/reference/gs-buffer-security-check)
