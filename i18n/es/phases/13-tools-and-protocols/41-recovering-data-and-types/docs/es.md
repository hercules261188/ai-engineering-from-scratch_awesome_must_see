# Recuperación de datos y tipos: estructuras, matrices, objetos de C++ y DWARF

> Un descompilador imprime`a->balance`cuando tiene una tabla de tipo y `*(int *)(param_1 + 4)`Esta lección construye la tabla a partir de los accesos, lee la verdadera de DWARF 5, y muestra dónde están de acuerdo los dos.

**Type:** Build
**Languages:** Python, C, C++, x86-64 assembly
**Prerequisites:** Phase 13 · 33 (disassembly and control flow), Phase 13 · 34 (lifting to pseudocode)
**Time:** ~140 minutes

## Objetivos de aprendizaje

- Lea un acceso de miembro de estructura como registro base más desplazamiento, y un acceso de matriz como base más índice veces paso, de las reglas de dirección de Intel SDM.
- Aplicar las reglas de alineación del Sistema V a la mano a `struct account`y explicar cada byte de relleno.
- Construye un pase de recuperación de tipo que convierta los accesos en una lista en offsets de campo, anchos, tipos, un paso y un tamaño, con la evidencia para cada uno.
- Escriba un lector DWARF 5 para `.debug_info`¿ Qué ?`.debug_abbrev`¿ Qué ?`.debug_str_offsets`, y `.debug_addr`de un objeto relocable, y comprobarlo con `llvm-dwarfdump`¿ Qué ?
- Comparar el diseño recuperado con el diseño DWARF y nombrar cada diferencia como evidencia faltante, un acceso fusionado, o un tipo no comprobable.
- Lea un vtable de Itanium C++, su tipo de información y sus nombres desfigurados de las reubicaciones, y resuelve una llamada virtual a una ranura.

## El problema

Usted compilado`account.c`Sin`-g`y preguntó a un agente qué.`bump`El agente envió el objeto a Ghidra 12.1.4 y recibió esto:

```c
void bump(long param_1,int param_2)
{
  *(int *)(param_1 + 4) = param_2 + *(int *)(param_1 + 4);
  return;
}
```

La misma fuente compilada con `-g`da esto desde el mismo decompiler:

```c
int bump(account *a,int by)
{
  a->balance = by + a->balance;
  return a->balance;
}
```

Las instrucciones son idénticas en ambos objetos. La diferencia es una tabla de tipo. Con la tabla el descompilador sabe que el primer parámetro apunta a una estructura de 32 bytes cuyo campo en el offset 4 es un 4 byte `int`nombre`balance`Sin él , el decompiler sólo sabe que algo de 8 bytes de ancho llegó .`%rdi`y que 4 bytes en `+4`se leían y se escribían.

Todo en esa tabla tiene una fuente.`addl 0x4(%rax), %ecx`El ancho 4 proviene de la`l`El nombre proviene de un`DW_TAG_member`Entrada en `.debug_info`Esta lección construye ambas mitades: un pase que recupera lo que las instrucciones demuestran y un lector que decodifica lo que el compilador grabó.

```figure
re-41-struct-recovery
```

## Un acceso de miembro es un desplazamiento

El volumen 1 del SDM de Intel, sección 3.7.5, define la dirección efectiva como `Offset = Base + (Index * Scale) + Displacement`, con una escala de 1, 2, 4 o 8 y un desplazamiento de 8, 16 o 32 bits.`RIP + Displacement`. Un acceso a miembros de estructuras utiliza la primera forma sin índice: el registro base contiene la dirección del objeto y el desplazamiento es el desplazamiento de miembros.

¿ Qué ?`bump`En el`-O0`. La lista de `llvm-objdump -d -r`Se dice:

```text
       4: 48 89 7d f8                  	movq	%rdi, -0x8(%rbp)
       e: 48 8b 45 f8                  	movq	-0x8(%rbp), %rax
      12: 03 48 04                     	addl	0x4(%rax), %ecx
      15: 89 48 04                     	movl	%ecx, 0x4(%rax)
```

Descifrar`03 48 04`con las reglas de la lección 33. `03`¿ Es verdad ?`ADD r32, r/m32`- ModRM .`48`¿ Es verdad ?`01 001 000`¿ Qué es esto ?`mod = 01`se refiere a un operando de memoria con un desplazamiento de 8 bits, `reg = 001`¿ Es verdad ?`%ecx`¿ Qué ?`r/m = 000`¿ Es verdad ?`%rax`El byte de desplazamiento es `04`La instrucción lee 4 bytes en `%rax + 4`y los agrega a `%ecx`- El .`l`el sufijo que `llvm-objdump`Impresiones es el tamaño de la operanda, así que el campo es de 4 bytes de ancho.`%rax`vino de la ranura del marco en `-0x8(%rbp)`, y ese espacio recibido`%rdi`, el primer argumento de un número entero bajo la convención de llamadas del Sistema V. Sigue tres hechos: el primer parámetro es un puntero, el objeto al que apunta tiene un campo de 4 bytes en el offset 4, y `bump`lee y escribe ese campo.

Nada en estos bytes dice`int`¿ Qué ?`balance`, o`struct account`No hay nada que diga cuánto es el objeto. Los 28 bytes restantes de la estructura son invisibles para el`bump`¿ Qué ?

## Las reglas de diseño del sistema V

El x86-64 psABI, sección 3.1.2, da tres reglas para los agregados. "Las estructuras y las uniones asumen la alineación de su componente más estrictamente alineado. Cada miembro se asigna al offset disponible más bajo con la alineación apropiada. El tamaño de cualquier objeto es siempre un múltiplo de la alineación del objeto".`char`1 y 1, `short`2 y 2, `int`4 y 4, `double`8 y 8, cualquier puntero 8 y 8.

Aplicar las normas a las estructuras en`account.c`¿Qué es esto ?

```c
struct account {
    char kind;
    int balance;
    short code;
    double rate;
    const char *label;
};
```

| Member | Size | Alignment | Lowest offset that fits | Padding before |
|---|---|---|---|---|
| `kind` | 1 | 1 | 0 | 0 |
| `balance` | 4 | 4 | 4 | 3 (bytes 1 to 3) |
| `code` | 2 | 2 | 8 | 0 |
| `rate` | 8 | 8 | 16 | 6 (bytes 10 to 15) |
| `label` | 8 | 8 | 24 | 0 |

El último miembro termina en 32. La alineación más estricta es 8, y 32 es un múltiplo de 8, por lo que el tamaño es 32 sin relleno de cola. Nueve bytes de los 32 son relleno, y el psABI dice "El contenido de cualquier relleno es indefinido".

El orden de los miembros determina el tamaño.`double, pointer, int, short, char`el relleno compensa 0, 8, 16, 20, 22 y termina en 23, que ronda hasta 24. `sysv_layout()`En el`code/main.py`aplica las tres reglas y produce ambos resultados.`ctypes`el módulo aplica las mismas reglas a través de la convención del compilador host, y `ctypes.sizeof`¿ Qué ?`ctypes.alignment`, y el `.offset`de cada campo dar 32, 8, y `0, 4, 8, 16, 24`El programa mismo imprime la tercera copia cuando se construye y se ejecuta en esta máquina:

```text
sizeof 32 align 8
kind 0 balance 4 code 8 rate 16 label 24
sum 315 tag -3 projected 318.75 label_length 7
```

La arm AAPCS64 establece las mismas reglas agregadas en su sección 5.10.1, y el documento arm64 de Apple mantiene estos cinco tamaños escalares, por lo que los números coinciden en los dos objetivos.`-target x86_64-unknown-linux-gnu`para que el lector ELF de la lección 33 y el lector DWARF de abajo trabajen en ellos.

