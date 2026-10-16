# El levantamiento al Pseudocodo: IR, SSA y estructuración desde cero

> Cuando construyes la derivación tú mismo, cada línea de esa salida se convierte en algo que puedes comprobar.

**Type:** Build
**Languages:** Python, C
**Prerequisites:** Phase 13 · 32 (ELF from scratch), Phase 13 · 33 (disassembly and control flow)
**Time:** ~150 minutes

## Objetivos de aprendizaje

- El elevador decodificó instrucciones x86-64 a una pequeña IR con banderas explícitas, cargas, tiendas, ramas, llamadas y devoluciones.
- Convierta los espacios de marco en variables locales y los registros de argumentos del Sistema V en parámetros.
- Construye un SSA mínimo con funciones phi en las fronteras de dominio, luego sustituye, simplifique y elimine el código muerto.
- Recuperarse`if`¿ Qué ?`if`con`else`¿ Qué ?`while`, y `do while`de la gráfica de flujo de control con postdominadores y bordes traseros.
- Explica el multiplicador por `0x66666667`como una división por 10 y probar la identidad.
- Lea el pseudocodo de Ghidra para las mismas funciones línea por línea y nombre la instrucción, o la suposición, detrás de cada diferencia.

## El problema

Le preguntaste a Rea el pseudocodo de `percent_change`En el`ledger-x86_64-O1.o`Ghidra 12.1.4 respondió en unos diez segundos:

```c
undefined1  [16] percent_change(int param_1,int param_2,ulong param_3)

{
  long lVar1;
  undefined1 auVar2 [16];
  undefined1 auVar3 [16];
  
  if (param_1 != 0) {
    lVar1 = (long)((param_2 - param_1) * 100);
    auVar3._0_8_ = lVar1 / (long)param_1 & 0xffffffff;
    auVar3._8_8_ = lVar1 % (long)param_1 & 0xffffffff;
    return auVar3;
  }
  auVar2._8_8_ = 0;
  auVar2._0_8_ = param_3;
  return auVar2 << 0x40;
}
```

La fuente tiene cuatro líneas de largo.`int`valores y devoluciones uno `int`Ghidra informa un valor de retorno de 16 bytes y un tercer parámetro que la fuente no tiene. Los dieciséis bytes de código de máquina son correctos, y Ghidra decodificó cada uno de ellos correctamente. La firma es una conjetura, y nada en el texto lo marca como uno.

rea devuelve este texto como el `procedure_pseudo_code`El resultado y se une una limitación al registro. La limitación dice "Pseudocode es Ghidra decompiler salida, no fuente original y no texto equivalente a Hopper salida".`compare_functions`La herramienta difiere de los pseudocodes como líneas de texto. Su propio registro de dimensiones dice "Assembly es texto opaco del proveedor" y que la normalización de la reubicación no está disponible. rea no puede mirar dentro del texto, y tampoco puede un agente que sólo lo lea.

El registro de DX-Ball de rea muestra el costo.`FUN_00406400(void)`llamando`__ftol()`La vista de instrucciones mostró una entrada de pila y una expresión x87. El agente siguió funcionando sólo porque las dos vistas no estaban de acuerdo.

Esta lección construye la derivación que el decompiler oculta. comienza con las 56 instrucciones que la lección 33 decodificó, elevalas a un IR, renombre en SSA, simplifica y estructura. Luego coloca su pseudocodo junto al de Ghidra para`percent_change`¿ Qué ?`grade`, y `total`Las pruebas compilan su salida con clang y ejecutarlo contra la fuente en una cuadrícula de entrada.

```figure
lifting-stages-percent-change
```

## La Fijación y el Oráculo

`code/fixtures/ledger.c`es el elemento compartido de esta parte del curso:

```c
int percent_change(int before, int after) {
    if (before == 0) return 0;
    return (after - before) * 100 / before;
}

int grade(int score) {
    switch (score / 10) {
    case 10:
    case 9: return 4;
    case 8: return 3;
    case 7: return 2;
    case 6: return 1;
    default: return 0;
    }
}

int total(const int *values, int count) {
    int sum = 0;
    for (int i = 0; i < count; i++) sum += values[i];
    return sum;
}
```

`grade_label`Las tres funciones anteriores son las que esta lección prueba.

El objeto fue construido con `clang -target x86_64-unknown-linux-gnu -O1 -fno-asynchronous-unwind-tables -c ledger.c`- El .`-O0`el objeto se sienta al lado para la sección de la ranura de pila. `code/fixtures/README.md`En el código de configuración de la red de datos de Ghidra se incluyen los comandos SHA-256 de cada objeto y el comando rea que produjo las fichas de Ghidra.

El oráculo para el decodificador es `llvm-objdump -d -r`. Su texto para ambos objetos se compromete como `ledger-x86_64-O1.objdump.txt`y `ledger-x86_64-O0.objdump.txt`Una prueba analiza esos archivos y compara cada instrucción por dirección, por bytes y por texto de AT&T.`-O1`y todas las instrucciones en `-O0`- ¿ Cuándo ?`llvm-objdump`se instala en el camino fijado, una segunda prueba lo ejecuta en vivo y compara de nuevo.

Aquí está .`percent_change`Como lo imprime el oráculo:

```text
0000000000000000 <percent_change>:
       0: 85 ff                        	testl	%edi, %edi
       2: 74 09                        	je	0xd <percent_change+0xd>
       4: 29 fe                        	subl	%edi, %esi
       6: 6b c6 64                     	imull	$0x64, %esi, %eax
       9: 99                           	cltd
       a: f7 ff                        	idivl	%edi
       c: c3                           	retq
       d: 31 c0                        	xorl	%eax, %eax
       f: c3                           	retq
```

## Etapa 1: Bytes a las instrucciones

La lección 33 construyó el decodificador. Esta lección mantiene una pequeña copia para que el código se ejecute por sí mismo. La copia cubre exactamente los códigos opcodes que se emiten para`ledger.c`En el`-O0`y `-O1`Para cualquier otra cosa que se eleva`DecodeError`con la dirección y el byte del código op.

El decodificador sigue el orden de bytes del SDM Intel, Volumen 2, sección 2.1. Los prefijos vienen primero, luego el código op, luego ModR/M, luego SIB, luego el desplazamiento, luego el inmediato.`0x40`¿ Qué ?`0x4F`, que debe estar directamente antes del código op. REX.W selecciona una operanda de 64 bits, REX.R extiende la `reg`En el campo de referencia, REX.X extiende el índice SIB y REX.B extiende `r/m`La sección 2.1.3 define los tres campos ModR/M `mod`¿ Qué ?`reg`, y `r/m`, y los tres campos SIB `scale`¿ Qué ?`index`, y `base`En modo de 64 bits,`mod = 00`con`r/m = 101`se refiere a un desplazamiento de 32 bits en relación con la siguiente instrucción (sección 2.2.1.6).

