# Desmontaje y flujo de control: de bytes a gráfico que puedes comprobar

> Una lista de desmontaje es una afirmación sobre qué bytes son instrucciones, y esta lección construye el decodificador y el gráfico que compruebe la afirmación.

**Type:** Build
**Languages:** Python, C, x86-64 assembly
**Prerequisites:** Phase 13 · 32 (ELF from scratch)
**Time:** ~150 minutes

## Objetivos de aprendizaje

- Decodificar el subconjunto de instrucciones x86-64 en las fijas del libro mayor de las reglas del volumen 2 del capítulo 2 de Intel SDM.
- Prueba el decodificador contra `llvm-objdump -d`en todas las instrucciones de cada edificio.
- Realice una barrida lineal y una descensía recursiva sobre los mismos bytes y explique dónde cada uno está equivocado.
- Cortar las instrucciones en bloques básicos y construir un gráfico de flujo de control con bordes tipografados.
- Computa los dominadores inmediatos con el algoritmo iterativo de Cooper, Harvey y Kennedy, y encuentra los bucles desde los bordes posteriores.
- Recuperar los objetivos de un salto indirecto de las entradas de reubicación que apuntan a su mesa.

## El problema

Usted compilado`ledger.c`En el`-O0`y preguntó a un agente qué.`grade`El agente llamó a un desmonteador, leyó la lista y respondió que la función termina en`jmpq *%rax`Luego escribió un pseudocodo con un solo`return 0`¿ Qué ?

Abres el mismo objeto en`llvm-objdump -d`y ver cuatro bloques después del salto que almacenan 4, 3, 2 y 1. La lista los tenía todo el tiempo. Nada en la lista dice cómo la ejecución llega a ellos, por lo que la herramienta los trató como código muerto.

Una lista responde a una sola pregunta: ¿quién byte forma qué instrucción? No dice qué instrucciones se ejecutan, en qué orden o desde dónde. Para responder a ellas se necesitan tres estructuras más: los bytes que el flujo de control puede alcanzar, los bloques básicos y los bordes entre ellos.

El rea 6.5.0 no tiene desmontaje propio. Para un análisis profundo, su README le pide que traiga Ghidra, Hopper o IDA. Para la inspección fuera de línea, rea también puede llamar a un objdump GNU o un Rizin que instales. Su gráfico de llamadas se detiene en objetivos no resueltos: "Las llamadas/llamadas contienen solo funciones resueltas.... El flujo informático sin objetivo sigue siendo desconocido". Esta lección construye la parte que produce esos hechos, para que puedas juzgarlos cuando una herramienta los informa.

```figure
re-33-sweep-vs-descent
```

## El formato de instrucciones tiene una regla por campo

Intel SDM Volumen 2, la figura 2-1 da el orden de byte de cada instrucción. Primero vienen prefijos opcionales, luego uno a tres bytes de código opcional, luego un byte ModRM opcional y un byte SIB opcional. Se sigue un desplazamiento de 0, 1, 2 o 4 bytes, luego un inmediato de 0, 1, 2 o 4 bytes. La instrucción completa será de un máximo de 15 bytes (SDM 2.3.11). AMD APM Volumen 3, Capítulo 1, Secciones 1.1 a 1.7 describen el mismo formato en menos páginas y son una buena verificación cruzada.

Cada campo decide lo que significa el siguiente campo. Un decodificador es por lo tanto una cadena de pequeñas decisiones, y cada decisión tiene una oración en el manual detrás de ella.

### Los prefijos se dividen en cuatro grupos