## Lea el resto de la estructura a mano

`bump`Ve un campo. Las otras cuatro funciones en `account.c`En el caso de las direcciones de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la dirección de la`account-x86_64-O0-g.objdump.txt`¿ Qué ?

`sum_balances`indexa una matriz de la estructura:

```text
      51: 48 8b 45 f8                  	movq	-0x8(%rbp), %rax
      55: 48 63 4d ec                  	movslq	-0x14(%rbp), %rcx
      59: 48 c1 e1 05                  	shlq	$0x5, %rcx
      5d: 48 01 c8                     	addq	%rcx, %rax
      60: 8b 40 04                     	movl	0x4(%rax), %eax
```

`%rax`¿ Es verdad ?`list`¿ Qué ?`%rcx`¿ Es verdad ?`i`de signo extendido a 64 bits, y `shlq $0x5`y la multiplicamos por 32.`addq`¿ Qué ?`%rax`¿ Es verdad ?`list + 32 * i`, y la carga lee 4 bytes en el desplazamiento 4 del elemento `i`. El conteo de cambios es el único lugar en este objeto donde el tamaño de la estructura aparece como un número.`0x49`a la `jmp 0x49`En el`0x72`¿ Qué ?

`tag`Se lee dos campos estrechos:

```text
      8c: 0f be 00                     	movsbl	(%rax), %eax
      98: 0f bf 48 08                  	movswl	0x8(%rax), %ecx
```

`movsbl`carga un byte en el desplazamiento 0 y se extiende de signo, por lo que el campo en 0 es un número entero de 1 byte firmado. `movswl`carga dos bytes en el offset 8 con extensión de signo, por lo que el campo en 8 es un número entero de 2 bytes firmado.`-O1`El primer acceso se hace `cmpb $0x6c, (%rdi)`, que todavía dice 1 byte en el desplazamiento 0 y no dice nada sobre el signo.

`projected`dice un flotante:

```text
      cc: f2 0f 2a 40 04               	cvtsi2sdl	0x4(%rax), %xmm0
      d5: f2 0f 59 40 10               	mulsd	0x10(%rax), %xmm0
```

`cvtsi2sdl`Converte un número entero de 32 bits firmado de la memoria a un doble, lo que confirma que el offset 4 tiene un número entero de 4 bytes firmado. `mulsd`se multiplica por el doble de la memoria, por lo que el offset 16 tiene un valor de punto flotante de 8 bytes.

`label_length`lee un puntero y luego lo sigue:

```text
      f3: 48 8b 40 18                  	movq	0x18(%rax), %rax
      f7: 48 63 4d f4                  	movslq	-0xc(%rbp), %rcx
      fb: 80 3c 08 00                  	cmpb	$0x0, (%rax,%rcx)
```

El `movq`Se lee 8 bytes en el offset 24. Por sí solo que es un integer de 8 bytes o un puntero.`cmpb`El valor de la memoria que se utiliza como dirección es un puntero.`cmpb`se lee 1 byte en el índice `%rcx`de la misma, y en `-O1`el puntero está avanzado con `leaq 0x1(%rcx), %rcx`dentro del bucle, por lo que los elementos apuntados están a 1 byte de distancia.

Si juntamos las cinco funciones, la estructura es`{ i8 at 0, i32 at 4, i16 at 8, f64 at 16, pointer at 24 }`Las diferencias entre 1 y 3 y 10 y 15 son exactamente el relleno previsto por las reglas de psABI.`const char *`, el tipo declarado del byte firmado en 0, que la fuente escribe `char`, y que las cinco funciones reciben el mismo tipo.`main`, que pasa un objeto de marco a los cinco.`-O0`Las páginas de llamadas hacen que sea visible: `leaq -0x70(%rbp), %rdi`seguido por `addq $0x20, %rdi`antes de`callq bump`, y `addq $0x40, %rdi`antes de`callq tag`Los desventajos son 0,`0x20`, y `0x40`son tres elementos de una matriz, y el `memcpy`de `0x60`los bytes que lo llenan es 96 bytes, que es tres elementos de 32.

## El pase

`trace_function()`En el`code/main.py`El control de la función de un registro de 64 bits o de un espacio de marco como `-0x8(%rbp)`Cada titular tiene uno de los tres valores.`Pointer`es una raíz más una offset constante más un conjunto de pasos, donde un paso registra que un múltiple desconocido de tantos bytes se agregó.`Index`es un número entero de valor desconocido que es un múltiplo de una escala.

El estado comienza con los seis registros de argumentos del Sistema V .`Pointer("param0", 0)`por el`Pointer("param5", 0)`Entonces cada instrucción aplica una regla:

- ¿ Qué es esto ?`movq`Una escritura de 32 bits o más estrecha en un registro lo hace un `Index`con escala 1.
- ¿ Qué es esto ?`leaq`con una base de puntero produce un puntero con el desplazamiento añadido, y con un registro de índices añade `scale * index_scale`A los pasos.`leaq`cuya base es `%rbp`o `%rsp`nombra un objeto de marco, que es cómo `main`Pases `rows`¿ Qué ?
- `shlq $n`en un`Index`multiplica su escala por `2^n`, y `imulq $k`se multiplica por `k`¿ Qué ?
- `addq`de un puntero e índice mantiene el puntero y añade la escala del índice a los pasos. `addq $k`a un indicador se mueve su descenso y se registra también como incremento de dicho registro.
- Una llamada hace que cada registro guardado por el llamador sea desconocido, y los registros de argumentos que contienen punteros se registran como un sitio de llamada.
- Cualquier operando de memoria cuyo registro base contiene un puntero se convierte en un `Access`con la raíz y el desplazamiento del puntero, el ancho y el tipo de la mnemónica, y una bandera de la tienda.

La amplitud y la bondad vienen del mnemónico solo.`access_width()`mapas de la `b`¿ Qué ?`w`¿ Qué ?`l`, y `q`los sufijos a 1, 2, 4 y 8 bytes, `movsbl`y `movswl`para las cargas estrechas firmadas, `movzbl`a los no firmados, el `sd`y `ss`instrucciones de SSE escalar para flotadores de 8 y 4 bytes, `cvtsi2sdl`a una fuente de un número entero de 4 bytes firmada, y el de 16 bytes `movaps`familia a un vector. `leaq`y el `nop`los formularios no son accesos.

```python
if memory.base in FRAME_REGISTERS:
    return Pointer(f"frame({memory.base}{format_signed(memory.displacement) if memory.displacement else ''})", 0)
base = env.get(memory.base)
if not isinstance(base, Pointer):
    return Unknown
strides = set(base.strides)
if memory.index is not None:
    index_value = env.get(memory.index)
    strides.add((index_value.scale if isinstance(index_value, Index) else 1) * memory.scale)
return Pointer(base.root, base.offset + memory.displacement, frozenset(strides), base.origin)
```

Una carga de 8 bytes a través de un puntero produce un nuevo puntero cuyos nombres raíces el campo de que proviene, como `param0->0x18`. Cuando ese valor se utiliza más tarde como base, o avanzado con `leaq`, el campo de que viene se marca como un puntero probado. Esa es la regla que convierte el descalzo 24 de "8 bytes" en "puntero".

### Cuatro tipos de pruebas de pasos

`infer_layout()`fusiona los accesos de cada función de testigo y recoge el paso de cuatro fuentes, cada una con la instrucción que lo produjo.