| Opcode bytes | Form | Fixture example |
|---|---|---|
| `01 29 31 39 85 89` | `op Ev, Gv` | `29 fe` `subl %edi, %esi` |
| `03 2b 3b 8b 8d` | `op Gv, Ev` | `03 04 97` `addl (%rdi,%rdx,4), %eax` |
| `63` with REX.W | `movsxd Gv, Ev` | `48 63 c7` `movslq %edi, %rax` |
| `69` and `6b` | `imul Gv, Ev, imm` | `48 69 c0 67 66 66 66` |
| `83 /0 /5 /7` | `add sub cmp Ev, imm8` | `83 c1 fa` `addl $-0x6, %ecx` |
| `c1 /5 /7` | `shr sar Ev, imm8` | `48 c1 f9 22` `sarq $0x22, %rcx` |
| `f7 /7` | `idiv Ev` | `f7 ff` `idivl %edi` |
| `ff /0 /4` | `inc Ev`, `jmp Ev` | `48 ff c2` `incq %rdx` |
| `70` to `7f`, `eb`, `e8` | `jcc rel8`, `jmp rel8`, `call rel32` | `74 09` `je 0xd` |
| `99`, `c3`, `90`, `0f 1f` | `cltd`, `retq`, `nop` | `66 0f 1f 44 00 00` `nopw (%rax,%rax)` |

El `leaq (%rip), %rcx`En el`grade+0x23`Es 7 bytes con un desplazamiento cero.`R_X86_64_PC32.rodata+0x10`en compensación `0x36`El elevador lo resuelve en la etapa 3.

## Etapa 2: Bloques y dominadores

Un bloque básico es una secuencia de declaraciones con una entrada en la parte superior y una salida en la parte inferior. Esa es la definición de Allen (1970), y cada papel de descompilador lo reutiliza. `build_blocks()`Reúne líderes: la entrada de la función, cada objetivo de salto dentro de la función, y cada dirección después de un salto o retorno. Cada bloque obtiene sucesores de su última instrucción.`jmp`Tiene uno.`ret`No tiene ninguna, y un bloque que termina en una instrucción ordinaria pasa al siguiente líder.

`total`En el`-O1`tiene cinco bloques:

| Block | Instructions | Successors |
|---|---|---|
| `0x40` | `testl %esi, %esi` then `jle 0x5c` | `0x5c`, `0x44` |
| `0x44` | `movl %esi, %ecx`, `xorl %edx, %edx`, `xorl %eax, %eax`, `nopw` | `0x50` |
| `0x50` | `addl (%rdi,%rdx,4), %eax`, `incq %rdx`, `cmpq %rdx, %rcx`, `jne 0x50` | `0x50`, `0x5b` |
| `0x5b` | `retq` | exit |
| `0x5c` | `xorl %eax, %eax` then `retq` | exit |

`immediate_dominators()`Es el algoritmo iterativo de Cooper, Harvey y Kennedy (2001). Caminará los bloques en orden posterior inverso y intersectará a los dominadores de los predecesores procesados hasta que nada cambie.`total`Dice que`0x44`domina .`0x50`También dice que`0x50`se domina a través del borde trasero, que es cómo la etapa 8 encuentra el bucle.

La misma función se ejecuta dos veces más. `dominance_frontiers()`se utiliza para la SSA en la etapa 5. `post_dominators()`lo ejecuta en el gráfico invertido con un nodo de salida virtual para encontrar uniones en la etapa 8.

## Etapa 3: Elevación a una pequeña IR

El IR tiene seis tipos de declaraciones: `Assign`¿ Qué ?`Store`¿ Qué ?`Branch`¿ Qué ?`Jump`¿ Qué ?`IndirectJump`, y `Return`. Expresiones son constantes, variables, operaciones binarias con ancho, lanzamientos (`sext`¿ Qué ?`zext`¿ Qué ?`trunc`), y cargas. existen cinco formas más que se simplificarán más adelante: `Flags`¿ Qué ?`Cond`¿ Qué ?`Concat`¿ Qué ?`SignFill`, y `Symbolic`¿ Qué ?

Los registros son variables nombradas por su familia de 64 bits.`%eax`es un `Assign`¿ Qué ?`rax`con ancho 32, y una lectura de `%eax`es una lectura de `rax`En el modo de 64 bits, una escritura de 32 bits se extiende a cero en el registro completo, por lo que una lectura posterior de 64 bits del mismo valor es un `zext`El elevador no conoce la definición de alcance, por lo que registra el ancho de lectura y permite a SSA insertar el molde en la etapa 5.

| Instruction | IR statements |
|---|---|
| `subl %edi, %esi` | `flags = cmp(esi, edi)` then `esi = esi - edi` |
| `testl %edi, %edi` | `flags = test(edi, edi)` |
| `je 0xd` | `if je(flags) goto 0xd else goto 0x4` |
| `xorl %eax, %eax` | `eax = 0` |
| `cltd` | `edx = signfill(eax)` |
| `idivl %edi` | `dividend = edx:eax`, `eax = dividend / edi`, `edx = dividend % edi` |
| `movslq %edi, %rax` | `rax = sext64(edi)` |
| `sarq $0x22, %rcx` | `rcx = rcx >>s 34` |
| `shrq $0x3f, %rax` | `rax = rax >>u 63` |
| `addl (%rdi,%rdx,4), %eax` | `flags = add(eax, load32(rdi + rdx * 4))` then `eax = eax + load32(rdi + rdx * 4)` |
| `leaq (%rip), %rcx` with its relocation | `rcx = &.rodata+0x14` |
| `pushq %rbp`, `movq %rsp, %rbp`, `subq $0x10, %rsp`, `popq %rbp` | nothing, frame bookkeeping |
| `retq` | `return rax` |

### Las banderas son una variable

Cada instrucción que establece los códigos de condición escribe una variable llamada `flags`, y cada salto condicional lo lee.`cmp a, b`escribe `cmp(a, b)`, una `test a, b`escribe `test(a, b)`, y `sub`escribe lo mismo `cmp`forma como efecto secundario.`jcc`Se convierte en`Branch(Cond(cc, flags), target, fallthrough)`El código de condición determina al operador de comparación cuando ambos se encuentran en la etapa 6:

| Code | After `cmp(a, b)` | After `test(a, a)` |
|---|---|---|
| `je` | `a == b` | `a == 0` |
| `jne` | `a != b` | `a != 0` |
| `jl`, `jge`, `jle`, `jg` | signed `<`, `>=`, `<=`, `>` | signed compare with 0 |
| `jb`, `jae`, `jbe`, `ja` | unsigned `<`, `>=`, `<=`, `>` | unsigned compare with 0 |

`xorl %eax, %eax`se levanta directamente a `eax = 0`Las banderas que pone están escritas como:`add`Las banderas de estilo que no salta en el accesorio dice.`xor`El idioma es importante en `grade`, donde Ghidra lo perdió.

### C.L.T. y I.D.V.

`cltd`llena`%edx`con el signo de `%eax`- ¿ Qué ?`idivl %edi`divide el par de 64 bits `%edx:%eax`por `%edi`, pone el coeficiente en `%eax`, y el resto en `%edx`El levantador escribe el par en un nombre temporal`dividend`La declaración restante luego lee la vieja`%eax`En la SSA la mitad superior aparece como`signfill`El simplificador reduce el par a la mitad baja, y la división se convierte en un 32-bit `/`¿ Qué ?

### Las ranuras de pila en -O0

En el`-O0`Clang mantiene a todos los locales en el marco.`percent_change`las tiendas `%edi`¿ Qué ?`-0x8(%rbp)`y `%esi`¿ Qué ?`-0xc(%rbp)`El levantador vigila para que se encuentren.`pushq %rbp`seguido por `movq %rsp, %rbp`Después de ese prólogo, una ópera de memoria con base.`%rbp`, sin índice, y un desplazamiento se convierte en una variable denominada `local_<hex>`- ¿ Qué ?`-0x8(%rbp)`¿ Es verdad ?`local_8`¿ Qué ?`-0xc(%rbp)`¿ Es verdad ?`local_c`¿ Qué ?

Las instrucciones del marco no producen IR, y tampoco lo hacen.`subq`y `addq`En el`%rsp`Cualquier otro operando de memoria se mantiene un`Load`o `Store`¿ Qué ?

### Traslados

El `leaq (%rip), %rcx`En el`grade`El desplazamiento en compensación`0x36`dice`R_X86_64_PC32.rodata+0x10`El enlace escribirá .`S + A - P`En el campo.`S`es la dirección de `.rodata`¿ Qué ?`A`es el agregado `0x10`, y `P`La instrucción luego calcula `next_ip + disp = (P + 4) + S + A - P = S + A + 4`¿ Qué ?

El levantador aplica la misma aritmética sin un enlace.`0x10 + 4 = 0x14`La IR dice que`rcx = &.rodata+0x14`, y la mesa allí tiene los cinco .`int`valores `1, 2, 3, 4, 4`¿ Qué ?

## Etapa 4: Parámetros de la convención de convocatoria

El sistema V AMD64 psABI, sección 3.2.3, pasa los argumentos de clase INTEGER en `%rdi`¿ Qué ?`%rsi`¿ Qué ?`%rdx`¿ Qué ?`%rcx`¿ Qué ?`%r8`¿ Qué ?`%r9`El resultado regresa en`%rax`Y luego ...`%rdx`En el apartado 3.2.1 se dice que `%rbp`¿ Qué ?`%rbx`, y `%r12`por el`%r15`pertenece al que llama y debe ser conservado.

`collect_params()`aplica una regla: un registro de argumentos que se lee antes de cualquier escritura en la función es un parámetro. El registro más alto de tal fija el recuento, por lo que una función que lee `%rsi`y nunca .`%rdi`todavía tiene dos parámetros.`percent_change`Leía`%edi`y `%esi`, así que lo ha hecho .`param_1`y `param_2`- ¿ Qué ?`grade`sólo se lee `%edi`- ¿ Qué ?`total`Leía`%rdi`como una base de 64 bits y `%esi`como un recuento de 32 bits.

Ahora es Ghidra.`param_3`Tiene una explicación.`idivl`escribe `%rdx`, y el psABI permite una vuelta de 128 bits en INTEGER `%rax:%rdx`En un archivo de objetos no hay un llamador que diga cuánto es el valor de retorno.

En el camino cero,`xorl %eax, %eax`Sólo escribe .`%eax`Ghidra lee .`%rdx`allí como una entrada de la función, y esa entrada se convirtió en `param_3`Las instrucciones son correctas, la firma es una suposición sobre las partes que las instrucciones dejan abiertas.

## Etapa 5: SSA, un nombre por definición

Después de levantar,`grade`escribe `rax`cuatro veces y `rcx`- ¿Qué? - Tres veces. Cytron, Ferrante, Rosen, Wegman y Zadeck (1991) definen la forma de asignación única estática. Cada variable se asigna exactamente una vez, y las funciones phi en los puntos de unión seleccionan entre las definiciones que las alcanzan. Su construcción tiene dos pasos. La sección 5.1 coloca las funciones phi utilizando fronteras de dominio, y la sección 5.2 renombra cada definición y uso con un paseo sobre el árbol dominador.`insert_phis()`y `rename()`Sigue esas dos secciones.

La lista de la SSA de `total`muestra lo que produce la construcción:

```text
block 0x44 -> 0x50
  ecx_1 = esi_0
  edx_1 = 0
  eax_1 = 0
block 0x50 -> 0x50, 0x5b
  eax_2 = phi(0x44: eax_1, 0x50: eax_3)
  rdx_2 = phi(0x44: edx_1, 0x50: rdx_3)
  flags_2 = phi(0x44: flags_1, 0x50: flags_5)
  flags_3 = add(eax_2, load32(rdi_0 + rdx_2 * 4))
  eax_3 = eax_2 + load32(rdi_0 + rdx_2 * 4)
  flags_4 = inc(rdx_2, 1)
  rdx_3 = rdx_2 + 1
  flags_5 = cmp(zext64(ecx_1), rdx_3)
  if jne(flags_5) goto 0x50 else goto 0x5b
```

La versión 0 de un registro es su valor al momento de la entrada. `esi_0`y `rdi_0`El encabezado del bucle`0x50`tiene dos predecesores, el bloque de configuración y sí mismo, por lo que cada variable con una definición en el bucle obtiene un phi allí.`flags_2`Phi está muerto a la llegada.`flags`tiene definiciones en ambos predecesores, y la etapa 6 lo elimina porque nada lo lee.

El paso de renombramiento también fija los anchos. `edx_1 = 0`es una definición de 32 bits y `rdx_3 = rdx_2 + 1`es de 64 bits, por lo que el phi `rdx_2`La lectura de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de la letra de`%rcx`En el`cmpq %rdx, %rcx`es de 64 bits mientras que su definición `ecx_1 = esi_0`es de 32 bits, por lo que el uso se hace `zext64(ecx_1)`Cada elenco en el pseudocodo final proviene de esta regla.

