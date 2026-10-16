# El paisaje de ingeniería inversa: historia, familias de herramientas y cómo se construye un descompiler

> Cinco herramientas leen el mismo objeto y dan cuatro direcciones, tres bloques contados, y dos respuestas para un interruptor, y esta lección construye el arnés que muestra cuál es cuál.

**Type:** Build
**Languages:** Python, C, x86-64 assembly, Java (one Ghidra script)
**Prerequisites:** Phase 13 · 32 (ELF from scratch), Phase 13 · 33 (disassembly and control flow), Phase 13 · 34 (lifting to pseudocode)
**Time:** ~150 minutes

## Objetivos de aprendizaje

- Coloque las principales herramientas del campo en una línea de tiempo datada de 1973 a 2026 y nombre a la familia a la que pertenece cada una.
- Nombre las ocho etapas de un decompiler y el término que Ghidra, Binary Ninja, Hex-Rays, radar2, rizin y angr usan para cada etapa.
- Levante una instrucción en ESIL, RzIL, P-code y VEX y explique cuáles de ellas escriben banderas con entusiasmo y cuáles almacenan un trozo.
- Reducir las direcciones de cinco herramientas a una `.text`la reducción de una función.
- Construye un arnés de comparación que lea las salidas de la herramienta capturadas y informa de acuerdo sobre los arranques de instrucciones, bloques, bordes y arranques de la función.
- Nombre, de la evidencia capturada, donde cada herramienta inventa código, pierde código, o se detiene en un salto indirecto.

## El problema

Pides a un agente que resuma lo que dicen tres herramientas de ingeniería inversa .`grade`En el`ledger-x86_64-O1.o`El agente ejecuta cada herramienta, lee tres listas y informa que la función se encuentra en `0x08000050`en una sola herramienta, `0x400010`en otro, y `0x100010`En el tercer, concluye que las herramientas no están de acuerdo sobre dónde está la función.

Las herramientas coinciden. El objeto es relocable y no tiene dirección de carga, por lo que cada herramienta elige una. Subtraer la dirección que cada herramienta ha dado `.text`y cada respuesta es `0x10`El agente comparó direcciones crudas porque nada le dijo lo que significaba la dirección en cada herramienta.

Más tarde el mismo agente lee`grade`en el `-O0`construye y informa bloques básicos 3, 8, 8 y 9 de cuatro herramientas.`jmp rax`Dos herramientas leen la tabla de saltos a través de sus reubicaciones. Una herramienta divide un bloque en el salto para colocar una etiqueta. Un conteo de bloques sin la razón detrás de él es un número sin reclamo.

Esta lección es el mapa que necesitas antes de confiar en cualquiera de esos números. Las lecciones 32 a 36 construyeron un decodificador, un elevador, un registro de evidencia y una reconstrucción. La segunda mitad de la pista los pone junto a las herramientas que la gente usa. Para ello se necesita la historia del campo, las familias en las que caen las herramientas, el pipeline que cada descompilador ejecuta, y un arnés que compara lo que las herramientas dicen sobre los objetos que usted mismo compila.

```figure
re-37-field-map
```

## Cincuenta y tres años en pasos fechados

El campo es más antiguo que la mayoría de sus herramientas.

### 1973 a 1991: primero los desmontables, luego los desmontables

Las primeras tesis de descompilación son de 1973. Hollander en Stanford y Housel en Purdue las escribieron, y ambas son conocidas hoy a través de citas.`adb`El tutorial en Bell Labs en 1977, y el desembolso enviado con UNIX V7 en 1979.`DEBUG`, con un desmonteador 8086, lanzado en PC-DOS 1.00 en agosto de 1981.

Los desmontables de lotes llegaron después. Sourcer, de V Communications en 1988, ejecutó múltiples pases sobre un programa DOS y escribió una lista de comentarios.`objdump`El primer artículo de la IDA fue escrito por Ilfak Guilfanov en enero de 1991 y publicado como software compartido el 20 de mayo de 1991.

### 1991 a 2009: desmontables interactivos y los primeros descompiladores

Cristina Cifuentes terminó "Técnicas de Compilación Reversa" en la QUT en julio de 1994.`dcc`, llevó i80286 programas DOS a C, y su trabajo de 1995 con Gough estableció las fases que todos los decompiladores posteriores guardaron. REC siguió en 1997 y Boomerang en 2002, el primer decompilador de código abierto, bajo una licencia BSD.

IDA se convirtió en un producto comercial en DataRescue en 1996, obtuvo un SDK de complementos en 1999, y se trasladó a Hex-Rays, fundada en 2005. El Hex-Rays Decompiler 1.0 fue lanzado el 17 de septiembre de 2007. En el mismo año Mike Van Emmerik presentó "Asignación única estática para la descompilación", la tesis que puso a SSA en el centro del flujo de datos de los descompiladores. La "Comparación estructural de objetos ejecutables" (2004) de Halvar Flake comenzó la familia de diferencias, y BinDiff salió a la venta en agosto de 2004.

radare comenzó en febrero de 2006 como un editor forense de hex con offsets de 64 bits. radare2, la reescritura modular bajo LGPL-3.0, fue lanzada por primera vez el 7 de marzo de 2009.

### 2009 a 2019: marcos y representaciones intermedias

LLDB fue anunciado como un subproyecto LLVM el 8 de junio de 2010. BAP, la plataforma de análisis binario con su lenguaje intermedio BIL, apareció en CAV 2011. Google adquirió zynamics, los fabricantes de BinDiff, el 1 de marzo de 2011. Hopper apareció en la Mac App Store a finales de 2011. Matt Godbolt comenzó Compiler Explorer en 2012 para que un programador de C++ pudiera ver el ensamblaje producido por un compilador.

Angr recibió su primera línea de código en agosto de 2013. El documento de Phoenix (Schwartz, Lee, Woo, Brumley, USENIX Security 2013) mostró estructuramiento que conserva la semántica con 40 `goto`DREAM (Yakdan y otros, NDSS 2015) eliminó el código de`goto`las declaraciones con estructuración independiente de patrones. Vector 35 fue fundada el 14 de enero de 2015, y Binary Ninja salió a la venta el 23 de julio de 2016 con su Low Level IL y una API Python. Angr fue lanzado públicamente en 2015 y ocupó el tercer lugar en el DARPA Cyber Grand Challenge en agosto de 2016. BinDiff se convirtió en gratuito en marzo de 2016. El Cutter 1.0 dio a radar2 una interfaz gráfica Qt en diciembre de 2017. En 2018, Guilfanov presentó "Decompiler internals: microcode" en Recon, la primera descripción pública del Hex-Rays IR.

### 2019 a 2026: descompileros abiertos y el giro de la investigación

Ghidra 9.0 fue lanzado por la NSA en la Conferencia RSA el 5 de marzo de 2019 bajo Apache-2.0, con fuente completa en GitHub un mes después. En el mismo año, el angr 8.19.2.4 añadió un decompiler preliminar. Binary Ninja 2.0 envió su decompiler de IL de alto nivel el 20 de mayo de 2020. Rizin fue anunciado como un tenedor de radar2 el 8 de diciembre de 2020, y el equipo de Cutter se trasladó a él. IDA Freeware 7.6 agregó un descompilador x64 en la nube en mayo de 2021.