| Kind | Where it comes from | Build that shows it |
|---|---|---|
| `scale` | an `Index` with scale above 1 reached a pointer, from `shlq $0x5` or `imulq` | `-O0`, `shlq $0x5, %rcx` at `0x59` |
| `loop-step` | a 64-bit increment of the base or index register inside a loop that also contains the access | `-O1`, `addq $0x20, %rdx` at `0x24` |
| `offset-delta` | same-width accesses through one base register inside one loop, at offsets a constant apart | `-O2`, eight loads 32 apart |
| `call-site` | the offsets at which one frame object is passed to the witness functions | `-O0`, `main` passes `rows`, `rows+0x20`, `rows+0x40` |

Un bucle es un retroceso .`jmp`En el caso de los datos de acceso, el incremento se calcula sólo cuando el rango de bucle contiene tanto el incremento como el acceso.`addl $0x1, %eax`que avanza.`i`En el`-O0`, porque es un 32 bits de adición y porque el acceso no pasa por `%eax`El paso final es el mayor divisor común de todos los valores encontrados. El tamaño es el paso cuando el paso es al menos tan grande como el final del campo final.

### El bucle vectorizado en -O2

En el`-O2`Clang vectorizado `sum_balances`. El cuerpo del bucle de`0x50`¿ Qué ?`0xaa`Se lee ocho elementos por pase:

```text
      3b: 48 8d 87 e4 00 00 00         	leaq	0xe4(%rdi), %rax
      50: 66 0f 6e 50 80               	movd	-0x80(%rax), %xmm2
      55: 66 0f 6e 98 60 ff ff ff      	movd	-0xa0(%rax), %xmm3
      61: 66 0f 6e 90 40 ff ff ff      	movd	-0xc0(%rax), %xmm2
      69: 66 0f 6e a0 20 ff ff ff      	movd	-0xe0(%rax), %xmm4
      7d: 66 0f 6e 10                  	movd	(%rax), %xmm2
      81: 66 0f 6e 58 e0               	movd	-0x20(%rax), %xmm3
      8a: 66 0f 6e 50 c0               	movd	-0x40(%rax), %xmm2
      8f: 66 0f 6e 60 a0               	movd	-0x60(%rax), %xmm4
      a0: 48 05 00 01 00 00            	addq	$0x100, %rax
      aa: 75 a4                        	jne	0x50 <sum_balances+0x40>
```

`%rax`comienza en`list + 0xe4`Los ocho desplazamientos colocan las cargas en compensaciones .`0x04, 0x24, 0x44, 0x64, 0x84, 0xa4, 0xc4, 0xe4`de la`list`, que son`4, 36, 68, 100, 132, 164, 196, 228`El incremento es`0x100`Un pase que registra compensaciones al verlas informa ocho campos y una estructura de al menos 232 bytes.

`fold_unrolled()`En un bucle, toma los accesos de la misma anchura a través de un registro base. Sus compensaciones son`d`aparte, aquí 32. Si el paso del bucle es `k * d`Y hay exactamente`k`Los cambios de la velocidad de la velocidad de los motores de carga son distintos, el bucle se desarrolla.`k`veces, y cada acceso es el mismo elemento de acceso en el offset `(offset - lowest) mod d`Aquí tienes .`256 = 8 * 32`con ocho desplazamientos, así que todas las ocho cargas se doblan en el desplazamiento 4, y los registros de pruebas de paso`offset-delta 32`¿ Qué ?`unrolled 32`, y `loop-step 256`El ciclo de residuos escalares en `0xe0`añade `scale 32`de la`shlq $0x5, %rdx`y `loop-step 32`de la`addq $0x20, %rdx`El divisor común más grande de los cinco valores es 32.

La figura tiene un botón que desactiva el plegado.`-O2`Este estado es lo que un lector ve que confía en el desplazamiento sin el bucle.

## DWARF 5 Desde los bytes

El `-g`construcción de `account.c`contiene ocho secciones de desactivación. `.debug_info`es 493 bytes, `.debug_abbrev`179, `.debug_str`230, `.debug_str_offsets`124, `.debug_addr`El número 112,`.debug_line`312, `.debug_line_str`12, y `.debug_frame`264. La introducción de las listas de normas DWARF 5 `.debug_str_offsets`¿ Qué ?`.debug_addr`, y `.debug_line_str`como nuevo en la Versión 5. El lector en `code/main.py`Necesita los primeros cinco.

### El encabezado de la unidad

La sección 7.5.1.1 de la norma da el título completo de la unidad de compilación: `unit_length`¿ Qué ?`version`¿ Qué ?`unit_type`¿ Qué ?`address_size`, y `debug_abbrev_offset`Los primeros doce bytes de `.debug_info`son:

```text
e9 01 00 00   unit_length = 0x1e9 = 489
05 00         version = 5
01            unit_type = DW_UT_compile
08            address_size = 8
00 00 00 00   debug_abbrev_offset = 0
```

El `unit_type`el byte es nuevo en la versión 5, y el estándar lo dice en la misma sección. La longitud cuenta los bytes después de sí mismo, por lo que la siguiente unidad comenzaría en `4 + 489 = 493`, el final de la sección. `llvm-dwarfdump --debug-info`Imprime los mismos cinco valores en su `Compile Unit:`la línea, y el primer ensayo analiza esa línea y compara.

### Las abreviaturas

La sección 2.1 dice que cada entrada de información de depuración "se compone de una etiqueta de identificación y una serie de atributos". La entrada en sí solo almacena un código de abreviatura.`.debug_abbrev`(sección 7.5.3) que nombra la etiqueta, dice si los niños siguen, y enumera cada atributo con su forma. Este objeto tiene 14 declaraciones. Dos cosas aquí:

```text
[12] DW_TAG_structure_type  DW_CHILDREN_yes
     DW_AT_name              DW_FORM_strx1
     DW_AT_byte_size         DW_FORM_data1
     DW_AT_decl_file         DW_FORM_data1
     DW_AT_decl_line         DW_FORM_data1
[13] DW_TAG_member           DW_CHILDREN_no
     DW_AT_name              DW_FORM_strx1
     DW_AT_type              DW_FORM_ref4
     DW_AT_decl_file         DW_FORM_data1
     DW_AT_decl_line         DW_FORM_data1
     DW_AT_data_member_location  DW_FORM_data1
```

`parse_abbreviations()`Se lee código ULEB128, etiqueta ULEB128, un byte de niños, luego atributo y forma pares hasta que un `0, 0`En la sección 7.5.6 se dan los códigos de formulario: `data1`es un byte, `ref4`es un descenso de 4 bytes en relación con la unidad, `strx1`es un índice de 1 byte en la tabla de compensaciones de cadenas, `sec_offset`y `data4`son 4 bytes, `addrx`es un índice ULEB128 en la tabla de direcciones, `exprloc`es una longitud ULEB128 y que muchos bytes, y `flag_present`no tiene bytes en absoluto.

### Descifrar el miembro del balance a mano

La entrada de la estructura para `account`comienza en el offset `0x195`con los bytes `0c 15 20 00 03`El código abreviado es 12, la estructura.`DW_AT_name`es el índice de cadenas `0x15`¿ Qué ?`DW_AT_byte_size`¿ Es verdad ?`0x20`= 32, `DW_AT_decl_file`es 0, y `DW_AT_decl_line`es 3, que es la línea de `struct account {`en la fuente.

La segunda entrada de miembros comienza en `0x1a3`con`0d 10 88 01 00 00 00 05 04`¿Qué es esto ?

1. `0d`es la abreviatura 13, un miembro.
2. `10`¿ Es verdad ?`DW_AT_name`¿ Cómo ?`strx1`: índice de cuerdas 16.
3. `88 01 00 00`¿ Es verdad ?`DW_AT_type`¿ Cómo ?`ref4`: compensación `0x188`desde el inicio de la unidad, que es el `DW_TAG_base_type`Entrada nombrada `int`¿ Qué ?
4. `00`¿ Es verdad ?`DW_AT_decl_file`y `05`¿ Es verdad ?`DW_AT_decl_line`: línea 5 de `account.c`¿ Qué ?
5. `04`¿ Es verdad ?`DW_AT_data_member_location`: compensación 4.