La parte "lite" es la salida de la SSA.`coalesce_webs()`Se puede dividir en copias paralelas, donde dos phis en una unen valores de intercambio, y la lección no maneja ese caso. La sección 7 de Cytron describe la salida general a través del colorado.

## Etapa 6: Simplificar, sustituir y eliminar el código muerto

`optimize()`El simplificador reescribe expresiones de abajo hacia arriba con estas reglas:

| Before | After |
|---|---|
| `Cond(je, cmp(a, b))` | `a == b`, and the same for the other nine codes |
| `Cond(jle, test(a, a))` | `a <= 0` |
| `x + (-6)` | `x - 6` |
| `trunc32(zext64(x))` where `x` is 32-bit | `x` |
| `trunc32(a + b)` on 64-bit operands | `trunc32(a) + trunc32(b)` |
| `(signfill(x): x) / d` | `x / d` at width 32 |
| `load32(base + idx * 4)` | `base[idx]` |
| `trunc32(p >>s 34) + trunc32(p >>u 63)` with `p = sext64(x) * 0x66666667` | `x / 10` |

La sustitución hacia adelante se mueve una definición en su único uso. La regla tiene tres condiciones. El uso está en el mismo bloque. El uso es una declaración y no una phi. Cuando la definición contiene una carga, ninguna tienda o llamada se encuentra entre la definición y el uso. Una constante o una dirección de sección se sustituye en cualquier lugar excepto en una phi, porque un argumento phi tiene que permanecer una variable para la web.

La eliminación de código muerto elimina una asignación o una phi cuyo destino no tiene uso y cuya expresión no tiene llamada.`percent_change`el resto `edx_2 = dividend_1 % edi_0`Está muerto, así que va primero.`edx_1 = signfill(eax_1)`con un solo uso, por lo que se sustituye por `dividend_1 = edx_1:eax_1`¿ Qué ?

La regla de pareja entonces reduce `signfill(eax_1):eax_1`¿ Qué ?`eax_1`, y la división se convierte en`eax_1 / edi_0`Ahora .`eax_1 = esi_1 * 100`tiene un solo uso y `esi_1 = esi_0 - edi_0`El uso de los datos de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información de la información.`flags`Las definiciones se han sustituido en las condiciones de rama y desaparecen.

```c
return (param_2 - param_1) * 100 / param_1;
```

### El multiplicador 0x66666667

`grade`Divide la puntuación por 10, y el clang emite no `idiv`- Emite .`imulq $0x66666667, %rax, %rax`, entonces`sarq $0x22, %rcx`en una copia, entonces `shrq $0x3f, %rax`Granlund y Montgomery, "División por enteros invariantes utilizando la multiplicación" (PLDI 1994), describen el método.`m`y un turno .`s`- Por cada uno .`x`en el rango, `(x * m) >> s`debe ser igual `x / d`¿ Qué ?

Aquí tienes .`m = 0x66666667 = 1717986919`y `s = 34`La aritmética se confirma:`2^34 / 10 = 1717986918.4`, y el techo es`1717986919`El error es`m * 10 - 2^34 = 6`Para que la identidad se mantenga en cada 32 bits .`x`, los tiempos de error `2^31`Debe permanecer por debajo.`2^34`, por lo que el error debe ser al máximo 8.`(x * m) >> 34`igual `floor(x / 10)`por cada uno de los no negativos `int`¿ Qué ?

Para negativo `x`El piso se aleja de cero mientras que C se dirige hacia cero. `shrq $0x3f`extrae el bit de signo del producto, que es 1 exactamente cuando `x`Si se añade correcta el redondeo.

`magic_divisor(m, s)`prueba un par de candidatos.`d`como el cuotiente entero de `2^s`por `m`y trata de`d`y `d + 1`. acepta una`d`Sólo cuando`m`es igual al límite máximo de `2^s / d`y el error de la unión se mantiene.`s = 33`da .`d = 5`, lo cual es correcto, porque `0x66666667`es también el techo de `2^33 / 5`Una prueba verifica la identidad contra`c_div(x, 10)`En los últimos años, el número de puntos de referencia de la`int`- El rango.

Con el reconocimiento en,`grade`Leía`ecx_1 = param_1 / 10 - 6;`Con el reconocimiento apagado, la demo imprime la forma en bruto:

```c
rax_1 = (long)param_1 * 0x66666667;
ecx_1 = (int)(rax_1 >> 34) + (int)((unsigned long)rax_1 >> 63) - 6;
```

Cuando se lee el código de un compilador que no ha estudiado, la forma en bruto es la honesta.

## Etapa 7: Tipos de uso

El código de máquina tiene anchos y no tipos. `infer_types()`Una red cuyas definiciones son de 32 bits de ancho es`int`, y una web de 64 bits es `long`Una red que es la base de un acceso de elementos`base[idx]`con elementos de 4 bytes es `int *`El tipo de puntero se extiende a lo largo de copias y fichas, así que `local_8 = param_1`En el`-O0`hace a ambos .`int *`. El tipo de devolución es el ancho del valor en el `return`las declaraciones.

El resultado para `total`¿ Es verdad ?`int total(int *param_1, int param_2)`Ghidra escribió`int total(long param_1,uint param_2)`El levantador usó la carga escalada como evidencia de la señal.`*(int *)(param_1 + uVar2 * 4)`- Ni siquiera puede ver .`const`, y ninguno puede decir .`int *`de la`unsigned *`¿ Qué ?

Lee, Avgerinos y Brumley (TIE, NDSS 2011) recopilar el tipo de marco como resolución de restricciones sobre una red. El resultado es un límite superior e inferior en lugar de una conjetura. Noonan, Loginov y Cok (Retypd, PLDI 2016) añaden polimorfismo y recursión.`const`En esta lección se utilizan las dos reglas más baratas de esa línea de trabajo y se nombran lo que no pueden ver.

La aritmética sin firmar se mantiene en las expresiones y no en los tipos de variables. `ja`después de`cmpl $0x4, %ecx`Imágenes como `(unsigned int)ecx_1 > 4`, y `shrq`Imágenes como `(unsigned long)rax_1 >> 63`Las variables permanecen firmadas para que el lector vea el valor del registro y el único lugar donde la instrucción la trata como sin firmar.

## Etapa 8: La estructuración a partir del gráfico