BinDiff 8 fue lanzado con código abierto bajo Apache-2.0 el 25 de septiembre de 2023. SAILR (Basco y otros, USENIX Security 2024) se convirtió en el estructurador predeterminado de angr y argumentó que sólo es falso `goto`Las declaraciones deben ser eliminadas, porque Linux 6.1 tiene 3.754 de ellos en su fuente. Binary Ninja 4.0 añadió una edición gratuita en febrero de 2024, y IDA 9.0 añadió `idalib`Los trabajos de decompillador aprendidos se extienden desde SLaDe (2023) hasta LLM4Decompile (2024) hasta DecompileBench (2025) y dos estudios de 2026 sobre la legibilidad asistida por agentes y sobre decompiladores que recompilan más y conservan menos.

Las versiones que se ejecutaron en esta lección son las actuales el 14 de octubre de 2026: Ghidra 12.1.4 (21 de septiembre de 2026), radar2 6.2.4 (5 de octubre de 2026), rizin 0.9.1 (29 de junio de 2026), y angr 10.0.3 (10 de octubre de 2026). Binary Ninja 6.0 "Krypton" (3 de septiembre de 2026) e IDA 9.4 (13 de julio de 2026) son las herramientas cerradas actuales.

## Las familias de herramientas

Una herramienta pertenece a una familia por lo que produce. Las familias se superponen en un programa, y la superposición es donde comienza la mayor confusión.

### Desmontables

Un barrido lineal decodifica desde el primer byte de una sección de código hasta el último. GNU `objdump -d`y `llvm-objdump -d`Trabaja de esta manera, y ninguno construye un gráfico de flujo de control.`objdump`trata los símbolos como límites de instrucciones y varía entre ellos. binutils 2.47 (26 de julio de 2026) es GPL-3.0 o posterior. LLVM 23.1.3 es Apache-2.0 con la excepción LLVM.`llvm-objdump`como "Apple LLVM versión 17.0.0", y esa es la copia que esta lección capturó.

Un desmonte interativo ejecuta descensos recursivos de entradas conocidas, agrega heurísticas para arranques de funciones y mantiene una base de datos.`Disassembler`Además de los analizadores de análisis automático), Ninja binario (descenso recurrente, luego barrido lineal sobre regiones desconocidas), Hopper, radar2 y rizin (`aa`¿ Qué ?`aaa`¿ Qué ?`af`), con Cutter e iaito como GUIs, todos están en esta familia. Andriesse y otros (USENIX Security 2016) midieron nueve de ellos en binarios generados por el compilador. La precisión de instrucción fue cercana al 100 por ciento, y el duro primitivo fue el inicio de la función, con tasas falsas negativas de 20 por ciento o más.

### Descompileros

Descompileros comerciales: el Hex-Rays Decompiler (doce familias de objetivos de x86 a RISC-V), HLIL de Binary Ninja con su renderización Pseudo C y la vista de pseudo código de Hopper. Descompiler abierto: el Ghidra decompiler, un motor C++ bajo Apache-2.0 que r2ghidra y rz-ghidra reutilizan como un plugin y que pypcode reutiliza como un elevador; el descompiler angr en AIL con SAILR estructurado bajo BSD-2-Clause; r2dec y jsdec en JavaScript; Boomerang y `dcc`Como historia. Radar2 también tiene un incorporado`pdc`, "pseudo desmontaje de salida en sintaxis similar a C". Rizin no tiene un descompilador incorporado.`pdc`no existe en rizina 0.9.1, y `pdg`necesita el plugin rz-ghidra.

### Desarregladores

GDB (1986) con el GEF y pwndbg front-end, LLDB (2010), WinDbg con Time Travel Debugging (2017), OllyDbg (congelado en 2014), x64dbg (GPL-3.0, 2014), el Ghidra Debugger con sus conectores Python, el Binary Ninja debugger y los debuggers incorporados de radar2 y rizin.

### Enfrentamientos de análisis binario

Angr (VEX IR, ejecución simbólica, `CFGFast`y `CFGEmulated`, decompiler), BAP (BIL, OCaml), Dyninst, Jakstab y ByteWeight son los marcos que Andriesse nombró.`import binaryninja`, y la IDA a través de `idalib`¿ Qué ?

### Bibliotécnicas de formato y motores

LIEF 1.0.0 (12 de julio de 2026, Apache-2.0) analiza los formatos ELF, PE, Mach-O, COFF y Android. CLE, "CLE Loads Everything", es el cargador de angr. Capstone 5.0.9 (28 de mayo de 2026, BSD-3-Clause) decodifica; ensambles Keystone 0.9.2 (GPL-2.0); emulados Unicorn 2.1.4 (GPL-2.0). radare2 6.2.4 paquetes Capstone 5, y rizin 0.9.1 proveedores Capstone, Zydis, y árbol-sitter.

### Herramientas de difusión y espectadores del lado del compilador

BinDiff 8 (Apache-2.0) con su BinExport 12 plugin para IDA Pro, Binary Ninja y Ghidra; Diaphora 3.4.3 (11 de octubre de 2026, AGPL-3.0) como un plugin de IDA; `radiff2`y `rz-diff`. El Explorador de Compiladores (BSD-2-Clause) muestra el conjunto que un compilador emite para un fragmento fuente, que es la dirección hacia adelante de todo lo demás en esta lista.

### Las versiones, las licencias y lo que esta lección fue

| Tool | Version read on 2026-10-14 | Licence | Headless entry | Ran here |
|---|---|---|---|---|
| Ghidra | 12.1.4 | Apache-2.0 | `analyzeHeadless`, `pyghidra.start()` | yes, on all four objects |
| radare2 | 6.2.4 | LGPL-3.0 | `r2 -qc '<cmds>' file`, `rabin2` | yes |
| rizin | 0.9.1 | LGPL-3.0 | `rizin -qc '<cmds>' file`, `rz-bin` | yes |
| angr | 10.0.3 | BSD-2-Clause | `angr.Project` in Python | yes |
| llvm-objdump | Apple LLVM 17.0.0 | Apache-2.0 with LLVM exception | `llvm-objdump -d -r` | yes |
| Binary Ninja | 6.0 "Krypton" | commercial; Free edition for non-commercial use | `binaryninja.load()` with a Commercial, Ultimate, or headless licence | no, documentation only |
| IDA Pro | 9.4 | commercial subscription; IDA Free for non-commercial x86 and x86-64 | `idat`, `ida -A -S`, `idalib` | no, documentation only |
| Hopper | 6.6.0 | commercial, perpetual | none documented | no, documentation only |
| Cutter | 2.5.0 | GPL-3.0 | GUI over rizin | no |

La edición gratuita de Binary Ninja sólo descompila x86, x86-64, ARMv7 y AArch64 y no tiene API. IDA Free no tiene IDAPython, no SDK, y no `idalib`Cada declaración sobre esas dos herramientas en esta lección proviene de su documentación y notas de liberación, y la figura las marca como no funcionan.