En la sección 5.7.6 se dice que una entrada miembro "puede tener un atributo DW_AT_data_member_location o un atributo DW_AT_data_bit_offset" y que una constante entera "es el desplazamiento en bytes desde el principio de la entidad que contiene". Ese byte es el mismo número que el desplazamiento en `addl 0x4(%rax), %ecx`, y `compare_layouts()`es la función que pone los dos lados a la vez.

### Las cadenas y direcciones pasan a través de dos tablas

El índice de cuerdas 16 no es un descenso en `.debug_str`El lector toma`DW_AT_str_offsets_base`de la entrada de unidad, que es 8, y lee la entrada de 4 bytes en `8 + 4 * 16`En el`.debug_str_offsets`Esa entrada contiene el descenso de`"balance"`En el`.debug_str`La misma indirección se aplica a las direcciones.`sum_balances`comienza con los bytes `07 07`: abreviatura 7, un subprograma, entonces `DW_AT_low_pc`¿ Cómo ?`addrx`Indice 7. La entrada de 8 bytes en `8 + 8 * 7`En el`.debug_addr`¿ Es verdad ?`0x30`, la dirección de la función en `.text`Las primeras seis entradas de esa tabla son las direcciones de las literalmente de seis cuerdas.

En un objeto relocable ninguna tabla tiene sus valores finales.`.debug_str_offsets`es cero en el archivo, y `.rela.debug_str_offsets`tiene 29 `R_X86_64_32`las reubicaciones contra `.debug_str`con el real de compensación en el adjunto. `.debug_addr`tiene 13 `R_X86_64_64`las reubicaciones contra `.text`y `.rodata.str1.1`- ¿ Qué ?`.debug_info`tiene cuatro reubicaciones propias: la abreviatura de compensación, `DW_AT_str_offsets_base`¿ Cómo ?`.debug_str_offsets+0x8`¿ Qué ?`DW_AT_stmt_list`, y `DW_AT_addr_base`¿ Cómo ?`.debug_addr+0x8`- ¿ Qué ?`ElfObject.relocated()`se aplica `S + A`Para los tipos de 32 bits y 64 bits antes de que el lector analice una sección. Un símbolo de sección tiene valor 0 en un archivo objeto, por lo que el campo recibe el agregado. `llvm-dwarfdump`El índice de resolución de la información de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de datos de la fuente de referencia de la fuente de datos de la fuente de referencia de la fuente de referencia de la fuente de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de referencia de la fuente de referencia de la fuente de referencia de referencia de la fuente de referencia de la fuente de referencia de la fuente de referencia de la fuente de referencia de la fuente de referencia de referencia de la fuente de referencia de la fuente de referencia de la fuente de la fuente de referencia de la fuente de referencia de referencia de la fuente de la fuente de referencia de la fuente de referencia de la fuente de referencia de la fuente de la fuente de referencia de la fuente de referencia de la fuente de la fuente de referencia de la fuente de referencia de la fuente de referencia de la fuente de la fuente de referencia de la fuente de la fuente de referencia de la fuente de la fuente de la fuente de referencia de la fuente de referencia de la fuente de la fuente de referencia de la fuente de la fuente de la fuente de la fuente de referencia de la fuente de referencia de la fuente de la fuente de referencia de la fuente de la fuente de la fuente de referencia de la fuente de la fuente de la fuente de la fuente de referencia de la fuente de la fuente de la fuente de referencia de la fuente de la fuente de referencia de la fuente de la fuente de la fuente de la fuente de referencia de la fuente de la

### Tipos, funciones y ubicaciones

`DwarfInfo.type_name()`sigue`DW_AT_type`Las referencias.`label`puntos de los miembros en un `DW_TAG_pointer_type`, que apunta a una `DW_TAG_const_type`, que señalan la`char`tipo base, por lo que el nombre se compone de `const char *`- ¿ Qué ?`type_size()`retorno `DW_AT_byte_size`para tipos y estructuras de base, 8 para un puntero, y contar veces el tamaño de un elemento para una matriz. `type_kind()`reduce el `DW_AT_encoding`de un tipo base a `int`con un signo, o `float`, y un tipo de puntero para `pointer`, que es el vocabulario que utiliza la comparación.

`DwarfInfo.functions()`Se lee seis .`DW_TAG_subprogram`Las entradas.`bump`se ejecuta desde `0x0`¿ Qué ?`0x21`, que es `DW_AT_low_pc`más el `DW_AT_high_pc`Su base de marco es`DW_OP_reg6 RBP`, y su parámetro `a`tiene la expresión de ubicación `91 78`¿ Qué es esto ?`DW_OP_fbreg`con el valor SLEB128 `0x78`, que es -8. Ese es el espacio del marco `-0x8(%rbp)`en el que se almacena la lista `%rdi`en la dirección `0x4`La información de descomposición y las instrucciones describen la misma ranura desde dos lados.

## Comparar las dos estructuras

`compare_layouts()`El miembro que no tiene campo es `no-evidence`Un campo de un tamaño diferente es`size-mismatch`Un miembro de puntero cuyo campo nunca fue desreferenciado es`pointer-unproven`. Un tipo que no esté de acuerdo, como un float donde las instrucciones utilizan aritmética de números enteros, es `kind-mismatch`. Los campos inferidos que comienzan dentro de un miembro o dentro de los relleno son `unexpected`El tamaño compara el paso con`DW_AT_byte_size`¿ Qué ?

Los números de las tres construcciones, desde `python3 code/main.py`¿Qué es esto ?

| Build | Accesses through the first parameter | Fields | Stride evidence | Members matched | Size |
|---|---|---|---|---|---|
| `-O0` | 11 | 5 | `scale 32`, `call-site 32`, `call-site 64` | 5 of 5 | 32, match |
| `-O1` | 9 | 5 | `loop-step 32` | 5 of 5 | 32, match |
| `-O2` | 17 | 5 after folding, 12 before | `offset-delta 32`, `unrolled 32`, `loop-step 256`, `scale 32`, `loop-step 32` | 5 of 5 | 32, match |

Los huecos inferidos son bytes de 1 a 3 y 10 a 15 en cada construcción, y esos son exactamente los bytes de relleno del diseño DWARF. Cada campo inferido se encuentra en un múltiplo de su propio tamaño, que es lo que requieren las reglas de psABI, y `natural_alignment_ok()`- Eso lo registra.

Añadir las funciones una a la vez y la imagen cambia.`bump`solo el diseño es un campo en el offset 4, el tamaño es desconocido, y el límite inferior es 8. `sum_balances`añade el paso. `tag`añade los desventajas 0 y 8. `projected`añade el flotador a 16. `label_length`añade el puntero a 24 y demuestra que es un puntero. `main`añade los sitios de llamadas en `-O0`y no añade nada a `-O1`y `-O2`La figura pasa por este orden para cada construcción.

Tres cosas permanecen fuera de alcance.`const`No está en las instrucciones.`label`es sólo "elementos 1 byte aparte" de la `leaq 0x1(%rcx), %rcx`en el bucle de cuerdas, que es consistente con `char`y con `unsigned char`y `_Bool`- En el`-O1`El signo del byte en el desplazamiento 0 es desconocido, porque `cmpb $0x6c, (%rdi)`Las pruebas de aprobación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de evaluación de los resultados de los estudios de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de la investigación de los Estados Unidos`int8`allí y`i8`En el`-O0`, donde `movsbl`Estas son las lecciones de hechos 35 registros como desconocidos.

## Objetos de C++