Cifuentes (1994), capítulo 6, coincide con las regiones del gráfico en comparación con las formas de un lenguaje de alto nivel. Los bucles vienen primero (6.6.1), luego los condicionals bidireccionales (6.6.2), luego los condicionals n-direccionales (6.6.3). Yakdan, Eschweiler, Gerhards-Padilla y Smith (NDSS 2015) toman el otro camino. Derivan la condición bajo la cual cada nodo ejecuta y construyen la estructura a partir de esas condiciones, sin formas fijas. Su motivación es una medida. Hex-Rays en una muestra producida 1.571 `goto`Las declaraciones en 49.514 líneas, una por 32 líneas, y su estructuración no produjo ninguna de las 1.530 funciones de núcleo.

`structure()`Es un miembro pequeño de la primera familia. Necesita dos hechos de la etapa 2.`Branch`El borde trasero es un borde de un bloque que domina su fuente. El bucle natural es el conjunto de bloques que llegan a la fuente del borde trasero sin pasar el encabezado.

Por un`Branch`con unirse `j`¿Qué es esto ?

- Si ...`j`es el objetivo de salto, emitir`if (!cond) { fallthrough region }`y continuar en `j`¿ Qué ?
- Si ...`j`es el paso de caída, emitir`if (cond) { target region }`y continuar en `j`¿ Qué ?
- Si ...`j`es la salida de función, ambos brazos regresan.`if (cond) {... }`y el otro brazo después de él, sin un `else`Este es el regreso temprano en`percent_change`y `total`¿ Qué ?
- De lo contrario emitir`if (cond) { target } else { fallthrough }`y continuar en `j`- Esto es ...`percent_change`En el`-O0`, donde ambas armas se almacenan .`local_4`y nos vemos en el epílogo.

Para un encabezado de bucle:

- Si el encabezado termina en un `Branch`con un sucesor fuera del bucle, emitir `while (cond) { body }`- ¿ Qué ?`total`En el`-O0`tiene esta forma: el encabezado compara `local_14`con`local_c`y las hojas en `jge`¿ Qué ?
- Si el encabezado es también el bloque con el borde trasero, o el bloqueo sale en su lugar, emitir `do { body } while (cond);`- ¿ Qué ?`total`En el`-O1`tiene esta forma: bloque `0x50`añade, incrementa, compara y se vuelve a sí mismo.

Un `IndirectJump`termina un bloque sin sucesores.`grade`En el`-O0`Es una verdadera mesa de saltos.`jmpq *%rax`a través de cinco compensaciones relativas en `.rodata+0x0`En esta ocasión, el levantador imprime una nota.`0x76`¿ Qué ?`0x7f`¿ Qué ?`0x88`, y `0x91`Los resultados de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de la ciencia de`switch`¿ Qué ?

### ¿Qué hizo -O1 con el interruptor?

En el`-O1`Clang no guardó una mesa de salto para `grade`- Se calcula .`score / 10 - 6`y comparó el resultado con 4 como un número sin firmar. Luego cargó el valor de retorno de una tabla de cinco entradas de `int`En el`.rodata+0x14`La comparación sin firmar es una instrucción que cubre ambas cosas .`score / 10 < 6`y `score / 10 > 10`Una diferencia negativa es un valor sin firmar grande, por lo que no se realiza el mismo cheque.

La mesa se mantiene .`1, 2, 3, 4, 4`para los cuotientes 6 a 10.`switch`con cinco `return`las declaraciones se convirtieron en un `if`El elevador emite exactamente eso, y llama la mesa.`lookup_14`después de su compensación de sección.

La longitud de la tabla es heurística.`table_values()`Leer de la mesa de inicio a la siguiente `.rodata`compensa que cualquier `.text`En este objeto, el siguiente desplazamiento referido es `0x28`, la tabla de `grade_label`, así que`lookup_14`¿ Qué ?`(0x28 - 0x14) / 4 = 5`La comparación sin firmar contra 4 está de acuerdo. Ghidra tampoco sabe la longitud.`*(uint *)(&DAT_001000b4 + (ulong)uVar2 * 4)`, una dirección de byte más un offset, y dejó los cinco valores para que el lector busque.

## Junto con Ghidra

La demostración imprime cada par.`code/fixtures/ghidra/*.json`, los registros de la evidencia que rea regresó el 2026-10-14.`npx -y rea-agents@6.5.0 decompile ledger-x86_64-O1.o <function> --provider ghidra --format json`Cada disco lleva`confidence: observed`y `authority: shipped-artifact`El texto se observa. Las afirmaciones dentro del texto se derivan, y los registros no dicen cuáles.

### %_cambio

```text
lifter                                         | ghidra
-----------------------------------------------+-----------------------------------------------
int percent_change(int param_1, int param_2)   | undefined1  [16] percent_change(int param_1,int param_2,ulong param_3)
{                                              | {
  if (param_1 == 0) {                          |   long lVar1;
    return 0;                                  |   undefined1 auVar2 [16];
  }                                            |   undefined1 auVar3 [16];
  return (param_2 - param_1) * 100 / param_1;  |   if (param_1 != 0) {
}                                              |     lVar1 = (long)((param_2 - param_1) * 100);
                                               |     auVar3._0_8_ = lVar1 / (long)param_1 & 0xffffffff;
                                               |     auVar3._8_8_ = lVar1 % (long)param_1 & 0xffffffff;
                                               |     return auVar3;
                                               |   }
                                               |   auVar2._8_8_ = 0;
                                               |   auVar2._0_8_ = param_3;
                                               |   return auVar2 << 0x40;
                                               | }
```

| Difference | Instruction or guess |
|---|---|
| Return type `undefined1 [16]` against `int` | Guess. `idivl` writes `%rax` and `%rdx`, and no caller fixes the return width. Both registers were kept. |
| `param_3` | Guess that follows from the first. On the zero path only `%eax` is written, so `%rdx` is read as an input. |
| `lVar1 / (long)param_1 & 0xffffffff` against `/ param_1` | Instruction. `cltd` plus `idivl` is a 64-bit division of a sign-extended 32-bit value, truncated to 32 bits. Both texts describe it. Ours reduced the pair with the `signfill` rule. |
| `lVar1 % (long)param_1` kept | Follows from the return type. The remainder in `%rdx` is dead in our analysis and alive in Ghidra's. |
| `if (param_1 != 0)` with the zero case after it | Instruction order. `je 0xd` jumps to the zero return. Ghidra put the computed arm first, we put the smaller arm first. Both are correct. |
| `return auVar2 << 0x40` | Guess. A 16-byte value with `param_3` in the low half, shifted into the high half, is how the decompiler expresses `%rax = 0` with `%rdx` unknown. |

La versión reparada en `code/fixtures/ghidra/ledger-x86_64-O1.ghidra-repaired.c`La prueba de la fuente de entrada de 81 pares de datos.

### grado