## El oleoducto de descompiler, etapa por etapa

Cifuentes nombró las fases en 1994: análisis de sintaxis, análisis semántico, generación de código intermedio, generación de gráficos de flujo de control, análisis de flujo de datos, análisis de flujo de control y generación de código. Las dividió en una parte delantera dependiente de la máquina, una parte delantera independiente de la máquina y del lenguaje, y una parte posterior dependiente del lenguaje. Desde entonces, cada descompilador mantiene esa forma y renombre las piezas. La siguiente tabla muestra las ocho etapas que utiliza esta lección y el nombre que cada herramienta les da. La figura muestra la misma tabla con las columnas de herramientas seleccionables.

| Stage | Ghidra | Binary Ninja | Hex-Rays / IDA | radare2 | rizin | angr |
|---|---|---|---|---|---|---|
| Loader | `ElfLoader`, `MachoLoader`, `PeLoader`; `.ldefs` picks the language | `BinaryView`, `BinaryViewType` | loader module (`LDSC`) | RBin (`rabin2`) | RzBin (`rz-bin`) | CLE (`cle.Loader`, `main_object`) |
| Disassembly | `Disassembler`; SLEIGH `.slaspec` compiled to `.sla`; analyzers "Disassemble Entry Points", "Function Start Search" | architecture plugin; Recursive Descent, then linear sweep phases | processor module (`ana.cpp`, `emu.cpp`, `out.cpp`); autoanalysis queues; FLIRT | RArch plugin to `RAnalOp`; `aa`, `aaa`, `af` | RzAnalysis plugin; `aa`, `aaa`, `af` | Capstone blocks; `CFGFast` function start search |
| CFG recovery | "Generate Basic Blocks and the CFG"; `BlockGraph`; "Decompiler Switch Analysis" | `Function`, `BasicBlock`; "Control Flow Graph (CFG) Analysis" phase | `mbl_graph_t`, ready at `MMAT_LOCOPT` | `afb`, `agf`, `aab` | `afb`, `agf`, `aab` (Nucleus) | `CFGFast`, `CFGEmulated`; `kb.functions`; IRSB |
| Lifting | "Generate Raw P-code"; Varnodes in `ram`, `register`, `const`, `unique` | Lifted IL, LLIL, Mapped MLIL, MLIL | microcode (`minsn_t`, `mop_t`); `MMAT_GENERATED` | ESIL (`asm.esil`, `ae`) | RzIL (`aoi`, `aez`) | VEX (`pyvex.lift`, IRSB), then AIL |
| SSA | `Heritage`, `ActionHeritage`; `MULTIEQUAL`, `INDIRECT` | LLIL SSA, MLIL SSA, HLIL SSA; `HLIL_VAR_PHI` | no named SSA; use-def data flow in `MMAT_GLBOPT1` to `3` | none; ESIL is built for emulation | none; RzIL is built for emulation | `_stage_transform_to_ssa_level0` and `level1`; `SeqNodeDephication` |
| Data flow and types | `ActionDeadCode`, `ActionInferTypes`, the `Rule` pool; prototype from `.cspec` | MLIL constant propagation, variables and types; `analyzeReturns` | "Transform microcode"; `MMAT_CALLS`, `MMAT_GLBOPT`, `MMAT_LVARS`; TIL | `afv`, `afc`, `aaft`; ESIL emulation | `afv`, `afc`, `aft` | `CallingConventionAnalysis`, `VariableRecoveryFast`, `Typehoon` |
| Structuring | `CollapseStructure`; `BlockIf`, `BlockWhileDo`, `BlockSwitch`, `BlockGoto` | `HLIL_IF`, `HLIL_WHILE`, `HLIL_SWITCH`; AST at `root` | "Generate ctree"; structural analysis; goto regions | `pdc`; `pdd` (r2dec); `pdg` (r2ghidra) | `pdg` (rz-ghidra); jsdec | `RegionIdentifier`; `RecursiveStructurer` with `SAILRStructurer` |
| Emission | "Emit Final C Tokens"; `PrintC`, `ActionNameVars`, `ActionSetCasts` | HLIL rendering; Pseudo C, Pseudo Rust, Pseudo Python | "Beautify ctree; Print ctree"; `cfunc_t` | text of `pdc`, `pdd`, `pdg` | `pdg` text | `CStructuredCodeGenerator`; `codegen.text` |

### Cargador

El cargador analiza el contenedor, mapea secciones y lee símbolos y reubicaciones. En un objeto relocable nada tiene una dirección todavía, por lo que el cargador elige uno.`0x08000000`, que pone`.text`En el`0x08000040`porque la sección comienza en el archivo de compensación `0x40`. lugares de CLE `.text`En el`0x400000`- El de Ghidra.`ElfLoader`Lo coloca en `0x100000`- ¿ Qué ?`llvm-objdump`Imprenta desviaciones de sección y ninguna base en absoluto.

Los cargadores también difieren en cuanto a las reubicaciones. Ghidra los procesa a la importación y CLE los aplica. radar2 imprime una pista cuando abre el objeto: no se aplicaron las reubicaciones, y `bin.relocs.apply=true`Lo que un cargador hace con las reubicaciones decide lo que cada etapa posterior puede ver, y se verá el efecto en el `-O0`Salta abajo de la mesa.

### Desmontaje

La lección 33 construyó ambos métodos.`llvm-objdump`ejecuta descensos recursivos y luego añade una búsqueda de inicio de función. La documentación de Binary Ninja nombra las fases en orden: Descenso recurrente, Análisis de objetivos de llamadas, Análisis CFG como "desmontaje lineal de las regiones desconocidas" y Análisis de llamadas de cola. Ghidra ejecuta sus analizadores por prioridad, con "Desaconstruir puntos de entrada" y "Función de inicio de búsqueda" temprano. La IDA mantiene colas de autoanálisis, cada una con prioridad, y firmas FLIRT para el código de la biblioteca. ¿Qué es eso?`CFGFast`Los documentos de cuatro fuentes de función comienzan: símbolos, un preámbulo de exploración, objetivos de llamada y un escaneo completo de los bytes restantes.

### Recuperación de CFG

Cada herramienta corta los bloques de la manera que la lección 33 hizo, con una diferencia que se medirá.`call`, porque su IRSB es un bloque de entrada única, de salida múltiple y una llamada es una salida.`grade_label`En el`-O0`, que llama`grade`En el caso de los Estados miembros, el informe de la Comisión de Asuntos Exteriores de la Unión Europea (UEA) se aplica a las dos partes de la misma.

### El levantamiento

El código de elevación de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de la base de datos de datos de la base de datos de la base de datos de datos de la base de datos de datos de la base de datos de datos de la base de datos de datos de la base de datos de datos de la base de datos de datos de la base de datos de datos de la base de datos de datos de la base de datos de datos de la base de datos de datos de datos de la base de datos de datos de datos de la base de datos de datos de datos de la base de datos de datos de datos de la base de datos de datos de datos de datos de la base de datos de datos de datos de datos de datos de datos de datos de la base de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de la base de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos de datos`MMAT_GENERATED`¿ Qué ?`MMAT_LVARS`. Binary Ninja se eleva a Lifted IL, luego dobla las banderas en condiciones en LLIL, y MLIL "traducen registros y acceso de memoria en variables".