El MDS 2.1.1 incluye los prefijos heredados.`F0`bloqueo y los prefijos de repetición `F2`y `F3`. El grupo 2 tiene las seis superposiciones del segmento (`2E`¿ Es verdad ?`%cs`El Grupo 3 es `66`, el tamaño de la operanda.`67`, el tamaño de la dirección de anulación.

El manual dice que se puede usar un prefijo de cada grupo.`ledger-x86_64-O0.o`en compensación `0xf2`Leía`66 66 66 66 66 2e 0f 1f 84 00 00 00 00 00`, cinco copias de `66`en una fila. `llvm-objdump`Imprima como `nopw %cs:(%rax,%rax)`El decodificador recoge cada byte prefijo que ve y luego pregunta cuáles importan.

`66`Para las instrucciones de SSE es un prefijo obligatorio que selecciona el código op, por lo que `66 0f ef c0`¿ Es verdad ?`pxor %xmm0, %xmm0`y no un 16 bits nada.`F3 0f 6f`¿ Es verdad ?`movdqu`El decodificador primero busca el par de prefijos obligatorios y el segundo byte de opcode. Sólo cuando ese par no tiene entrada lo hace`66`Es decir, una operanda de 16 bits.

### REX añade cuatro bits y debe sentarse al lado del código op

La SDM 2.2.1 define REX como un byte de `40`¿ Qué ?`4F`con la forma`0100WRXB`- ¿ Qué ?`W`selecciona una operanda de 64 bits. `R`se extiende el campo de reglamentación de la ModRM. `X`se extiende el campo del índice SIB. `B`Extiende el campo ModRM r/m, el campo base SIB o el registro en el byte de código optico (cuadro 2-4). REX debe aparecer justo antes del código optico o los bytes de escape, y se ignora un REX seguido de un prefijo heredado.

En el`48 89 e5`el byte `48`es REX con `W`El código op `89`¿ Es verdad ?`MOV r/m, r`, y ModRM `e5`selecciona`%rsp`en el`%rbp`. Retirando el REX y los mismos bytes son `movl %esp, %ebp`¿ Qué ?

REX cambia también los nombres de los registros de byte.`%spl`¿ Qué ?`%bpl`¿ Qué ?`%sil`, y `%dil`en lugar de`%ah`¿ Qué ?`%ch`¿ Qué ?`%dh`, y `%bh`El decodificador guarda dos mesas por esa razón.

### Los códigos de acceso se encuentran en dos mapas

Un código op de un byte es el caso común. `0F`Escapes al mapa de dos bytes, y `0F 38`o `0F 3A`Los sistemas de opticidad de los sistemas de opticidad de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de las cámaras de luz de luz de las cámaras de luz de las cámaras de luz de luz de las cámaras de luz de las cámaras de luz de luz de las cámaras de luz de luz de las cámaras de luz de luz de luz de las cámaras de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de luz de`/digit`En el manual.`83 /7`¿ Es verdad ?`cmp`con un instantáneo de 8 bits, `83 /5`¿ Es verdad ?`sub`, y `FF /4`es el indirecto`jmp`¿ Qué ?

El decodificador en `code/main.py`cubre los códigos opcodes en los cuatro objetos fijos más algunos vecinos.`mov`¿ Qué ?`lea`¿ Qué ?`movslq`¿ Qué ?`movzbl`¿ Qué ?`imul`, los turnos,`test`, y `idiv`. continúa con `push`¿ Qué ?`pop`¿ Qué ?`call`¿ Qué ?`ret`, los dieciséis saltos condicionales, el multi-byte `nop`, y cinco códigos opticos SSE2. Cualquier otro código optico aumenta`DecodeError`Un decodificador que adivina es peor que uno que se detiene.

### ModRM y SIB seleccionan los operandos

El MDS 2.1.3 divide el MDRM en `mod`(bites 7:6), `reg`(5:3), y `r/m`(2:0) `mod = 11`significa que r/m es un registro. `mod = 00`¿ Qué ?`01`, y `10`memoria media sin desplazamiento, un desplazamiento de 8 bits, o un desplazamiento de 32 bits. `r/m = 100`significa que un byte SIB sigue.`scale`(7:6), `index`(5:3), y `base`(2:0), y un índice de `100`significa que no hay registro de índices.

Dos filas especiales en el cuadro 2-2 son de uso en modo de 64 bits. `mod = 00`con`r/m = 101`No es`(%rbp)`. Es un desplazamiento de 32 bits en relación con la siguiente instrucción (SDM 2.2.1.6). `mod = 00`con una base de SIB de `101`es un desplazamiento de 32 bits sin registro de base.

¿ Qué ?`8d 44 08 fa`de la`grade`En el`-O0`El código op.`8D`¿ Es verdad ?`lea`- ModRM .`44`¿ Es verdad ?`01 000 100`: un desplazamiento de 8 bits, reg `%eax`, y un SIB sigue.`08`¿ Es verdad ?`00 001 000`: escala 1, índice `%rcx`, base `%rax`. El byte de desplazamiento `fa`es -6. El texto de AT&T es `leal -0x6(%rax,%rcx), %eax`, que calcula .`score / 10 - 6`, el índice de interruptores.

### El desplazamiento es donde el editor de enlaces escribe

`48 8d 0d 00 00 00 00`¿ Es verdad ?`leaq (%rip), %rcx`Los cuatro bytes cero son el campo de desplazamiento de un operando relativo al RIP. En un objeto relocable son poseedores de lugar.`.rela.text`en compensación `0x69`dice`R_X86_64_PC32.rodata-0x4`. El psABI (cuadro 4.9) define `R_X86_64_PC32`¿ Cómo ?`S + A - P`: la dirección del símbolo más el adjetivo menos la dirección del campo.

El procesador agrega el campo a la dirección de la siguiente instrucción, por lo que el objetivo final es `S + A + (end - P)`- Por esto .`lea`el campo es los últimos cuatro bytes, así que `end - P`es 4 y el objetivo es `.rodata + (-4) + 4 =.rodata + 0x0`. El adendo de -4 compensa esos cuatro bytes de antemano.`.rodata+0x10`en el `-O1` `lea`medio `.rodata+0x14`¿ Qué ?

El decodificador registra el desplazamiento del campo de desplazamiento dentro de cada instrucción. `resolve_reference()`Se trata de una función que se desplaza en la tabla de reubicación y informa donde el operando apunta después de vincular.`leaq (%rip), %rcx`En la mesa en`.rodata+0x0`" . " .

### Las instantáneas son de 32 bits como máximo, con una excepción

SDM 2.2.1.5 dice que los instantáneos permanecen al máximo 32 bits en modo de 64 bits y son de signo extendido.`MOV r64, imm64`(El artículo`B8+r`con`REX.W`), que la sintaxis AT&T llama `movabsq`. El decodificador lee 8 bits inmediatos como firmado para aritmética (`83 c1 fa`¿ Es verdad ?`addl $-0x6, %ecx`) y sin firmar para los recuentos de turnos y selectores de mezcla (`pshufd $0xee`Eso coincide con qué .`llvm-objdump`Las huellas.

Cerca de las ramas,`push`, y `pop`por defecto a un operando de 64 bits e ignora `66`(MDS 2.2.1.7) Por eso el decodificador las nombra `pushq`¿ Qué ?`popq`¿ Qué ?`callq`, y `retq`sin leer un prefijo de tamaño operand.

## Descifrar una instrucción a mano

Antes de confiar en el código, decodifica.`48 69 c0 67 66 66 66`de la`grade`En el papel.

1. `48`¿ Está en ?`40`¿ Qué ?`4F`, así que es REX con `W = 1`El tamaño de la operanda es de 64 bits.
2. `69`¿ Es verdad ?`IMUL r, r/m, imm32`¿ Qué ?
3. `c0`es ModRM `11 000 000`: formulario de registro, reg `%rax`, r/m `%rax`¿ Qué ?
4. Los siguientes cuatro bytes son el inmediato, pequeño-endian: `0x66666667`¿ Qué ?
5. La orden de AT&T es inmediata, fuente, destino: `imulq $0x66666667, %rax, %rax`¿ Qué ?

Eso es siete bytes, y `llvm-objdump`Imprime el mismo texto. La constante es la recíproca que permite que el clang se divide por 10 con un multiplicador y el `sarq $0x22`que sigue, que cambia por 34. Se encontrará con ese par de nuevo en la lección 34, donde el elevador vuelve a convertirlo en`score / 10`¿ Qué ?

El decodificador hace estos cinco pasos en `Decoder.decode()`¿ Qué ?`_State.modrm()`, y `_State.finish()`. Leerlos en ese orden y emparejar cada rama con la sección de SDM nombrada en su docstring.

## Barrear lineal

Schwarz, Debray y Andrews describen el barrido lineal en la sección 3.1 de "Desmontaje de código ejecutable revisado" (WCRE 2002).`objdump`funciona de esta manera, y también lo hace `llvm-objdump`¿ Qué ?

```python
def linear_sweep(code: bytes, start: int = 0, end: int | None = None) -> SweepResult:
    position = start
    while position < end:
        try:
            instruction = decode(code[:end], position)
        except DecodeError:
            undecodable.append((position, code[position]))
            position += 1
            continue
        instructions.append(instruction)
        position = instruction.end
```

El método tiene un modo de falla, y la figura 1 del papel lo muestra. Los bytes que son datos se decodifican como instrucciones, y el decodificador luego pierde los límites reales de instrucciones por un tiempo. En x86, el decodificador generalmente se sincroniza de nuevo dentro de unos pocos bytes, lo que hace que el error sea difícil de ver. La detección solo ocurre en un código opálico inválido, y la conclusión del documento es que estos algoritmos "fracasan silenciosamente".

El barrido es también el oráculo para esta lección.`llvm-objdump -d`en cada instrucción de cada fijación, y `llvm-objdump`El texto capturado vive en`code/fixtures/*.objdump.txt`. El ensayo analiza cada línea en direcciones, bytes y texto, deja de lado los comentarios después `#`Los cuatro objetos contienen 86, 56, 84 y 11 instrucciones, 237 en total, y el decodificador coincide con todos ellos.

## Descenso recurrente

El segundo método se llama recurrente. Comienza en un punto de entrada conocido y decodifica hasta que se haga un transferencia de control. Para un salto directo, continúa en el objetivo. Para una rama condicional, continúa tanto en el objetivo como en la siguiente instrucción, y para una llamada, continúa después de ella, porque el call regresa. Para un regreso, deténgase. Mantenga un conjunto de direcciones visitadas para que termine el paseo.

```python
if instruction.flow == "branch":
    worklist.append(instruction.target)
    address = instruction.end
elif instruction.flow == "indirect-jump":
    table = resolver(instruction, instructions) if resolver else None
    if table is None:
        unresolved.append(instruction)
        break
```

El modo de falla es el contrario. Un salto indirecto no tiene un objetivo en sus bytes. El paseo se detiene allí, y cada bloque alcanzable solo a través del salto permanece desconocido. La figura 2 del papel muestra un segundo peligro: un conjunto de objetivos derivado de relocalizaciones que apunta al medio de una instrucción.`main.py`registra ese caso como un conflicto y detiene ese camino, por lo que la superposición es visible en lugar de decodificarse silenciosamente dos veces.

Andriesse y otros midieron nueve desmontables en 981 binarios generados por compiladores (USENIX Security 2016). Los datos en línea y el código superpuestos eran más raros en la salida real del compilador que la literatura asumió, y el primitivo duro fue la recuperación de límites de función.`grade`Es común.

## Un recinto donde el barrido es incorrecto y el descenso es correcto

`code/fixtures/sweep-trap.s`es 24 bytes de conjunto escrito a mano con una tabla de cuatro bytes dentro del cuerpo de la función.`movzbl (%rcx,%rax), %eax`y salta sobre él.

```asm
lookup:
        movslq  %edi, %rax
        leaq    .Ltable(%rip), %rcx
        movzbl  (%rcx,%rax), %eax
        jmp     .Ldone
.Ltable:
        .byte   0x31, 0xc0, 0x90, 0x48
.Ldone:
        addl    $1, %eax
        retq
```

Construye con `clang -target x86_64-unknown-linux-gnu -c sweep-trap.s`Los cuatro bytes de la tabla fueron elegidos para que se decodificaran como instrucciones plausibles.`31 c0`¿ Es verdad ?`xorl %eax, %eax`- ¿ Qué ?`90`¿ Es verdad ?`nop`- ¿ Qué ?`48`es un prefijo REX, así que el barrido se une con los tres bytes del real `addl $1, %eax`y huellas`addq $0x1, %rax`Los capturados .`llvm-objdump`la salida muestra exactamente que:

```text
       e: eb 04                        	jmp	0x14 <lookup+0x14>
      10: 31 c0                        	xorl	%eax, %eax
      12: 90                           	nop
      13: 48 83 c0 01                  	addq	$0x1, %rax
      17: c3                           	retq
```

Nada en esta lista parece roto. cada byte decodificado, cada mnemónica es real, y el`retq`La lista sigue siendo incorrecta en tres aspectos.`0xe`objetivos `0x14`, y ninguna instrucción en la lista comienza en `0x14`La instrucción que funciona en`0x14`es un 32 bits `addl`, y la lista muestra un 64 bits `addq`los cuatro bytes de la tabla aparecen como código.

Descenso recurrente de la`lookup`El símbolo decodifica seis instrucciones, sigue el salto a `0x14`, y informes `addl $0x1, %eax`No reclaman ninguno de los cuatro bytes de la mesa y ninguno de los ocho bytes de relleno antes .`plus_one`- ¿ Qué ?`trap_report()`En el`main.py`Imprime la comparación:

```json
"linearSweep": {
  "instructions": 9,
  "atJumpTarget": {"starts": "0x13", "text": "addq\t$0x1, %rax", "startsAtTarget": false},
  "claimedDataBytes": 4,
  "claimedPaddingBytes": 8
},
"recursiveDescent": {
  "instructions": 6,
  "atJumpTarget": "addl\t$0x1, %eax",
  "claimedDataBytes": 0,
  "claimedPaddingBytes": 0
}
```

El examen`test_sweep_and_objdump_share_the_same_mistake`afirma que `llvm-objdump`Impreso`addq $0x1, %rax`En el`0x13`El oráculo es un barrido lineal y hereda su punto ciego.

## Bloques básicos

Allen definió el bloque básico en "Análisis de flujo de control" (1970). Es una secuencia de instrucciones que se introduce solo en su primera instrucción y se deja solo en su última. Yakdan y otros lo reanudan para descompilación en la Sección II.A.2 de "No más Gotos" (NDSS 2015), con el nodo de encabezado como la entrada de función.

La construcción es un escaneo de dos pasos sobre las instrucciones decodificadas, ordenadas por dirección. El primer paso recoge líderes. Un líder es la entrada, una rama o un objetivo de salto, un objetivo de mesa o la instrucción después de cualquier transferencia de control. El segundo paso inicia un nuevo bloque en cada líder y en cada hueco en la secuencia de direcciones.

```python
for instruction in self.instructions:
    if instruction.address in leaders or current is None or current.end != instruction.address:
        current = Block(instruction.address, [])
        self.blocks[instruction.address] = current
    current.instructions.append(instruction)
```

Los bordes siguen de la última instrucción de cada bloque.`jump`Una rama condicional tiene dos, etiquetados.`taken`y `fallthrough`Una instrucción simple al final de un bloque pasa al siguiente bloque, lo que ocurre cuando la siguiente instrucción es un blanco de rama.`table`, o ninguno, y el bloque se registra en `unresolved`¿ Qué ?

Dos hechos más salen del mismo escaneo. Un bloque que la entrada no puede alcanzar es inaccesible, y el barrido sobre.`grade`En el`-O0`Un bloque cuya dirección de inicio cae dentro de una instrucción de la lista de otro método es el caso de superposición de la trampa.

## El gráfico de flujo de control del total

`total`En el`-O1`es 31 bytes y 13 instrucciones:

```text
      40: 85 f6                        	testl	%esi, %esi
      42: 7e 18                        	jle	0x5c <total+0x1c>
      44: 89 f1                        	movl	%esi, %ecx
      46: 31 d2                        	xorl	%edx, %edx
      48: 31 c0                        	xorl	%eax, %eax
      4a: 66 0f 1f 44 00 00            	nopw	(%rax,%rax)
      50: 03 04 97                     	addl	(%rdi,%rdx,4), %eax
      53: 48 ff c2                     	incq	%rdx
      56: 48 39 d1                     	cmpq	%rdx, %rcx
      59: 75 f5                        	jne	0x50 <total+0x10>
      5b: c3                           	retq
      5c: 31 c0                        	xorl	%eax, %eax
      5e: c3                           	retq
```

Los líderes son`0x40`(entrada), `0x44`(después de la `jle`), `0x50`(Objetivo de la`jne`), `0x5b`(después de la `jne`), y `0x5c`(Objetivo de la`jle`5 bloques, 5 bordes:

| From | To | Kind | Why |
|---|---|---|---|
| `0x40` | `0x44` | fallthrough | `jle` not taken, count is positive |
| `0x40` | `0x5c` | taken | count is zero or negative, return 0 |
| `0x44` | `0x50` | fallthrough | the `nopw` is alignment inside the block, execution passes through it |
| `0x50` | `0x50` | taken | `jne 0x50`, the loop |
| `0x50` | `0x5b` | fallthrough | loop exit, return the sum |

El de seis bytes `nopw`En el`0x4a`es relleno para que el bucle encaje en `0x50`Está en un límite de 16 bytes. Es accesible, por lo que pertenece al bloque`0x44`El un-byte .`nop`En el`0x5f`Después de la última .`retq`La barrida sobre la función hace un sexto bloque para ella, y `unreachable()`lo enumera.

En el`-O2`La misma función es de 140 bytes con dos bucles.`0x80`añade ocho valores por pase con `paddd`, y el resto escalar corre en `0xc0`El CFG tiene dos bordes traseros.`0x80 -> 0x80`y `0xc0 -> 0xc0`, y la suite de pruebas comprueba ambos.

## Dominatores con el algoritmo iterativo

Un bloque .`d`domina un bloque `b`Cuando cada camino desde la entrada a `b`pasa por aquí.`d`El dominador inmediato de`b`es el dominador de `b`que es lo más cercano a `b`La noción se remonta a Prosser (1959) a Allen (1970), y es la estructura que la lección 34 necesita colocar.`if`y `while`- Sí, es cierto.

Cooper, Harvey y Kennedy escribieron "Un algoritmo de dominación simple y rápido" (2001). Argumentan que la formulación de flujo de datos, ejecutada en postorden inverso con una representación compacta, supera a Lengauer-Tarjan (1979) en gráficos bajo unos mil nodos. Su artículo dice que su propósito es "no presentar un nuevo algoritmo". Muestra que los algoritmos simples pueden ganar en la práctica y son buenas herramientas de enseñanza. Una función CFG casi nunca tiene mil nodos.

```python
def intersect(first: int, second: int) -> int:
    while first != second:
        while number[first] > number[second]:
            first = idom[first]
        while number[second] > number[first]:
            second = idom[second]
    return first

changed = True
while changed:
    changed = False
    for start in order[1:]:
        processed = [p for p in self.predecessors(start) if p in idom]
        candidate = processed[0]
        for other in processed[1:]:
            candidate = intersect(other, candidate)
        if idom.get(start) != candidate:
            idom[start] = candidate
            changed = True
```

`number`posee la posición posterior a la orden inversa de cada bloque, con la entrada en 0. `intersect`El periódico escribe la comparación con números de orden posterior y `<`Este código utiliza números de orden posterior inverso, por lo que la comparación es `>`Todo lo demás sigue el pseudocodo del periódico.

Para`total`En el`-O1`El resultado es `idom(0x44) = 0x40`¿ Qué ?`idom(0x50) = 0x44`¿ Qué ?`idom(0x5b) = 0x50`, y `idom(0x5c) = 0x40`El borde .`0x50 -> 0x50`es un borde trasero porque su objetivo domina su fuente, y un borde trasero marca un bucle (Allen 1970). `back_edges()`El examen lo devuelve.`test_dominators_on_a_hand_built_diamond_with_a_loop`ejecuta el mismo código en una secuencia de 16 bytes con un punto de unión, así que `intersect`ve dos predecesores.

## La tabla de salto en grado en -O0

En el`-O0`clang compila el `switch`En el`grade`a la expresión de la mesa de salto independiente de la posición:

```text
      5d: 83 e8 04                     	subl	$0x4, %eax
      60: 77 38                        	ja	0x9a <grade+0x6a>
      62: 48 8b 45 f0                  	movq	-0x10(%rbp), %rax
      66: 48 8d 0d 00 00 00 00         	leaq	(%rip), %rcx
		0000000000000069:  R_X86_64_PC32	.rodata-0x4
      6d: 48 63 04 81                  	movslq	(%rcx,%rax,4), %rax
      71: 48 01 c8                     	addq	%rcx, %rax
      74: ff e0                        	jmpq	*%rax
```

Leemos esto como cuatro hechos.`subl $0x4`seguido por `ja`es una verificación de límites: los índices superiores a 4 se van al caso predeterminado en `0x9a`, así que la tabla tiene cinco entradas.`leaq`con su reubicación pone la dirección de la tabla en `%rcx`, y la sección anterior mostró que la dirección es `.rodata+0x0`- El .`movslq`carga una entrada de 32 bits firmada escalada por 4.`addq`añade la base de la tabla, por lo que cada entrada es la distancia entre la tabla y su bloque de caja.

En el archivo objeto cada byte de la tabla es cero.`.rela.rodata`¿Qué es esto ?

```text
RELOCATION RECORDS FOR [.rodata]:
OFFSET           TYPE                     VALUE
0000000000000000 R_X86_64_PC32            .text+0x91
0000000000000004 R_X86_64_PC32            .text+0x8c
0000000000000008 R_X86_64_PC32            .text+0x87
000000000000000c R_X86_64_PC32            .text+0x82
0000000000000010 R_X86_64_PC32            .text+0x86
```

Aplicar`S + A - P`de entrada `k`en compensación `4k`El valor almacenado es `.text + A - (.rodata + 4k)`, y el código se agrega `.rodata + 0`al mismo, así que el objetivo es`.text + A - 4k`. Entrada 0 es `0x91 - 0 = 0x91`, la entrada 1 es `0x8c - 4 = 0x88`, y la entrada 2 es `0x87 - 8 = 0x7f`Las entradas 3 y 4 son `0x82 - 12 = 0x76`y `0x86 - 16 = 0x76`Los casos 6, 7, 8, 9 y 10 de la`score / 10`ir a los bloques que almacenan 1, 2, 3, 4 y 4. Que coincide con la línea fuente C por línea, incluyendo el caso compartido para 9 y 10.

`recover_table()`¿Qué hace esta aritmética?`jump_table_resolver()`coincide con el idioma hacia atrás de la `jmpq`- Encuentra el`addq`que hizo el registro, el `movslq`que lo cargó, el `leaq`que hizo la base, y el límite `ja`. La descenso luego empuja los cuatro objetivos distintos y continúa. Un ejecutable vinculado tiene valores finales en la tabla y no se reubican. La misma función luego lee las entradas firmadas de 32 bits y agrega la dirección de la tabla, y `evidence`registros de la ruta que produjo los objetivos.

Los números para `grade`En el`-O0`, desde `python3 code/main.py cfg code/fixtures/ledger-x86_64-O0.o grade`¿Qué es esto ?

| Method | Instructions | Blocks | Edges | Bytes | What is wrong |
|---|---|---|---|---|---|
| linear sweep | 34 | 9 | 7 | 118 code + 10 padding | 5 unreachable blocks, no edge into the four case blocks, padding decoded as `nopw` |
| recursive descent | 25 | 3 | 2 | 82 code | 36 code bytes never reached, `jmpq *%rax` at `0x74` unresolved |
| descent with the table | 33 | 8 | 11 | 118 code | nothing |

Con la mesa el árbol dominador está completo.`idom`de cada bloque de casos es `0x62`, el bloque con el salto.`idom`de `0xa1`es la entrada `0x30`, porque el camino predeterminado de `0x9a`alcanza `0xa1`Sin pasar .`0x62`¿ Qué ?

## Lo que hizo el optimizador para calificar en -O1

Ya verás .`grade`Los bytes dicen algo más específico.`-O1`y `-O2`No hay saltos indirectos .`grade`en absoluto:

```text
      2c: 83 f9 04                     	cmpl	$0x4, %ecx
      2f: 77 0c                        	ja	0x3d <grade+0x2d>
      31: 89 c8                        	movl	%ecx, %eax
      33: 48 8d 0d 00 00 00 00         	leaq	(%rip), %rcx
		0000000000000036:  R_X86_64_PC32	.rodata+0x10
      3a: 8b 04 81                     	movl	(%rcx,%rax,4), %eax
      3d: c3                           	retq
```

El complemento de reubicación `0x10`más el 4 habitual pone la mesa en `.rodata+0x14`La carga es una llanura .`movl`, y los bytes que hay`01 00 00 00 02 00 00 00 03 00 00 00 04 00 00 00 04 00 00 00`. El compilador sustituyó los cinco bloques de caja con una tabla de los cinco valores de retorno. Hay una reubicación, una tabla y una verificación de límites. El flujo de control es de tres bloques y tres bordes con una rama condicional. `find_value_tables()`se informa como un `value-table`con entradas `[1, 2, 3, 4, 4]`y `evidence: "bytes"`¿ Qué ?

`grade_label`En el`-O2`El proyecto de ley de la Unión Europea se ha desarrollado en el marco de la política de la Unión Europea.`labels`array de cinco fichas de cadena se convirtió en una tabla de cinco 32 bits de compensación en `.rodata+0x0`Cada entrada tiene una`R_X86_64_PC32`reubicación en contra de la sección de cadena, y la función agrega la dirección de la tabla a la entrada que carga. `recover_table()`lee esas cinco reubicaciones y devoluciones de compensación `0, 2, 4, 6, 8`en el`.rodata.str1.1`, que se mantienen .`F`¿ Qué ?`D`¿ Qué ?`C`¿ Qué ?`B`, y `A`. La misma aritmética que recupera una tabla de saltos recupera una tabla de datos relativa, porque ambas son `S + A - P`en un descenso conocido.

La lección de las dos construcciones es nombrar lo que muestran los bytes. Una tabla de saltos cambia el gráfico de flujo de control. Una tabla de valores cambia una carga. Llamando a ambos "el interruptor" oculta la diferencia que la lección 34 tiene que reconstruir.

## Lo que Rea y Ghidra te dan aquí

rea 6.5.0 con el proveedor de Ghidra 12.1.4 devuelve el texto de lista de una función de `procedure_assembly`como una cuerda.`inspect_native_instruction`devuelve los bytes, longitud, texto de decodificación, tokens de operand, referencias tipadas y flujo de una instrucción, con el retorno clasificado a partir de p-code. `xrefs`devuelve las direcciones de origen únicas ordenadas de Ghidra's `ReferenceManager.getReferencesTo`- ¿ Qué ?`analyze_function`lleva el CFG y `body`de la función. `procedure_info`da al organismo como rangos inclusivos y advierte que "las lagunas y las colas compartidas no deben tratarse como bytes propiedad".

Cada uno de estos hechos es el descenso recursivo de Ghidra más sus analizadores, serializados.`instruction_flow_evidence`¿ Qué ?`function_references`, y `jump_table_evidence`Cuando Ghidra no encuentra la tabla, el flujo es "desconocido", y los bloques de caso faltan a los llamadores, llamadas y el CFG.

Ahora tienes el trabajo.`python3 code/main.py cfg`En el caso de que el proveedor de datos de la misma fuente de datos haya obtenido una información de la misma fuente de datos, comparar los comienzos de bloque, el conteo de bordes y la tabla recuperada con lo que devuelve el proveedor.

Una nota sobre el decodificador como programa. Capstone 6.0.0-Alpha10 (2026-07-21) fija fuera de los límites lee y escribe bajo CVE-2026-55893 y CVE-2026-55894, y 5.0.9 (2026-05-28) fijas CVE trasladadas. Un decodificador lee bytes que no escribió. El límite de 15 bytes, el control de truncado en `_Cursor.byte()`, y la negativa a adivinar un código opcode desconocido protege`main.py`de entrada para la que no fue construida.

## Construye el mismo

Ejecutar la demostración:

```bash
cd phases/13-tools-and-protocols/33-disassembly-and-control-flow
python3 code/main.py
```

El resultado es un informe JSON. Para cada objeto fijo da el acuerdo del oráculo. Para cada función da instrucciones, bloque y recuento de borde bajo los tres métodos. También enumera bloques no alcanzables, saltos indirectos no resueltos, bordes posteriores, dominadores inmediatos y cualquier tabla. Terminará con la comparación de trampas.

Forma esperada:

```json
{
  "fixtures": {
    "ledger-x86_64-O0.o": {
      "oracle": "86/86 instructions agree with llvm-objdump",
      "functions": {
        "grade": {
          "range": "0x30-0xa6",
          "paddingBytes": 10,
          "linearSweep": {"instructions": 34, "blocks": 9, "edges": 7, "unreachableBlocks": ["0x76", "0x7f", "0x88", "0x91", "0xa6"], "...": "..."},
          "recursiveDescent": {"instructions": 25, "blocks": 3, "edges": 2, "unresolvedIndirectJumps": ["0x74"], "...": "..."},
          "recursiveDescentWithTables": {"instructions": 33, "blocks": 8, "edges": 11, "...": "..."},
          "tables": [{"kind": "jump-table", "table": ".rodata+0x0", "count": 5, "entries": ["0x91", "0x88", "0x7f", "0x76", "0x76"], "evidence": "relocation", "...": "..."}]
        }
      }
    }
  },
  "sweepTrap": {"...": "..."}
}
```

Leer .`code/main.py`en este orden:

1. `ElfObject`Leer los encabezados de la sección, los símbolos y `.rela.*`las entradas y `symbolize()`Nombrar una dirección en el camino `llvm-objdump`- Sí, lo hace.
2. `Decoder.decode()`Leer prefijos y REX, luego despachos en el mapa de un byte o dos byte de código optico.
3. `_State.modrm()`se lee ModRM, SIB y el desplazamiento, y se registra el desplazamiento de campo de compensación.
4. `resolve_reference()`se aplica `S + A - P`a la reubicación en un desplazamiento o en un campo inmediato.
5. `linear_sweep()`y `recursive_descent()`Los dos métodos, cada uno con su fallo registrado.
6. `recover_table()`y `jump_table_resolver()`leer una tabla a través de sus reubicaciones o sus bytes.
7. `ControlFlowGraph`construye líderes, bloques y bordes de tipografía, y `dominators()`es el algoritmo iterativo.
8. `analyze_function()`ejecuta todos los métodos en una función y `compare_with_oracle()`- ¿Qué? - Comprueba el decodificador.

Hay dos órdenes más.`python3 code/main.py disasm FILE.o`Imprima una lista con el objetivo de reubicación después de cada instrucción que tenga uno. `python3 code/main.py cfg FILE.o FUNCTION`Imprime el análisis de una función.

## Usalo

Utilice los tres métodos en este orden en un nuevo objeto.

1. En primer lugar, haz la comparación del oráculo.`llvm-objdump`en una instrucción simple tiene un error, y cada estructura posterior la hereda.
2. Ejecutar el descenso recursivo de cada símbolo de función y leer `unresolved`Cada entrada es un salto o llamada indirecta cuyos objetivos no tienes.
3. Para cada salto no resuelto, busca el idioma de la mesa y la reubicación que lo apunta. Si el idioma no coincide, lea los bytes antes del salto y escriba lo que vea. Esa nota es la primera desconocida en la lección 35.
4. Comparar el barrido y el bajado byte por byte. los bytes que sólo el barrido reclama son relleno o datos. los bytes que ninguna reclama son inaccesibles o pertenecen a una función que no ha encontrado.
5. Construye el CFG desde la descenda con tablas recuperadas y lea los bordes traseros.`while`o `for`¿ Qué ?

Mantenga los archivos de texto del oráculo con los objetos. Cuando se mueva a una nueva versión de compilador, volver a capturarlos, porque una lista cambiada es un fijo cambiado.

## Laboratorio interactivo

### Laboratorio A: decodificar en papel, luego comprobar

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/33-disassembly-and-control-flow/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
decode(bytes.fromhex("48 8d 0d 00 00 00 00"), 0).text
decode(bytes.fromhex("8d 44 08 fa"), 0).text
decode(bytes.fromhex("66 66 66 66 66 2e 0f 1f 84 00 00 00 00 00"), 0).length
```

Escriba lo que espera antes de cada llamada.`48`de la primera y ver el cambio de tamaño de la operanda.`decode(bytes.fromhex("0f 0a"), 0)`y leer el `DecodeError`¿ Qué ?

### Laboratorio B: la reubicación en una localidad

```python
elf = load_fixture("ledger-x86_64-O0.o")
code = elf.section(".text").data
lea = decode(code, 0x66)
lea.displacement_offset
resolve_reference(elf, lea)
```

La referencia dice:`.rodata+0x0`Cambiar la dirección a `0x116`En el`grade_label`y leer la referencia en `.data`Entonces decodifica el `callq`En el`0x10e`El campo inmediato lleva`R_X86_64_PLT32 grade-0x4`, y el objetivo resuelto es `0x30`, que es `grade`¿ Qué ?

### Laboratorio C: observe los dos métodos no coinciden

```python
trap = load_fixture("sweep-trap.o")
text = trap.section(".text").data
[i.text for i in linear_sweep(text, 0, 0x20).instructions]
[i.text for i in recursive_descent(text, [0], 0x20).sorted_instructions()]
```

Cuenta las instrucciones en cada lista y encuentra la que comienza en `0x13`Entonces cambia el último byte de la tabla en `sweep-trap.s`de la`0x48`¿ Qué ?`0xe8`El barrido ahora se traga el`addl`y el `retq`en un `callq`La función parece no tener retorno.

### Laboratorio D: dominadores a mano

```python
analysis = analyze_function(load_fixture("ledger-x86_64-O1.o"), "total")
analysis.cfg.edges
analysis.cfg.reverse_postorder()
analysis.cfg.dominators()
analysis.cfg.back_edges()
```

Dibujar los cinco bloques en papel y calcular los dominadores antes de mirar el resultado.`grade`En el`-O0`y explicar por qué.`idom(0xa1)`¿ Es verdad ?`0x30`Y no lo es .`0x62`¿ Qué ?

## Laboratorio de práctica

Añadir un sitio de llamada indirecta a los aparatos y ampliar el análisis.

Requisitos:

- Escriba una función C que llama a través de un puntero de función elegido de una pequeña matriz.`-O0`y `-O1`y comprometen los objetos con sus `llvm-objdump`¿Qué es eso?
- Haga que el decodificador coincida con el oráculo en ambos objetos.
- Extenderse`recursive_descent()`Así que una llamada indirecta registra el sitio de llamada en `calls`con`None`como el objetivo y continúa después de él.
- Escriba un resolver que lea la matriz de punteros de función a través de sus reubicaciones y informa las posibles llamadas.
- Añadir el conjunto de llamadas al informe de función como `indirectCalls`con el tipo de evidencia para cada objetivo.
- Escriba una prueba que falla cuando el resolver devuelve un objetivo que no es un símbolo de función.

El éxito significa`python3 code/main.py cfg`Nombran a todas las posibles llamadas con la reubicación que lo prueba, y la comparación del oráculo todavía pasa a cada objeto.

## Lección Artefacto

Esta lección incluye `outputs/skill-disassembly-and-control-flow.md`. Utilice para convertir un objeto y un nombre de función en una lista comprobada, bloques, bordes tipados, dominadores y todas las transferencias no resueltas. Requiere una comparación de oráculos, ambos métodos de desmontaje, contabilidad de byte, evidencia de reubicación para cada tabla recuperada y una declaración de lo que permaneció desconocido.

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/33-disassembly-and-control-flow
python3 code/main.py
python3 code/tests/test_main.py
```

La verificación deberá demostrar:

- el decodificador está de acuerdo con `llvm-objdump -d`en todas las instrucciones 237 de los cuatro objetos de fijación
- REX.W selecciona operandas de 64 bits y REX.R y REX.B amplían los campos del registro
- ModRM y SIB renderizan la operanda de memoria AT&T, incluida la escala, el índice y el desplazamiento firmado
- se registra un desplazamiento relativo al PIR con su desplazamiento de campo dentro de la instrucción
- repetidos prefijos heredados y un decodeo de segmento de anulación como el oráculo
- un obligatorio `66`prefijo selecciona el código op de SSE y deja los registros de 32 bits
- Las instantáneas de 8 bits son ampliadas por signo para aritmética y sin firmar para el conteo de turnos
- una instrucción de más de 15 bytes, una instrucción truncada y un aumento de código op desconocido `DecodeError`
- `R_X86_64_PC32`en un `lea`Resolve la cuestión y `R_X86_64_PLT32`en un `call`se resuelve a la llamada
- El barrido decodifica la tabla de trampas como código y no tiene instrucciones en el objetivo de salto
- El descenso nunca reclama la mesa o el relleno y lee `addl $0x1, %eax`en el objetivo
- `llvm-objdump`comete el mismo error que el barrido en la trampa
- una entrada dentro de una instrucción se registra como un conflicto
- un salto indirecto no resuelto detiene el descenso y se enumera
- `total`En el`-O1`tiene cinco bloques, cinco bordes tipados, un borde trasero, y los dominadores inmediatos esperados
- un diamante hecho a mano con un bucle da a los dominadores el algoritmo define
- `total`En el`-O2`tiene dos bucles
- `grade`En el`-O0`produce una mesa de salto de cinco entradas de `.rela.rodata`con objetivos `0x91, 0x88, 0x7f, 0x76, 0x76`
- Sin la mesa .`grade`En el`-O0`Tiene tres bloques y un salto sin resolver
- `grade`En el`-O1`y `-O2`es una tabla de valores `[1, 2, 3, 4, 4]`sin saltos indirectos
- `grade_label`En el`-O2`lee una tabla de cuerdas relativa a través de cinco reubicaciones
- La figura interactiva incorpora los mismos datos que el análisis produce

## Modos de falla de producción

| Failure | What the weak tool reports | What the analysis must prove |
|---|---|---|
| Data inside `.text` | A plausible listing with every byte decoded | The descent claims no data byte and the jump target starts an instruction |
| Padding after the last return | A `nopw` instruction at the end of the function | The bytes are unreachable from the entry |
| Indirect jump through a table | "jmp rax" and four dead blocks | Table address from the `lea` relocation, entry count from the bounds check, targets from `.rela.rodata` |
| Switch turned into a value table | "a jump table with a relocation" | A `movl` load from `.rodata+0x14`, three blocks, no indirect transfer |
| Call with a zero displacement | `callq 0x113`, the next instruction | `R_X86_64_PLT32 grade-0x4` resolves to `0x30` |
| Decoder guesses an unknown opcode | A mnemonic that looks right | `DecodeError` with the address, and the sweep records the byte |
| Instruction longer than 15 bytes | A very long "instruction" | The cursor raises before reading byte 16 |
| Branch into the middle of an instruction | Two overlapping listings | A conflict record with both addresses |
| Loop reported as straight-line code | No `while` in the pseudocode | A back edge whose target dominates its source |
| Oracle captured from a different compiler | Tests pass on stale text | The fixture README names the exact compiler and command |

## Regla de funcionamiento

Descifrar contra un oráculo, controlar el flujo de entrada de entradas conocidas y explicar cada byte como reclamado, relleno, datos o inaccesible. Recuperar cada objetivo indirecto de una reubicación o una tabla a la que puede apuntar.

## Leer más

- [Intel 64 and IA-32 Architectures Software Developer's Manual, Volume 2, Chapter 2: Instruction Format](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [AMD64 Architecture Programmer's Manual Volume 3, Chapter 1: Instruction Encoding](https://docs.amd.com/v/u/en-US/24594_3.38_APM_Vol3_PUB)
- [System V Application Binary Interface, AMD64 Architecture Processor Supplement, Table 4.9: Relocation Types](https://gitlab.com/x86-psABIs/x86-64-ABI)
- [Schwarz, Debray, Andrews, "Disassembly of Executable Code Revisited" (WCRE 2002)](https://www2.cs.arizona.edu/~debray/Publications/disasm.pdf)
- [Andriesse et al., "An In-Depth Analysis of Disassembly on Full-Scale x86/x64 Binaries" (USENIX Security 2016)](https://www.usenix.org/conference/usenixsecurity16/technical-sessions/presentation/andriesse)
- [Allen, "Control Flow Analysis" (1970)](https://doi.org/10.1145/800028.808479)
- [Cooper, Harvey, Kennedy, "A Simple, Fast Dominance Algorithm" (2001)](https://www.cs.tufts.edu/~nr/cs257/archive/keith-cooper/dom14.pdf)
- [Lengauer, Tarjan, "A fast algorithm for finding dominators in a flowgraph" (1979)](https://doi.org/10.1145/357062.357071)
- [Capstone 6.0.0-Alpha10 release notes](https://github.com/capstone-engine/capstone/releases/tag/6.0.0-Alpha10)
- rea 6.5.0 fuente en etiqueta `3b79af2`que esta lección describe: `README.md`¿ Qué ?`docs/ghidra-provider.md`¿ Qué ?`docs/ghidra-dos.md`¿ Qué ?`docs/reverse-engineering-tools.md`¿ Qué ?`bridge/ghidra/ReaGhidraBridge.java`¿ Qué ?`src/contracts/officialToolContracts.ts`¿ Qué ?`src/ghidra/GhidraAnalysisProfile.ts`¿ Qué ?
