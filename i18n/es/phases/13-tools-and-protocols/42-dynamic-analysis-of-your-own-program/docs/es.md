# Análisis dinámico de su propio programa: debuggers, rastros y confirmación de afirmaciones estáticas

> Una observación sobre una carrera de una construcción es un punto de ruptura. Esta lección construye el arnés que convierte esas observaciones en veredictos confirmados, refutados y no observados sobre las afirmaciones que las lecciones estáticas hacen.

**Type:** Build
**Languages:** Python, C, arm64 assembly, lldb command language
**Prerequisites:** Phase 13 · 33 (disassembly and control flow), Phase 13 · 35 (evidence, confidence, and unknowns)
**Time:** ~130 minutes

## Objetivos de aprendizaje

- Explicar lo que un depurador necesita del sistema operativo en Linux (`ptrace`) y en macOS (puertos de tareas de Mac y puertos de excepción), y por qué una cáscara de arena no puede lanzar un desinfectante.
- Establezca puntos de interrupción de software por nombre y por dirección, establezca un punto de vigilancia de hardware y lea los límites que el procesador impone a ambos.
- Emplear lldb en modo batch desde Python con un gancho de parada que registra registros, memoria y variables de marco en cada parada, y analizar la transcripción de nuevo en una línea de tiempo.
- Compruebe las afirmaciones estáticas de las lecciones 32, 33, 34 y 41 contra esa línea de tiempo: registros de argumentos, offsets de campo estructural, que borde una rama condicional tomó, y que llamadas ocurrieron.
- Ejecutar el mismo plan contra un`-O2`Construir, leer puntos de ruptura que resuelvan y nunca golpean, y decir lo que hace y no prueba.
- Guarde y cargue un archivo de núcleo, y nombre lo que un núcleo de estilo pila deja fuera.
- Mapa de los conjuntos de comandos gdb e lldb, y ubicar `strace`¿ Qué ?`ltrace`¿ Qué ?`bpftrace`¿ Qué ?`dtrace`, y `rr`en el sistema operativo correcto con sus versiones y licencias.
- Encuentra los hechos de firma de código de macOS que permiten a lldb deshacer un binario que construyó usted mismo, y las dos condiciones bajo las cuales se niega.

## El problema

Lección 41 Leer `struct account`de DWARF y dijo `balance`Se encuentra en el desplazamiento 4. la lección 33 construyó el gráfico de flujo de control de`classify`y dijo que la primera rama va a la`return -1`La lección 32 dice que los dos primeros argumentos llegan a la`x0`y `x1`Cada una de esas frases se deriva de bytes. Ninguno de ellos se observó.

Un agente entonces lee el mismo programa y informa "llamadas principales"`bump`¿ Qué ?`classify`, y `sum_balances`La tabla de símbolos enumera los tres, por lo que la oración parece segura.`-O2`El proceso imprime sus cuatro líneas y salidas.`hit count = 0`¿ Qué ?

El agente no estaba equivocado sobre la fuente, estaba equivocado sobre la construcción de la que se le preguntó.`sum_balances(rows, 3)`La única manera de saber que para este binario en esta máquina es ejecutarlo bajo un depurador y mirar.

Esta lección construye un arnés que hace la mirada.`account.c`, lee la lista, planifica una sesión de Ildb, la ejecuta y califica cada reclamación estática en función de lo que muestran las paradas. La calificación tiene tres resultados. `confirmed`significa una parada que muestra que la afirmación es cierta para esta carrera. `refuted`significa que una parada demostró que era falsa.`not-observed`La lección 35 almacena cada veredicto como un`observed`registro con autoridad `controlled-replay`¿ Qué ?

```figure
re-42-run-timeline
```

## Un depurador es tres servicios de sistema y una mesa

Un desinfectador hace cuatro cosas a un proceso del que no posee el código: detiene el proceso, lee y escribe sus registros, lee y escribe su memoria y se le dice cuándo se detiene el proceso de nuevo. El kernel proporciona cada uno de estos como un servicio, y los servicios difieren según el sistema operativo.

En Linux el servicio es `ptrace(2)`. La página manual dice que "proporciona un medio por el cual un proceso (el rastreador) puede observar y controlar la ejecución de otro proceso". `PTRACE_TRACEME`deja que un niño anuncie que su padre lo rastreará,`PTRACE_ATTACH`toma un proceso no relacionado, `PTRACE_PEEKDATA`"una palabra en la dirección de la memoria del rastreador",`PTRACE_GETREGS`lee los registros y `PTRACE_SINGLESTEP`ejecuta una instrucción. Los límites del módulo Yama adjuntan: `/proc/sys/kernel/yama/ptrace_scope`en su defecto de 1 significa "el proceso objetivo debe ser un descendiente del llamador", 2 significa que sólo un administrador puede adjuntar, y 3 significa que nadie puede.

En macOS `ptrace`Es una interfaz más pequeña.`PT_TRACE_ME`¿ Qué ?`PT_ATTACHEXC`¿ Qué ?`PT_STEP`¿ Qué ?`PT_CONTINUE`, y `PT_KILL`, y dice que con `PT_ATTACHEXC`"Las señales del niño van al padre como excepciones de Mach".`ptrace`En absoluto. El Ildb's `debugserver`llamadas`task_for_pid()`para obtener un puerto de tarea Mach para el proceso, entonces `task_set_exception_ports()`Así que los puntos de ruptura llegan como mensajes de excepción.`taskgated`Decide quién puede llamar .`task_for_pid`¿ Qué ?

El costo de ese diseño se puede ver en una transcripción que esta lección mantiene. La carcasa de la sesión se ejecuta bajo una caja de arena de la cintura de seguridad. El mismo plan que produce once paradas fuera de la caja de arena produce esto dentro de ella:

```text
(lldb) run
error: process exited with status -1 (attach failed ((os/kern) invalid address))
```

El proceso fue creado y luego no pudo ser adoptado. El error es sobre los derechos del desarreglador, y no dice nada sobre el programa. El arnés lo registra como un fracaso de lanzamiento, con cero paradas y sin veredictos.

La tabla es la tabla de puntos de ruptura. El descomposición recuerda qué byte de instrucción reemplazó en qué dirección, por lo que puede poner el original de nuevo cuando continúa, y cuenta los hits por ubicación. `breakpoint list`Imprenta esa mesa, y el arnés lo lee al final de cada carrera.

## Puntos de ruptura: un byte o una palabra sustituida

Un punto de ruptura de software es una instrucción que el procesador atrapa, escrito sobre los primeros bytes de la instrucción que desea detenerse en.`INT3`, código op `CC`, de un byte de largo, que el manual de Intel describe como "intencionado para llamar al manipulador de excepciones de desacceso". En arm64 es `BRK #imm16`, una instrucción de cuatro bytes cuya excepción "registra la excepción en ESR_ELx, utilizando el valor EC 0x3C".`BRK`La instrucción misma, por lo que una parada informa la dirección de la instrucción sustituida y no la siguiente.

Lldb establece puntos de interrupción por nombre, por archivo y línea, o por dirección.

```text
(lldb) breakpoint set -n bump
Breakpoint 1: where = account-O0`bump + 12 at account.c:13:19, address = 0x00000001000004bc
```

`bump + 12`Es la cuarta instrucción.`sub sp, sp, #0x10`¿ Qué ?`str x0, [sp, #0x8]`, y `str w1, [sp, #0x4]`, la configuración de marco que la tabla de líneas DWARF atribuye a la línea de apertura de la función.`x0`y `x1`Aún mantienen los argumentos, porque el prólogo sólo los copió.