`shapes.cpp`define `Shape`con un destructor virtual, un puro virtual `area()`, y un virtual `name()`, y `Rect`, que se deriva de `Shape`y añade dos `int`El Itanium C++ ABI rige cómo Clang establece estos en x86-64 Linux.

### Nombres equivocados

La sección 5.1.2 del ABI dice que "un nombre se corrompe con prefijos `_Z`El objeto tiene 19 símbolos de este tipo.`_ZNK4Rect4areaEv`Se lee como `_Z`, entonces`N`para un nombre anidado, `K`para una función de miembro const, `4Rect`y `4area`como nombres de fuentes con prefijo de longitud, `E`para cerrar el nombre anidado, y `v`para una lista de parámetros vacíos. `llvm-cxxfilt`huellas`Rect::area() const`- ¿ Qué ?`_Z8describeRK5Shapei`¿ Es verdad ?`describe(Shape const&, int)`¿ Qué es esto ?`R`es una referencia, `K`es const, `5Shape`es la clase,`i`¿ Es verdad ?`int`Los nombres especiales en la sección 5.1.4 son:`_ZTV`para una mesa de trabajo, `_ZTI`para información de tipo, y `_ZTS`para la cadena de nombres de tipoinfo.`C1`¿ Qué ?`C2`¿ Qué ?`D0`¿ Qué ?`D1`, y `D2`, así que`_ZN4RectD0Ev`y `_ZN4RectD2Ev`ambas impresas como `Rect::~Rect()`: el destructor de eliminación y el destructor de objeto base comparten un nombre de fuente.

`Demangler`En el`code/main.py`cubre ese subconjunto más los códigos de tipo de construcción de la sección 5.1.5.2 y el `S_`Las pruebas se realizan en los 19 nombres y se comparan con los capturados.`llvm-cxxfilt --no-strip-underscore`La bandera es importante en macOS, donde la herramienta elimina un subtítulo principal por defecto y luego no reconoce `Z8describe...`¿ Qué ?

### El vtable de sus reubicaciones

La sección 2.5.2 de la ABI enumera los componentes de una tabla virtual en orden: offsets de llamada virtual, offsets de base virtual, offset a arriba, el puntero typeinfo y luego los punteros de función virtual. El punto de dirección, el valor que tiene vptr de un objeto, es la dirección del primer puntero de función (sección 2.6.2). `Rect`no tiene bases virtuales, por lo que su tabla se compensa a arriba, tipo de información y funciones.

El símbolo`_ZTV4Rect`En el objeto cada byte es cero. El contenido está en `.rela.data.rel.ro._ZTV4Rect`¿Qué es esto ?

```text
0000000000000008 R_X86_64_64              _ZTI4Rect
0000000000000010 R_X86_64_64              _ZN4RectD2Ev
0000000000000018 R_X86_64_64              _ZN4RectD0Ev
0000000000000020 R_X86_64_64              _ZNK4Rect4areaEv
0000000000000028 R_X86_64_64              _ZNK4Rect4nameEv
```

La ranura 0 mantiene el desplazamiento hasta arriba, que es 0 y no necesita una reubicación.`+16`De ahí, la ranura 0 es el destructor de objetos completos, la ranura 1 es el destructor de borrado, la ranura 2`area`, apartado 3 `name`- ¿ Qué ?`read_vtable()`se lee el tamaño del símbolo, las reubicaciones en su rango y el cero en el desplazamiento 0. `Shape`La mesa tiene la misma forma que la de`__cxa_pure_virtual`en la ranura 2, que es la función de tiempo de ejecución que una llamada virtual pura alcanza.

### RTTI

En la sección 2.9.4 se describe `std::type_info`como un vptr seguido de un indicador al nombre de tipo de descomposición, y enumera las clases derivadas. `_ZTI4Rect`tiene tres traslados: `_ZTVN10__cxxabiv120__si_class_type_infoE+0x10`¿ Qué ?`_ZTS4Rect`, y `_ZTI5Shape`El primero es el vtable de`__si_class_type_info`, la clase que utiliza el ABI para un tipo con una única base pública no virtual en el punto de compensación cero, y el `+0x10`Es el punto de dirección de esa tabla.`4Rect`, que es el nombre del tipo de mangled, y `name()`en un `type_info`La tercera es la base.`_ZTI5Shape`usos `__class_type_info`y no tiene puntero de base. `read_typeinfo()`devuelve estos tres hechos.

### La llamada virtual

`describe`En el`-O0`hace una llamada virtual:

```text
       f: 48 8b 7d f8                  	movq	-0x8(%rbp), %rdi
      13: 48 8b 07                     	movq	(%rdi), %rax
      16: ff 50 10                     	callq	*0x10(%rax)
```

El primero .`movq`La segunda lectura de 8 bytes en el desplazamiento 0 del objeto, que es el vptr, porque el ABI lo coloca primero en una clase con funciones virtuales.`callq`pasa por aquí .`0x10`pasado el VPT, y `0x10 / 8`es la ranura 2.`find_virtual_calls()`coincide con el patrón y informa a ambos candidatos: `Rect::area() const`de la`_ZTV4Rect`y `__cxa_pure_virtual`de la`_ZTV5Shape`El que se ejecuta depende del objeto, por lo que la llamada es indirecta.

DWARF registra la misma ranura.`DW_TAG_subprogram`las entradas para `area`en ambas clases de transporte`DW_AT_vtable_elem_location (DW_OP_constu 0x2)`¿ Qué ?`name`lleva la ranura 3, y los destructores llevan la ranura 0. `Rect`es un `DW_TAG_structure_type`de 16 bytes con una `DW_TAG_inheritance`Entrada para `Shape`en compensación 0 y miembros `w`a las 8 y `h`A las 12 .`Shape`es 8 bytes con un miembro artificial `_vptr$Shape`Dos derivadas independientes dan la ranura 2 para `area`: la tabla de reubicación y las entradas de desactivación.

En el`-O1`El objeto no tiene nada de vtable. `main`construcciones `Rect`como local, así que el compilador resolvió `r.name()`y `describe(r, 2)`No se puede tomar la dirección de la tabla, por lo que la tabla no se emite.`describe`todavía contiene `movq (%rdi), %rax`y `callq *0x10(%rax)`, porque su parámetro puede ser cualquier`Shape`. Una tabla v que falta es un hecho sobre un archivo de objeto, y la prueba `test_the_optimised_object_keeps_the_call_pattern_but_drops_the_vtables`- ¿Qué? - Comprueba ambas mitades.

El ABI de Microsoft difiere en cada uno de estos detalles.`?`y codificar la convención de llamada, así que `void test(void)`Compilado como C++ `__cdecl`¿ Es verdad ?`?test@@ZAXXZ`El localizador RTTI se encuentra en `vfptr[-1]`como un `_RTTICompleteObjectLocator`, y los símbolos son`??_7`para un vftable y `??_R4`La documentación de Microsoft dice que "no hay estándares para nombrar en C++ entre los proveedores de compiladores".

## Cómo las herramientas presentan los tipos recuperados

Cada fila que tiene un nombre de salida fue producida en esta máquina y capturada en`code/fixtures/tools/`Las filas para las herramientas que no están instaladas aquí describen su documentación y lo dicen.