### SSA

La tesis de Van Emmerik enumera lo que SSA compra un descompilador: propagación de expresiones, parámetro e identificación de retorno sin una suposición de ABI, análisis de conservación, análisis de tipo como un problema de flujo de datos escaso, y el análisis de saltos e llamadas indirectos.`Heritage`y añade dos operaciones que sólo existen en el gráfico,`MULTIEQUAL`para el phi y`INDIRECT`En el caso de los efectos secundarios, el Ninja binario expone una forma de SSA en cada nivel de IL.`Clinic`.Hex-Rays no nombra un formulario SSA y utiliza el flujo de datos de definición en sus pases de optimización global.

### Flujo y tipos de datos

El bucle de simplificación principal de Ghidra se ejecuta "Eliminar código muerto", "Propagar tipos locales" y "Reescribir término" hasta que nada cambie, con la reescritura realizada por un grupo de `Rule`Los rayos transforman el microcodo, detecta los argumentos de llamada en `MMAT_CALLS`, y asigna variables locales en `MMAT_LVARS`. Angr se ejecuta .`CallingConventionAnalysis`¿ Qué ?`VariableRecoveryFast`, y `Typehoon`La lección 41 trata de esta etapa.

### Estructuración

Ghidra's `CollapseStructure`Busca subgrafos que coincidan con una estructura, los desmorona en un nodo, y cuando se atasca elimina un borde y lo marca desestructurado.`if`¿ Qué ?`while`¿ Qué ?`do`, y `switch`La Comisión de Agricultura y Desarrollo de la Agricultura de la Comunidad Europea (CEDEFOP) ha adoptado una directiva que establece que las regiones de entrada y salida únicas se deben a las regiones de la Comunidad Europea y a las regiones de la Comunidad Europea.`SAILRStructurer`por defecto, con `PhoenixStructurer`y `DreamStructurer`La lección 34 fue escrita por un estructurador a mano.

### Emisos

Ghidra emite tokens C a través de `PrintC`con una impresora Oppen bonita después `ActionNameVars`y `ActionSetCasts`. Hex-Rays embellece e imprime el árbol. Binary Ninja hace HLIL como Pseudo C, Pseudo Rust o Pseudo Python. angr devuelve `codegen.text`. Imágenes de radar2`pdc`¿ Qué ?`pdd`, o`pdg`¿ Qué ?

## Una instrucción en cuatro IR

¿ Qué ?`48 83 ec 10`¿ Qué ?`subq $0x10, %rsp`Decodifica en papel con las reglas de la lección 33.

1. `48`es REX con `W = 1`La operanda es de 64 bits.
2. `83`es el código opmético aritmético del grupo 1 con un inmediato de 8 bits.
3. `ec`es ModRM `11 101 100`- ¿ Qué ?`mod = 11`es el formulario de registro. `reg = 101`¿ Es verdad ?`/5`, que es `sub`- ¿ Qué ?`r/m = 100`¿ Es verdad ?`%rsp`¿ Qué ?
4. `10`Es el inmediato, 16.

Cuatro elevadores recibieron esos cuatro bytes.

ESIL de radar2 6.2.4 es una cadena separada por comas de 30 tokens:

```text
16,0x8000000000000000,-,!,16,rsp,-=,63,$o,^,of,:=,63,$s,sf,:=,$z,zf,:=,$p,pf,:=,64,$b,cf,:=,4,$b,af,:=
```

Es una máquina de apilar.`16,rsp,-=`Se escribe cada bandera con un operador interno: `$o`sobrecarga,`$s`el signo,`$z`cero,`$p`la paridad,`$b`¿Por qué no me lo prestas?

El RzIL de rizina 0.9.1 es una expresión S con 14 `set`efectos:

```text
(seq (set op1 (var rsp)) (set op2 (bv 64 0x10)) (set sub (- (var op1) (var op2))) (set rsp (var sub))
     (set _result (var sub)) (set pf (! (lsb (let _val (cast 8 false (var _result)) ...))))
     (set zf (is_zero (var _result))) (set sf (msb (var _result))) ... (set cf (|| ...)) (set of (|| ...)) (set af (|| ...)))
```

Cada bandera es una expresión booleana sobre bitvectores.`let`¿ Qué ?

Código P de la especificación SLEIGH `x86:LE:64:default`a través del código pyp 4.0.1 es nueve operaciones después de la `IMARK`¿Qué es esto ?

```text
CF = RSP < 0x10
OF = sborrow(RSP, 0x10)
RSP = RSP - 0x10
SF = RSP s< 0x0
ZF = RSP == 0x0
unique[58300:8] = RSP & 0xff
unique[58400:1] = popcount(unique[58300:8])
unique[58500:1] = unique[58400:1] & 0x1
PF = unique[58500:1] == 0x0
```

Registro de transferencias con temporarios en el `unique`Las banderas se calculan a la instrucción, ansiosamente, y`CF`y `OF`se calculan antes de que la restancia se sobreponga `RSP`¿ Qué ?

VEX de pyvex 10.0.3 es siete declaraciones después de la`IMark`¿Qué es esto ?

```text
t2 = GET:I64(offset=48)
t0 = Sub64(t2,0x0000000000000010)
PUT(offset=144) = 0x0000000000000008
PUT(offset=152) = t2
PUT(offset=160) = 0x0000000000000010
PUT(offset=168) = 0x0000000000000000
PUT(offset=48) = t0
```

No aparece ninguna bandera. VEX nombres se registran por su compensación en el estado invitado, y compensación 48 es `rsp`Los beneficios de 144 a 168 son`cc_op`¿ Qué ?`cc_dep1`¿ Qué ?`cc_dep2`, y `cc_ndep`El elevador almacena el código de operación 8, que es la restancia de 64 bits, y los dos operandos.`pcode.writesFlags`es verdad y `vex.computesFlags`es falso en el informe de arnés, y por qué VEX es el más corto de los cuatro.