Por dirección, el Ildb necesita saber a qué módulo pertenece la dirección.`breakpoint set -a 0x100000504`antes de que el proceso exista es una dirección de carga sin sección detrás de ella, y en una primera prueba se quedó `unresolved`durante toda la carrera con `hit count = 0`El tipo de forma que utiliza el arnés se denomina módulo:

```text
(lldb) breakpoint set -a 0x100000504 -s account-O0
Breakpoint 4: address = 0x0000000100000504
```

Con `-s`El número es una dirección de archivo dentro `account-O0`, lldb agrega la diapositiva de carga, y después de la carrera se lee la ubicación `resolved, hit count = 2`. La lista de la lección 33 da direcciones de archivos, así que este es el formulario que conecta una lista estática a una parada dinámica.

### Descifrar la rama a mano

La primera rama condicional en `classify`En el`-O0`Es la palabra.`36f800a8`En el`0x1000004f0`, que `llvm-objdump`Imágenes como `tbz w8, #0x1f, 0x100000504`. Descifrarlo de la codificación de Arm A64 de `TBZ`¿ Qué ?

1. El 31 es`b5`, el bit alto del número de bits. Es 0.
2. Los 30 a 25 bits son `011011`, el grupo de prueba y rama. bit 24 es 0, que selecciona `TBZ`Y no lo es .`TBNZ`¿ Qué ?
3. Los bits 23 a 19 son `b40`, los cinco bits más bajos del número de bits: `11111`= 31. Con `b5`= 0 el bit probado es 31, el bit de signo de un valor de 32 bits.
4. Los bits de 18 a 5 son `imm14`¿ Qué es esto ?`00000000000101`= 5. La compensación de ramas es `imm14 * 4`- 20 - 20 - 20`0x14`, así que el objetivo es`0x1000004f0 + 0x14 = 0x100000504`¿ Qué ?
5. Los bits 4 a 0 son `Rt`¿ Qué es esto ?`01000`= 8, así que el registro es `w8`¿ Qué ?

`w8`se cargó una instrucción antes por `ldr w8, [x8, #0x4]`, que es `a->balance`en el despacho 4. `TBZ`Las ramas cuando el bit es cero. La rama se toma cuando el bit de signo está claro, que es `balance >= 0`La fuente escribió:`if (a->balance < 0) return -1;`, por lo que el compilador invirtió la condición: el borde tomado saltar el `return -1`El bloque, y el caída a través de `0x1000004f4`La afirmación estática que lleva el arnés es, por lo tanto, una afirmación de borde con un predicado:`a->balance >= 0`El ascensor de la segunda rama funciona de la misma manera.`subs w8, w8, #0xc8`computaciones`balance - 200`y `b.le 0x100000524`se toma cuando eso es al máximo cero, que es `balance <= 200`¿ Qué ?

## Los puntos de vigilancia son hardware y pocos

Un punto de vigilancia detiene el proceso cuando se accede a la memoria en una dirección. No puede ser una instrucción reemplazada, porque cualquier instrucción en cualquier lugar podría hacer el acceso. Los procesadores por lo tanto llevan un pequeño conjunto de comparadores de direcciones. En x86-64 son los registros de descomposición:`DR0`por el`DR3`Tener "las direcciones de hasta 4 puntos de intervalo", y `DR7`El sistema de control de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de datos de los pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares de pares`DBGWVR<n>_EL1`, que "contiene la dirección virtual de datos utilizada para la coincidencia de puntos de vigilancia", y `DBGWCR<n>_EL1`para los controles, y el manual de Arm dice "Una implementación puede incluir entre 2-64 puntos de vigilancia".

El Ildb informa el número que se le ha dado:

```text
(lldb) watchpoint set variable a->balance
Watchpoint created: Watchpoint 1: addr = 0x16fdfe00c size = 4 state = enabled type = m
(lldb) watchpoint list
Number of supported hardware watchpoints: 4
```

Cuatro en esta máquina.`0x16fdfe00c`¿ Es verdad ?`x0 + 4`: el indicador estructural en el `bump`parada más el desplazamiento DWARF de `balance`- ¿ Qué ?`size = 4`¿ Es el `int`- ¿ Qué ?`type = m`Cuando las ranuras se agotan, el LLDB rechaza el siguiente punto de vigilancia, y debido a que una región que no está alineada o no tiene una potencia de dos puede tomar dos ranuras, el número de puntos de vigilancia que puede establecer puede ser menor que cuatro.

Los puntos de vigilancia de datos informan después del acceso. El manual de Intel dice que "una instrucción que escribe memoria sobrescribe los datos originales antes de que se genere la excepción de defecto generada por un punto de ruptura de datos", y lldb guarda el valor antiguo en su propia caché para imprimir ambos:

```text
Watchpoint 1 hit:
old value: 250
new value: 255
* thread #1, queue = 'com.apple.main-thread', stop reason = watchpoint 1
    frame #0: 0x00000001000004d0 account-O0`bump(a=0x000000016fdfe008, by=5) at account.c:14:12