| Tool | Version, licence | Where types live | On our fixtures |
|---|---|---|---|
| Ghidra | 12.1.4, 2026-09-21, Apache-2.0 | Data Type Manager with built-in, user-defined, and derived types; `.gdt` archives; types used by a program are copied into it | With DWARF: `/DWARF/account.c/account size=32 align=8` and `a->balance = by + a->balance;`. Without: `*(int *)(param_1 + 4)` with `long param_1`, `tag(char *param_1)`, and `*(char **)(param_1 + 0x18)` |
| IDA Pro | 9.4, 2026-07-13, commercial; IDA Free is non-commercial with a cloud decompiler for x86 only | Local Types window, `.til` type libraries, DWARF and PDB import into a `dwarf` or `pdb` folder | Not installed. Documentation only |
| Binary Ninja | 6.0, 2026-09-03, commercial with a Free edition | Type editing, header import and export, type archives, debug info from DWARF and PDB, type libraries, platform types | Not installed. Documentation only |
| radare2 | 6.2.4, 2026-10-05, LGPLv3 | `ts`, `tsc`, `afvd` | `ts` and `tsc account` printed nothing for the relocatable object with `bin.dbginfo=true`; `pdf` shows `mov dword [rax + 4], ecx` with source line comments from DWARF |
| rizin | 0.9.1, 2026-06-29, LGPL core | `ts`, `tsc`, `afvd` | `tsc account` printed `struct account { char kind; int balance; short code; double rate; const char *label; };` from DWARF; the build without `-g` lists only its bundled header types |
| lldb | lldb-1703.0.236.21 | `image lookup -t` | `byte-size = 32` and the full `compiler_type` text from the ELF object |
| gdb | not installed here | `ptype /o` prints "the offsets and sizes of fields in a struct" with holes marked | Documentation only |
| pyelftools | 0.33, public domain | Python API over `.debug_info` | A 43-line script prints the same five members and `unit version 5` |
| llvm-dwarfdump | Apple LLVM 17.0.0 | `--debug-info`, `--debug-abbrev`, `--debug-str-offsets` | The oracle for every reader test |

La salida de Ghidra sin DWARF muestra las mismas dos reglas en funcionamiento.`char *`al parámetro de `tag`porque la función compara un byte a través de ella, y `char **`al campo en `0x18`Porque ...`label_length`Desreferencia el valor cargado y compara los bytes. Esas son las mismas dos reglas que el paso en esta lección aplica: ancho de la instrucción, puntero de la dereferencia. La documentación del decompiler de Ghidra dice que los varnodos de código p "no tienen necesariamente un tipo de datos asociado con ellos" y que el decompiler "al final asigna un tipo de datos formal" por propagación. La opción "Inferir indicadores constantes" extiende la misma propagación a las constantes. Ghidra 12.1 añadió la recuperación de bitfield al descompilador, que las fichas de esta lección no ejercen.

La tabla hecha a mano es la otra mitad de la historia. Ghidra's Data Type Manager, IDA's Local Types, Binary Ninja's type archives, y rizin's `td`El pase te dice qué escribir y qué campos estás adivinando.

rea 6.5.0 con su proveedor Ghidra devuelve el texto del descompilador como un registro. Todo lo que está en la fila Ghidra se aplica a él, y el registro lleva los nombres de tipos sin la evidencia que los produjo.`python3 code/main.py infer`en la misma lista le da esa evidencia.

## Investigación de la interferencia de tipo

El paso aquí es la mitad de generación de restricciones de un sistema de inferencia de tipo, restringido a un estruto alcanzado desde un parámetro.

- TIE (Lee, Avgerinos, Brumley, NDSS 2011) enmarcó la reconstrucción de tipo como generación de restricciones y resolución sobre una IR binaria, midió la conservaduría y precisión, y comparó con un descompilador comercial en 87 programas de núcleos.
- Retypd (Noonan, Loginov, Cok, PLDI 2016) añadió subtipo, polimorfismo y tipos recursivos, y informó reconstrucción del puntero `const`Una implementación de código abierto existe bajo el nombre de GrammaTech.
- OSPREY (Zhang y otros, IEEE S&P 2021) trata cada ubicación de memoria como una variable aleatoria e inferen los tipos más probables.
- DIRTY (Chen y otros, USENIX Security 2022) y ReSym (Xie y otros, CCS 2024) producen descompiladores postproceso con modelos aprendidos, y ReSym informa con una precisión del 72,9% en diseños estructurales completos.
- RecStruct (Chen y otros, USENIX Security 2026) recupera estructos anidados sin un descompilador, con ejecución simbólica insensible al flujo y unión-encuentra sobre ranuras de pila.

La estructura de la pila es lo primero que salta el pase de esta lección.`-O1`¿ Qué ?`main`¿ Qué es ?`rows`en la pila y el código en línea dice `0x24(%rsp)`¿ Qué ?`0x40(%rsp)`¿ Qué ?`0x48(%rsp)`¿ Qué ?`0x30(%rsp)`, y `0x38(%rsp)`Los modulos 32 son 4, 0, 8, 16 y 24: la misma estructura, sin parámetros para sembrar.

## Construye el mismo

Ejecutar la demostración:

```bash
cd phases/13-tools-and-protocols/41-recovering-data-and-types
python3 code/main.py
```

La salida es un informe JSON con cinco partes. `dwarf`El título de la unidad, las estructuras y las funciones de la`-O0`Objeto.`oracles`tiene el diseño de las reglas de psABI, el `ctypes`La distribución, la distribución DWARF, la ejecución nativa y un `agree`bandera.`builds`contiene los accesos, la disposición inferida y la comparación para cada uno de los tres objetos. `cxx`contiene los nombres desfigurados, ambos vtables, ambos registros de tipoinfo, la llamada virtual y las ranuras DWARF. `tools`enumera las salidas de la herramienta capturadas.

Forma esperada:

```json
{
  "dwarf": {"unit": {"unitLength": 489, "version": 5, "unitType": "DW_UT_compile", "addressSize": 8, "abbrevOffset": 0}, "...": "..."},
  "oracles": {"agree": true, "psabiRules": {"size": 32, "align": 8, "...": "..."}, "...": "..."},
  "builds": {
    "O0": {"layout": {"stride": 32, "size": 32, "gaps": [1, 2, 3, 10, 11, 12, 13, 14, 15], "...": "..."}, "comparison": {"matches": 5, "members": 5, "...": "..."}},
    "O1": {"...": "..."},
    "O2": {"...": "..."}
  },
  "cxx": {"virtualCalls": [{"slot": 2, "candidates": {"Shape": "__cxa_pure_virtual", "Rect": "_ZNK4Rect4areaEv"}, "...": "..."}], "...": "..."}
}
```

Leer .`code/main.py`en este orden:

1. `ElfObject`Se leían secciones, símbolos y reubicaciones, y `relocated()`se aplica `S + A`en una sección de descomposición.
2. `parse_abbreviations()`y `DwarfInfo._parse_unit()`Lea el encabezado, la tabla de abreviaturas y las entradas, luego resuelva `strx`y `addrx`a través de las dos mesas.
3. `DwarfInfo.struct_layouts()`y `functions()`convertir las entradas en `StructLayout`y `FunctionInfo`registros, con `type_name()`¿ Qué ?`type_size()`, y `type_kind()`¿ Qué ?
4. `parse_listing()`¿ Qué ?`parse_memory()`, y `access_width()`Leer el `llvm-objdump`¿Qué es eso?
5. `trace_function()`es la máquina del estado abstracto, y `fold_unrolled()`y `infer_layout()`convertir los accesos en un diseño.
6. `compare_layouts()`produce las filas de estado, y `sysv_layout()`y `ctypes_layout()`son los otros dos oráculos.
7. `Demangler`¿ Qué ?`read_vtable()`¿ Qué ?`read_typeinfo()`, y `find_virtual_calls()`cubrir el objeto C++.

Hay cuatro órdenes más.`python3 code/main.py dwarf FILE.o`Imprime las estructuras y funciones de cualquier objeto relocable de DWARF 5. `python3 code/main.py infer LISTING.txt [FILE.o]`ejecuta el pase en una lista y, con un objeto, compara con su DWARF. `python3 code/main.py demangle NAME...`y `python3 code/main.py vtable FILE.o`cubrir la mitad de C ++.