```text
lifter                                         | ghidra
-----------------------------------------------+-----------------------------------------------
static const int lookup_14[5] = {1, 2, 3, 4, 4}; | uint grade(int param_1)
int grade(int param_1)                         | {
{                                              |   uint uVar1;
  int eax_1;                                   |   uint uVar2;
  int ecx_1;                                   |   
                                               |   uVar2 = param_1 / 10 - 6;
  ecx_1 = param_1 / 10 - 6;                    |   uVar1 = -(param_1 >> 0x1f) ^ (uint)((ulong)((long)param_1 * 0x66666667) >> 0x3f);
  eax_1 = 0;                                   |   if (uVar2 < 5) {
  if ((unsigned int)ecx_1 <= 4) {              |     uVar1 = *(uint *)(&DAT_001000b4 + (ulong)uVar2 * 4);
    eax_1 = lookup_14[(unsigned long)(unsigned int)ecx_1]; |   }
  }                                            |   return uVar1;
  return eax_1;                                | }
}                                              |
```

| Difference | Instruction or guess |
|---|---|
| `param_1 / 10 - 6` in both | Instruction. Both recognized the `0x66666667` idiom. |
| `uVar1 = -(param_1 >> 0x1f) ^ (uint)(... >> 0x3f)` against `eax_1 = 0` | Instruction `xorl %eax, %eax` at `grade+0x1a`. The register is zero. Ghidra expressed the `xor` as the sign of `param_1` exclusive-or the sign bit of the product, two spellings of the same bit, and did not cancel them. The value is 0 for every `int`, and the test proves it over 141 scores. A reader who trusts the text believes the default depends on the score. |
| `uVar2 < 5` against `(unsigned int)ecx_1 <= 4` | Instruction `cmpl $0x4, %ecx` then `ja`. Same condition. Ghidra typed `uVar2` as `uint`, we cast at the use. |
| `*(uint *)(&DAT_001000b4 + (ulong)uVar2 * 4)` against `lookup_14[...]` | Instruction plus relocation. Ghidra loaded the object with `.rodata` at `0x001000a0`, so `0x001000b4` is `.rodata + 0x14`. It applied the relocation and kept a byte address. We named the table and read its five values. |
| `uint` return against `int` | Guess on both sides. The instructions only say 32 bits. |

### total

```text
lifter                                         | ghidra
-----------------------------------------------+-----------------------------------------------
int total(int *param_1, int param_2)           | int total(long param_1,uint param_2)
{                                              | {
  int eax_1;                                   |   int iVar1;
  int ecx_1;                                   |   ulong uVar2;
  long rdx_1;                                  |   
                                               |   if (0 < (int)param_2) {
  if (param_2 <= 0) {                          |     uVar2 = 0;
    return 0;                                  |     iVar1 = 0;
  }                                            |     do {
  ecx_1 = param_2;                             |       iVar1 = iVar1 + *(int *)(param_1 + uVar2 * 4);
  rdx_1 = 0;                                   |       uVar2 = uVar2 + 1;
  eax_1 = 0;                                   |     } while (param_2 != uVar2);
  do {                                         |     return iVar1;
    eax_1 = eax_1 + param_1[rdx_1];            |   }
    rdx_1 = rdx_1 + 1;                         |   return 0;
  } while ((unsigned long)(unsigned int)ecx_1 != rdx_1); | }
  return eax_1;                                |
}                                              |
```

| Difference | Instruction or guess |
|---|---|
| `int *param_1` against `long param_1` | Inference on our side from `addl (%rdi,%rdx,4), %eax`. Ghidra kept the width. |
| `uint param_2` and `(int)param_2` against `int param_2` | Ghidra chose unsigned because `cmpq %rdx, %rcx` compares the zero-extended count. We kept `int` and cast at the compare. |
| `do { } while` in both | Instruction `jne 0x50` from the block to itself. The compiler rotated the loop. The `-O0` object keeps the `while`. |
| `ecx_1 = param_2` kept against `param_2 != uVar2` | Our substitution rule stays inside one block. The copy `movl %esi, %ecx` is in the setup block and the compare is in the loop, so the variable stays. Ghidra propagated it. |
| `(unsigned long)(unsigned int)ecx_1` | Instruction. `movl %esi, %ecx` zero-extends, and the 64-bit compare reads the full register. |

El examen de comportamiento recopila el texto de Ghidra para `grade`y `total`No se ha cambiado.`typedef`líneas y una definición de `DAT_001000b4`desde los veinte bytes en `.rodata+0x14`Ambas funciones coinciden con la fuente en la red.

## Lo que dicen las investigaciones sobre la lectura de Pseudocodo

LLM4Decompile (EMNLP 2024) midió Ghidra solo en una re-executibilidad del 20,12 por ciento en HumanEval-Decompile. Alrededor de una de cada cinco funciones recopiló y aprobó las afirmaciones que vienen con el índice de referencia. Al refinar la producción de Ghidra con su modelo 6.7B aumentó eso al 52.74 por ciento. DecompileBench (Findings of ACL 2025) ejecutó seis decompiladores industriales y seis enfoques de LLM en 23,400 funciones de 130 programas. Los resultados del LLM se leeron mejor y tenían una corrección de funcionalidad 52,2% menor.

Liu, Raff y Micinski (arXiv 2609.05370, septiembre 2026) explican la brecha. Utilizaron 300 funciones de biblioteca de GitHub y 287 funciones basadas en CVE. El modelo de refinamiento más fuerte aumentó la tasa de construcción de Ghidra de 75 a 90 por ciento mientras que la tasa de coincidencia de comportamiento cayó de 74 a 62 por ciento. Los candidatos que pasaron todas las pruebas en el índice de referencia todavía divergieron en las entradas despejadas. Eso ocurrió en el 4,9 por ciento de los casos en ocho sistemas, y en hasta el 13 por ciento para un sistema. Hasta una décima parte de las vulnerabilidades reveladas desaparecieron del código refinado.

Las causas que rastrean son simples: campos inventados, tipos, calles y guardias reemplazaron las incógnitas visibles que el decompiler había dejado en el texto.

Esta es la razón por la que el elevador imprime .`(unsigned long)(unsigned int)ecx_1`Imprime una nota en lugar de un nombre más bonito.`switch`para el salto no resuelto, y `lookup_14`En lugar de una etiqueta que un modelo inventaría.`code/main.py`- El de Ghidra.`undefined1 [16]`Es feo y honesto.`int percent_change(int, int)`El mismo modelo producirá una firma equivocada en la siguiente función.

## Construye el mismo