```

La parada está en`0x1000004d0`, la instrucción después de `str w8, [x9, #0x4]`En el`0x1000004cc`Una peculiaridad vale la pena conocer antes de analizar la salida.`watchpoint set`, huellas de Ildb `Watchpoint 1 hit:`con un valor antiguo vacío y `new value: 250`Esa es la lectura inicial, y el arnés ignora un golpe sin un valor antiguo.

## El paso, el detener y el modo de batch

La gramática de comando de Ildb es `<noun> <verb> [-options] [arguments]`Los verbos de paso son`thread step-in`(El artículo`s`), `thread step-over`(El artículo`n`), `thread step-inst`(El artículo`si`), `thread step-inst-over`(El artículo`ni`), y `thread step-out`(El artículo`finish`¿ Qué es esto ?`thread until 12`corre a una línea.`thread return <expr>`El arnés no camina, se detiene en las direcciones previstas y deja que un gancho de parada haga la lectura:

```text
target stop-hook add -o "register read x0 x1 pc" -o "memory read -s4 -fx -c24 $x0" -o "frame variable -P 1"
```

`-o`Se pueden repetir, y las órdenes se ejecutan en orden en cada parada.`register read`Nombrar los dos registros de argumentos y el contador de programas. `memory read -s4 -fx -c24 $x0`Imprima 24 palabras de cuatro bytes en hex desde la dirección en `x0`, que es 96 bytes, tres `struct account`las filas. `frame variable -P 1`Imprime los argumentos y los locales y expande los punteros a un nivel, así que `a`muestra sus cinco campos.

El modo de lote es `lldb -b`El manual dice:`--batch`ejecuta los comandos desde `-s`¿ Qué ?`-S`¿ Qué ?`-o`, y `-O`y luego se detiene, y que si un comando se detiene "debido a una señal o accidente, el descomposición volverá a la llamada interactiva". El arnés escribe el plan a un archivo y lo pasa con `-s`. Una situación importante: `settings set interpreter.stop-command-source-on-error false`Sin ella, el primero.`continue`Después de que el proceso haya salido, termina todo el archivo, y `breakpoint list`Nunca se ejecuta.`error: Process must be launched.`y continúa.

Dos hechos sobre la forma de salida deciden cómo funciona el parser.`Process N stopped`bloque con su `frame #0`El parser recopila registros, memoria y líneas variables en un pedazo pendiente y une el pedazo al siguiente.`frame #0`Una pieza cuyo contador de programa está en`dyld`En el`_dyld_start`es la parada de entrada antes de que el programa se ejecute, y no `frame #0`En segundo lugar, un punto de ruptura con`--auto-continue`El sistema de control de los puntos de intervalo de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de velocidad de los sistemas de control de velocidad de los sistemas de control de velocidad de velocidad de velocidad de los sistemas de control de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad de velocidad`continue`en su lugar.

## El plan viene de la lista

`Plan.commands()`construye el archivo de comandos desde el `llvm-objdump -d`El texto del binario se ejecutará.`account-O0`¿Qué es esto ?

```text
breakpoint set -n bump
breakpoint set -n classify
breakpoint set -n sum_balances
breakpoint set -a 0x100000504 -s account-O0
breakpoint set -a 0x1000004f4 -s account-O0
breakpoint set -a 0x100000524 -s account-O0
breakpoint set -a 0x100000514 -s account-O0
target stop-hook add -o "register read x0 x1 pc" -o "memory read -s4 -fx -c24 $x0" -o "frame variable -P 1"
run
watchpoint set variable a->balance
watchpoint list
continue
...
breakpoint list
watchpoint list -v
```

Los tres puntos de ruptura de nombre son las entradas de la función.`Listing.conditional_branches()`encuentra en `classify`¿ Qué es esto ?`tbz`En el`0x1000004f0`y `b.le`En el`0x100000510`El punto de vigilancia se fija en la primera parada, que el plan espera que sea `bump`- Dieciséis .`continue`El presupuesto no es el conocimiento del programa. Es un techo.

Las reclamaciones se construyen a partir de los mismos insumos por `claims_for()`¿Qué es esto ?

| Id | Kind | Claim | Source |
|---|---|---|---|
| `arg-registers` | argument-register | at every entry the first arguments arrive in `x0` and `x1` | lesson 32 and AAPCS64 |
| `field-offsets` | field-offset | `struct account` is 32 bytes with `kind` at 0, `balance` at 4, `code` at 8, `rate` at 16, `label` at 24 | lesson 41 and the DWARF dump |
| `branch-1` | branch-taken | `tbz` at `0x1000004f0` goes to `0x100000504` when `a->balance >= 0`, otherwise to `0x1000004f4` | lesson 33 edges and the lesson 34 lift |
| `branch-2` | branch-taken | `b.le` at `0x100000510` goes to `0x100000524` when `a->balance <= 200`, otherwise to `0x100000514` | lesson 33 edges and the lesson 34 lift |
| `calls` | call-edge | `main` calls `bump` once, `classify` three times, `sum_balances` once | source text and `nm` |
| `store-once` | store | `bump` writes `a->balance` once and the new value is the old value plus `by` | lesson 34 pseudocode |
| `stride` | struct-stride | `list[i].balance` sits at `list + 32*i + 4` | lesson 41 and the psABI aggregate rule |
| `output` | behaviour | the program prints `class 0 1 -1` and `sum 315` | source arithmetic |

Los offsets de campo provienen de`struct_layout()`, que dice `DW_TAG_structure_type`con`DW_AT_byte_size (0x20)`y los cinco.`DW_TAG_member`las entradas con su `DW_AT_data_member_location`El plan nunca escribe el número 4.

## La carrera en -0: once paradas

La primera parada es la`bump`El gancho imprimió esto antes del bloqueo de parada:

```text
      x0 = 0x000000016fdfe008
      x1 = 0x0000000000000005
      pc = 0x00000001000004bc  account-O0`bump + 12 at account.c:13:19
0x16fdfe008: 0x00000073 0x000000fa 0x00000009 0x00000000
0x16fdfe018: 0x00000000 0x3ff40000 0x00000719 0x00000001
0x16fdfe028: 0x0000006c 0xffffffd8 0x00000003 0x00000000
0x16fdfe038: 0x00000000 0x40000000 0x00000721 0x00000001
(account *) a = 0x000000016fdfe008 {
  kind = 's'
  balance = 250
  code = 9
  rate = 1.25
  label = 0x0000000100000719 "savings"
}
(int) by = 5
```

Lea las ocho palabras de la primera fila contra las offsets DWARF. La palabra 0 en offset 0 es `0x73`, que es `'s'`, el `kind`la palabra 1 en el offset 4 es `0xfa`= 250, `balance`La palabra 2 en el descalzo 8 es `0x9`¿ Qué ?`code`, una `short`En la mitad baja con dos bytes de relleno por encima de ella.`double`Las palabras 4 y 5 en el offset 16 son `0x3ff4000000000000`, que es la codificación IEEE-754 de 1.25. las palabras 6 y 7 en el offset 24 son `0x100000719`, el `label`Indicador en `__TEXT`Las siguientes ocho palabras comienzan con`0x6c`¿ Qué es esto ?`'l'`y `0xffffffd8`= -40: la tercera fila, `loan`, 32 bytes más tarde. Cada número está de acuerdo con`frame variable`, así que`_check_layout()`registros `field-offsets`¿ Cómo ?`confirmed`en esta parada, y lo hace de nuevo en cada una de las once paradas donde`x0`tiene un indicador estructural.

`x0`igual `a`y `x1`igual `by`, así que`arg-registers`El arnés sólo compara los argumentos declarados.`classify`Se detiene`x1`Aún lee .`0x5`, un resto de `bump`, y `classify`El registro en el que no hay ningún argumento es ruido, y la alegación de la lección 32 no dice nada al respecto.

La segunda parada es el punto de vigilancia, el antiguo valor 250, el nuevo valor 255, `by`= 5. Un golpe en toda la carrera, así que `store-once`Luego vienen tres activaciones de`classify`, cada uno con su parada de entrada y una o dos paradas de borde:

| Stop | Address | Stop reason | What it tests |
|---|---|---|---|
| 2 | `0x1000004e8` | breakpoint 2.1 | entry, `a->balance` = 100, `x0` = `a` |
| 3 | `0x100000504` | breakpoint 4.1 | `tbz` taken, 100 >= 0, `branch-1` confirmed |
| 4 | `0x100000524` | breakpoint 6.1 | `b.le` taken, 100 <= 200, `branch-2` confirmed |
| 5 | `0x1000004e8` | breakpoint 2.1 | entry, `a->balance` = 255 |
| 6 | `0x100000504` | breakpoint 4.1 | `tbz` taken, 255 >= 0 |
| 7 | `0x100000514` | breakpoint 7.1 | `b.le` fell through, 255 > 200 |
| 8 | `0x1000004e8` | breakpoint 2.1 | entry, `a->balance` = -40 |
| 9 | `0x1000004f4` | breakpoint 5.1 | `tbz` fell through, -40 < 0 |
| 10 | `0x100000544` | breakpoint 3.1 | `sum_balances` entry, `x1` = 3 |

`_check_branches()`Los grupos de paradas en activaciones, lee el equilibrio en la entrada, y para cada parada de borde calcula el sucesor de la predicada levantada predice.`branch-1`Confirmado tres veces.`branch-2`El punto de control de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la velocidad de la

En la parada 10 el gancho lee 24 palabras de `list`- ¿ Qué ?`_check_stride()`toma palabra 1, palabra 9 y palabra 17, que son compensaciones 4, 36, y 68: 100, 255, y -40.`classify`Se puede ver en la misma dirección, sumarlas a 315, y comparar la suma con la `sum 315`en línea el programa impreso. Luego el proceso sale con estado 0 y `breakpoint list`muestra recuentos de hits de 1, 3, 1 para los nombres y 2, 1, 1, 1 para los cuatro bordes.`confirmed`, con 25 récords detrás, 23 de ellos atados a una parada y dos a la carrera terminada.

## La carrera en -O2: cero paradas

La misma fuente en `-O2`Es un programa diferente.`llvm-objdump -d`de `main`tiene cuatro .`bl`instrucciones, todo para`printf`En lugar de las llamadas que tiene .`mov w8, #-0x1`¿ Qué ?`mov w8, #0x1`, un cero a través de `xzr`, y `mov w8, #0x13b`, que es 315. El compilador calcula todos los resultados en el momento de compilar. `nm`listas todavía `_bump`¿ Qué ?`_classify`, y `_sum_balances`, porque son funciones externas y las copias fuera de línea deben existir para que otros objetos puedan llamar.`DW_TAG_subprogram`entradas para las cuatro funciones y cero `DW_TAG_inlined_subroutine`las entradas, así que incluso la información de defecto no registra la línea.

El plan para esta construcción tiene tres puntos de ruptura de nombre y ningún punto de ruptura de borde. `classify`En el`-O2`es seis instrucciones,`ldr`¿ Qué ?`cmp`¿ Qué ?`cset`¿ Qué ?`cmp`¿ Qué ?`csinv`¿ Qué ?`ret`El borde reclama de la`-O0`Las listas no tienen dirección en este binario, y el arnés no las lleva. Una reclamación sobre un borde pertenece a la construcción cuya lista tiene el borde.

La carrera:

```text
(lldb) breakpoint set -n bump
Breakpoint 1: where = account-O2`bump at account.c:13:16, address = 0x0000000100000460
(lldb) run
sizeof 32 align 8
offsets 0 4 8 16 24
class 0 1 -1
sum 315
Process 48444 exited with status = 0 (0x00000000)
(lldb) watchpoint set variable a->balance
error: Command requires a process which is currently stopped.
(lldb) breakpoint list
1: name = 'bump', locations = 1, resolved = 1, hit count = 0
```

Cada punto de ruptura resuelto. La dirección existe, el símbolo existe, y el Ildb escribió un`BRK`El número de muertos es cero porque ninguna información llegó a la dirección.

| Claim | Verdict | Record |
|---|---|---|
| `calls` | refuted | after a full run with exit status 0, entry stops bump 0, classify 0, sum_balances 0 |
| `output` | confirmed | the four lines and exit status 0 |
| `arg-registers` | not-observed | no stop tested it, hit counts all 0 |
| `field-offsets` | not-observed | no stop tested it |
| `store-once` | not-observed | no stop in bump, so the watchpoint was never set |
| `stride` | not-observed | no stop in sum_balances |

Sé exacto sobre lo que dice la refutación.`main`llama a las tres funciones en este programa. Para esta construcción y esta carrera no lo hace, y el arnés comprobó que la carrera se completó, porque un recuento de golpes de cero antes de la salida significaría sólo que la carrera no había alcanzado la llamada todavía. La afirmación no dice que las funciones sean código muerto, y la carrera tampoco dice que: se exportan y otro objeto podría llamarlos. Las otras cuatro alegaciones no se confirman ni se refutan. El estruto todavía tiene`balance`en el despacho 4 en esta construcción, y el `-O2`DWARF dice que sí, pero no hay parada en esta carrera de lectura de memoria para mostrarlo. `not-observed`mantiene esa distinción, y la lección 35 mantiene abierto lo desconocido hasta que una carrera lo observa.

## Los residuos de la base

Un archivo central es la memoria del proceso y los registros escritos en disco, para que el desarreglador pueda leer un programa detenido después de que el programa se haya ido.`core(5)`dice "Un pequeño número de señales que causan una terminación anormal de un proceso también hacen que un registro del estado en el núcleo del proceso sea escrito en disco", en`/cores/core.pid`"siempre y cuando el proceso terminado tuviera permiso de escritura en el directorio, y el directorio existiera".`sysctl`informes `kern.coredump: 1`y `kern.corefile: /cores/core.%P`¿ Qué ?`/cores`existe con el modo `drwxrwxr-t root admin`, y el caparazón `ulimit -c`No se escribe ningún núcleo del núcleo hasta que se eleva el límite con `ulimit -c unlimited`, o en todo el sistema con `launchctl limit core unlimited`¿ Qué ?

El Ildb escribe un núcleo bajo demanda sin accidente.`bump`punto de ruptura:

```text
(lldb) process save-core --style stack account-O0-stack.core
Saving 49152 bytes of data for memory region at 0x16fdf4000

Modified-memory or stack-memory only corefile created.  This corefile may 
not show library/framework/app binaries on a different system, or when 
those binaries have been updated/modified. Copies are not included
in this corefile.  Use --style full to include all process memory.
```

El archivo fue de 53.248 bytes.

```text
(lldb) target create "./account-O0" --core "account-O0-stack.core"
(lldb) bt
* thread #1, stop reason = ESR_EC_BRK_AARCH64 (fault address: 0x0)
  * frame #0: 0x00000001000004bc account-O0`bump(a=0x000000016fdfe008, by=5) at account.c:13:19
    frame #1: 0x0000000100000640 account-O0`main at account.c:40:5
    frame #2: 0x000000019a949d54 dyld`start + 7184
(lldb) frame variable *a
(account) *a = (kind = 's', balance = 250, code = 9, rate = 1.25, label = "")
(lldb) frame variable a->label
(const char *) a->label = 0x0000000100000719 ""
```

La pista y los campos enteros están intactos, porque viven en la pila que el núcleo salvó. `label`Impresiones como una cadena vacía con su puntero intacto.`0x100000719`es correcto; los bytes a los que apunta son los literales `"savings"`en el `__TEXT`El ejecutable es un segmento de la cadena, y un núcleo de estilo pila no incluye ese segmento. el puntero es una observación. la cadena no está disponible, y lldb no lo invente. `--style full`incluye toda la memoria de proceso, y un núcleo completo de este programa de 34 KB en la máquina de investigación fue de 2.1 GB, porque el proceso mapea la caché compartida.`ESR_EC_BRK_AARCH64`: el núcleo registró el `BRK`excepción que el punto de ruptura planteó.

En Linux el núcleo escribe el núcleo de acuerdo con `/proc/sys/kernel/core_pattern`, una plantilla con `%p`para el PID, `%e`para el nombre del comando, `%s`para la señal, y `%t`por el tiempo, y un líder `|`las distribuciones del sistema utilizan esa tubería para`systemd-coredump`, y `coredumpctl list`¿ Qué ?`info`¿ Qué ?`dump`, y `debug`Recuperar lo que guardó.`gdb prog core`y `lldb -c core prog`ambos abren el resultado.

## Gdb y Ildb lado a lado

Los dos debuggers hacen el mismo trabajo con palabras diferentes. `COMMAND_MAP`en el arnés se encuentran 35 filas, impresas por `python3 code/main.py commands`- Las que utiliza esta lección:

| Task | gdb | lldb | lldb alias |
|---|---|---|---|
| break at a function | `break bump` | `breakpoint set -n bump` | `b bump` |
| break at an address in one module | `break *0x1000004f0` | `breakpoint set -a 0x1000004f0 -s account-O0` | |
| run commands at every stop | `define hook-stop` ... `end` | `target stop-hook add -o '...'` | |
| list breakpoints with hit counts | `info breakpoints` | `breakpoint list` | `br l` |
| watch a variable for writes | `watch a->balance` | `watchpoint set variable a->balance` | `wa s v a->balance` |
| list watchpoints | `info watchpoints` | `watchpoint list -v` | `wa l -v` |
| read one register | `p $x0` | `register read x0` | |
| read memory as words | `x/8xw 0x16fdfe008` | `memory read -s4 -fx -c8 0x16fdfe008` | `x -s4 -fx -c8 0x16fdfe008` |
| show arguments | `info args` | `frame variable --no-locals` | |
| step one instruction | `stepi` | `thread step-inst` | `si` |
| run until the function returns | `finish` | `thread step-out` | `finish` |
| write a core file | `gcore core` | `process save-core --style stack core` | |
| open a core file | `gdb account-O0 core` | `lldb -c core account-O0` | |

El de la Ildb `p`es un alias de `dwim-print`, que imprime una variable simple a través de la`frame variable`camino y cae de nuevo a `expression`para cualquier otra cosa.`v`¿ Es verdad ?`frame variable`- Prefiero .`frame variable`en un arnés: lee ubicaciones DWARF y nunca ejecuta código en el defecto.

Las versiones y licencias en 2026-10-14. lldb lleva el número de versión LLVM, y la versión actual de llvm.org es LLVM 23.1.3 de 2026-10-06, bajo la Licencia Apache 2.0 con excepciones LLVM. Homebrew lo envía como la fórmula separada `lldb`Las herramientas de la Línea de Comando de esta nave máquina .`lldb-1703.0.236.21`, la propia numeración de Apple, con un Python 3.9.6 incorporado, y el lldb de Apple ya está firmado para deshacerse. GDB 18.1 fue lanzado el 2026-09-25 bajo GPL-3.0 o posterior.`.gdb_index`, AArch64 `FPMR`, y Python `gdb.Corefile`GDB no tiene soporte nativo para los procesos de silicio de Apple hasta 18.1: una serie de 15 parches "Suporte inicial AArch64-Darwin" fue publicada en octubre de 2025, y en febrero de 2026 un mantenedor estableció la ruta para aterrizarlo a través de la asignación de derechos de autor y los parches BFD primero.`gdb`La fórmula tiene botellas arm64 y se construye con un falso `--target=x86_64-apple-darwin20`, y la solicitud de extracción que las añadió dice "no es posible debug aarch64-darwin binarios a través de GDB". Cualquier debugger no Apple en macOS también necesita un certificado autofirmado y el `com.apple.security.cs.debugger`derecho, o `task_for_pid`se falla con "(os/kern) falla (0x5)". el lugar de gdb en este curso es Linux, donde Debian estable navega gdb 16.3 para arm64, y donde `rr`¿Qué es eso?

## Trazación: Lo que el programa pidió al núcleo

Un debuger muestra el estado del programa en una parada. Un rastreador muestra los efectos del programa en el sistema sin detenerlo por mucho tiempo. Las herramientas son por sistema operativo.

| Tool | OS | Version | Licence | What it records |
|---|---|---|---|---|
| `strace` | Linux | 7.2, 2026-08-18 | LGPL-2.1-or-later | system calls with arguments and return values |
| `ltrace` | Linux | 0.7.3 last tarball, git snapshots in Debian | GPL-2.0 | dynamic library calls |
| `bpftrace` | Linux | v0.27.0, 2026-09-10 | Apache-2.0 | kernel and user probes through eBPF |
| `dtrace` and `dtruss` | macOS | `Sun D 1.19` in macOS 26.2 | CDDL, from Sun's OpenSolaris | D probes; `dtruss` prints system calls |

`dtruss`dice en su manual que "dado que esto utiliza DTrace, sólo los usuarios con privilegios raíz pueden ejecutar este comando".`dtrace(1)`Agrega que "Con la configuración predeterminada de Protección de Integridad del Sistema, los programas D no están permitidos para acceder a los valores de direcciones del núcleo o el contenido de la memoria del núcleo" y que las acciones destructivas no están permitidas bajo SIP. `sudo dtruss ./account-O0`En esta máquina no se ejecutó en esta lección, porque el arnés funciona sin raíz, y la lección registra que como no verificado.`write`por cada uno `printf`línea" es una pregunta de rastreo, "ha hecho`x1`Mantenga 5 en `bump`" es una pregunta de depuración.

## Grabación y reproducción

`rr`registra una carrera una vez y la reproduce deterministicamente bajo gdb, con ejecución inversa: fija un punto de vigilancia en el valor que le importa y `reverse-cont`se vuelve hacia atrás a la escritura. El proyecto se describe a sí mismo como "Registras un fracaso una vez, luego descompásas la grabación, deterministicamente, tantas veces como quieras". La versión actual es 5.9.0 de 2025-02-13, bajo la licencia MIT con un bloque BSD-2-Clause. Es sólo Linux. Necesita contadores de rendimiento de hardware para contar ramas, por lo que se ejecuta en Intel Nehalem y más tarde, ciertos núcleos AMD Zen y "ciertas microarquitecturas AArch64 (por ejemplo. ARM Neoverse N1 o la serie M de Apple Silicon) " bajo Linux. Dentro de una máquina virtual los contadores deben ser virtualizados, y el wiki del proyecto dice "Ninguna otra virtualización en Mac tampoco lo hace", por lo que Docker y Colima en esta máquina no pueden ejecutarlo. Pernosco es el servicio comercial basado en grabaciones de rr.

Gdb tiene su propio grabador de software. `record full`registros "un registro de la ejecución del proceso" con un límite predeterminado de 200.000 instrucciones, y `record btrace`En esta máquina macOS, la transcripción guardada del arnés es el registro: el mismo plan en el mismo binario produce las mismas paradas en el mismo orden, y el dispositivo prueba el orden enpinado.

## Lo que prueba una carrera

Una ejecución prueba una entrada en una construcción. La norma C define el comportamiento indefinido como "comportamiento, en el uso de una construcción de programa no portátil o errónea o de datos erróneos, para los cuales este documento no impone requisitos", por lo que un programa que sobrepasa un programa firmado `int`puede imprimir legítimamente diferentes resultados en `-O0`y `-O2`La lección 36 se encontró con esto .`percent_change`UBSan convierte la pregunta en una parada:`-fsanitize=signed-integer-overflow -fsanitize-trap=undefined`hace que el desbordamiento "execute una instrucción de trampa", que aterriza en el Ildb como una parada con una razón.

El código optimizado también mueve lo que desea ver. El manual de GDB dice que "el optimizador ha reorganizado su código; el desarreglador le muestra lo que realmente está allí" y que una variable optimizada fuera de existencia es una "GDB nunca ve".`<optimized out>`, y su fuente separa eso de `<unavailable>`: optimizado "no existe en realidad en el programa", no disponible "existe en el programa (a nivel de máquina), pero GDB no tiene medios para llegar a él".`<variable not available>`Cuando el contador de programas está fuera de cada rango de la lista de ubicaciones DWARF de la variable. Un punto de vigilancia en un local que vive en un registro nunca se dispara, porque un registro no es memoria.`-O2`El caso extremo es que toda la función se dobló y el silencio de un punto de vigilancia que nunca se puso no prueba nada.

El arnés escribe tres tipos de registro y nunca un cuarto.`confirmed`Una parada que lo probó y no estuvo de acuerdo es`refuted`Una afirmación que no se puede probar es`not-observed`El libro mayor de la lección 35 acepta las dos primeras como`observed`registros con autoridad `controlled-replay`, y mantiene el tercero como un desconocido abierto.

## Desarreglar solo sus propias construcciones en macOS

En el silicio de Apple, el enlace indica que es para ti.`ld(1)`dice de `-adhoc_codesign`: "El estándar para Apple Silicon binarios es ser codificado ad hoc". `codesign -dvvv account-O0`en las huellas de los accesorios:

```text
Format=Mach-O thin (arm64)
CodeDirectory v=20400 size=387 flags=0x20002(adhoc,linker-signed) hashes=9+0 location=embedded
Signature=adhoc
TeamIdentifier=not set
```

`0x20002`¿ Es verdad ?`CS_ADHOC`(El artículo`0x2`) más `CS_LINKER_SIGNED`(El artículo`0x20000`), "Automáticamente firmado por el enlace" en el encabezado de firma de código de Apple. `codesign -d --entitlements -`Imprima nada después del nombre ejecutable: el binario no tiene derechos. Lldb lo lanzó, golpeó cada punto de ruptura, estableció un punto de vigilancia de hardware y escribió un núcleo, con `DevToolsSecurity -status`el informe "El modo de desarrollo está actualmente desactivado" y la protección de la integridad del sistema habilitada.

Dos derechos controlan quién puede desarmar qué, y se sientan en lados opuestos. `com.apple.security.cs.debugger`es para el desarreglador: "Un valor booleano que indica si la aplicación es un desarreglador y puede unirse a otros procesos o obtener puertos de tareas".`debugserver`La Comisión ha adoptado una decisión de la Comisión de 16 de diciembre de 2005.`codesign -d --entitlements -`en las listas de copias de las herramientas de línea de comando `com.apple.private.cs.debugger`¿ Qué ?`com.apple.private.set-exception-port`¿ Qué ?`com.apple.private.thread-set-state`, y `com.apple.private.logging.diagnostic`- ¿ Qué ?`com.apple.security.get-task-allow`El código Xcode lo inyecta en las edificaciones de desarreglo, lo deshace de la exportación, y la notarización rechaza un binario que todavía lo tiene. Los ingenieros de Apple establecen la regla para la negativa: el puerto de tareas se niega cuando el objetivo tiene el tiempo de ejecución endurecido habilitado y no lo tiene.`get-task-allow`- Una llanura .`clang`Si alguna vez firmas un acuerdo con`codesign --options runtime`, refirmarlo con `codesign -s - -f --entitlements get-task-allow.plist`antes de hacer el depósito.

Tres hechos más cerran la sección. Primero, puede ver el diálogo "Developer Tools Access necesita tomar el control de otro proceso para continuar el depuración" la primera vez en una sesión de inicio de sesión; `sudo DevToolsSecurity -enable`elimina la contraseña adicional para los miembros de la `admin`o `_developer`Los grupos. Segundo, no deshabilita SIP para deshabilitar su propio código. Apple dice "No necesita deshabilitar SIP para ejecutar y deshabilitar aplicaciones de Xcode", y el único componente SIP que toca el deshabilitamiento es el `csrutil enable --without debug`En tercer lugar, la caja de arena del cinturón de seguridad en esta sesión bloquea`task_for_pid`En su totalidad, y el dispositivo `sandbox-O0.lldb.txt`Una carrera que no puede comenzar es un hecho ambiental, y el arnés lo informa como tal.

Todo en esta lección se aplica a los programas que compila. Los mismos comandos se unirían a otro proceso sólo si su propietario opta por, y nada aquí enseña cómo cambiar eso.

## Construye el mismo

Ejecutar la demostración:

```bash
cd phases/13-tools-and-protocols/42-dynamic-analysis-of-your-own-program
python3 code/main.py
```

La salida es un informe JSON sobre las transcripciones guardadas. Para cada construcción enumera los puntos de ruptura con su resolución y recuento de hits, el recuento de puntos de vigilancia de hardware, cada parada con sus registros y argumentos, el estado de salida, la salida del programa, los errores de impresión, y las reclamaciones con sus veredictos y los registros de evidencia detrás de ellos. Añade el viaje de ida y vuelta del núcleo, los resultados de la API Python, el fallo de la caja de arena y qué herramientas externas están ausentes en esta máquina.

Forma esperada:

```json
{
  "program": "account.c",
  "variants": {
    "O0": {
      "binary": "account-O0",
      "breakpoints": {"1": {"spec": "name=bump", "address": "0x1000004bc", "resolved": true, "hits": 1}, "...": "..."},
      "hardwareWatchpoints": 4,
      "watchpoints": {"1": {"address": "0x16fdfe00c", "size": 4, "type": "m", "hits": [[250, 255]], "hitCount": 1}},
      "stops": [{"index": 0, "pc": "0x1000004bc", "function": "bump", "kind": "entry", "registers": {"x0": "0x16fdfe008", "x1": "0x5"}, "...": "..."}],
      "exitStatus": 0,
      "claims": [{"id": "arg-registers", "verdict": "confirmed", "confirmed": 5, "refuted": 0, "not-observed": 0, "...": "..."}],
      "evidence": [{"claim": "arg-registers", "verdict": "confirmed", "stop": 0, "observation": "bump: x0 = 0x16fdfe008 and a = 0x000000016fdfe008, x1 = 0x5 and by = 5"}]
    },
    "O2": {"stops": [], "claims": [{"id": "calls", "verdict": "refuted", "...": "..."}], "...": "..."}
  },
  "core": {"savedAt": {"function": "bump", "pc": "0x1000004bc"}, "labelString": "", "...": "..."},
  "types": {"python": "3.9.6", "size": 32, "...": "..."},
  "sandbox": {"stops": 0, "errors": ["process exited with status -1 (attach failed ((os/kern) invalid address))"]},
  "absentTools": ["bpftrace", "gdb", "llvm-objdump", "ltrace", "rr", "strace"]
}
```

Leer .`code/main.py`en este orden:

1. `Listing`analiza el `llvm-objdump -d`texto en funciones e instrucciones, y `Function.conditional_branches()`encuentra las ramas con sus objetivos y fall-throughs.
2. `struct_layout()`lee la estructura y las entradas de los miembros de la `llvm-dwarfdump`¿Qué es eso?
3. `lift_condition()`transforma una rama condicional en un predicado, la lección 34 paso para `tbz`¿ Qué ?`cbz`, y las formas de comparación y rama.
4. `Plan.commands()`escribe el archivo de comandos de lldb de la lista, y `Plan.lldb_command()`es la invocación del lote.
5. `parse_transcript()`camina la salida en orden y construye `Stop`registros con registros, memoria, variables y valores de puntos de vigilancia, más la tabla de puntos de ruptura y el estado de salida.
6. `claims_for()`La Comisión ha adoptado una decisión de la Comisión de 16 de diciembre de 2005.`evaluate()`las calificaciones de ellos para parar por parar con el `_check_*`Funciones.
7. `rollup()`reduce los registros a un veredicto por demanda, con la negación que tiene prioridad sobre la confirmación.
8. `capture()`y `capture_extras()`producido los accesorios, y `live()`repite todo el ciclo en una nueva construcción cuando `clang`y `lldb`puede correr.

Hay cuatro órdenes más.`python3 code/main.py plan O0`Imprima el archivo de comandos para una construcción. `python3 code/main.py commands`Imprima la tabla de GDB y LDB. `python3 code/main.py tools`informes sobre las herramientas externas `PATH`- ¿ Qué ?`python3 code/main.py live O0`compilaciones `account.c`En un directorio temporal, los planes de su nueva lista, ejecuta lldb, y califica las reclamaciones contra esa carrera.

## Usalo

Utilice el arnés en un programa propio en este orden.

1. Construye con `-g`Mantenga el binario, su dSYM o su DWARF, y el `llvm-objdump -d`Te mandaré un mensaje de texto juntos.
2. Escriba las reclamaciones antes de ejecutar. Cada una nombra una función, un registro o una dirección o un offset, un valor o borde esperado, y la lección o documento de la que proviene. Una reclamación que escribe después de ver la parada describe la parada y no prueba nada.
3. Generar el plan de la lista y leerlo. Cada dirección en él debe rastrear a una línea en la lista. Si un punto de ruptura no tiene tal línea, eliminarlo.
4. ejecuta una vez en modo batch y mantén la transcripción con la línea de comandos y la versión de depuración en su encabezado. La transcripción es la evidencia. Los veredictos se derivan de ella, y cualquiera puede volver a derivarlos.
5. Grado. Lea todos.`not-observed`Las razones le dicen qué afirmaciones necesitan otra entrada, otra construcción o otro punto de ruptura.
6. ejecuta la construcción optimizada con las mismas reclamaciones. espera puntos de ruptura resueltos con cero hits donde el compilador se alineó, y mantenga esas reclamaciones abiertas.
7. Introduzca los registros confirmados y refutados en el libro mayor de la lección 35 como `observed`, autoridad `controlled-replay`Las no observadas se quedan desconocidas con el número de puntos de ruptura.

## Laboratorio interactivo

### Laboratorio A: leer una parada a mano

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/42-dynamic-analysis-of-your-own-program/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
t = load_transcript("session-O0.lldb.txt")
s = t.stops[0]
hex(s.pc), s.function, s.reason, s.registers
[hex(w) for w in s.memory[:8]]
s.struct_pointer().fields
```

Antes de la segunda línea, escriba qué palabra espera que tenga 250 y qué dos palabras tienen 1,25.`struct.unpack("<d", struct.pack("<II", s.memory[4], s.memory[5]))`y comparar. Entonces mira`t.stops[1].watch_hit`y `t.stops[1].pc`, y encontrar esa dirección en `account-O0.objdump.txt`Es la instrucción después de la tienda.

### Laboratorio B: la rama y su sucesor

```python
plan = make_plan("O0")
plan.branches()
classify = plan.listing.function("classify")
[lift_condition(classify, b, "a->balance")[0] for b in plan.branches()]
[(hex(s.pc), s.struct_pointer().field_int("balance")) for s in t.stops_in("classify")]
```

Combine cada parada de borde con su parada de entrada y decide, antes de mirar la evidencia, qué sucesor predice el predicado.`[e.to_dict() for e in evaluate(plan, t, struct_layout(open("fixtures/account-O0.dwarfdump.txt").read(), "account")) if e.claim.startswith("branch")]`y comparar.

### Laboratorio C: la carrera que nunca se detuvo

```python
o2 = load_transcript("session-O2.lldb.txt")
len(o2.stops), o2.exit_status, o2.hits()
o2.errors[:3]
make_plan("O2").branches()
rollup(evaluate(make_plan("O2"), o2, struct_layout(open("fixtures/account-O2.dwarfdump.txt").read(), "account")))
```

Explica cada uno `not-observed`En sus propias palabras, entonces abra.`account-O2.objdump.txt`y encontrar las cuatro constantes en `main`que reemplazó las llamadas.`DW_TAG_inlined_subroutine`En el`account-O2.dwarfdump.txt`con`count_tags()`¿ Qué ?

### Laboratorio D: rompe la evidencia

```python
text = open("fixtures/session-O0.lldb.txt").read().replace("x1 = 0x0000000000000005", "x1 = 0x0000000000000006", 1)
broken = parse_transcript(text)
[e.to_dict() for e in evaluate(plan, broken, struct_layout(open("fixtures/account-O0.dwarfdump.txt").read(), "account")) if e.claim == "arg-registers"][:2]
```

Un registro editado hace que el primero .`arg-registers`registro a `refuted`y el rolloup con él, mientras que las otras cuatro entradas permanecen `confirmed`. Entonces borra el `old value: 250`línea antes de la parada de vigilancia y vigilancia `store-once`Caer a`not-observed`Un gradado que no puede fallar no es una calificación.

Si el Ildb puede lanzar un proceso en su caparazón, termine con `python3 main.py live O0`y `python3 main.py live O2`Las direcciones de la pila serán diferentes de las fichas.

## Laboratorio de práctica

Extenda el arnés con una afirmación sobre el bucle en `sum_balances`¿ Qué ?

Requisitos:

- Encuentra el borde posterior del bucle de `sum_balances`en el `-O0`en la lista con `conditional_branches()`y añadir un punto de ruptura en el encabezado del bucle.
- Añadir una reclamación que el encabezado ejecuta `count + 1`tiempo para `count`= 3, que es la lección 33 borde trasero más la prueba de salida.
- Extenda el gancho de parada o el plan para que el gancho lea la ranura de la pila del contador de bucle en cada parada de encabezado, y escriba un `_check_loop()`que registra la secuencia de valores de contra.
- Recuperar la transcripción con `python3 main.py capture O0`La prueba del plan fallará hasta que el archivo de comandos comprometido coincida con el nuevo plan, que es el punto de esa prueba.
- Añadir una prueba que edita un valor de contra en la transcripción y afirma que la reclamo gira `refuted`¿ Qué ?

El éxito significa que el informe muestra los puntos de la encabezada en orden con sus valores de contra, y el `-O2`Run mantiene la nueva reclamación `not-observed`con la razón de que el bucle fue vectorizado o doblado.

## Lección Artefacto

Esta lección incluye `outputs/skill-dynamic-analysis-of-your-own-program.md`. Utilice para convertir un programa que ha construido, su lista y una lista de reclamaciones estáticas en una transcripción de depuración guardada y un veredicto por reclamación, con la parada que produjo cada veredicto y la razón de cada reclamación que no se observó.

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/42-dynamic-analysis-of-your-own-program
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

La suite se ejecuta sin Ildb.`code/fixtures/`La verificación debe demostrar:

- el analizador de la lista encuentra las cuatro funciones con sus rangos y las dos ramas condicionales de `classify`En el`-O0`con objetivo y caída
- el ascensor de `tbz w8, #0x1f`¿ Es verdad ?`a->balance >= 0`y el ascensor de `subs`Además`b.le`¿ Es verdad ?`a->balance <= 200`
- `main`En el`-O0`llamadas`bump`¿ Qué ?`classify`tres veces, y `sum_balances`, y `main`En el`-O2`Sólo llamadas .`printf`
- x86-64 `j<cc>`Las mnemónicas son condicionales y `jmp`y `callq`no son
- la descarga DWARF produce una estructura de 32 bytes con miembros en 0, 4, 8, 16 y 24
- el `-O2`DWARF tiene cuatro subprogramas y cero entradas de subroutine en línea, y `nm`sigue listando las tres funciones
- el plan es igual al archivo de comandos comprometido para ambas construcciones, y el encabezado de transcripción registra el comando Lldb que lo ejecutó
- el `-O0`transcript tiene once paradas en las funciones esperadas, estado de salida 0, y las cuatro líneas de salida
- la salida del gancho está conectada a la parada que la sigue, con `x1`= 5, memoria comenzando en `x0`, y la cadena de etiqueta
- la tabla de puntos de ruptura tiene resolución y cuenta de impacto 1, 3, 1, 2, 1, 1, 1
- El punto de vigilancia es de 4 bytes de tipo modifique en `x0 + 4`, golpeado una vez con el valor antiguo 250 y el nuevo valor 255, y cuatro ranuras de hardware se reportan
- el `-O2`La transcripción tiene cero paradas, tres puntos de ruptura resueltos con cero hits, sin punto de vigilancia y un error por no utilizado `continue`
- el lanzamiento en caja de arena falla antes de cualquier parada con el error de adjuntación
- El viaje de ida y vuelta del núcleo mantiene el rastro y `balance`= 250 y pierde la cuerda detrás `label`
- La API Python lee los cinco campos, el tamaño 32, y la función se extiende sin un proceso
- una transcripción escrita a mano x86-64 analiza con `rdi`¿ Qué ?`rsi`, y `rip`
- Cada reclamación se confirma en `-O0`con registros 5, 11, 3, 2, 1, 1, 1, 1
- la evidencia de la rama nombra el valor, la condición y el borde
- la evidencia de campo de compensación decodifica las palabras a través del diseño DWARF
- la prueba de la tienda y los pasos llevan el valor viejo y nuevo y los saldos de las tres filas
- el `-O2`ejecuta refuta los bordes de llamada y deja las otras afirmaciones no observadas
- se refuta un valor de registro editado, no se observa un impacto en el punto de vigilancia eliminado y se refuta la ganancia de la recolección
- El mapa de comandos tiene formas gdb e lldb para cada fila, y la tabla de hardware nombra cuatro registros de defecto x86
- la demostración muestra herramientas ausentes en lugar de fallas, y el modo en vivo informa la herramienta ausente por nombre
- Los datos de las figuras provienen de ambas sesiones y el módulo del sitio incorpora los mismos datos

## Modos de falla de producción

| Failure | What the weak report says | What the harness must prove |
|---|---|---|
| Resolved breakpoint with zero hits | "the function runs" or "the function is dead" | a full run with exit status, hit counts, and the verdict `refuted` for the call edge only |
| Register read at the wrong stop | "`x1` = 5 inside `classify`" | compare only declared arguments at the entry stop |
| Struct read without offsets | a column of hex words | every word mapped to a `DW_AT_data_member_location` and compared with `frame variable` |
| Branch graded from the mnemonic | "`tbz` means taken when negative" | the lifted predicate from the encoding and the stop at the successor |
| Watchpoint on a register-resident local | "no write happened" | the watchpoint was set on an address and was hit, or the record is `not-observed` |
| Auto-continue breakpoints | a clean transcript with no stops | plain breakpoints, a stop hook, and a continue budget |
| Address breakpoint without a module | `unresolved, hit count = 0` | `-s MODULE` so the listing address resolves to a load address |
| First command error ends the batch | no `breakpoint list` at the end | `interpreter.stop-command-source-on-error false` |
| Stack-style core read as full memory | `label = ""` reported as the program's value | the core style, the pointer, and the segment it points into |
| Sandboxed shell | "lldb is broken" | `attach failed ((os/kern) invalid address)` recorded as an environment fact |
| Hardened target without `get-task-allow` | "macOS forbids debugging" | the two conditions from Apple's rule, and a re-sign of your own binary |
| Transcript discarded after grading | verdicts with no source | the transcript file, its command line, and the debugger version |

## Regla de funcionamiento

Planifique las paradas de la lista, ejecute una vez en modo batch, mantenga la transcripción y califique cada reclamación estática en cada parada que pueda probarla. Escriba confirmado o refutado solo desde una parada, escriba no observado con los recuentos de hits y la razón, y nunca deje que un punto de ruptura resuelto se interponga para una carrera. Debugue solo las construcciones que hizo.

## Leer más

- [LLDB Tutorial: command grammar, breakpoints, watchpoints, stop hooks](https://lldb.llvm.org/use/tutorial.html)
- [lldb(1) manual page: --batch, --one-line, --source, --core](https://lldb.llvm.org/man/lldb.html)
- [GDB to LLDB Command Map](https://lldb.llvm.org/use/map.html)
- [LLDB Python Reference: SBTarget, SBFrame, SBValue, and the embedded interpreter](https://lldb.llvm.org/use/python-reference.html)
- [LLDB Caveats: the Python build lldb links against](https://lldb.llvm.org/resources/caveats.html)
- [LLVM 23.1.3 release](https://github.com/llvm/llvm-project/releases/tag/llvmorg-23.1.3)
- [GDB 18.1 release announcement, 2026-09-25](https://sourceware.org/pipermail/gdb-announce/2026/000150.html)
- [GDB manual: Process Record and Replay](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Process-Record-and-Replay.html)
- [GDB manual: Debugging Optimized Code](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Optimized-Code.html)
- [GDB wiki: Permissions on Darwin, code signing a debugger](https://sourceware.org/gdb/wiki/PermissionsDarwin)
- [gdb-patches: Initial AArch64-Darwin support series, October 2025](https://sourceware.org/pipermail/gdb-patches/2025-October/222196.html)
- [Homebrew gdb formula pull request adding arm64 bottles](https://github.com/Homebrew/homebrew-core/pull/209753)
- [ptrace(2), Linux manual page](https://man7.org/linux/man-pages/man2/ptrace.2.html)
- [lldb debugserver MachTask.mm: task_for_pid and task_set_exception_ports](https://github.com/llvm/llvm-project/blob/main/lldb/tools/debugserver/source/MacOSX/MachTask.mm)
- [Intel SDM Volume 2: INT n, INTO, INT3, INT1](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Intel SDM Volume 3: debug registers DR0 to DR7 (Debug, Branch Profile, TSC, and Resource Monitoring Features)](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Arm A64 ISA: BRK, Breakpoint instruction](https://developer.arm.com/documentation/ddi0602/latest/Base-Instructions/BRK--Breakpoint-instruction-)
- [Arm A64 ISA: TBZ, Test bit and branch if zero](https://developer.arm.com/documentation/ddi0602/latest/Base-Instructions/TBZ--Test-bit-and-branch-if-zero-)
- [Arm Architecture Reference Manual, D2.9 Watchpoint exceptions](https://developer.arm.com/documentation/ddi0487/latest/)
- [Apple ld(1): -adhoc_codesign](https://keith.github.io/xcode-man-pages/ld.1.html)
- [Apple ld64 LinkEdit.hpp: CS_LINKER_SIGNED](https://github.com/apple-oss-distributions/ld64/blob/59a99ab60399c5e6c49e6945a9e1049c42b71135/src/ld/LinkEdit.hpp)
- [Apple: Debugging Tool Entitlement, com.apple.security.cs.debugger](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.cs.debugger)
- [Apple: Hardened Runtime](https://developer.apple.com/documentation/security/hardened-runtime)
- [Apple: Resolving common notarization issues, the get-task-allow entitlement](https://developer.apple.com/documentation/security/resolving-common-notarization-issues)
- [Apple: Disabling and Enabling System Integrity Protection](https://developer.apple.com/documentation/security/disabling-and-enabling-system-integrity-protection)
- [Apple Developer Forums: debugserver may not get the taskport of a hardened process](https://developer.apple.com/forums/thread/127544)
- [Apple DevToolsSecurity(8)](https://keith.github.io/xcode-man-pages/DevToolsSecurity.8.html)
- [Apple core(5)](https://keith.github.io/xcode-man-pages/core.5.html)
- [Apple Technical Note TN2124: Mac OS X Debugging Magic, core dumps](https://developer.apple.com/library/archive/technotes/tn2124/_index.html)
- [lldb `process save-core` help text](https://github.com/llvm/llvm-project/pull/143126)
- [core(5), Linux manual page](https://man7.org/linux/man-pages/man5/core.5.html)
- [coredumpctl(1)](https://man7.org/linux/man-pages/man1/coredumpctl.1.html)
- [Apple dtrace(1)](https://keith.github.io/xcode-man-pages/dtrace.1.html)
- [Apple dtruss(1m)](https://keith.github.io/xcode-man-pages/dtruss.1m.html)
- [strace releases](https://github.com/strace/strace/releases)
- [ltrace](https://ltrace.org/)
- [bpftrace CHANGELOG](https://github.com/bpftrace/bpftrace/blob/master/CHANGELOG.md)
- [rr project](https://rr-project.org/)
- [rr 5.9.0 release](https://github.com/rr-debugger/rr/releases/tag/5.9.0)
- [rr wiki: Building And Installing, hardware and VM requirements](https://github.com/rr-debugger/rr/wiki/Building-And-Installing)
- [Clang UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)
- [ISO/IEC 9899:2024 working draft N3220, clause 3.5 undefined behavior](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3220.pdf)
- [DWARF Debugging Information Format Version 5](https://dwarfstd.org/doc/DWARF5.pdf)
- [Procedure Call Standard for the Arm 64-bit Architecture (AAPCS64)](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst)
- [System V Application Binary Interface, AMD64 Architecture Processor Supplement](https://gitlab.com/x86-psABIs/x86-64-ABI)
