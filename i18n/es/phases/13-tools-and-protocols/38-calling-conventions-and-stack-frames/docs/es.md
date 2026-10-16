# Convenciones de llamadas y marcos de pila: Sistema V, Windows x64 y AArch64

> Una desmontaje muestra qué registros una función lee. Sólo el ABI dice lo que esos registros significan, y el ABI es elegido por el encabezado del objeto, por lo que esta lección lee el encabezado primero y luego verifica cada registro contra la regla que lo puso allí.

**Type:** Build
**Languages:** Python, C, x86-64 and AArch64 assembly
**Prerequisites:** Phase 13 · 32 (ELF from scratch), Phase 13 · 33 (disassembly and control flow)
**Time:** ~140 minutes

## Objetivos de aprendizaje

- Seleccione la convención de llamada del encabezado de objeto: ELF x86-64 es el sistema V psABI, COFF AMD64 es Microsoft x64, ELF AArch64 es AAPCS64, Mach-O arm64 es la variante de Apple.
- Decodifique el prólogo de una función byte por byte para cada ABI y coloque cada registro guardado, la dirección de devolución y el puntero de marco en su desvio del puntero de entrada.
- Inserir los registros de parámetros de una función de los registros que lee antes de escribirlos, y las ranuras de argumentos de pila de las cargas por encima del puntero de pila de entrada.
- Cuéntale a las secuencias de registro independientes del Sistema V desde las ranuras de posición de Microsoft x64, y tanto desde los contadores NGRN y NSRN de AAPCS64.
- Verifique cada inferencia con la firma de origen, la regla en la especificación y los propios datos de desvio del compilador en `.pdata`¿ Qué ?`.xdata`, y `.eh_frame`¿ Qué ?
- Comparar cómo Ghidra, IDA, Binary Ninja, radar2, rizin y angr nombre y recuperar los mismos hechos, y registrar donde su salida no está de acuerdo con la fuente.

## El problema

Usted compilado`convention.c`para Linux x86-64 en `-O2`y pidió a radar2 6.2.4 los argumentos de `eight_args`Respondió con siete:`rdi`¿ Qué ?`rsi`¿ Qué ?`rdx`¿ Qué ?`r8`¿ Qué ?`r9`¿ Qué ?`rsp+0x8`, y `rsp+0x10`La fuente C tiene ocho parámetros.`%rcx`exactamente una vez, en `leaq (%rdx,%rcx), %rax`, y la recuperación variable de la herramienta no contaba un registro que sólo aparece dentro de una dirección.

Luego abrió la versión de Windows del mismo archivo en rizin 0.9.1.`amd64`y encontró`r8`¿ Qué ?`rcx`¿ Qué ?`rdx`El objeto es COFF, así que la convención es Microsoft x64, el cuarto argumento de registro es `r9`Las dos herramientas pierden registros diferentes en la misma fuente.

Una lista es una secuencia de instrucciones. Un convenio de llamada es un contrato que se encuentra fuera de la lista: qué registros llevan argumentos, que el solicitante debe conservar, dónde se encuentran la dirección de retorno y los argumentos de pila, y qué debe dejar el solicitante por encima del indicador de pila. Cuatro documentos definen los contratos que importan aquí. El sistema V AMD64 psABI (versión 1.0, edificación de 12 de marzo de 2025) secciones 3.2.1 a 3.2.3. La convención de llamadas de Microsoft x64, el uso de la pila y las páginas de prólogo y epílogo (fechas de página de 2018-12-17 a 2025-03-19). El Armo AAPCS64 libera 2025Q4 (23 de enero de 2026) las secciones 6.1 a 6.9. Las notas de la plataforma arm64 de Apple. Esta lección lee esas reglas, las aplica a un archivo C compilado de tres maneras, y construye el analizador que convierte una lista en una lista revisada de ubicaciones de parámetros, registros guardados y offsets de cuadros.

```figure
re-38-three-frames
```

## Un titular escoge el libro de reglas

`convention.c`tiene doce funciones: `leaf_add`¿ Qué ?`eight_args`¿ Qué ?`mixed`¿ Qué ?`by_value`¿ Qué ?`big_ret`¿ Qué ?`sum_varargs`¿ Qué ?`keeps`, y cinco llamantes de `extern`Las estructuras utilizan`long long`, porque`long`Es 4 bytes en el objetivo MSVC y 8 bytes en los demás, y una estructura que cambia de tamaño cambios convención.`-target x86_64-unknown-linux-gnu`¿ Qué ?`x86_64-pc-windows-msvc`, y `aarch64-unknown-linux-gnu`En el`-O0`y `-O2`, y `arm64-apple-macos`En el`-O2`No hay encabezados de Linux o Windows instalados, por lo que el archivo declara lo que necesita e incluye solo los propios de Clang `stdarg.h`y `stddef.h`¿ Qué ?

`describe_object()`En el`code/main.py`lee el número mágico de 4 bytes y un campo de máquina y nombra el contrato:

| Header | Field | Value | Rulebook |
|---|---|---|---|
| `7f 45 4c 46` ELF | `e_machine` at offset 0x12 | 62 | System V AMD64 psABI |
| `64 86` COFF | `Machine` at offset 0 | 0x8664 | Microsoft x64 |
| `7f 45 4c 46` ELF | `e_machine` | 183 | AAPCS64 |
| `cf fa ed fe` Mach-O 64 | `cputype` at offset 4 | 0x0100000c | AAPCS64 with Apple differences |

Los tres contratos no están de acuerdo en casi todas las filas de la tabla siguiente.

| Rule | System V psABI | Microsoft x64 | AAPCS64 |
|---|---|---|---|
| Integer arguments | `%rdi %rsi %rdx %rcx %r8 %r9`, next free register | `RCX RDX R8 R9`, by position | `x0` to `x7`, by NGRN |
| Floating-point arguments | `%xmm0` to `%xmm7`, next free register | `XMM0` to `XMM3`, by position | `v0` to `v7`, by NSRN |
| Integer result | `%rax`, then `%rdx` | `RAX` | `x0`, then `x1` |
| Floating-point result | `%xmm0`, then `%xmm1` | `XMM0` | `v0` |
| Result too large for registers | address passed in `%rdi` "as if it were the first argument", returned in `%rax` | pointer in `RCX`, arguments shifted right, returned in `RAX` | address in `x8`, `x0` to `x7` untouched |
| Callee-saved | `%rbp %rbx %r12` to `%r15` | `RBX RBP RDI RSI RSP R12` to `R15`, `XMM6` to `XMM15` | `r19` to `r29`, `SP`, low 64 bits of `v8` to `v15` |
| First stack argument on entry | `8(%rsp)` | `40(%rsp)`, above a 32-byte shadow store | `0(sp)` |
| Below the stack pointer | 128-byte red zone | "considered volatile" | no red zone (Apple: 128 bytes) |
| Alignment at the call | `(%rsp + 8)` multiple of 16 on entry | 16-byte aligned outside prolog and epilog | `SP mod 16 = 0` |
| Variadic marker | `%al` holds an upper bound on vector registers used | floats duplicated in the integer register | none in registers; Apple uses 8-byte stack slots |

## Sistema V: Dos secuencias de registro independientes y una zona roja