Ejecutar la demostración:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode
python3 code/main.py
```

La demostración decodifica las cuatro funciones de la`-O1`Objeto y compara cada instrucción con la cometida `llvm-objdump`¿Qué es lo que se dice?`percent_change`¿ Qué ?`grade`, y `total`Imprime el IR antes de SSA, la lista SSA con funciones phi y el pseudocodo al lado del texto Ghidra.`grade`con el reconocimiento de división apagado, el `-O0`versiones de `percent_change`¿ Qué ?`total`, y `grade_label`, y un resumen:

```json
{
  "functions": ["grade", "grade_label", "percent_change", "total"],
  "instructions": 56,
  "blocks": {"percent_change": 3, "grade": 3, "total": 5, "grade_label": 4},
  "phis": {"percent_change": 0, "grade": 1, "total": 2, "grade_label": 1},
  "objdumpDifferences": 0
}
```

Leer .`code/main.py`en este orden:

1. `read_object()`se lee los encabezados de la sección, la tabla de símbolos y el `.rela`secciones de un objeto relocable ELF64.
2. `decode_one()`y `render_att()`Descifrar una instrucción y imprimirlo en el camino `llvm-objdump`- Sí, lo hace.
3. `build_blocks()`¿ Qué ?`immediate_dominators()`, y `dominance_frontiers()`construye los hechos gráficos.
4. `lift_insn()`convertir una instrucción en declaraciones de IR y resolver las reubicaciones.
5. `insert_phis()`y `rename()`construir SSA e insertar lanzamientos de ancho.
6. `make_simplifier()`¿ Qué ?`substitute_once()`, y `remove_dead_code()`Corren hacia adentro .`optimize()`¿ Qué ?
7. `collect_params()`¿ Qué ?`coalesce_webs()`, y `infer_types()`nombre y escribir las redes.
8. `structure()`con`post_dominators()`y `natural_loops()`emite el cuerpo C.
9. `lift_function()`se unen las etapas y `lift_all()`Levanta un objeto entero con firmas de llamadas.

## Usalo

Para levantar una función de su propio objeto, construye con `clang -target x86_64-unknown-linux-gnu -c`y llamar`lift_all(read_object(path))`Cuando el decodificador se eleva`DecodeError`, lee el byte de código optico del mensaje y encuentra en el mapa de código optico SDM. Luego añada una entrada a las tablas en la parte superior del archivo. Cuando el elevador eleva en un mnemónico, añada una regla a `lift_insn()`Cuando el constructor emite un`goto`, imprimir el gráfico con `IrPrinter`Cada adición es pequeña y cada una se comproba con`llvm-objdump`o el clang por los ensayos existentes.

Cuando se instale Ghidra, ejecuta rea en la misma función y coloca los dos textos uno al lado del otro. Llena la tabla de diferencia antes de creer cualquiera de los textos. Una diferencia con una instrucción detrás de ella es una elección de presentación. Una diferencia sin instrucción detrás de ella es una suposición, y la suposición puede ser de ambos lados.

## Laboratorio interactivo

### Laboratorio A: una función a través de cinco etapas

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
obj = read_object(FIXTURES / "ledger-x86_64-O1.o")
lifted = lift_function(obj, "percent_change")
for insn in lifted.decoded: print(f"0x{insn.address:x}: {insn.raw.hex(' ')}  {insn.att()}")
print(lifted.ir_text)
print(lifted.ssa_text)
print(lifted.pseudocode)
```

Sigue .`%esi`de la`subl %edi, %esi`¿ Qué ?`esi_1 = esi_0 - edi_0`¿ Qué ?`(param_2 - param_1)`El byte .`fe`es el ModR/M de la `subl`, con `mod = 11`¿ Qué ?`reg = edi`, y `r/m = esi`¿ Qué ?

### Laboratorio B: el lenguaje de división

```python
print(lift_function(obj, "grade", recognize_division=False).pseudocode)
print(magic_divisor(0x66666667, 34), magic_divisor(0x66666667, 33), magic_divisor(0x66666667, 31))
```

La primera línea imprime el multiplicador bruto y los cambios.`10 5 None`Descubra por qué el mismo multiplicador se divide por 5 con un cambio menos, y por qué el cambio 31 no cumple con la línea de error.

### Laboratorio C: un nuevo objeto detiene el decodificador

Construir el dispositivo en `-O2`y levantarlo:

```bash
clang -target x86_64-unknown-linux-gnu -O2 -fno-asynchronous-unwind-tables -c fixtures/ledger.c -o "$TMPDIR/ledger-O2.o"
python3 -c 'from main import *; from pathlib import Path; import os; lift_all(read_object(Path(os.environ["TMPDIR"]) / "ledger-O2.o"))'
```

El decodificador se detiene en `0x56`con`opcode 81 is outside the fixture subset`- Código de opciones .`81`Es el formulario de grupo 1 con una inmediata de 32 bits.`andl $0x7ffffff8, %edx`, y Clang lo usó en el bucle vectorizado de `total`- Añade a la parte de la parte .`0x83`En el`decode_one()`El subconjunto crece una entrada a la vez, y cada entrada se comproba con `llvm-objdump`¿ Qué ?

### Laboratorio D: Valor predeterminado de Ghidra en grado

```python
print(ghidra_pseudocode("grade"))
for score in (-25, 0, 59, 60, 100, 115):
    sign = -(score >> 31)
    product_sign = ((score * 0x66666667) & ((1 << 64) - 1)) >> 63
    print(score, sign ^ product_sign)
```

Cada fila imprime 0, Ghidra `uVar1`La expresión es correcta e ilegible, y la instrucción `xorl %eax, %eax`lo dice en dos bytes.

## Laboratorio de práctica

Añadir`switch`recuperación para `grade`En el`-O0`¿ Qué ?

Requisitos:

- Lea las cinco reubicaciones en `.rela.rodata`de `ledger-x86_64-O0.o`, cada uno `R_X86_64_PC32.text+offset`, y calcular los cinco objetivos del caso desde la tabla en `.rodata+0x0`¿ Qué ?
- Añadir un`Switch`terminador en la IR con la expresión de índice, la lista de objetivos y el valor predeterminado de la `ja`antes de la carga de la mesa.
- Dale al bloque de interruptor un sucesor por objetivo para que los dominadores, posdominadores y SSA sigan trabajando.
- Emite`switch (local_10) { case 0:... }`con uno `case`por objetivo y el epílogo compartido después de la unión.
- Guarde la nota para cualquier salto indirecto cuya mesa no pueda atar.
- Compile el resultado con clang y compare con la fuente sobre la cuadrícula de puntuación en las pruebas.

El éxito significa`grade`En el`-O0`y `grade`En el`-O1`producen la misma salida en la red, con una `switch`Y no .`goto`¿ Qué ?

## Artículo