## Usalo

En un nuevo objeto que construiste tú mismo:

1. Captura .`llvm-objdump -d -r`El pase lee ese texto.
2. - ¿ Qué ?`python3 code/main.py infer`En el listado. Leer primero la línea de testigos: nombra las funciones cuyo primer parámetro se utilizó como base de memoria. Si una función falta, su parámetro nunca alcanzó un operando de memoria, o llegó a uno a través de un camino que el paso no rastrea.
3. Lea cada campo con su número de acceso. Un acceso es un testigo.
4. Lea la evidencia de los pasos.`call-site`o `scale`La entrada es fuerte.`loop-step`La entrada sin otro tipo es débil, porque un bucle desarrollado informa el paso de varios elementos.`unrolled`aparece, compruebe el pliegue a mano.
5. Si un`-g`construir existe, ejecutar la comparación y leer cada fila que no es `match`- ¿ Qué ?`no-evidence`fila nombra un miembro no se ha tocado función.`unexpected`nombre de filas a un acceso que el pase no pudo colocar.
6. Escriba el diseño en el administrador de tipos de su herramienta como un estructurado, y registre qué campos, nombres y tipos son suposiciones.

Mantenga los tres oráculos a su alcance.`ctypes`, y DWARF no está de acuerdo en un diseño que usted escribió, usted ha encontrado un estructurado, un campo de bits, o un objetivo con reglas diferentes.

## Laboratorio interactivo