psABI 3.2.3 clasifica cada ocho bytes de cada argumento como INTEGER, SSE o MEMORY y luego asigna registros de dos listas que no interactúan. "INTEGER" toma "el siguiente registro disponible de la secuencia %rdi, %rsi, %rdx, %rcx, %r8 y %r9". SSE toma `%xmm0`¿ Qué ?`%xmm7`. "Si no hay registros disponibles para cualquier ocho bytes de un argumento, todo el argumento se pasa en la pila", y los argumentos de memoria "se empujan en la pila en orden inverso (derecha a izquierda).

`eight_args(a, b, c, d, e, f, g, h)`En el`-O2`es 27 bytes:

```text
      10: 48 01 fe                     	addq	%rdi, %rsi
      13: 48 8d 04 0a                  	leaq	(%rdx,%rcx), %rax
      17: 48 01 f0                     	addq	%rsi, %rax
      1a: 4c 01 c0                     	addq	%r8, %rax
      1d: 4c 01 c8                     	addq	%r9, %rax
      20: 48 03 44 24 08               	addq	0x8(%rsp), %rax
      25: 48 03 44 24 10               	addq	0x10(%rsp), %rax
      2a: c3                           	retq
```

Seis cajas, luego dos cargas en `8(%rsp)`y `16(%rsp)`. La figura 3.3 del psABI pone "argumento de memoria 8byte 0" en `16(%rbp)`¿ Cuándo ?`%rbp`es la base del marco, que es `8(%rsp)`en la entrada antes de cualquier empuje. `g`es a las 8, `h`a los 16 años.

`mixed(int i, double d, long long l, float f)`En el caso de las listas de los dos, las listas de los dos indicios son:`%edi`y `%rsi`El doble y el flotante toman`%xmm0`y `%xmm1`La primera instrucción es:`cvtsi2sd %edi, %xmm2`, y `%xmm2`Es un registro de rasguño que la función escribe antes de leer.`cvtsi2sd`como una lectura inventa un tercer argumento de punto flotante. radar2 hizo exactamente eso y informó`mixed`¿ Cómo ?`(double arg1, double arg2, double arg3)`En el`xmm0`¿ Qué ?`xmm1`¿ Qué ?`xmm2`¿ Qué ?

`by_value(struct pair p)`pasa un estruto de 16 bytes como dos 8 bytes INTEGER en `%rdi`y `%rsi`, y lo devuelve de la misma manera: `leaq 0x1(%rdi), %rax; leaq -0x1(%rsi), %rdx; retq`- ¿ Qué ?`big_ret(long long seed)`devuelve 32 bytes, que es la memoria de clase, y el psABI dice "el llamador proporciona espacio para el valor de retorno y pasa la dirección de este almacenamiento en %rdi como si fuera el primer argumento". `seed`se mueve a`%rsi`, y la función comienza con `movq %rdi, %rax`porque "En la devolución %rax contendrá la dirección que ha sido introducida por el solicitante en %rdi".

La zona roja es la otra firma del Sistema V. psABI 3.2.2: "La zona de 128 bytes más allá de la ubicación señalada por %rsp se considera reservada y no puede ser modificada por los manipuladores de señales o interrupciones".`-O0`¿ Qué ?`eight_args`¿ Es verdad ?`pushq %rbp; movq %rsp, %rbp`y luego seis tiendas a `-0x8(%rbp)`por el`-0x30(%rbp)`sin ninguna`subq`Las tiendas están a 48 bytes de la señal de pila.`-O2`construcción de `sum_varargs`asigna 88 bytes con `subq $0x58, %rsp`y luego escribe el registro de la zona de guardar en `-0x58(%rsp)`hasta`-0x80(%rsp)`: 128 bytes por debajo del indicador de pila, toda la zona roja.`-mno-red-zone`y `leaf_add`ganancias de un `pushq %rax`y un `addq $0x8, %rsp`para mantener sus 8 bytes de locales por encima del puntero de pila.

Las llamadas variádicas añaden un registro. "%al se utiliza como argumento oculto para especificar el número de registros vectoriales utilizados", un límite superior de 0 a 8. `call_varargs`lo pone con `xorl %eax, %eax`En el`-O2`y `movb $0x0, %al`En el`-O0`- ¿ Qué ?`sum_varargs`lo lee primero: `testb %al, %al; je`saltos ocho `movaps %xmmN`Se guarda cuando no se utilizó registro de vectores.

## Microsoft x64: Posiciones, una tienda de sombras y ninguna zona roja

Las reglas de Microsoft son posicionales. "Los argumentos integrales se pasan en los registros RCX, RDX, R8 y R9. Los argumentos de puntos flotantes se pasan en XMM0L, XMM1L, XMM2L y XMM3L. " y "Cualquier argumento de punto flotante y de doble precisión en los primeros cuatro parámetros se pasan en XMM0 - XMM3, dependiendo de la posición". El mismo `mixed`llega como `%ecx`¿ Qué ?`%xmm1`¿ Qué ?`%r8`¿ Qué ?`%xmm3`: ranura 0 es una int, así que `RCX`La ranura 1 es un doble, así que `XMM1`La ranura 3 es flote, así que `XMM3`- ¿ Qué ?`XMM0`y `XMM2`No te muevas.

El llamador también debe dejar espacio. "El llamador siempre debe asignar suficiente espacio para almacenar cuatro parámetros de registro, incluso si el llamador no toma tantos parámetros". Las páginas lo llaman la tienda de sombra o las direcciones de casa, y las cuatro ranuras son "de manera efectiva propiedad de la función llamada". En la entrada `0(%rsp)`es la dirección de devolución, `8(%rsp)`¿ Qué ?`39(%rsp)`son las cuatro casas, y el primer argumento de pila es en `40(%rsp)`- ¿ Qué ?`eight_args`En el`-O2`Leía`0x28(%rsp)`¿ Qué ?`0x30(%rsp)`¿ Qué ?`0x38(%rsp)`¿ Qué ?`0x40(%rsp)`: 40, 48, 56, 64.

El lado de la llamada muestra la aritmética.`call8`En el`-O2`¿ Es verdad ?`subq $0x48, %rsp`y luego tiendas de 5, 6, 7, 8 en `0x20(%rsp)`¿ Qué ?`0x38(%rsp)`antes de`movl $1, %ecx`72 bytes es 32 para la tienda de sombra, 32 para cuatro argumentos de pila, y 8 para que la dirección de retorno sea en un límite de 16 bytes. `call_varargs`sólo registra argumentos y todavía asigna `subq $0x28, %rsp`: 32 para la tienda de sombras y 8 para la alineación. `keeps`empuja tres registros no volátiles (24 bytes) y luego `subq $0x20, %rsp`, así que eso`ext`encuentra su almacén de sombras de 32 bytes debajo de la dirección de retorno.

Los estructuras siguen su propia regla. "Cualquier argumento que no encaje en 8 bytes, o no sea 1, 2, 4, o 8 bytes, debe pasar por referencia". y "Argumentos de 16 bytes se pasan por referencia". `by_value`por lo tanto recibe un indicador de una copia en `%rdx`, después de un indicador de resultados oculto en `%rcx`El número de puntos de referencia de la base de datos de la base de datos de la base de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de`call_pair`construye la copia en `0x20(%rsp)`con`movaps`, pasa `leaq 0x20(%rsp), %rdx`, pasa el buffer de resultados con `leaq 0x38(%rsp), %rcx`, y lee`.b`de vuelta de`0x40(%rsp)`¿ Qué ?