Esta lección produce`outputs/skill-lifting-to-pseudocode.md`. Utilice para levantar una función con una derivación para cada línea y para llenar la tabla de diferencia con el texto de un descompilador. El artefacto requiere el desmonte de referencia, las reubicaciones, la convención de llamada, la composición de composición, la comparación de comportamiento y un veredicto para cada línea diferente.

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

La suite tiene 25 pruebas.

- el decodificador está de acuerdo con el compromiso `llvm-objdump`texto en cada instrucción de ambos objetos, y con un vivo `llvm-objdump`cuando se instala
- REX, ModR/M, SIB y el desplazamiento de campo de compensación de `leaq (%rip), %rcx`se decodifican como se describe en el MDS
- un código op fuera del subconjunto aumenta `DecodeError`En vez de adivinar
- El lector de ELF enumera las cuatro funciones y las tres `.text`reubicaciones
- `total`tiene cinco cuadras,`0x44`domina .`0x50`, y `0x50`es un encabezado de bucle con sí mismo como el cuerpo
- `%rdi`y `%rsi`¿ Cómo se hace ?`param_1`y `param_2`, y `param_1`En el`total`¿ Es verdad ?`int *`
- `test`y `cmp`con`je`¿ Qué ?`jle`¿ Qué ?`ja`, y `jge`¿ Cómo se hace ?`==`¿ Qué ?`<=`, sin firmar .`>`, y `<`
- SSA coloca dos funciones Phi en vivo en el encabezado de bucle de `total`y ninguno en .`percent_change`
- el pseudocódigo de `percent_change`¿ Qué ?`grade`, y `total`igual al texto esperado
- `grade`Leía`param_1 / 10 - 6`con reconocimiento en y el bruto multiplicarse con reconocimiento fuera
- `0x66666667`con el turno 34 dividido por 10, con el turno 33 dividido por 5, y la identidad se mantiene en la cuadrícula
- las ranuras de pila en `-O0`¿ Cómo se hace ?`local_4`¿ Qué ?`local_8`¿ Qué ?`local_c`, y el marcador de marco nunca aparece
- la mesa en `.rodata+0x14`Leía`1, 2, 3, 4, 4`a través de la reubicación
- el salto indirecto en`grade`En el`-O0`se informa con sus cuatro bloques sin levantar
- Las lámparas de Ghidra contienen el retorno de 16 bytes, el `param_3`, el no simplificado .`xor`, y el `long param_1`
- las seis unidades emitidas se compilan con `clang -std=c11 -Wall -Werror`por`x86_64-unknown-linux-gnu`
- las funciones levantadas, el texto reparado de Ghidra, y la fuente imprimen las mismas 232 líneas sobre la cuadrícula de entrada

## Modos de falla de producción

| Failure | What the weak reading reports | What the derivation must show |
|---|---|---|
| Decompiler widens the return value | "returns a 16-byte struct" | `idivl` writes `%rdx`, and no caller fixes the width |
| Decompiler adds a parameter | "takes three arguments" | The register is read on one path only because the return width was guessed |
| Idiom goes unrecognized | "multiplies by a magic constant" | The multiplier is the ceiling of `2^s / d` and the error bound holds |
| Idiom is recognized and a `xor` is lost | "default depends on the input" | `xorl %eax, %eax` writes zero, two spellings of one bit cancel |
| Pointer typed as integer | "takes a long" | A scaled 4-byte load from the register is pointer evidence |
| Table length unknown | "reads past the table" | The next referenced offset or the section end bounds it, and the range check agrees |
| Loop shape misread | "while loop" | The header has no exit, the latch jumps back, so it is `do while` |
| Indirect jump guessed | "switch with five cases" | The targets come from `.rela.rodata`, or the jump is reported as unresolved |
| Substitution across a store | "one expression" | A loaded value moved past a store or a call changes meaning |
| Refinement model edits the text | "compiles and looks right" | Build rate up, behaviour match down, invented guards replaced visible unknowns |
| Decoder guesses an unknown opcode | "disassembly looks fine" | `DecodeError` with the address and the byte, then one table entry checked against `llvm-objdump` |

## Regla de funcionamiento

Trate el pseudocodo como una derivación que no ha visto. Reproduce a partir de las instrucciones con un elevador que puede leer. Coloque los dos textos uno al lado del otro, y escriba la instrucción o la adivinación detrás de cada línea que difiere. Compile ambas y ejecute ambas contra la fuente cuando lo tengas. Una línea sin instrucción detrás de ella permanece como una adivinación sin importar cuán limpia lea.

## Leer más

- [Intel SDM Volume 2, chapter 2, Instruction Format](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [System V AMD64 psABI, section 3.2, Function Calling Sequence](https://gitlab.com/x86-psABIs/x86-64-ABI)
- [Cytron et al., Efficiently Computing Static Single Assignment Form and the Control Dependence Graph, TOPLAS 1991](https://doi.org/10.1145/115372.115320)
- [Cooper, Harvey, Kennedy, A Simple, Fast Dominance Algorithm, 2001](https://www.cs.tufts.edu/~nr/cs257/archive/keith-cooper/dom14.pdf)
- [Cifuentes, Reverse Compilation Techniques, PhD thesis, QUT 1994](https://eprints.qut.edu.au/36820/)
- [Yakdan et al., No More Gotos, NDSS 2015](https://www.ndss-symposium.org/ndss2015/ndss-2015-programme/no-more-gotos-decompilation-using-pattern-independent-control-flow-structuring-and-semantics/)
- [Granlund, Montgomery, Division by Invariant Integers using Multiplication, PLDI 1994](https://doi.org/10.1145/178243.178249)
- [Lee, Avgerinos, Brumley, TIE, NDSS 2011](https://www.ndss-symposium.org/ndss2011/tie-principled-reverse-engineering-of-types-in-binary-programs/)
- [Noonan, Loginov, Cok, Polymorphic Type Inference for Machine Code, PLDI 2016](https://arxiv.org/abs/1603.05495)
- [LLM4Decompile, EMNLP 2024](https://arxiv.org/abs/2403.05286)
- [DecompileBench, Findings of ACL 2025](https://aclanthology.org/2025.findings-acl.1194/)
- [Liu, Raff, Micinski, When LLM Decompilers Recompile More and Preserve Less, 2026](https://arxiv.org/abs/2609.05370)
- [rea 6.5.0 Ghidra provider notes](https://github.com/morluto/rea/blob/rea-agents-6.5.0/docs/ghidra-provider.md)
- rea 6.5.0 fuente en etiqueta `3b79af2`que esta lección describe: `src/domain/functionComparisonDimensions.ts`¿ Qué ?`website/evidence/dx-ball-sound-pan.md`¿ Qué ?