### Laboratorio A: decodificar la entrada de miembro

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/41-recovering-data-and-types/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
elf = load_object("account-x86_64-O0-g.o")
dwarf = DwarfInfo(elf)
dwarf.header()
elf.section(".debug_info").data[0x1a3:0x1ac].hex(" ")
dwarf.dies[0x1a3].attributes
dwarf.type_name(dwarf.type_of(dwarf.dies[0x1a3]))
```

Decodifica los nueve bytes en papel con la abreviatura 13 antes de mirar el diccionario.`elf.section(".debug_str_offsets").data[8:16].hex()`, y el trasladado,`elf.relocated(".debug_str_offsets")[8:16].hex()`, y explicar la diferencia.

### Laboratorio B: rastrear una función

```python
listing = load_listing("account-x86_64-O0-g.objdump.txt")
trace = trace_function("label_length", listing["label_length"])
[(hex(a.address), a.root, a.offset, a.width, a.kind) for a in trace.accesses]
trace.pointer_uses
trace_function("sum_balances", listing["sum_balances"]).loops
```

Escriba, antes de ejecutar, que acceso prueba que el offset 24 es un puntero.`-O1`lista y encontrar el `leaq 0x1(%rcx), %rcx`En el`trace.increments`¿ Qué ?

### Laboratorio C: doblar el bucle vectorial

```python
o2 = load_listing("account-x86_64-O2-g.objdump.txt")
trace = trace_function("sum_balances", o2["sum_balances"])
sorted({a.offset for a in trace.accesses})
folded, evidence = fold_unrolled(trace, trace.accesses)
sorted({a.offset for a in folded}), [(e.kind, e.value, e.text) for e in evidence]
```

Entonces corre .`analyze_build("O2", fold=False)["comparison"]`y contar los `unexpected`Cambiar la regla de plegado para que requiere que el paso del bucle sea exactamente`k * d`con`k`al menos 2, y confirmen que el `-O1`el bucle, con un acceso y el paso 32, se deja solo.

### Laboratorio D: el tablero y los nombres

```python
shapes = load_object("shapes-x86_64-O0-g.o")
read_vtable(shapes, "_ZTV4Rect")
read_typeinfo(shapes, "_ZTI4Rect")
demangle("_ZNK4Rect4areaEv"), demangle("_Z8describeRK5Shapei")
find_virtual_calls(load_listing("shapes-x86_64-O0-g.objdump.txt"), [read_vtable(shapes, "_ZTV5Shape"), read_vtable(shapes, "_ZTV4Rect")])
```

Calcule la ranura desde `callq *0x10(%rax)`en papel, luego confirme que contra`DwarfInfo(shapes).struct("Rect").virtual_methods`Entonces cargue .`shapes-x86_64-O1-g.o`y tratar`read_vtable`Leer el libro.`ElfError`y explicar por qué el símbolo está ausente.

## Laboratorio de práctica

Recupera la estructura de la pila en `main`En el`-O1`¿ Qué ?

Requisitos:

- Extenderse`trace_function()`Así que una región de ranura de marco puede ser una raíz.`movaps`Tiendas que llenan`0x0(%rsp)`¿ Qué ?`0x5f(%rsp)`de la`.data.rel.ro`, que es el inicializador de 96 bytes de `rows`¿ Qué ?
- Atribuye los accesos en línea `0x24(%rsp)`¿ Qué ?`0x40(%rsp)`¿ Qué ?`0x48(%rsp)`¿ Qué ?`0x30(%rsp)`, y `0x38(%rsp)`y reportar sus compensaciones modulo un candidato paso.
- Derivar el paso 32 del bucle en `0x120`¿ Qué ?`0x12c`, donde `%r14`comienza a las 4 y avanza a las `0x20`hasta que`0x64`, y mostrar que el bucle se une`0x64`es tres elementos más el campo de compensación.
- Comparar el resultado con el diseño de DWARF y alcanzar cinco coincidencias, y registrar qué tipos permanecen no probados.
- Añadir una prueba que falla cuando la raíz del marco se atribuye a un parámetro.

El éxito significa`python3 code/main.py infer`en el `-O1`listas de listados `main`como un testigo con una raíz de marco, y la comparación muestra cinco coincidencias de`main`¿Por qué no?

## Lección Artefacto

Esta lección incluye `outputs/skill-recovering-data-and-types.md`. Utilice para convertir una lista y una configuración opcional de defecto en un diseño estructural con evidencia por campo, un paso con sus fuentes, una comparación con DWARF y una lista de lo que permaneció desconocido. Requiere la tabla de acceso, la prueba de paso, la verificación de plegado, el oráculo DWARF cuando existe, y una declaración de nombres y calificadores que ninguna instrucción puede proporcionar.

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/41-recovering-data-and-types
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

La verificación deberá demostrar:

- El título de la unidad, las 14 abreviaturas y los cinco elementos de compensación están de acuerdo con `llvm-dwarfdump`
- Los nombres llegan a través de los relocalizados `.debug_str_offsets`mesa y la unidad `str_offsets_base`y `addr_base`son 8
- Seis funciones llevan los rangos, `DW_OP_reg6 RBP`, y `DW_OP_fbreg`las ubicaciones que muestra el vertedero, y sus direcciones bajas son iguales a la función de la lista comienza
- un objeto sin`.debug_info`eleva, y el decodificador LEB128 maneja `0x78`como -8
- `pyelftools`y `lldb`imprime el mismo diseño que el lector
- las reglas de psABI, `ctypes`, DWARF, y la carrera nativa dan compensaciones `0, 4, 8, 16, 24`y tamaño 32, y el estruct reordenado es de 24 bytes
- el parser de la lista lee 13 instrucciones en `bump`, adjuntos .`R_X86_64_PLT32 bump-0x4`a la llamada en `0x194`, y encuentra el salto hacia atrás en `0x72`
- operandos de memoria, anchos, tipos y signos decodificados del texto AT&T y el mnemónico
- `bump`En el`-O0`lee su puntero hacia atrás desde la ranura del marco y hace tres accesos en el despacho 4
- El paso viene de `shlq $0x5`En el`-O0`, desde `addq $0x20, %rdx`En el`-O1`, y de ocho cargas dobladas por un paso de 256 en `-O2`
- `label_length`prueba el puntero en el desplazamiento 24 desreferenciándolo
- `main`pasa un objeto de marco en compensaciones 0, `0x20`, y `0x40`En el`-O0`y no llama nada más que`printf`En el`-O1`
- Cada construcción coincide con cinco de los cinco miembros con tamaño 32 y huecos iguales al relleno DWARF
- sin doblar el `-O2`crear informes siete campos más allá de byte 32 y no tamaño
- El desangler reproduce todos los 19 `llvm-cxxfilt`cuerdas
- tanto las tablas de vtables, tanto los registros de tipoinfo, y la llamada virtual en `0x16`Resolver a la ranura 2, y DWARF está de acuerdo
- el `-O1`Objeto C++ mantiene el patrón de llamada y no tiene vtable
- Las capturas de Ghidra, rizin y radar2 muestran la estructura con DWARF y los desplazamientos crudos sin ella
- La figura interactiva incorpora los mismos datos que el análisis produce

## Modos de falla de producción

| Failure | What the weak tool reports | What the analysis must prove |
|---|---|---|
| Unrolled loop | Eight fields 32 bytes apart and a 232-byte struct | Loop step 256 equals 8 times the offset delta 32, so one field at offset 4 |
| Stride read from one increment | `sizeof` 256 from `addq $0x100` | The greatest common divisor of every stride source, with the scalar tail's `addq $0x20` |
| 8-byte field called a pointer | `void *` at offset 24 | A later access uses the loaded value as a base, or the kind stays `int64` |
| Pointer called an integer | `long` at offset 24 | The dereference at `0xfb`, recorded as pointer proof |
| Narrow compare | `char` with a sign | `cmpb` carries no sign; only `movsbl` or `movzbl` does |
| Five functions assumed to share a type | One struct with no evidence of aliasing | Call sites in `main` that pass one frame object, or an explicit analyst assertion |
| Strings read from `.debug_str` by index | Garbage names from a relocatable object | `str_offsets_base` plus the relocated `.debug_str_offsets` entry |
| Addresses read as numbers | `low_pc` of 0 for every function | `addrx` through the relocated `.debug_addr` table |
| DWARF 4 object | A reader that misparses the header | Version check on the second header field; 4 has no `unit_type` byte |
| Missing vtable | "the class has no virtual functions" | The `-O1` object inlined every call; the `-O0` object has the table and its relocations |
| Vtable slot read from the wrong base | Off-by-two slot numbers | The address point is 16 bytes past the symbol, after offset-to-top and typeinfo |
| Mangled name stripped | `Z8describeRK5Shapei` left as is | `--no-strip-underscore` on macOS, or a demangler that checks for `_Z` |
| Struct typed from one tool's guess | `char *param_1` taken as the field type | The width and the dereference, written as evidence, and the name marked as a guess |

## Regla de funcionamiento

Recupere un campo de un acceso y un paso de un bucle, y adjunta la instrucción a cada uno. Doble los bucles desarrollados antes de confiar en un desplazamiento que pase el paso. Lea DWARF a través de sus abreviaturas y sus tablas reubicadas, nunca indexando en una sección de cadena. Comparar campo por campo y escribir cada miembro sin evidencia, cada puntero no probado, y cada nombre como desconocido. En un objeto de C ++, lea la tabla de sus reubicaciones y la ranura de la llamada, luego use DWARF para confirmar la ranura que ha leído.

## Leer más

- [System V Application Binary Interface, AMD64 Architecture Processor Supplement, section 3.1.2: Data Representation](https://gitlab.com/x86-psABIs/x86-64-ABI)
- [Procedure Call Standard for the Arm 64-bit Architecture (AAPCS64), section 5.10: Composite Types](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst)
- [Writing ARM64 code for Apple platforms](https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms)
- [Intel 64 and IA-32 Architectures Software Developer's Manual, Volume 1, section 3.7.5: Specifying an Offset](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [DWARF Debugging Information Format Version 5, sections 2.1, 5.7.6, 7.5.1.1, 7.5.3, 7.5.6](https://dwarfstd.org/doc/DWARF5.pdf)
- [Itanium C++ ABI, sections 2.5 (Virtual Table Layout), 2.9 (Run-Time Type Information), 5.1 (External Names)](https://itanium-cxx-abi.github.io/cxx-abi/abi.html)
- [Microsoft Learn: Decorated names](https://learn.microsoft.com/en-us/cpp/build/reference/decorated-names?view=msvc-170)y [Name Decoration](https://learn.microsoft.com/en-us/cpp/error-messages/tool-errors/name-decoration?view=msvc-170)
- [llvm-dwarfdump command guide](https://llvm.org/docs/CommandGuide/llvm-dwarfdump.html)y [llvm-cxxfilt command guide](https://llvm.org/docs/CommandGuide/llvm-cxxfilt.html)
- [Python ctypes: Structure layout, sizeof, alignment](https://docs.python.org/3/library/ctypes.html)
- [Ghidra Data Type Manager](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/Base/src/main/help/help/topics/DataTypeManagerPlugin/data_type_manager_description.htm)¿ Qué ?[Decompiler Concepts](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/Decompiler/src/main/help/help/topics/DecompilePlugin/DecompilerConcepts.html)¿ Qué ?[Decompiler Options](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/Decompiler/src/main/help/help/topics/DecompilePlugin/DecompilerOptions.html), y [Ghidra 12.1.4 release](https://github.com/NationalSecurityAgency/ghidra/releases/tag/Ghidra_12.1.4_build)
- [IDA 9.1 user guide: Types](https://docs.hex-rays.com/9.1/user-guide/types)¿ Qué ?[IDA 9.4 release post](https://hex-rays.com/blog/ida-9.4-release-a-new-dyld-shared-cache-swift-analysis-new-teams-add-on-and-more-), y [IDA Free](https://hex-rays.com/ida-free)
- [Binary Ninja user guide: Types](https://docs.binary.ninja/guide/types/)y [Binary Ninja 6.0 Krypton](https://binary.ninja/2026/09/03/binary-ninja-6.0-krypton.html)
- [radare2 6.2.4 release](https://github.com/radareorg/radare2/releases/tag/6.2.4)y [rizin v0.9.1 release](https://github.com/rizinorg/rizin/releases/tag/v0.9.1)
- [GDB manual: Examining the Symbol Table, ptype /o](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Symbols.html)
- [pyelftools on PyPI](https://pypi.org/project/pyelftools/)
- [Lee, Avgerinos, Brumley, "TIE: Principled Reverse Engineering of Types in Binary Programs" (NDSS 2011)](https://www.ndss-symposium.org/ndss2011/tie-principled-reverse-engineering-of-types-in-binary-programs/)
- [Noonan, Loginov, Cok, "Polymorphic Type Inference for Machine Code" (PLDI 2016)](https://arxiv.org/abs/1603.05495)y [the retypd repository](https://github.com/GrammaTech/retypd)
- [Zhang et al., "OSPREY: Recovery of Variable and Data Structure via Probabilistic Analysis for Stripped Binary" (IEEE S&P 2021)](https://ieeexplore.ieee.org/document/9519451/)
- [Chen et al., "Augmenting Decompiler Output with Learned Variable Names and Types" (USENIX Security 2022)](https://www.usenix.org/conference/usenixsecurity22/presentation/chen-qibin)
- [Xie et al., "ReSym: Harnessing LLMs to Recover Variable and Data Structure Symbols from Stripped Binaries" (CCS 2024)](https://dl.acm.org/doi/10.1145/3658644.3670340)
- [Chen et al., "RecStruct" (USENIX Security 2026)](https://www.usenix.org/conference/usenixsecurity26/presentation/chen-yuxin)