No hay zona roja. "Toda memoria más allá de la dirección actual de RSP se considera volátil: el sistema operativo, o un desarreglador, puede sobrescribir esta memoria durante una sesión de desarreglamiento del usuario, o un manipulador de interrupciones. "`-O0` `leaf_add`Necesita 8 bytes de locales y los consigue con `pushq %rax`en la entrada y `popq %rcx`antes de`retq`, dos instrucciones de un byte en lugar de una `subq`y un `addq`. La página de uso de la pila define la hoja que puede hacer esto: "Una función de hoja es aquella que no requiere una entrada de tabla de funciones. No puede hacer cambios en ningún registro no volátil, incluido RSP".

Variadic llama a sus registros. "Para las funciones vararg o no prototipo, cualquier valor de punto flotante debe duplicarse en el correspondiente registro de uso general". `sum_varargs`En el`-O2`las tiendas `%rdx`¿ Qué ?`%r8`¿ Qué ?`%r9`¿ Qué ?`0x18(%rsp)`¿ Qué ?`0x20(%rsp)`¿ Qué ?`0x28(%rsp)`Después de uno .`pushq %rax`, que es la entrada de compensaciones 16, 24, 32: las viviendas de los parámetros del segundo, tercer y cuarto registro.`va_list`¿ Es verdad ?`leaq 0x18(%rsp), %rax`, el hogar de `%rdx`, el primer argumento sin nombre.

## AAPCS64, y lo que Apple cambia

AAPCS64 6.8.2 recorre una lista de reglas con dos contadores. C.9: un argumento integral "se copia a los bits menos significativos en x[NGRN]" y incrementos NGRN. C.1 hace lo mismo para argumentos de puntos flotantes con NSRN. C.12: un compuesto de al máximo `8 - NGRN`las palabras dobles se "copian en registros de uso general consecutivos". C.13 establece "El NGRN se establece en 8" una vez que un argumento se va a la pila, por lo que nada vuelve a registros después. `eight_args`usos `x0`¿ Qué ?`x7`y nunca toca la pila.`mixed`¿ Es verdad ?`w0`¿ Qué ?`d0`¿ Qué ?`x1`¿ Qué ?`s1`- ¿ Qué ?`by_value`¿ Es verdad ?`x0`y `x1`Entrando y saliendo.

6.9 dice que un resultado que se pasaría en registros "se devuelve en los mismos registros", de lo contrario "La dirección del bloque de memoria se pasará como argumento adicional a la función en x8". `big_ret`escribe `str x0, [x8]`¿ Qué ?`stur q0, [x8, #0x8]`¿ Qué ?`str x9, [x8, #0x18]`y nunca escribe.`x0`- ¿ Qué ?`call_big`conjuntos `mov x8, sp`, y `seed`Se queda en el .`x0`, porque 6.9 añade "no hay ningún requisito para que el callee preserve el valor almacenado en x8" y `x8`no es uno de los registros de argumentos.

6.4.5.2: en una interfaz pública "SP mod 16 = 0". La pila es completamente descendente y "No se permite acceder (para leer o escribir) a la región inactiva" debajo de ella, por lo que el Linux `-O0` `leaf_add`emite`sub sp, sp, #0x10`para 8 bytes de locales donde el Sistema V utilizó la zona roja.

6.4.6 define el registro de cuadros: "un registro de cuadros de dos valores de 64 bits en la pila ... el doble palabra dirigido más bajo debe indicar el registro de cuadros anterior y el doble palabra dirigido más alto debe contener el valor pasado en LR al ingresar a la función actual". y "La ubicación del registro de cuadros dentro de un marco de pila no está especificada".`keeps`En el`-O2`en Linux pone el registro en la parte inferior con `stp x29, x30, [sp, #-0x30]!; mov x29, sp`. La misma función en `-O0`lo pone en la cima con `sub sp, sp, #0x20; stp x29, x30, [sp, #0x10]; add x29, sp, #0x10`¿ Qué ?

Las notas arm64 de Apple mantienen los registros y la alineación y cambian el marco, la zona roja y las llamadas variadas. "El registro de marcador de marco (x29) siempre debe abordar un registro de marco válido. Algunas funciones, como las funciones de hoja o llamadas de cola, pueden optar por no crear una entrada en esta lista".`keeps`ahorra`x22, x21, x20, x19`primero y pone el récord en la cima: `stp x29, x30, [sp, #0x20]; add x29, sp, #0x20`. "La plataforma de reserva registro x18. No utilice este registro. " "La zona roja ARM64 consiste en los 128 bytes inmediatamente debajo del puntero de pila", y el clang todavía emitido `sub sp, sp, #0x10`para el `-O0`Los argumentos variádicos se asignan "a la cantidad apropiada de ranuras de pila de 8 bytes" y "el tipo va_list es un alias para char*". `call_varargs`en las tiendas de Apple 10, 20, 30 a `[sp]`¿ Qué ?`[sp, #8]`¿ Qué ?`[sp, #0x10]`y sólo pasa.`w0`; la construcción de Linux pasa `w1`¿ Qué ?`w2`¿ Qué ?`w3`y llamadas de cola con `b extva`La manzana .`sum_varargs`Leía`add x8, sp, #0x10`, el indicador de la pila del llamador, como su `va_list`El Linux `sum_varargs`ahorra`x1`¿ Qué ?`x7`y `q0`¿ Qué ?`q7`en un marco de 224 bytes e inicializa los campos de estructura `__gr_offs`-56 y `__vr_offs`-128 con uno `mov x8, #-0x38; movk x8, #0xff80, lsl #32`¿ Qué ?

Los símbolos también difieren. Mach-O prefija cada símbolo C con `_`, así que la lista muestra`_keeps`y la primera función aparece bajo la etiqueta privada `ltmp0`El analizador renombra las etiquetas a través de la tabla de símbolos.

## Decodifica un prólogo a mano, tres veces

`keeps(int n)`llamadas`ext(i)`en un bucle y mantiene `n`¿ Qué ?`i`, y `acc`En la llamada, cada ABI debe guardar registros.`-O2`Construir para cada objetivo.

Sistema V, 4 bytes:

1. `55`¿ Es verdad ?`push r64`con el registro en los tres bits más bajos: `50 + 5`¿ Qué ?`%rbp`- Profundidad 8,`%rbp`guardado en el desplazamiento de entrada -8.
2. `41 56`¿ Qué es esto ?`41`es REX con `B`conjunto, `56`¿ Es verdad ?`50 + 6`. REX.B se extiende de 6 a 14: `pushq %r14`En -16.
3. `53`¿ Es verdad ?`50 + 3`¿ Qué es esto ?`pushq %rbx`en -24.