Los mismos cuatro elevadores en .`48 63 c7`¿ Qué ?`movslq %edi, %rax`, la primera instrucción de `grade`En el`-O1`, dar 5 tokens ESIL (`32,edi,~,rax,=`), 1 RzIL `set`, 1 operación con código P (`RAX = sext(EDI)`La lista de instrucciones de Ghidra, capturada a través de la letra sin cabeza, imprime la misma operación que la de Ghidra.`(register, 0x0, 8) INT_SEXT (register, 0x38, 4)`: registro de compensación 0 es `RAX`, compensación `0x38`¿ Es verdad ?`RDI`, y los tamaños son 8 y 4 bytes.

`83 f9 04`¿ Qué ?`cmpl $0x4, %ecx`, desde el registro de fronteras .`grade`, da 30 tokens ESIL, 13 RzIL `set`Los efectos de la aplicación de los métodos de control de la información son:`cc_op`7, la subtracción de 32 bits. Nada escribe.`%ecx`Los dos anchos de restas que encontrarás en VEX son 7 para 32 bits y 8 para 64 bits, y el traductor que construyes lleva esa tabla.

## Cuatro bases de direcciones para una función

Antes de que el arnés pueda comparar cualquier cosa tiene que estar de acuerdo consigo mismo sobre dónde está una función.`grade`En el`ledger-x86_64-O1.o`¿ Qué ?

| Tool | Address of `.text` | Address of `grade` | Offset |
|---|---|---|---|
| `llvm-objdump` | `0x0` | `0x10` | `0x10 - 0x0 = 0x10` |
| radare2, rizin | `0x08000040` | `0x08000050` | `0x08000050 - 0x08000040 = 0x10` |
| angr | `0x400000` | `0x400010` | `0x400010 - 0x400000 = 0x10` |
| Ghidra | `0x100000` | `0x100010` | `0x100010 - 0x100000 = 0x10` |

La tabla de símbolos dice:`grade`¿ Qué es ?`.text + 0x10`Cada herramienta se encuentra de acuerdo una vez que se elimina la base.`iSj`en radare2 y rizin, el`.text`La entrada de la sección en la captura de angr, la `.text`bloqueo de memoria en la captura de Ghidra, y cero para el barrido.`test_every_tool_places_grade_at_offset_0x10_from_its_own_base`corrige ese hecho.

## Donde las herramientas están de acuerdo y dónde se dividen

El arnés en `code/main.py`Se leyeron las capturas, se redujo cada dirección y se comparó.

### El caso de acuerdo: libro mayor en -O1

Las cuatro constructoras de gráficos informan de bloques 3, 3, 5 y 4 y 2, 3, 5 y 4 bordes, y los conjuntos de bordes son idénticos, pares por pares, a través de radar2, rizin, angr y Ghidra.`total`Son las que la lección 33 dibujó a mano.

La única diferencia es en el relleno.`66 90`después de`grade`¿ Qué ?`90`después de`total`) son decodificadas por el barrido, que no tiene razón para detenerse, y por el angr, que crea `sub_40003e`y `sub_40005f`Radar2, Rizin y Ghidra no reclaman ninguno de ellos.

### El caso vectorizado: libro mayor en -O2

`total`En el caso de los gráficos, el circuito SSE2 y el circuito de residuos escalares de la lección 33 se cortan de la misma manera en todas partes.

### La tabla de saltos: grado en -O0

Aquí las herramientas se dividen, y la división es la lección.

| Tool | Blocks | Edges | What the capture says |
|---|---|---|---|
| radare2 | 3 | 2 | the block at `0x62` has no successor; bytes `0x76` to `0x9a` are never reached; 8 instruction starts missing |
| rizin | 8 | 11 | the block at `0x62` carries a `switch_op` with 5 cases and 4 distinct targets |
| angr | 8 | 7 | the case blocks exist, and the transfer at `0x62` goes to an unresolvable target node |
| Ghidra | 9 | 12 | `switchD` label at `0x74` splits the block; four `COMPUTED_JUMP` flows to `0x76`, `0x7f`, `0x88`, `0x91` |

Los cuatro decodificaron las mismas 33 instrucciones cuando llegaron a ellas.`jmp rax`Y dejó 36 bytes sin reclamar, y los nombres de arneses que van desde el código perdido. Rizin y Ghidra encontraron los cuatro objetivos distintos que la lección 33 lee de las cinco reubicaciones en`.rela.rodata`. Angr enumera los bloques de la caja dentro `grade`y todavía no tiene ventaja de la barra de salto a ellos; su captura registra la transferencia en `0x62`como lo que lleva a un nodo objetivo insoluble, y su descompilador imprimió un computador `goto`Ghidra tiene un bloque y un borde más que rizin para el mismo flujo de control porque su análisis de interruptor colocó una etiqueta en el `jmp rax`y el modelo de bloque comienza un nuevo bloque en una etiqueta.

El gráfico de la lección-33 tenía 8 bloques y 11 bordes. rizin coincide exactamente. El punto es ver que 9 y 11 y 7 y 2 no son cuatro medidas de código encontradas. Son cuatro respuestas a cuán lejos una herramienta siguió un salto indirecto y dónde eligió cortar.

### Datos en código: trampa de barrido

`lookup`es 24 bytes con los 4 bytes de la tabla `31 c0 90 48`En el`0x10`El arnés enumera sus diferencias con la mayoría: comienza extra en`0x10`¿ Qué ?`0x12`, y `0x13`, y un comienzo perdido en `0x14`Eso es lo que .`addq $0x1, %rax`Radar2, Rizin y Ghidra reportan 6 comienzos, 2 bloques, 1 borde y salida.`0x10`¿ Qué ?`0x14`No reclamado. Angr informa los mismos 2 bloques y 1 borde para `lookup`y luego inventar.`sub_400010`en la tabla bytes: 2 instrucciones en un bloque de 4 bytes que termina en `0x14`, donde el segundo bloque de `lookup`comienza, por lo que el byte REX en `0x13`El último hallazgo del arnés para este objeto es que ningún constructor de gráficos decodifica.`0x10`¿ Qué ?`0x14`, por lo que esos bytes se sientan dentro de una función y son datos.

### Inventado y omitido, por herramienta

En los cuatro objetos el arnés nombra 9 funciones que Angr inventa sin símbolo, cada una en el relleno o en la mesa de trampas.`grade`En el`-O0`. Nombra 9 rangos que el barrido lineal decodifica y ningún constructor de gráficos trata como código: los ocho huecos de relleno más la tabla de trampas.

## Tres Decompiladores en grado

Tres de las herramientas capturadas emiten texto similar a C para `grade`En el`-O1`La fuente es`switch (score / 10)`con cinco casos, compilados a partir de un control de límites y una tabla de valores de devolución.

Ghidra 12.1.4:

```c
uint grade(int param_1)
{
  uint uVar1;
  uint uVar2;

  uVar2 = param_1 / 10 - 6;
  uVar1 = -(param_1 >> 0x1f) ^ (uint)((ulong)((long)param_1 * 0x66666667) >> 0x3f);
  if (uVar2 < 5) {
    uVar1 = *(uint *)(&DAT_001000b4 + (ulong)uVar2 * 4);
  }
  return uVar1;
}
```

Angr 10.0.3:

```c
extern unsigned int g_4000b4[4];

unsigned int grade(int a0)
{
    unsigned int v1;  // ecx
    unsigned int v2;  // eax

    v1 = (unsigned int)((long long)(a0 * 1717986919) >> 0x22) + (unsigned int)(a0 * 1717986919 >> 63) - 6;
    v2 = 0;
    if (v1 <= 4)
        v2 = g_4000b4[v1];
    return v2;
}
```

Radar2 6.2.4, `pdc`¿Qué es esto ?

```c
void sym.grade (int64_t arg1) {
    rax = edi
    rax = rax * 0x66666667
    rcx = rax
    rcx >>= 0x22
    rax >>>= 0x3f
    ecx += eax
    ecx += 0xfffffffa
    eax = 0
    v = ecx - 4
    if (((unsigned) v) <= 0) {
        eax = ecx
        rcx = rip
        eax = dword [rcx + rax*4]
    }
    return
}
```