Tres empujes son 24 bytes. La dirección de retorno agrega 8, y 32 es un múltiplo de 16, así que el `callq ext`dentro del bucle está alineado sin una `subq`No hay ninguna .`movq %rsp, %rbp`, así que esta construcción no tiene indicador de marco.`%rbp`como el registro de llamadas salvadas que contiene `acc`(El artículo`xorl %ebp, %ebp`¿ Qué ?`addl %eax, %ebp`El epílogo es el espejo:`5b 41 5e 5d c3`¿ Qué ?

Microsoft x64, 7 bytes:

1. `56`¿ Es verdad ?`pushq %rsi`en -8. `57`¿ Es verdad ?`pushq %rdi`En -16.`53`¿ Es verdad ?`pushq %rbx`En -24, los tres son no volátiles aquí, y`RDI`y `RSI`Se trataba de registros de argumentos del sistema V.
2. `48 83 ec 20`: REX.W, código op `83 /5`con un 8 bits inmediato, ModRM `ec`¿ Es verdad ?`11 101 100`¿ Qué es esto ?`subq $0x20, %rsp`- Profundidad 56.

56 más la dirección de regreso es 64, alineado. Los 32 bytes son la tienda de sombra que`ext`La página de epílogo sólo permite "añadir RSP, constante o lea RSP, constante [FPReg], seguido de una serie de cero o más 8 bytes de registro pops y un retorno o un jmp", y los bytes son `48 83 c4 20 5b 5f 5e c3`¿ Qué ?

AArch64, 16 bytes en cuatro palabras, poco-endian:

1. `a9bd7bfd`¿ Es verdad ?`stp`con pre-índice. Bits 31:30 `10`Seleccione registros de 64 bits, bits 25:23 `011`Seleccione pre-index, bit 22 es 0 para almacenar. imm7 en bits 21:15 es `1111010`, que es -6, escalado por 8: -48. Rt2 en 14:10 es 30, Rn en 9:5 es 31 (`sp`), Rt en 4:0 es 29. `stp x29, x30, [sp, #-0x30]!`: el puntero de pila cae 48, `x29`va a -48 y `x30`-40 por debajo.
2. `f9000bf5`¿ Es verdad ?`str`con un descenso sin firmar de 12 bits escalado por 8: imm12 a 21:10 es 2, Rn es `sp`, Rt es 21.`str x21, [sp, #0x10]`, compensación de entrada -32.
3. `910003fd`¿ Es verdad ?`add`inmediato: imm12 es 0, Rn es `sp`, Rd es 29.`mov x29, sp`: el marcador de marco apunta al registro en -48.
4. `a9024ff4`¿ Es verdad ?`stp`con un offset firmado: imm7 es 4, escalado a 32. Rt2 es 19, Rt es 20. `stp x20, x19, [sp, #0x20]`¿ Qué es esto ?`x20`en -16, `x19`en -8.

El marco es de 48 bytes: `x29`en 0, `x30`a las 8, `x21`en 16, 8 bytes de relleno en 24, `x20`en 32, `x19`La palabra más baja del registro apunta al registro anterior y la siguiente palabra tiene LR, como lo requiere 6.4.6. `cmp w0, #0x1`se encuentra entre la primera y la segunda tienda; el programador lo movió al prólogo, y el analizador tolera una comparación o una instrucción aritmética entre las operaciones de pila.

## El compilador escribió dos veces los mismos hechos

Un prólogo para analizar es una derivación. El compilador emite una segunda para el desvinculador, y los dos deben estar de acuerdo.

En Windows la segunda derivación es `.pdata`y `.xdata`Construye el`-O2`objeto sin`-fno-asynchronous-unwind-tables`y `llvm-objdump -h`muestra`.xdata`de 0x34 bytes y `.pdata`de 0x48 bytes. 0x48 es 72, seis `RUNTIME_FUNCTION`entradas de 12 bytes con tres `IMAGE_REL_AMD64_ADDR32NB`Las seis son las tres.`sum_varargs`¿ Qué ?`keeps`¿ Qué ?`call8`¿ Qué ?`call_varargs`¿ Qué ?`call_pair`, y `call_big`. Las otras seis funciones son hojas, y la página de manejo de excepciones dice "Si la búsqueda no encuentra una entrada de tabla de funciones, se supone que el código es parte de una función de hoja, y RSP se dirige directamente al puntero de retorno".

El `.xdata`bytes para `keeps`¿ Qué ?`01 07 04 00 07 32 03 30 02 70 01 60`- ¿ Qué ?`UNWIND_INFO`comienza con Versión en 3 bits y Banderas en 5 bits: versión 1, sin banderas. SizeOfProlog es 7, CountOfCodes es 4, FrameRegister y FrameOffset son 0.`UNWIND_CODE`es un desplazamiento en el prólogo y un byte de código op y información. `07 32`: en el offset 7, op 2 es `UWOP_ALLOC_SMALL`, info 3 significa `3 * 8 + 8 = 32`- Los bytes.`03 30`: op 0 es `UWOP_PUSH_NONVOL`, registro 3 es `rbx`- ¿ Qué ?`02 70`¿ Qué es esto ?`rdi`- ¿ Qué ?`01 60`¿ Qué es esto ?`rsi`Los códigos se almacenan en orden inverso de ejecución, y tienen el mismo nombre de tres empujes y la misma asignación de 32 bytes que los bytes `56 57 53 48 83 ec 20`- El .`sum_varargs`La entrada es `01 01 01 00 01 02 00 00`: un prólogo de 1 byte con un código, `UWOP_ALLOC_SMALL`Así es como el desvío ve.`pushq %rax`: una asignación, porque `%rax`Es volátil y nada se está salvando.

En Linux la segunda derivación es `.eh_frame`, que se define en el PSABI 4.2.4 y se codifica en la sección 6.4 del DWARF 5. `llvm-dwarfdump --eh-frame`en el objeto relocable imprime una FDE por función y resuelve la `R_X86_64_PC32`Los datos de los datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de datos de la red de datos de datos de la red de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de cuentas de cuentas de cuentas de cuentas de cuentas de cuentas de cuentas de cuentas de cuentas`keeps`¿Qué es esto ?

```text
  DW_CFA_def_cfa_offset: +16
  DW_CFA_def_cfa_offset: +24
  DW_CFA_def_cfa_offset: +32
  DW_CFA_offset: RBX -32
  DW_CFA_offset: R14 -24
  DW_CFA_offset: RBP -16
```

El CFA es el indicador de pila antes de la llamada, 8 por encima del indicador de pila de entrada, por lo que `RBP -16`es el descenso de entrada -8, `R14 -24`es -16, `RBX -32`es -24, y la compensación CFA más profunda de 32 es el marco de 24 bytes más la dirección de retorno.`keeps`dice`DW_CFA_def_cfa: W29 +48`con`W29 -48`¿ Qué ?`W30 -40`¿ Qué ?`W21 -32`¿ Qué ?`W20 -16`¿ Qué ?`W19 -8`. Allí el CFA es el propio puntero de la pila de entrada, por lo que los offsets son los offsets de marco sin conversión, y el CFA se define a partir de `x29`Mientras el registro de fotogramas está en vivo. `eh_frame_cross_check()`Se hace esta comparación para los 12 FDE de ambos objetos.`call8`bajo el Sistema V empuja dos argumentos salientes en el medio del cuerpo, por lo que su compensación CFA más profunda es 32 mientras que su marco de prólogo es 8, y el cheque compara el puntero de pila más profundo que el cuerpo alcanza, 24, más 8.

## Cómo las herramientas llaman esto

Cada herramienta en este campo tiene una estructura de datos para la convención de llamadas y un pase que asigna uno a una función. Los nombres difieren y las lagunas difieren. Las herramientas cerradas se describen a partir de su documentación; las abiertas se ejecutan en las fichas y su salida se captura junto a los objetos.

**Ghidra 12.1.4 (Apache-2.0).**Las reglas viven en archivos de especificaciones del compilador,`.cspec`, junto al lenguaje SLEIGH. `x86-64-gcc.cspec`declara una `<default_proto>`nombre`__stdcall`, "Derivado de la Interfaz Binaria de Aplicaciones del Sistema V, Suplemento de procesador de arquitectura AMD64 abril 2016", con `<pentry>`listas para `XMM0_Qa`¿ Qué ?`XMM7_Qa`y `RDI, RSI, RDX, RCX, R8, R9`, una entrada de pila en el offset 8, a `join_dual_class`regla para tipos de hasta 16 bytes, salidas `XMM0_Qa, XMM1_Qa, RAX, RDX`, y `hidden_return`para el resto.`x86-64-win.cspec`declaraciones`__fastcall`con cuatro `<group>`elementos que se emparejan `XMM0_Qa`con`RCX`¿ Qué ?`XMM1_Qa`con`RDX`, y así sucesivamente, que es la regla de posición, una entrada de pila en el offset 40, `convert_to_ptr`para las líneas de 3, 5, 6 o 7 bytes con el comentario "sólo se pueden empaquetar las líneas de tamaño 8,16,32,64 bits para registrarse", `unaffected`listaje `RBX RBP RDI RSI RSP R12`¿ Qué ?`R15`y `XMM6`¿ Qué ?`XMM15`, y un `localrange`de la pila 8 a 39, que es la tienda de sombras. `AARCH64.cspec`declaraciones`__cdecl`cuya primera entrada es `x8`con`storage="hiddenret"`, entonces`q0`¿ Qué ?`q7`y `x0`¿ Qué ?`x7`¿ Qué ?`convert_to_ptr`para las estructuras de 17 bytes o más, y `killedbycall`incluidos`x8`con el comentario "registro de ubicación de resultados indirectos, que no se refleja en la lista de pentry". La etapa final del descompilador se llama "Establecer el prototipo de la función"; los analizadores son "ID de convención de llamada" y "ID de parámetro del descompilador"; la opción del programa "Evaluación de prototipo" elige el modelo predeterminado.

**IDA Pro and IDA Free (Hex-Rays, commercial; current release 9.4, build 9.4.260714).**El tipo de función es una declaración C establecida a través de SetType, y "IDA utilizará la información del tipo para comentar el desmonte con la información sobre los argumentos de función".`int __usercall func@<ebx>(int x, int y@<esi>)`pone el resultado en `EBX`y `y`En el`ESI`, y `@<^12.4>`Nombrar una pila de ubicación por compensación y tamaño. `__userpurge`Es lo mismo con la limpieza de la pila de llamadas.`__spoils`para el conjunto de piezas de corte, `__hidden`¿ Qué ?`__return_ptr`, y `__struct_ptr`La acción "Decompile as call" del descompilador acepta una`__usercall`IDA Free sólo admite x86 y x86-64.

**Binary Ninja 6.0 "Krypton" (Vector 35, commercial, with a Free edition that decompiles x86, x86_64, ARMv7, and AArch64).**¿ Qué es esto ?`Platform`"contiene toda la información relacionada con el entorno de ejecución del binario, principalmente las convenciones de llamadas utilizadas, el sistema operativo y la arquitectura", y la lista incluye `linux-x86_64`¿ Qué ?`windows-x86_64`¿ Qué ?`linux-aarch64`, y `mac-aarch64`, cada uno con un `default_calling_convention`- ¿ Qué ?`CallingConvention`¿ Qué ?`int_arg_regs`¿ Qué ?`float_arg_regs`¿ Qué ?`callee_saved_regs`¿ Qué ?`caller_saved_regs`¿ Qué ?`int_return_reg`¿ Qué ?`high_int_return_reg`¿ Qué ?`float_return_reg`, y dos banderas que codifican las diferencias en esta lección directamente: `arg_regs_share_index`, "Cuando es verdadero, el argumento Nth consume la ranura Nth de las listas de registros enteros y flotantes independientemente de su tipo", y `stack_reserved_for_arg_regs`, "el espacio sombra/hogar utilizado por la convención de llamadas de Windows x64". `eligible_for_heuristics`¿ Qué ?`required_arg_regs`, y `required_clobbered_regs`El sistema de detección de convenciones heurísticas de las plataformas es el de los títulos C.`types/platform/`, uno por nombre de plataforma.

**radare2 6.2.4 (LGPL-3.0, run here).** `afcl`enumera las convenciones que el plugin de arquitectura conoce: `amd64 amd64syscall dlang golang golang_abi0 ms p9 reg swift`para x86-64 y `arm64 dlang golang p9 reg swift`para AArch64. `afc`informa sobre la convención de la función actual, `afcr`huellas`cc.amd64.arg0: rdi`por el`arg5: r9`y `ret0: rax`, y los perfiles son archivos SDB llamados`cc-ARCH-BITS.sdb`. Radar2 elegido .`amd64`para el objeto ELF y `ms`para el objeto COFF, que es la división correcta. `afv`Enumera argumentos y variables, y `afs`Imprime una firma. En el Linux `-O2`objeto `afv`Encontré siete argumentos para`eight_args`y se cayó .`rcx`; en el objeto de Windows que encontró `rcx, rdx, r8`y cuatro ranuras de pila y se cayó .`r9`Ambos registros faltantes aparecen sólo dentro de un`lea`- En el .`mixed`se informó `double arg1 @ xmm0, arg2 @ xmm1, arg3 @ xmm2`En Linux y en Windows no hay ningún argumento.`x0`¿ Qué ?`x7`por`eight_args`, y para`big_ret`se enumera`x0`y `double arg2 @ d1`donde la función dice `x8`y carga una constante en `q1`- Cada uno .`afs`La firma comienza con `void`, porque el tipo de retorno no se deduce.`-O0`, donde cada argumento se almacena en el marco,`afv`Encontré los ocho argumentos de `eight_args`con`rbp+0x10`y `rbp+0x18`para el par de pila.

**rizin 0.9.1 (LGPL-3.0, run here) and Cutter 2.5.0.** `afcl`las listas `amd64 amd64syscall ms reg swift`- ¿ Qué ?`afc`regresó .`amd64`para el objeto COFF, por lo que existe el perfil de Microsoft y no se seleccionó en el formato. `afi`Se han enumerado siete argumentos para `eight_args`en Linux, otra vez sin`rcx`, y en Windows `r8, rcx, rdx`con tres ranuras de pila, sin `r9`y sin la ranura en `0x40`- ¿ Qué ?`afv`no existe; el comando de lista es `afvl`Cutter es el extremo gráfico delantero sobre Rizin.

**angr 10.0.3 (BSD-2, run here).** `CFGFast`seguido por `CompleteCallingConventions(recover_variables=True)`conjuntos `function.calling_convention`¿ Qué ?`SimCCSystemVAMD64`¿ Qué ?`SimCCMicrosoftAMD64`, o`SimCCAArch64`, en formato binario, y `function.prototype`El documento recogió los ocho argumentos de la Comisión.`eight_args`En los tres objetivos, el indicador oculto de`big_ret`en x86-64 como `(BOT*, long long) -> BOT*`, el conjunto de referencia de las ventanas `by_value`¿ Cómo ?`(BOT*, BOT*) -> BOT*`, y el sistema V variado `sum_varargs`¿ Cómo ?`(long long, ...)`Sus huecos son diferentes de los radares2:`mixed`regresó como`(long long, long long, double, float)`En el sistema V y en el AArch64, los registros correctos se registran en el orden incorrecto, ya que la lista de números enteros se imprime antes que la lista de float; el AArch64 `big_ret`regresó como`(long long) -> double`, así que el `x8`se perdió la convención y las tiendas vectoriales se leyeron como resultado; y el AArch64 `sum_varargs`regresó con 16 parámetros, todo el registro de área de ahorro.

**Others.**Hopper 6.6.0 es un desmonteador y descompiler comercial para macOS y Linux; su manejo de convenciones no se ejecutó ni se documentó para esta lección.`frame info`y `register read`, que es la visión dinámica de la lección 42 de los mismos offsets. Compiller Explorer muestra las tres listas lado a lado de la fuente, que es la forma más rápida de confirmar una regla antes de buscar en bytes.

| Concept | Ghidra | IDA | Binary Ninja | radare2 / rizin | angr | this lesson |
|---|---|---|---|---|---|---|
| The convention object | `prototype` in a `.cspec` | calling convention keyword in the function type | `CallingConvention` on a `Platform` | `afc` name, `cc-ARCH-BITS.sdb` | `SimCC` subclass | `expected_locations(abi, name)` |
| Default per target | `<default_proto>` per cspec | compiler setting | `Platform.default_calling_convention` | `afc=`, `anal.cc` | by format | `describe_object()` |
| Argument registers | `<pentry>` list, `<group>` for positional | `@<reg>` in `__usercall` | `int_arg_regs`, `float_arg_regs`, `arg_regs_share_index` | `cc.NAME.argN` | `ARG_REGS`, `FP_ARG_REGS` | `INT_ARG_REGS`, `FLOAT_ARG_REGS` |
| Shadow store | `localrange` stack 8 to 39 | implicit | `stack_reserved_for_arg_regs` | `cc.ms.argn=stack` | `STACKARG_SP_DIFF` | `STACK_ARG_BASE[WIN64] = 40` |
| Hidden result pointer | `hidden_return`, `storage="hiddenret"` | `__return_ptr`, `__hidden` | return type too large for `int_return_reg` | not modelled | `BOT*` first argument | `HIDDEN_RETURN_REG` |
| Callee-saved | `<unaffected>` | `__spoils` lists the opposite | `callee_saved_regs` | profile | `CALLEE_CLEANUP` and arch | `CALLEE_SAVED` |
| Assigning one to a function | "Call Convention ID" analyzer, "Establish Function's Prototype" | user or type library | heuristic detection with `required_arg_regs` | `afc`, `afva`, `aaft` | `CallingConventionAnalysis` | read-before-write plus the rulebook |

## Construye el mismo

Ejecutar la demostración:

```bash
cd phases/13-tools-and-protocols/38-calling-conventions-and-stack-frames
python3 code/main.py
```

La salida es un informe JSON. Para cada uno de los siete objetos nombra el formato, la máquina y el ABI, y para cada una de las doce funciones da los bytes del prólogo, los registros guardados con compensaciones de entrada, el tamaño del marco, el puntero de marco, los registros de parámetros, los compensaciones de argumentos de pila, los derrames, el registro de retorno o puntero oculto, las profundidades de llamada y los registros y almacenes salientes de los llamadores. Cada función termina con sus controles: la regla, el valor esperado de la firma, el valor observado de los bytes y si están de acuerdo. El informe termina con los dos controles cruzados de desvío y la herramienta captura. En los 84 análisis de funciones, se realizan 489 controles y todos los 489 están de acuerdo.

Forma esperada:

```json
{
  "objects": {
    "convention-x86_64-windows-O2": {
      "format": "COFF", "machine": "AMD64", "abi": "Microsoft x64", "checks": "73/73 agree",
      "functions": {
        "keeps": {
          "prologue": ["56  pushq %rsi", "57  pushq %rdi", "53  pushq %rbx", "48 83 ec 20  subq $0x20, %rsp"],
          "frameBytes": 56,
          "savedRegisters": [{"register": "rsi", "entryOffset": -8}, {"register": "rdi", "entryOffset": -16}, {"register": "rbx", "entryOffset": -24}],
          "parameterRegisters": ["rcx"],
          "returnRegisters": ["rax"],
          "calls": [{"callee": "ext", "depth": 56}],
          "checks": [{"rule": "Microsoft x64: the caller must always allocate ...", "expected": "at least 32 bytes ...", "observed": [56], "agree": true}, "..."]
        }
      }
    }
  },
  "sehCrossCheck": ["..."],
  "ehFrameCrossCheck": {"...": "..."}
}
```

Leer .`code/main.py`en este orden:

1. `describe_object()`se lee el encabezado ELF, COFF o Mach-O y la tabla de símbolos y se devuelve el ABI y la lista de funciones.
2. `parse_listing()`Se divide el `llvm-objdump -d -r`texto en funciones, mantiene el objetivo de reubicación de cada llamada y salto, y desprende comentarios con el marcador de la sintaxis correcta: `#`para AT&T, `//`y `;`para el AArch64, donde `#`El Consejo de Ministros de la Unión Europea ha adoptado una decisión de la Comisión.
3. `effect_x86()`y `effect_arm()`convertir una instrucción en lecturas, direcciones lecturas, escrituras, cargas, tiendas, y un delta de indicador de pila. La última operanda AT&T es el destino; la primera operanda AArch64 es el destino excepto en tiendas, comparaciones y ramas. `lea`El destino de la navegación es el destino de la navegación.`mov`¿ Qué ?`cvtsi2sd`¿ Qué ?`setcc`, y el idioma cero .`xor r, r`Una llamada escribe todo el conjunto volátil. Una tienda preindexada se mueve.`sp`antes de que el acceso y una carga posterior a la indexación lo muevan después.
4. `analyze_function()`camina el cuerpo una vez hasta la primera.`ret`El eje de la señal de pila se encuentra por debajo de su valor de entrada, por lo que una memoria opera`disp(%rsp)`está en el punto de entrada de compensación `disp - depth`y `disp(%rbp)`¿ Qué es ?`frame_pointer + disp`. Registra registros de clases de argumentos que se leen antes de que se escriban, carga en o por encima del primer argumento de pila de la ABI, almacena registros de argumentos en el marco, accede por debajo del puntero de pila actual y la profundidad en cada llamada.
5. `_return_registers()`El registro de resultados se mantiene cuando el último acceso antes de la salida es una escritura.`rax`vivo en el`big_ret`en Microsoft x64, donde `movq %rcx, %rax`es seguido por tiendas a través de `(%rax)`¿ Qué ?
6. `expected_locations()`aplica los tres libros de reglas a una firma C: las dos listas de sistema V con clases de ocho bytes, las ranuras posicionales de Microsoft con estructuras de referencia secundaria y el contador AAPCS64 con `x8`para resultados superiores a 16 bytes.
7. `run_checks()`compara las dos y añade las reglas del marco: los registros registrados en llamadas deben guardarse, `(rsp + 8) % 16 == 0`o `sp % 16 == 0`en cada llamada, 32 bytes de sombra en cada llamada de Microsoft, ningún acceso por debajo del puntero de pila bajo Microsoft y AAPCS64 y como máximo 128 bytes bajo el Sistema V, y un epílogo que refleja el prólogo con el puntero de pila de vuelta a su valor de entrada.
8. `seh_cross_check()`y `eh_frame_cross_check()`leer los datos de desvío de los objetos construidos sin `-fno-asynchronous-unwind-tables`y compararlo con el prólogo de la investigación.

Hay tres órdenes más.`python3 code/main.py analyze convention-x86_64-windows-O2 keeps`Imprima una función. `python3 code/main.py object FILE`Imprime el veredicto del titular.`python3 code/main.py seh`Imprima el decodificado `UNWIND_INFO`de cada función que tiene una.

## Usalo

Utilice el analizador en este orden en un nuevo objeto que construyó.

1. - ¿ Qué ?`python3 code/main.py object FILE.o`Si el encabezador y tu comando de construcción no están de acuerdo, deténgase, uno de ellos está equivocado.
2. Captura .`llvm-objdump -d -r FILE.o`Leer los bytes del prólogo y los registros guardados antes de cualquier otra cosa, y comparar el tamaño del marco con los pulsos y los `sub`¿ Qué ?
3. Un registro que la firma no explica es un puntero oculto, un marcador variádico o un registro de rasguño que el analizador confundió con un argumento. El último caso es un error en la tabla de efectos.
4. Cada uno da el nombre de la regla en la que se apoya, así que lo primero que debe comprobar es la regla, luego los bytes, luego el analizador.
5. Cuando el objeto tenga datos de desvío, ejecuta la verificación cruzada. Dos derivadas que coinciden son la evidencia más fuerte que esta lección produce.
6. Cuando se instala otra herramienta, captura su salida con el comando exacto y compara. Cuando está de acuerdo, tiene tres derivadas. Cuando difiere, tiene un registro específico y una instrucción específica para mirar, y eso es un hallazgo para la lección 35.

## Laboratorio interactivo

### Laboratorio A: una instrucción, tres efectos

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/38-calling-conventions-and-stack-frames/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
effect(Instruction(0, "48 8d 04 0a", "leaq", "(%rdx,%rcx), %rax"), SYSV)
effect(Instruction(0, "f2 0f 2a d7", "cvtsi2sd", "%edi, %xmm2"), SYSV)
effect(Instruction(0, "a9bd7bfd", "stp", "x29, x30, [sp, #-0x30]!"), AAPCS64)
```

Escriba las lecturas, la dirección, las lecturas y el delta de pila que espera antes de cada llamada.`[sp, #-0x30]!`¿ Qué ?`[sp], #0x30`con`ldp`y ver el delta moverse de`sp_delta`¿ Qué ?`post_sp_delta`¿ Qué ?

### Laboratorio B: el marco de las retenciones

```python
info, listing = load_fixture("convention-aarch64-linux-O2")
keeps = analyze_function("keeps", listing["keeps"], info.abi)
keeps.saved
keeps.frame_pointer
[c.to_dict() for c in frame_cells(keeps)]
```

Compara las compensaciones con la decodificación manual de arriba.`convention-arm64-macos-O2`y ejecutar las mismas tres líneas; el registro se mueve a -16 y cuatro registros se sientan debajo de él.`convention-x86_64-linux-O2-fp`y leer .`saved`por`keeps`: el marcador de marco está de vuelta, y también es un `pushq %rax`cuyo único trabajo es hacer`(rsp + 8)`Un múltiplo de 16 en la llamada.

### Laboratorio C: lectura antes de escritura en mezcla

```python
info, listing = load_fixture("convention-x86_64-linux-O2")
mixed = analyze_function("mixed", listing["mixed"], info.abi)
mixed.param_registers
mixed.return_registers
expected_locations(SYSV, "mixed")["parameters"]
expected_locations(WIN64, "mixed")["parameters"]
```

La lista del Sistema V es `rdi, xmm0, rsi, xmm1`y la lista de Microsoft es `rcx, xmm1, r8, xmm3`Ahora cargue el objeto de Windows y ejecuta las tres primeras líneas de nuevo. Luego abra la captura de radar2 para el objeto de Linux y encontrar`[afv mixed]`Encuentra la instrucción que explica la tercera.

### Laboratorio D: segundo dictamen del desmantelador

```python
for row in seh_cross_check():
    print(row["function"], row["sizeOfProlog"], row["unwindPushes"], row["unwindAllocBytes"], row["agree"])
for row in eh_frame_cross_check("convention-x86_64-linux-O2-eh"):
    print(row["function"], row["cfiSaved"], row["cfiMaxOffset"], row["deepestStack"], row["agree"])
```

Encuentra el`pushq %rax`de `sum_varargs`En la tabla de SEH y explicar por qué el código de desviado lo registra como una asignación.`call8`en el `.eh_frame`¿Qué es lo que se dice?

## Laboratorio de práctica

Añade un cuarto objetivo y un cuarto libro de reglas.

Requisitos:

- Compilación `convention.c`con`-target x86_64-apple-macos -O2`y capturar su lista.`describe_object()`Para leer el Mach-O `cputype`0x01000007 y la mapear al Sistema V psABI, que Apple sigue en x86-64, con el `_`el prefijo del símbolo.
- Verifique si cada función coincide con los controles del Sistema V. Registra la única diferencia que encuentra en los prólogos: Darwin mantiene el puntero de marco en cada nivel de optimización.
- Añadir un`-fstack-protector-strong`Extenda el análisis del prólogo para que la carga canaria de`%fs:0x28`y la tienda debajo de los lugareños se registran como un `canary`la célula, y la comparación anterior `ret`como el cheque que guarda el epílogo.
- Añadir una prueba que falla cuando una tienda de canarios se cuenta como un argumento derramado.
- Escriba el marco de `with_canary`-Code de estilo a mano antes de ejecutar el analizador, y mantener el dibujo junto a la prueba.

El éxito significa`python3 code/main.py`El archivo de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de la red de datos de datos de la red de datos de datos de la red de datos de datos de la red de datos de datos de la red de datos de datos de la red de datos de datos de la red de datos de datos de la red de datos de datos de la red de datos de datos de datos de la red de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de datos de la red de datos de datos de datos de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red de red

## Lección Artefacto

Esta lección incluye `outputs/skill-calling-conventions-and-stack-frames.md`. Utilice para convertir un objeto que usted construyó, su listaje y las firmas de C en ubicaciones de parámetros, un diseño de marco, un registro de devolución y una verificación de cada inferencia contra la regla en que se basa y contra los datos de desvío del compilador. Requiere el veredicto de ABI del encabezado primero, prohíbe la lectura.`cvtsi2sd`o `xor r, r`Los objetivos son los siguientes:

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/38-calling-conventions-and-stack-frames
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

La verificación deberá demostrar:

- los encabezados ELF, COFF y Mach-O de los siete objetos denominados System V, Microsoft x64, AAPCS64 y Apple arm64
- la tabla de símbolos de cada objeto enumera las mismas doce funciones, con el Mach-O `_`prefijo y el `ltmp0`etiqueta resuelta
- el paresador de la lista mantiene `ext`como el objetivo de la llamada y mantiene AArch64 inmediato como `#-0x30`intacta
- `leaq (%rdx,%rcx), %rax`lee dos registros de direcciones y no carga nada; `xorl %eax, %eax`y `cvtsi2sd`son sólo escritos; un pre-indecadas `stp`Se mueve .`sp`antes de la tienda y después de la indexación `ldp`después de la carga
- `keeps`En el`-O2`¿ Es verdad ?`55 41 56 53`en el sistema V con un marco de 24 bytes y sin indicador de marco, `56 57 53 48 83 ec 20`en Microsoft x64 con un marco de 56 bytes, y `a9bd7bfd`Primero en AArch64 con el récord de -48
- La construcción de Apple pone el registro en -16 por encima de cuatro registros guardados
- Cada función tiene un marcador de marco en `-O0`en Linux y ninguno en `-O2`, y `-fno-omit-frame-pointer`Lo trae de vuelta con un empuje de alineación
- el `-O0`la hoja utiliza 8 bytes de zona roja bajo el sistema V, `pushq %rax`bajo Microsoft x64, y `sub sp, sp, #0x10`en virtud de la AAPCS64; `-mno-red-zone`elimina el uso de la zona roja
- Cada epílogo refleja su prólogo y el puntero de pila está de vuelta en la entrada
- `eight_args`Se lee seis registros y ranuras 8 y 16, cuatro registros y ranuras 40 a 64, o `x0`¿ Qué ?`x7`, en ambos niveles de optimización
- `mixed`¿ Es verdad ?`rdi, xmm0, rsi, xmm1`en el marco del sistema V y `rcx, xmm1, r8, xmm3`bajo Microsoft x64, con `xmm0`como resultado de ambos
- El 16 bytes estructura viaja en dos registros bajo el sistema V y AAPCS64 y por referencia detrás de un puntero oculto bajo Microsoft x64
- La estructura de 32 bytes regresa a través de `rdi`¿ Qué ?`rcx`, o`x8`
- La variada callee dice `%al`, hogares `rdx, r8, r9`en 16, 24, 32, ahorra `x1`¿ Qué ?`x7`y `q0`¿ Qué ?`q7`, o lee el indicador de la pila de llamadas en Apple
- los llamadores establecen los registros esperados, almacenan 7 y 8 en 0 y 8 o 5 a 8 en 32 a 56, cero `%al`para la llamada variada, y pasar la dirección del buffer de resultado en `rdi`¿ Qué ?`rcx`, o`x8`
- Todos los 489 controles están de acuerdo con sus reglas
- El seis .`UNWIND_INFO`Las entradas coinciden con el prólogo y las hojas no tienen ninguna
- Las 24 FDE coinciden con las compensaciones de registro guardado y el indicador de pila más profundo
- La herramienta captura los radares de registro2 `amd64`y `ms`, de Rizin `amd64`en el objeto COFF, el desaparecido `rcx`, los tres dobles, y los prototipos de Angr
- los datos de la figura incorporados en el módulo de sitio son iguales a la salida del analizador

## Modos de falla de producción

| Failure | What the weak tool reports | What the analysis must prove |
|---|---|---|
| COFF object read as System V | `rdi` and `rsi` as arguments that the function never reads | The header says 0x8664; the convention is Microsoft x64 |
| Register read only inside a `lea` | Seven arguments for an eight-argument function | Base and index registers are reads |
| `cvtsi2sd` destination counted as a read | A third double argument in `xmm2` | The first write to `xmm2` is a write; `mixed` has two SSE arguments |
| Positional slots read as sequences | `xmm0` named as the double argument on Windows | Slot 1 is `XMM1`, slot 3 is `XMM3` |
| `push rbp` taken as a frame pointer | A frame pointer in an `-O2` function that uses `rbp` as an accumulator | Only `mov %rsp, %rbp` after the push creates one |
| Red zone measured from the entry stack pointer | 216 bytes "below the stack pointer" | 128 bytes below the current stack pointer after an 88-byte `sub` |
| Shadow store treated as locals | Stack arguments at 8 to 32 | Microsoft stack arguments start at 40 |
| `lea` treated as a load | A stack argument at `0x60(%rsp)` in a variadic callee | `lea` computes `overflow_arg_area`; it loads nothing |
| Hidden pointer read as a normal argument | `big_ret(void *, long long)` | The pointer is stored through and copied to `rax`, `rax`, or stays in `x8` |
| Deepest CFA compared with the prologue frame | A mismatch on a caller that pushes outgoing arguments | Compare with the deepest stack pointer the body reaches |
| Convention asserted from the ISA | `amd64` for a COFF object | The format picks the rulebook, then the ISA picks the registers |
| Prototype trusted from one tool | Three doubles, or sixteen variadic parameters | The source signature or a second derivation must agree |

## Regla de funcionamiento

Lea el encabezado antes de la primera instrucción y nombre el libro de reglas. Coloque cada registro guardado y cada argumento en un desvio del puntero de entrada de la pila, y compruebe con la regla que lo puso allí y con los datos de desvio que el compilador escribió para el mismo marco. Trate el prototipo de una herramienta como una reclamación hasta que los bytes o la fuente lo confirmen.

## Leer más

- [System V Application Binary Interface, AMD64 Architecture Processor Supplement, Version 1.0 (sections 3.2.1 to 3.2.3, 4.2.4)](https://gitlab.com/x86-psABIs/x86-64-ABI)
- [Microsoft x64 calling convention](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention)
- [Microsoft x64 stack usage](https://learn.microsoft.com/en-us/cpp/build/stack-usage)
- [Microsoft x64 prolog and epilog](https://learn.microsoft.com/en-us/cpp/build/prolog-and-epilog)
- [Microsoft x64 exception handling: RUNTIME_FUNCTION, UNWIND_INFO, UNWIND_CODE](https://learn.microsoft.com/en-us/cpp/build/exception-handling-x64)
- [Arm Procedure Call Standard for the Arm 64-bit Architecture (AAPCS64), release 2025Q4](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst)
- [Apple: Writing ARM64 code for Apple platforms](https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms)
- [DWARF Debugging Information Format Version 5, section 6.4 Call Frame Information](https://dwarfstd.org/doc/DWARF5.pdf)
- [Clang driver frame-pointer defaults, `useFramePointerForTargetByDefault` in `CommonArgs.cpp`](https://github.com/llvm/llvm-project/blob/main/clang/lib/Driver/ToolChains/CommonArgs.cpp)
- [Clang command line reference: `-fomit-frame-pointer`, `-momit-leaf-frame-pointer`](https://clang.llvm.org/docs/ClangCommandLineReference.html)
- [Fedora 38 change: frame pointers by default](https://fedoraproject.org/wiki/Changes/fno-omit-frame-pointer)
- [Ubuntu 24.04: performance engineering with frame pointers by default](https://ubuntu.com/blog/ubuntu-performance-engineering-with-frame-pointers-by-default)
- [Ghidra compiler specification: `x86-64-gcc.cspec`](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Processors/x86/data/languages/x86-64-gcc.cspec)
- [Ghidra compiler specification: `x86-64-win.cspec`](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Processors/x86/data/languages/x86-64-win.cspec)
- [Ghidra compiler specification: `AARCH64.cspec`](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Processors/AARCH64/data/languages/AARCH64.cspec)
- [Hex-Rays: SetType and the `__usercall` convention](https://docs.hex-rays.com/ida-actions/settype)
- [Hex-Rays: Decompile as call](https://docs.hex-rays.com/ida-actions/hx_userdefinedcall)
- [Binary Ninja API 6.0: `callingconvention` module](https://api.binary.ninja/binaryninja.callingconvention-module.html)
- [Binary Ninja API 6.0: `platform` module](https://api.binary.ninja/binaryninja.platform-module.html)
- [Binary Ninja user documentation: platform types](https://docs.binary.ninja/guide/types/platformtypes.html)
- [The Official Radare2 Book: calling conventions](https://book.rada.re/analysis/calling_conventions.html)
- [The Official Radare2 Book: code analysis](https://book.rada.re/analysis/code_analysis.html)
- [angr documentation: decompiler analysis passes, including calling convention recovery](https://docs.angr.io/en/latest/analyses/decompiler.html)
- [radare2 6.2.4 release](https://github.com/radareorg/radare2/releases/tag/6.2.4)
- [rizin 0.9.1 release](https://github.com/rizinorg/rizin/releases/tag/v0.9.1)