Leemos los tres contra el oleoducto.`0x66666667`y el cambio de 34 de vuelta en`param_1 / 10`Angr levantó los mismos bytes y dejó la aritmética como`a0 * 1717986919 >> 0x22`. los radares2 `pdc`Es la emulación de ESIL más una pseudo sintaxis: nombres de registro, no variables, no tipos, y los límites de control impresos como `<= 0`Ghidra escribió la carga de la mesa a través de la letra.`DAT_001000b4`; en el caso de la declaración `g_4000b4[4]`En la actualidad, el número de casos en la lista de casos no se ha recuperado, ya que en la`-O1`No hay interruptor en los bytes.`-O0`, donde existe la mesa de saltos, Ghidra impreso `switch(param_1 / 10)`con casos 6 a 10 y por defecto, y angr impreso un computador `goto`¿ Qué ?

Rizin no tiene un descompilador propio, y los registros de captura que `pdc`El HLIL de Binary Ninja y el árbol de Hex-Rays son las dos salidas que esta lección no pudo capturar.

## Construye el mismo

Ejecutar la demostración:

```bash
cd phases/13-tools-and-protocols/37-the-reverse-engineering-landscape
python3 code/main.py
```

La salida es un informe JSON. Se abre con las versiones de herramientas grabadas en el momento de la captura y con las herramientas presentes en su máquina, que pueden estar ausentes. Luego, para cada uno de los cuatro objetos da las extensiones del símbolo, las lagunas de relleno, la base de cada herramienta utilizada, la función inicia cada herramienta encontrada o inventada, y para cada función el acuerdo de inicio de instrucción, el conteo de bloques, el conteo de bordes y los bordes que cada herramienta carece. Terminará con la contabilidad de byte por herramienta, las propias notas de las herramientas, una lista de resultados en frases simples y la sección IR: la salida del traductor para las tres instrucciones junto a los recuentos medidos en los ascensores capturados.

Forma esperada:

```json
{
  "captureTools": {"llvm-objdump": "Apple LLVM version 17.0.0", "radare2": "radare2 6.2.4 ...", "rizin": "rizin 0.9.1 ...", "angr": "10.0.3", "ghidra": "12.1.4"},
  "localTools": {"radare2": "absent", "...": "..."},
  "fixtures": {
    "ledger-x86_64-O0.o": {
      "bases": {"llvm-objdump": "0x0", "radare2": "0x8000040", "angr": "0x400000", "ghidra": "0x100000", "...": "..."},
      "functions": {"grade": {"blocks": {"radare2": 3, "rizin": 8, "angr": 8, "ghidra": 9}, "edges": {"radare2": 2, "rizin": 11, "angr": 7, "ghidra": 12}, "...": "..."}},
      "findings": ["radare2 never reaches bytes 0x76-0x9a inside grade that the other graph builders decode", "..."]
    }
  },
  "ir": {"translator": [{"text": "subq $0x10, %rsp", "flags": {"pcodeLike": "eager", "vexLike": "thunk"}, "...": "..."}], "observed": {"...": "..."}}
}
```

Leer .`code/main.py`en este orden:

1. `read_text_symbols()`lee los encabezados de la sección y el `STT_FUNC`símbolos de `.text`La tabla de símbolos es la única verdad de la arnés, y`TextSection.gaps()`derivando el relleno de ella.
2. `parse_objdump()`¿ Qué ?`parse_r2_family()`¿ Qué ?`parse_angr()`, y `parse_ghidra()`Cada uno convierte una captura en una`ToolView`: una base, las instrucciones como compensación a la longitud, y una `FunctionView`Cada parser sabe el vocabulario de una herramienta y nada más.
3. `majority()`y `ranges()`¿Son los dos auxiliares para que todo comparativo se utilice?
4. `function_report()`compara las instrucciones iniciales con la mayoría de los constructores de gráficos, luego cuenta los bloques, luego los conjuntos de bordes, y enumera lo que cada herramienta carece.
5. `byte_report()`cuenta cada byte como código dentro de un símbolo, relleno, no reclamado, inventado o perdido, y nombra los bytes dentro de símbolos que ningún constructor de gráficos decodifica.
6. `findings()`convierte esas frases en frases, y`compare()`Se elabora un informe por objeto.
7. `decode_small()`¿ Qué ?`to_pcode_like()`, y `to_vex_like()`El traductor: tres codificaciones en forma de registro, un estilo ansioso, un estilo de trozo.
8. `observed_ir()`Las medidas de los ascensores capturados, y `figure_data()`Se reúne la línea de tiempo, la tabla de escenarios y la evidencia que muestra la figura.

Hay tres órdenes más.`python3 code/main.py compare FIXTURE.o`Imprime el informe de un objeto. `python3 code/main.py ir 4883ec10`Imprime los dos formularios del traductor para una instrucción. `python3 code/main.py figure-data`Imprima los datos incrustados en la figura.

Las capturas fueron hechas una vez por `code/fixtures/capture_tools.py`, que registra cada comando y versión en `captures.json`Si tienes las herramientas, ejecuta el guión de captura para actualizar los archivos y lee los accesorios README para el entorno que necesita.

## Usalo

Use el arnés en un objeto nuevo en este orden.

1. Construye el objeto usted mismo y mantenga el comando del compilador. Lea su tabla de símbolos antes de abrir cualquier herramienta, para que sepa cuántas funciones hay y dónde está el relleno.
2. Captura cada herramienta una vez en modo sin cabeza. Almacene la salida en bruto y el comando exacto junto al objeto. Un desacuerdo que no pueda volver a leer no es un hallazgo.
3. Verifique la base. Seleccione una función y confirme que cada herramienta lo coloca en el mismo desplazamiento después de restar su `.text`Si no lo hace, deténgase y averigüe qué cargaba esa herramienta.
4. Si los constructores de gráficos no están de acuerdo aquí, el resto de la comparación es sobre la decodificación y no sobre el análisis.
5. Comparar bloques y bordes, y leer las notas de cada herramienta antes de juzgar un recuento.`switch_op`, una `COMPUTED_JUMP`, o un nodo objetivo insoluble explica la mayoría de las diferencias.
6. Cuenta para bytes. Padding que una herramienta afirma es una heurística en el trabajo. Bytes símbolo que una herramienta deja son un paso parado. Bytes símbolo que ningún constructor de gráficos afirma son datos.
7. Una descripción de la documentación es útil y es un tipo diferente de evidencia de una captura.

## Laboratorio interactivo

### Laboratorio A: una función, cuatro bases

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/37-the-reverse-engineering-landscape/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
views = load_views("ledger-x86_64-O1.o")
{tool: hex(view.base) for tool, view in views.items()}
{tool: hex(view.functions["grade"].start + view.base) for tool, view in views.items()}
```

Escriba las cuatro direcciones crudas antes de ejecutar la segunda línea y comprobarlos contra la mesa en esta lección.`grade`con`grade_label`y calcular a mano donde se encuentra en Ghidra.

### Laboratorio B: la mesa de salto desde cuatro lados

```python
report = compare("ledger-x86_64-O0.o")
report["functions"]["grade"]["blocks"]
report["functions"]["grade"]["edgesMissingPerTool"]["radare2"]
report["notes"]["rizin"]
report["notes"]["ghidra"]
report["bytes"]["radare2"]["misses"]
```

Cuenta los bordes que faltan en el radar y ajusta cada uno a una línea de la`-O0`Luego lee la nota de Ghidra y encuentra la`switchD`etiqueta en la captura de Ghidra.

### Laboratorio C: banderas ansiosas contra un tronco

```python
translate("48 83 ec 10")["pcodeLike"]
translate("48 83 ec 10")["vexLike"]
translate("83 f9 04")["vexLike"]
{row["name"]: (row["pcode"]["ops"], row["vex"]["statements"]) for row in observed_ir()["instructions"]}
```

Encuentra el`cc_op`constantes en ambas salidas y explicar por qué uno es 8 y el otro 7.`translate("48 01 c8")`y leer el `TranslateError`¿ Qué ?

### Laboratorio D: la trampa bajo cinco herramientas

```python
report = compare("sweep-trap.o")
report["functions"]["lookup"]["instructionStarts"]
report["bytes"]["dataInsideSymbols"]
report["functionStarts"]["angr"]
```

Descifrar los bytes `31 c0 90 48 83 c0 01`en el papel como un barrido, luego como un descenso después de la`jmp`Combine sus dos listados con el extra y comienza el informe de arnés.

## Laboratorio de práctica

Añade una quinta captura y extienda el arnés.

Requisitos:

- Elige una herramienta que esta lección no capturó y que puedas ejecutar en tu máquina: GNU `objdump`, LIEF con Capstone, una sesión Binary Ninja Free exportada a mano, o Ghidra con un conjunto de analizadores diferente. Captura su salida en los cuatro objetos con el comando exacto y la versión en `captures.json`¿ Qué ?
- Escriba un parser que convierta la captura en un `ToolView`con la base de la herramienta, instrucciones, funciones, bloques donde la herramienta las tiene, y bordes donde las tiene.
- Añadir la herramienta a `TOOLS`y, si construye un gráfico, para`DESCENT_TOOLS`Echa la demostración y lee lo que cambió en los conjuntos mayoritarios.
- Escriba una prueba que asegure que el bloque de la herramienta cuenta `ledger-x86_64-O1.o`y una prueba que asegura lo que hace con la mesa de trampas.
- Escriba un párrafo para la lección que dice, en palabras propias de la herramienta, lo que llama el cargador, el desmonteador y el CFG.

El éxito significa`python3 code/main.py`informe la nueva herramienta en `captureTools`, cada función en `ledger-x86_64-O1.o`¿ Qué es ?`allDescentToolsAgree`La Comisión ha presentado una serie de conclusiones sobre la`sweep-trap.o`Digamos lo que la nueva herramienta hizo con los bytes `0x10`¿ Qué ?`0x14`¿ Qué ?

## Lección Artefacto

Esta lección incluye `outputs/skill-the-reverse-engineering-landscape.md`Utilice para ejecutar varias herramientas en un objeto que haya construido, reducir sus direcciones a una base y informar, por función, donde coincidan en los comienzos de instrucciones, bloques y bordes, y cuál bytes cada herramienta invente o omite. Requiere capturas con comandos y versiones grabadas, una tabla de símbolos leída del archivo y una declaración de qué herramientas se describieron solo a partir de la documentación.

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/37-the-reverse-engineering-landscape
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

La verificación deberá demostrar:

- El lector ELF encuentra cuatro símbolos de función en `ledger-x86_64-O1.o`con tamaños 16, 46, 31 y 63, y dos huecos de relleno
- las capturas de barrido contienen 86, 56, 84 y 11 instrucciones para los cuatro objetos
- Radar2 y rizin reportan las mismas instrucciones, bloques y bordes en `ledger-x86_64-O1.o`, y sólo el radar2 tiene`pdc`texto
- marcas de angr`sub_40003e`y `sub_40005f`como funciones sin símbolo
- Los bloques de Ghidra para`total`son los cinco bloques de la lección 33 y sus flujos de llamada no son bordes de gráfico
- cada herramienta de los lugares `grade`en compensación `0x10`después de que se retire su base
- Cada herramienta está de acuerdo en cada instrucción que comienza en cada función de `ledger-x86_64-O1.o`
- Los cuatro constructores de gráficos informan de bloques 3, 3, 5, 4 y 2, 3, 5, 4 bordes allí, con conjuntos de bordes idénticos
- `total`En el`-O2`tiene 10 bloques y 13 bordes en cada constructor de gráficos
- `grade`En el`-O0`tiene 3, 8, 8 y 9 bloques y 2, 11, 7 y 12 bordes en radar2, rizin, angr y Ghidra, con las razones en las notas de cada herramienta
- `grade_label`En el`-O0`es un bloque en todas partes excepto angr, que termina un bloque en la llamada
- Angr inventa una función por hueco de relleno más una en la mesa de trampas, y ninguna otra herramienta inventa o pierde una función
- Las ganancias de barrido comienzan.`0x10`¿ Qué ?`0x12`¿ Qué ?`0x13`y pierde .`0x14`en la trampa, y ningún grafico constructor decodifica `0x10`¿ Qué ?`0x14`
- Los resultados nombran el rango perdido en radar2 y cada función inventada en angr
- el manifiesto registra una versión y un comando para todos los 21 archivos de captura
- Descompone tres herramientas `grade`, y sólo Ghidra dobla el lenguaje de división
- El traductor escribe CF, OF, SF, ZF, PF con entusiasmo en un estilo y un `cc_op`El otro no escribe ningún registro para`cmp`, y no hay bandera para `movslq`
- el traductor rechaza los formularios de memoria, otros códigos opcodes, bits REX adicionales y longitudes incorrectas
- de la traducción `cc_op`los números 8 y 7 son iguales a los del VEX capturado
- los ascensores capturados dan 9, 1, 10 operaciones de código P y 7, 4, 7 declaraciones VEX para las tres instrucciones
- La línea de tiempo se extiende desde 1973 hasta 2026 en orden, la tubería tiene ocho etapas y seis herramientas con un término en cada célula, y las dos herramientas cerradas se marcan como no se ejecutan
- La figura interactiva incorpora los mismos datos que el arnés produce

## Modos de falla de producción

| Failure | What the weak report says | What the harness must prove |
|---|---|---|
| Raw addresses from different tools | "the tools disagree about where grade is" | one offset after each base is removed, checked on one function |
| Block count as a measure of code found | "Ghidra found the most code" | the same instruction starts, with the extra block explained by a label |
| One tool as the oracle | "objdump shows the table bytes are instructions" | the sweep is a linear sweep; three graph builders leave the bytes unclaimed |
| Padding functions counted as discoveries | "angr found seven more functions" | every one sits in a padding gap or on data, and none has a symbol |
| A stopped walk reported as dead code | "the four case blocks are unreachable" | 36 bytes missed by one tool and reached by three |
| Edge counts compared without the notes | "rizin and Ghidra disagree on grade" | the same four computed targets, one extra split at a label |
| A decompiler's arithmetic read as the program | "the function multiplies by 1717986919" | the idiom from lesson 34, folded by one tool and left by another |
| A closed tool described as if run | "Binary Ninja reports five blocks" | the figure and the table mark it as documentation only |
| Captures without commands | "we ran radare2" | the manifest with version, command, and date for every file |
| An IR with no flags read as a lossy lift | "VEX dropped the flags" | the thunk in `cc_op`, `cc_dep1`, `cc_dep2`, evaluated at the branch |

## Regla de funcionamiento

Captura cada herramienta una vez con su comando y versión, reduce cada dirección a una base y compara las instrucciones comienzan antes de los bloques y los bloques antes de los bordes. Cuenta con cada byte como se afirma, relleno, datos o falta. Lea la propia nota de una herramienta antes de juzgar su número, y diga qué herramientas solo lees.

## Leer más

- [Cifuentes, "Reverse Compilation Techniques", PhD thesis, QUT (1994)](https://eprints.qut.edu.au/36820/)
- [Cifuentes and Gough, "Decompilation of Binary Programs", Software: Practice and Experience (1995)](https://doi.org/10.1002/spe.4380250706)
- [Van Emmerik, "Static Single Assignment for Decompilation", PhD thesis, University of Queensland (2007)](https://espace.library.uq.edu.au/view/UQ:158682)
- [Schwartz, Lee, Woo, Brumley, "Native x86 Decompilation Using Semantics-Preserving Structural Analysis and Iterative Control-Flow Structuring" (USENIX Security 2013)](https://www.usenix.org/conference/usenixsecurity13/technical-sessions/presentation/schwartz)
- [Yakdan et al., "No More Gotos" (NDSS 2015)](https://www.ndss-symposium.org/ndss2015/ndss-2015-programme/no-more-gotos-decompilation-using-pattern-independent-control-flow-structuring-and-semantics/)
- [Basque et al., "Ahoy SAILR! There is No Need to DREAM of C" (USENIX Security 2024)](https://www.usenix.org/conference/usenixsecurity24/presentation/basque)
- [Andriesse et al., "An In-Depth Analysis of Disassembly on Full-Scale x86/x64 Binaries" (USENIX Security 2016)](https://www.usenix.org/conference/usenixsecurity16/technical-sessions/presentation/andriesse)
- [Flake, "Structural Comparison of Executable Objects" (DIMVA 2004)](https://dl.gi.de/items/7f573931-cb51-49ef-a12c-0452a9fb0ea4)
- [Maranzano and Bourne, "A Tutorial Introduction to ADB", Bell Labs (1977)](http://squoze.net/UNIX/v7/files/doc/18_adb.pdf)
- [Stallman, KTH lecture on GDB entering distribution (30 October 1986)](https://www.gnu.org/philosophy/stallman-kth.html)
- [Hex-Rays, "IDA: celebrating 30 years of binary analysis innovation"](https://hex-rays.com/blog/ida-celebrating-30-years-of-binary-analysis-innovation)
- [Guilfanov, "Decompiler internals: microcode", Recon Brussels 2018 (slides)](https://www.recon.cx/2018/brussels/resources/slides/RECON-BRX-2018-Decompiler-internals-microcode.pdf)
- [Hex-Rays, microcode view and maturity levels](https://docs.hex-rays.com/core/decompiler/how-tos/microcode-view)
- [Hex-Rays, IDA 9.4 release notes](https://docs.hex-rays.com/release-notes/9_4)
- [Hex-Rays, IDA Free](https://hex-rays.com/ida-free)
- [Ghidra releases (12.1.4)](https://github.com/NationalSecurityAgency/ghidra/releases)
- [Ghidra P-code reference](https://github.com/NationalSecurityAgency/ghidra/blob/master/GhidraDocs/languages/html/pcoderef.html)
- [Ghidra decompiler source, main work flow and Action classes](https://github.com/NationalSecurityAgency/ghidra/tree/master/Ghidra/Features/Decompiler/src/decompile/cpp)
- [Ghidra analyzeHeadless README](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/RuntimeScripts/support/analyzeHeadlessREADME.md)
- [Binary Ninja, BNIL overview](https://docs.binary.ninja/dev/bnil-overview.html)
- [Binary Ninja, "Architecture Agnostic Function Detection in Binaries" (2017)](https://binary.ninja/2017/11/06/architecture-agnostic-function-detection-in-binaries.html)
- [Binary Ninja, batch processing and headless licences](https://docs.binary.ninja/dev/batch.html)
- [Binary Ninja 6.0 "Krypton" release post](https://binary.ninja/2026/09/03/binary-ninja-6.0-krypton.html)
- [Binary Ninja Free](https://binary.ninja/free/)
- [radare2 book, ESIL](https://book.rada.re/emulation/esil.html)
- [radare2 book, decompilers](https://book.rada.re/arch/decompile.html)
- [radare2 releases (6.2.4)](https://github.com/radareorg/radare2/releases)
- [rizin, RzIL design notes](https://github.com/rizinorg/rizin/blob/dev/doc/rzil.md)
- [rizin releases (0.9.1)](https://github.com/rizinorg/rizin/releases)
- [Announcing Rizin (8 December 2020)](https://rizin.re/posts/announcing-rizin/)
- [Cutter releases (2.5.0)](https://github.com/rizinorg/cutter/releases)
- [angr documentation, CFG recovery](https://docs.angr.io/en/latest/analyses/cfg.html)
- [angr documentation, the decompiler and its analysis passes](https://docs.angr.io/en/latest/analyses/decompiler.html)
- [angr documentation, intermediate representation (VEX)](https://docs.angr.io/en/latest/advanced-topics/ir.html)
- [angr on PyPI (10.0.3)](https://pypi.org/project/angr/)
- [pypcode on PyPI](https://pypi.org/project/pypcode/)
- [llvm-objdump command guide](https://llvm.org/docs/CommandGuide/llvm-objdump.html)
- [GNU objdump manual](https://sourceware.org/binutils/docs/binutils/objdump.html)
- [LIEF documentation](https://lief.re/doc/stable/index.html)
- [Capstone releases](https://github.com/capstone-engine/capstone/releases)
- [BinDiff releases (v8, open source)](https://github.com/google/bindiff/releases/tag/v8)
- [Diaphora](https://github.com/joxeankoret/diaphora)
- [Hopper Disassembler downloads](https://www.hopperapp.com/download.html)
- [Compiler Explorer](https://github.com/compiler-explorer/compiler-explorer)
- [LLDB announced as an LLVM subproject (2010)](https://blog.llvm.org/2010/06/new-lldb-debugger.html)
- [Boomerang decompiler](https://boomerang.sourceforge.net/)
- [Tan et al., "LLM4Decompile" (EMNLP 2024)](https://aclanthology.org/2024.emnlp-main.203/)
- [Gao et al., "DecompileBench" (Findings of ACL 2025)](https://aclanthology.org/2025.findings-acl.1194/)
- [Liu, Raff, Micinski, "When LLM Decompilers Recompile More and Preserve Less" (2026)](https://arxiv.org/abs/2609.05370)
