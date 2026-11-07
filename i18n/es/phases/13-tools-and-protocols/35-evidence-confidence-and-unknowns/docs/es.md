# Evidencias, confianza y desconocidos en análisis binario

> Un hallazgo sobre un programa compilado es un registro con un hash, una confianza, una autoridad y una lista de lo que no puede mostrar.

**Type:** Build
**Languages:** Python, C
**Prerequisites:** Phase 13 · 32 (ELF from scratch), Phase 13 · 33 (disassembly and control flow), Phase 13 · 34 (lifting to pseudocode)
**Time:** ~105 minutes

## Objetivos de aprendizaje

- Construir un registro de pruebas basado en los datos de rea.`Evidence`con la confianza observada, derivada y inferida.
- Computa IDs de contenido-hash que permanecen iguales cuando el archivo se mueve a otra carpeta.
- Registra lo que cada proveedor no puede mostrar como limitaciones que viajan con el registro.
- Rechazar el indicación de un resultado hasta que una ejecución observada de la función lo respalde.
- Mantenga los desconocidos residuales como registros revisados unidos por una cadena de hash.
- El proveedor de caché responde bajo una clave exacta y rechaza una reproducción obsoleta.

## El problema

¿ Qué le preguntas a un agente ?`grade`En el`ledger-x86_64-O1.o`El agente llama rea, lee el pseudocodo de Ghidra y responde 0. Un revisor pregunta cómo sabe el agente. El agente tiene un bloque de texto que Ghidra imprimió.`uVar1 = -(param_1 >> 0x1f) ^ (uint)((ulong)((long)param_1 * 0x66666667) >> 0x3f);`Los bytes en ese punto dicen`xor eax, eax`¿ Qué ?

La expresión y la instrucción producen 0 para cada entrada. El agente no sabía eso.[NDSS BAR 2026](https://www.ndss-symposium.org/wp-content/uploads/bar2026-58.pdf)En el estudio de 2026 se plantó texto dentro de los binarios y se encontró que los modelos propusieron la acción insegura plantada en 35 de los 40 casos ([arXiv 2607.12507](https://arxiv.org/abs/2607.12507)¿Qué es lo que se hace?

La respuesta de la agencia es que, en el caso de las llamadas de herramientas, cada llamada de éxito produce una.`Evidence`registro con un `confidence`de `observed`¿ Qué ?`derived`, o`inferred`, un `authority`, y una lista de `limitations`. La habilidad envasada le dice al agente: "Cada conclusión debe distinguir observaciones, inferencias y desconocimientos. Cita ID de Evidencia, preserva limitaciones y cobertura incompleta, y nunca implica que el análisis estático observado ejecución".

Tres cosas que realmente le deja a usted. Sus instantáneas almacenan en caché los resultados exactos de la consulta, no el análisis. Su verificación pasa una reclamación cuando una dimensión de comparación no cambia, y su propio comentario dice que un pase "no establece equivalencia global de implementación". Su libro mayor de obligaciones toma candidatos sólo de JavaScript y gráficos gestionados, por lo que una función nativa nunca alcanza.`verified`a través de ella.

Esta lección construye la pieza que falta en Python simple. Usted escribe un libro mayor que almacena todos los hechos sobre el dispositivo como un registro con un hash de contenido. Usted hace predicciones sobre`percent_change`y `grade`Importa dos registros reales reales y la instantánea de donde provienen, y su libro mayor recalcula cada hash rea escrito.

```figure
re-evidence-ledger
```

## Tres palabras para la confianza, cinco para la autoridad

`confidence`es un enum con tres valores, nunca un número. `authority`Cuando un productor no dice nada, rea utiliza`shipped-artifact`con`observed`¿ Qué ?

| Value | Meaning on the fixture | Records in this lesson |
|---|---|---|
| `observed` | A fact read from bytes or from a run | the ELF reader's section and symbol reads, the two Ghidra records, 17 host runs |
| `derived` | A fact computed from observed records | the switch table model, the division model, every unknown mutation |
| `inferred` | An expectation that no observation states yet | 18 predictions |
| `shipped-artifact` | The file on disk was the source | every read of `ledger-x86_64-O1.o` |
| `controlled-replay` | A run you controlled produced the fact | the host runs of `harness.c` |
| `analyst-inference` | Your own judgement | predictions and unknown mutations |
| `historical-reference` | An older source tree or document | none here |
| `external-service` | A service outside the file and the host | none here |

Dos confusiones cuestan más tiempo. La primera trata de pseudocodo como una observación del comportamiento. Es una observación de lo que Ghidra imprimió. La segunda trata de una lista vacía como prueba de ausencia.`code/main.py`codifica ambas reglas.

## El registro

Un registro tiene exactamente dieciséis campos, y rea 6.5.0 rechaza cualquier otro campo. `code/main.py`mantiene los mismos nombres que rea, por lo que un registro de ambos lados se valida en el otro.`relationship_coverage`, dice cuán completa es una lista de llamados o llamados.`complete`¿ Qué ?`partial`, o`unknown`, o`null`cuando la operación no informa de relaciones de llamadas.

| Field | What it holds |
| --- | --- |
| `evidence_id` | `ev_` and 64 hex digits, a hash of the content |
| `subject` | `name`, `digest.sha256`, `format`, `architecture`, `local_path`, or `null` |
| `provider` | `id`, `name`, `version` or `null` |
| `analysis_profile` | the provider's settings and their digest, or `null`, always present |
| `predicate_type` | a tag, `rea.analysis` by default |
| `operation`, `parameters` | the tool and its input |
| `raw_result`, `normalized_result` | provider output and the canonical result |
| `confidence`, `authority` | the enums above |
| `environment` | `id`, `platform`, `architecture`, `isolation`, or `null` |
| `limitations` | plain sentences |
| `relationship_coverage` | `status` `complete`, `partial`, or `unknown` with diagnostics, or `null` |
| `locations` | `address`, `address-range`, `artifact-path`, `file-offset`, `file-offset-range` |
| `evidence_links` | the `ev_` ids this record rests on |

Este es el verdadero disco que escribió rea para`grade`, cortar a los campos que importan aquí.`code/fixtures/rea-evidence-grade.json`, 37.573 bytes.

```json
{
  "evidence_id": "ev_67b6d3f992eaf49a2e16de084b6ebfdd9884e8c5c7f75e175b7bb20f4f748ce1",
  "subject": {
    "name": "ledger-x86_64-O1.o",
    "digest": { "sha256": "9a4a8082ba5e285a732552981a6935e0c5255725664f380e37c44ebfb04d803f" },
    "format": "elf",
    "architecture": "x86_64",
    "local_path": "/private/tmp/aiefs-re/phases/13-tools-and-protocols/35-evidence-confidence-and-unknowns/code/fixtures/ledger-x86_64-O1.o"
  },
  "provider": { "id": "ghidra", "name": "Ghidra", "version": "12.1.4" },
  "analysis_profile": {
    "provider": { "id": "ghidra", "name": "Ghidra", "version": "12.1.4" },
    "parameters": { "jump_table_evidence": "typed-case-default-blocks-v1", "import_mode": "ephemeral-source-immutable" },
    "digest": "9c7f6b24be69294e61da046e4583e3856cb9e40433b149c1bf8bb534d2fe03ef"
  },
  "operation": "analyze_function",
  "parameters": { "procedure": "grade" },
  "confidence": "observed",
  "authority": "shipped-artifact",
  "environment": null,
  "limitations": [
    "Default-space addresses use lowercase 0x-prefixed hexadecimal; other address spaces use <percent-encoded-space>:0x<hex>.",
    "Resolved reference metadata identifies computed, indirect, external, call, jump, and data edges; unresolved targetless flows remain unknown, and function classifications distinguish thunks and externals."
  ],
  "relationship_coverage": {
    "status": "complete",
    "diagnostics": [
      "Only resolved Ghidra call references are returned; unresolved computed or indirect calls remain unknown, while function classifications distinguish thunks and externals."
    ]
  },
  "locations": [],
  "evidence_links": []
}
```

La verdadera .`parameters`El objeto del perfil tiene 25 claves.`limitations`Una limitación es parte del contenido, así que dejar de lado una cambia la identificación, y su libro mayor rechazaría el registro.

Observe lo que el registro no dice. `environment`¿ Es verdad ?`null`porque Ghidra leyó un archivo y no corrió nada.`locations`está vacío porque este informe de operación se dirige a dentro `normalized_result`- ¿ Qué ?`relationship_coverage`tiene el estatus `complete`Su diagnóstico dice que sólo se devuelven referencias de llamadas resolvidas de Ghidra, y que las llamadas informáticas o indirectas no resueltas permanecen desconocidas.

## El Id es un hilo de contenido, y el camino no está en él

reacomputaciones `evidence_id`¿ Cómo ?`ev_`más SHA-256 sobre el RFC 8785 canónico JSON de una proyección del registro.`digest`¿ Qué ?`format`, y `architecture`y gota .`name`y `local_path`- Se cae .`relationship_coverage`cuando su valor es `null`, así que un registro sin relaciones de llamadas mantiene la misma identidad que antes de que el campo existiera.`raw_result`y cada limitación, se hacha.

RFC 8785 clasifica las claves de objetos, elimina el espacio blanco y fija las formas de número y cadena. Para los registros cuyos números son enteros, la biblioteca estándar de Python produce los mismos bytes:

```python
def canonical_json(value):
    return json.dumps(value, sort_keys=True, separators=(",", ":"), ensure_ascii=False, allow_nan=False)
```

`code/main.py`Rechaza los floats antes de que se hash, porque RFC 8785 y Python imprimen algunos floats de manera diferente.

| rea value | Recomputed by `code/main.py` |
|---|---|
| `ev_67b6d3f992eaf49a…` for `grade` | equal |
| `ev_e88b10dc0c6400e5…` for `percent_change` | equal |
| profile digest `9c7f6b24be69294e…` over `{provider, parameters}` | equal |
| `query_26122aebe9d8abae…` for the `grade` snapshot entry | equal |
| `query_0f95ef38038d3666…` for the `percent_change` entry | equal |

La misma observación de los mismos bytes obtiene la misma identificación en cualquier máquina y en cualquier carpeta.`local_path`En la lista anterior se nombran carpetas en un Mac. La identificación no cambia cuando se copia el dispositivo a `/home/you/re`- ¿ Qué ?`rebind_subject_path()`Te permite mostrar un disco bajo un nuevo camino.`sha256`¿ Qué ?`format`, o`architecture`La diferencia. rea 6.5.0 hace lo mismo con `rebindEvidenceToTargetPath`Cuando reproduce un registro de instantáneas para un objetivo en una nueva ruta.

`validate_evidence()`Recomputa el ID y rechaza una incompatibilidad con la frase de Rea, "El identificador semántico de la evidencia no coincide con su registro".`analysis_profile.provider`difiere de su `provider`- El examen .`test_tampered_id_and_mismatched_profile_are_rejected`Edita una línea de pseudocodo dentro de la real `grade`registrar y ver el libro mayor rechazarlo.

## Observe usted mismo el objeto

Antes de cualquier proveedor, el libro mayor lee el archivo. `observe_object()`analizará los encabezados de las secciones ELF64, `.symtab`¿ Qué ?`.strtab`, y `.rela.text`con`struct.unpack_from`Escribe una .`inspect_object`registro y uno `read_function_bytes`registro por símbolo de función.`observed`con autoridad `shipped-artifact`, porque la fuente era el archivo en el disco.

| Record | Content | Id |
|---|---|---|
| `inspect_object` | five sections with offsets, sizes, and hashes, four functions, three relocations | `ev_645e0a145f28bd5c…` |
| `read_function_bytes percent_change` | 16 bytes at `.text+0x00` | `ev_2572a7d254ad15cb…` |
| `read_function_bytes grade` | 46 bytes at `.text+0x10`, SHA-256 `ffcb75c2a7bb1104…` | `ev_a467c2427bfc0c56…` |
| `read_function_bytes total` | 31 bytes at `.text+0x40` | `ev_b1e845cea2543073…` |
| `read_function_bytes grade_label` | 63 bytes at `.text+0x60` | `ev_c50041be26e6b7f4…` |

`llvm-objdump -t`está de acuerdo: `grade`En el`0000000000000010`con tamaño `2e`Las tres reubicaciones en`.rela.text`¿ Qué ?`R_X86_64_PC32`contra`.rodata`en compensaciones `0x36`¿ Qué ?`0x84`, y `0x93`, con adiciones `0x10`¿ Qué ?`0x24`, y `-0x4`Cada disco lleva un`file-offset-range`ubicación y una limitación: "Los bytes de sección se leen desde el archivo en el disco. El lector no aplica ninguna reubicación y no ejecuta ningún código".

La lección 32 construye este lector en su totalidad. Aquí hay 60 líneas, porque esta lección es sobre lo que usted registra sobre la lectura.

## Derivar la tabla y decir que lo derivó

Los 46 bytes de `grade`contienen cuatro patrones que la lección 33 decodificó:

| Bytes | Instruction | Offset in `grade` |
|---|---|---|
| `48 69 c0 67 66 66 66` | `imul rax, rax, 0x66666667` | 3 |
| `48 c1 f9 22` | `sar rcx, 34` | 13 |
| `83 c1 fa` | `add ecx, -6` | 23 |
| `83 f9 04 77 0c` | `cmp ecx, 4` then `ja` | 28 |

Un multiplicador por `0x66666667`Después de un cambio de 34 se firma una división por 10. El archivo tiene el número 10.`derive_grade_model()`escribe ese hecho como una limitación: "El divisor 10 es una derivación del multiplicador 0x66666667 y el cambio 34. El archivo tiene el número 10. "

El `add ecx, -6`El primer caso se sitúa en 6.`cmp ecx, 4`con un salto sin firmar admite cinco casos, 6 a 10.

El `lea`En el`.text+0x33`lleva la reubicación con adenda `0x10`. Una PC32 reubica medidas desde el final del desplazamiento de 4 bytes, por lo que la tabla comienza en `.rodata+0x14`Cinco palabras de pequeño endio allí leyeron`1, 2, 3, 4, 4`- El interruptor en .`ledger.c`El compilador convirtió el interruptor en una tabla de búsqueda de valores de devolución.

El registro resultante `ev_605106bff2d9998f…`¿ Qué ?`confidence` `derived`, y su `evidence_links`El registro de bytes se encuentra en el libro mayor y el registro derivado tiene un enlace colgante.`ev_a467c242…`¿ Qué ?

Ahora compárate con Ghidra. Su pseudocodo para los mismos bytes:

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

Ghidra aplicó la reubicación y nombró la tabla `DAT_001000b4`, que es `.rodata+0x14`después de que se colocó`.text`En el`0x100000`. Recuperó la división por 10 y los cinco casos. Para el valor predeterminado imprimió una expresión que es siempre 0 y nunca dice 0.`native_value_flow`marca esas operaciones `is_dead: true`El registro es una fiel observación de Ghidra. El significado que se lee de él se deriva, y el valor que se espera de una carrera se deduce.

El `percent_change`El registro muestra el otro fracaso.`undefined1 [16] percent_change(int param_1,int param_2,ulong param_3)`La fuente dice:`int percent_change(int before, int after)`- ¿ Qué ?`idiv`escribe `EDX:EAX`, y el camino cero sale .`EDX`No se ha tocado, así que el descompilador dedujo una vuelta de 16 bytes y un tercer parámetro.`native_api.return_type.confidence`por`grade`¿ Es verdad ?`"low"`Tratar una firma recuperada como la inferencia del proveedor, y registrarla de esa manera.

## Las predicciones son registros inferidos

Una predicción es una afirmación sobre el comportamiento que ninguna observación ha declarado. `claim_suite()`construye dieciocho de ellos a partir de los modelos derivados y de la fuente. `record_prediction()`almacena cada uno como un disco con `confidence` `inferred`y `authority` `analyst-inference`, vinculado a los registros en los que se apoya.

| Claim | Predicted value | Rests on |
|---|---|---|
| `percent_change(12400, 15500)` | 25, from 310000 / 12400 | division model `ev_076a9167…` |
| `percent_change(0, 5)` | 0, the zero guard | division model |
| `percent_change(100, 50)` | -50 | division model |
| `percent_change(-7, 3)` | -142, because `idiv` truncates toward zero | division model |
| `grade(55)` | 0, index -1 fails the unsigned compare | table model `ev_605106bf…` |
| `grade(60)`, `grade(69)` | 1, 1 | table model |
| `grade(70)`, `grade(85)` | 2, 3 | table model |
| `grade(95)`, `grade(100)`, `grade(105)` | 4, 4, 4 | table model |
| `grade(110)`, `grade(-5)` | 0, 0 | table model |
| `total(1, 2, 3)`, `total()` | 6, 0 | source record `ev_2f1e6d25…` |
| `grade_label(85)` | `"B"` | source record and table model |
| `grade(score)` for 90 to 100 | 4 for all eleven inputs | table model |

`predict()`utiliza la semántica C, no la semántica Python. `c_div()`truncates hacia cero, y `wrap32()`Pone cada intermedio en 32 bits.`//`rodadas hacia el infinito negativo, así que `(3 - -7) * 100 // -7`La lección 36 hace que esta cuadrícula sea mucho más grande.

Cada predicción abre un desconocido residual.`observed`pruebas con autoridad `controlled-replay`y recomienda una sonda:`execute_function`La siguiente sección produce esa evidencia. La pregunta para la afirmación de primer grado dice "¿Se califica ((95) == 4 cuando se ejecuta la función compilada?"

## Compruebe si se comparan con una carrera

`code/fixtures/harness.c`incluye `ledger.c`y imprime un resultado por invocación. `observe_runs()`lo compila con la primera de `clang`¿ Qué ?`cc`, o`gcc`En el camino y ejecuta cada entrada en la cuadrícula una vez.

```json
{
  "operation": "execute_function",
  "parameters": { "function": "grade", "args": [95] },
  "normalized_result": { "result": 4, "stdout": "4\n", "exit_code": 0 },
  "confidence": "observed",
  "authority": "controlled-replay",
  "environment": { "id": "host-process", "platform": "darwin", "architecture": "arm64", "isolation": "process" },
  "limitations": [
    "The run executed a host build of ledger.c, not the x86-64 object. The claim about the object rests on the shared source and on the compiler.",
    "One run observes one input. It proves nothing about other inputs."
  ]
}
```

El tema de un registro de ejecución es el arnés binario, un archivo Mach-O arm64 en este Mac, con su propio SHA-256.`course-harness`con la línea de versión del compilador, aquí `Apple clang version 17.0.0 (clang-1700.6.3.2)`- El .`evidence_links`Nombre del registro fuente `ev_2f1e6d251543d017…`, porque la carrera y el objeto comparten`ledger.c`Y nada más.

La primera limitación se lee dos veces. El objeto x86-64 nunca se ejecutó en esta máquina. Lo que se ejecutó fue una segunda construcción de la misma fuente para otra arquitectura. Eso sigue siendo una observación, y es la más fuerte disponible sin un emulador, por lo que la afirmación pasa con esa frase adjunta. Un informe que omite la ha perdido la información que el lector necesita.

Las 17 entradas únicas coinciden con las predicciones.`code/fixtures/run-evidence.json`posee esos 17 registros y el registro de origen, exportado con `--record`En 2026-10-13. Cuando no hay compilador presente,`build_ledger()`El informe de la Comisión de Agricultura y Desarrollo de la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Comisión de Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la Comisión de Agricultura y la Agricultura, la`runs`campo.

## Reglas de promoción

Una reclamación pasa de inferida a apoyada sólo a través de ejecuciones observadas. `check_claim()`Busca por`execute_function`registros con la misma función y argumentos, compara cada resultado con la predicción y cuenta las entradas cubiertas:

| Status | Condition | rea's completion ledger |
|---|---|---|
| `pass` | every input has an agreeing run and none disagrees | `pass` |
| `fail` | any run disagrees | `fail` |
| `unknown` | some inputs have runs, others have none | `unknown` |
| `unsupported` | no input has a run | `unsupported` |

`promote_claim()`Es la forma estricta.`ClaimError`cuando cualquier identificación es `inferred`o `derived`. Se eleva cuando un id nombra una serie de otra función u otros argumentos. Se eleva cuando la lista está vacía y cuando las carreras cubren menos entradas que la afirmación.`grade(55)`, y ninguno puede promoverlo.`ev_7be90c0c…`en la figura y ver la reclamación caer a `unsupported`Mientras que dos registros sobre `grade`No se vuelva.

La afirmación de rango muestra el caso parcial. Sólo dos de sus once entradas, 95 y 100, aparecen en la cuadrícula de ejecución. `check_claim()`retorno `unknown`con`covered`2 de 11, y `promote_claim()`se niega con "2 de 11 entradas han observado ejecuciones". El libro mayor no redondea eso.

`completion_ledger()`Espejos rea `EvidenceCompletionLedger`Su identificación es`ecl_`más el hash de sus registros, y `complete`es verdad sólo cuando`pass`igual `total`Las huellas de demostración .`ecl_da64933ea13befed…`con`complete: false`, debido a esa reivindicación de un rango.

## Los desconocidos son registros con una cadena de hash

rea almacena un residuo desconocido como un registro estricto. `code/main.py`mantiene los mismos campos y las mismas invariantes.

El ID es`unk_`más SHA-256 por encima `{domain, question, scope_digest}`Dos analistas que hacen la misma pregunta sobre el mismo archivo obtienen la misma identificación.`scope_digest`es el SHA-256 del objeto.

Cada cambio es una nueva revisión.`previous_revision_digest` `null`Cada revisión posterior nombra a los`revision_digest`de la anterior, y la digestión cubre todo campo excepto a sí misma.`verify_unknown_chain()`El sistema de corrección de datos de la lista de cambios de datos de la red de datos de la red de corrección de datos de la red de datos de la red de corrección de datos de la red de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de la red de corrección de datos de corrección de datos de la red de corrección de corrección de datos de la red de corrección de corrección de datos de corrección de corrección de datos de la red de corrección de corrección de corrección de datos de corrección de corrección de corrección de corrección de datos de corrección de corrección de corrección de corrección de datos de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de corrección de

Cada mutación es en sí misma un registro de pruebas, como lo hace con el proveedor `rea-unknown-registry`Nuestros cargos .`confidence` `derived`¿ Qué ?`authority` `analyst-inference`Su limitación dice: "Una mutación registrada registra lo que el analista decidió.

Es por eso que los 81 registros de la demostración incluyen 36 registros de mutación derivados: 18 aberturas y 18 actualizaciones.

Se aplican tres invariantes en cada revisión:

- Sólo un`resolved`Un desconocido lleva un `resolution`, y sólo la revisión 1 no tiene un predecesor.
- ¿ Qué es esto ?`contradicted`nombres desconocidos al menos un identificador contradictorio, y ningún identificador aparece en ambas listas.
- ¿ Qué es esto ?`verified`la resolución nombra al menos una identificación de la prueba con la confianza y autoridad requeridas.

La tercera regla es donde el libro mayor se niega con más frecuencia. Las notas de flujo de trabajo de rea dicen "Sólo las pruebas observadas calificadas pueden verificar una resolución". `resolve_unknown()`Por lo tanto, rechaza el modelo de tabla, rechaza un registro observado con autoridad `shipped-artifact`, y acepta el registro de carrera `ev_f4ffbe8f922ec7fe…`. La revisión resuelta para `grade(95)`es la revisión 2, y su `previous_revision_digest`¿ Es verdad ?`ffca840b55a8a188…`, el resumen de la revisión 1.

`update_unknown()`¿ Qué es ?`expected_revision`y eleva .`revision-conflict`Cuando está envejecido, como es rea.`update_unknown`La reacción correcta es la que nos dan las notas de Rea: volver a leer la revisión actual y decidir de nuevo.

Un desconocido permanece abierto después de la demostración:`unk_f2eb0d7f84c80a19…`, "¿Clasifica la puntuación) == 4 para cada puntuación de 90 a 100 mantener cuando la función compilada se ejecuta?" Su estado es `investigating`, sus IDs de apoyo incluyen las dos carreras, y su sonda todavía nombra las nueve entradas sin una carrera.

## Un caché de respuestas no es una base de datos para análisis

Rea tiene tres cosas que se parecen a una caché.`-readOnly -deleteProject`y eliminado al cerrar.`AnalysisSnapshotCache`El archivo de instantáneas contiene hasta 10.000 entradas para una identidad binaria.`target`¿ Qué ?`binding`¿ Qué ?`entries`¿ Qué ?`workflow_entries`, y un `evidence_bundle`¿ Qué ?

La llave de caché es `query_`más SHA-256 por encima `{target, binding, operation, parameters}`- ¿ Qué ?`target`¿ Es verdad ?`{sha256, kind, format, architecture}`, y un `raw-binary`El objetivo también ha`entry_point`- ¿ Qué ?`binding`El análisis de los datos de los datos de los usuarios es el proveedor y todo el perfil de análisis, incluido el digesto.`jump_table_evidence`de la`typed-case-default-blocks-v1`a una `v2`, y el digesto cambia, así que cada clave cambia.

`code/fixtures/rea-snapshot.json`Es el archivo rea escrito durante las dos ejecuciones, 142.690 bytes. rea lo escribió con permisos exclusivos del propietario, y la copia comprometida es legible para todos. Tiene dos entradas y un paquete de pruebas con los mismos dos registros. `AnswerCache.import_snapshot()`recalcula cada uno `query_id`Y rechaza el archivo cuando cualquier difiere.`target`o `binding`Se diferencia de la activa, con la redacción de rea `profile_mismatch`Rechaza una instantánea cuya`format_version`No es`course-answer-cache-1`¿ Qué ?

Tres mediciones de esta máquina.`grade`La consulta tardó 8,6 segundos, con una nueva importación de Ghidra.`percent_change`La consulta contra el mismo instantáneo tomó 8,5 segundos, porque una consulta diferente es una falta de caché.`grade`La consulta tardó 2,1 segundos y devolvió el registro idéntico con el mismo `evidence_id`. La reproducción en memoria en rea añade la limitación "Cargado desde una instantánea de análisis local de REA; esta llamada no re-ejecutó al proveedor". Nuestro `lookup()`añade una oración del mismo tipo, para que el lector pueda distinguir una repetición de una respuesta fresca.

Una mutación termina la caché. en rea 6.5.0, un cambio de nombre o un comentario elimina la caché de instantáneas antes de que rea envíe la llamada al proveedor. Una respuesta fallida no puede probar que la escritura se haya vuelto a rodar, por lo que una escritura fallida también detiene la reproducción. Una importación de metadatos que cambia la base de datos hace lo mismo.

## Por qué un agente necesita esto

Un agente que maneja rea a través de MCP ve el registro completo de cada llamada y puede citar`evidence_id`En el estudio de NDSS BAR, mostrando contenido binario como datos solos, todavía quedaron 15 propuestas inseguras de 40, y las comprobaciones de procedencia con autorización de herramienta las eliminaron.[arXiv 2606.06838](https://arxiv.org/abs/2606.06838)En binarios con cientos de funciones, sólo entre el 10 y el 15 por ciento recibieron análisis real antes de que el agente produjo estubes.

El libro mayor responde a cada uno de estos con un registro, y el agente informa sólo lo que el libro mayor contiene.`unsupported`Una reclamación con algunas carreras es`unknown`Un mensaje de proveedor es`observed`No hay más que darle al agente el archivo de habilidades.`outputs/`Y hacer que el libro mayor sea el único lugar donde se escriben conclusiones que se aplica a Claude, Cursor, Codex, OpenClaw, Hermes, o cualquier agente que lea SKILL.md.

## Construye el mismo

Ejecutar la demostración de la biblioteca estándar:

```bash
cd phases/13-tools-and-protocols/35-evidence-confidence-and-unknowns
python3 code/main.py
```

La demostración lee el objeto, importa los dos registros rea, verifica la instantánea rea y deriva los dos modelos. Luego registra 18 predicciones, abre 18 desconocidas, compila y ejecuta el arnés, verifica cada reclamo y resuelve 17 desconocidas.

```json
{
  "records": 81,
  "by_confidence": { "observed": 25, "derived": 38, "inferred": 18 },
  "rea_snapshot": { "entries": 2, "replay_hit": true, "stale_profile_refused": true },
  "grade_table": [1, 2, 3, 4, 4],
  "runs": "live: Apple clang version 17.0.0 (clang-1700.6.3.2)",
  "claims": [ { "claim_id": "grade(95)", "status": "pass", "covered": "1/1" }, { "claim_id": "grade(90..100)", "status": "unknown", "covered": "2/11" } ],
  "completion_ledger_id": "ecl_da64933ea13befed9483f8b00447bb3f4ceb74324c01e2515eff03f3af3ca750",
  "complete": false,
  "unknowns": { "resolved": 17, "investigating": 1 }
}
```

Sin un compilador C,`runs`Leía`imported 17 recorded run records from run-evidence.json`y el resto es lo mismo.

Leer .`code/main.py`en este orden:

1. `canonical_json()`y `compute_evidence_id()`Mantenga la receta de hash y la proyección semántica.
2. `create_evidence()`y `validate_evidence()`construir y comprobar un registro.
3. `EvidenceLedger.record()`y `import_bundle()`almacenar registros y rechazar conflictos.
4. `open_unknown()`¿ Qué ?`update_unknown()`, y `resolve_unknown()`Mantenga la cadena de revisión.
5. `observe_object()`se lee el ELF, y `derive_grade_model()`Leía la mesa.
6. `claim_suite()`¿ Qué ?`predict()`, y `record_prediction()`escribe el nivel inferido.
7. `observe_runs()`transforma cada arnés en un registro observado.
8. `check_claim()`¿ Qué ?`promote_claim()`, y `completion_ledger()`aplicar las normas de promoción.
9. `AnswerCache`Las llaves responden como las instantáneas de Rea y rechaza las viejas.

## Usalo

Mantenga un libro mayor por objetivo y por investigación. Ponga el SHA-256 del objetivo en cada tema y el alcance de la pregunta en cada desconocido.`provider`Cuando reconstruyes el archivo, el sujeto se altera y cada registro se convierte en evidencia de otro archivo.

Escribe las limitaciones en el momento en que las conoces. El lector que no aplica ninguna reubicación lo dice. La ejecución de una construcción de host lo dice. El modelo que infere un divisor lo dice. Un registro escrito sin sus limitaciones parece más fuerte de lo que permiten los hechos, y un lector posterior no puede reparar eso.

Tratar a cada texto de proveedor como datos. Las cadenas y pseudocodes del objetivo son textos que el objetivo controla.`_meta.provenance.untrusted_content: true`Las instrucciones del servidor le dicen al modelo que trate el contenido objetivo como datos y que informe cualquier solicitud en él para ejecutar, instalar, descargar o enviar algo como un intento de inyección rápida.

## Laboratorio interactivo

### Laboratorio A: recomputar una identificación real

Desde el`code`directorio, Python abierto:

```bash
cd phases/13-tools-and-protocols/35-evidence-confidence-and-unknowns/code
python3 -q
```

- ¿Qué quieres decir ?

```python
from main import *
record = load_evidence_file(FIXTURES / "rea-evidence-grade.json")
record["evidence_id"] == compute_evidence_id(record)
record["subject"]["local_path"] = "/home/you/re/ledger-x86_64-O1.o"
record["evidence_id"] == compute_evidence_id(record)
record["limitations"].pop()
record["evidence_id"] == compute_evidence_id(record)
```

Las primeras dos comparaciones son ciertas. El camino no está en el hash. La tercera es falsa, porque una limitación está contenida.

### Laboratorio B: rechazar un ascenso

```python
ledger, report = build_ledger(live_runs=False)
claim = [c for c in claim_suite(ledger.find(operation="derive_switch_model")[0], ledger.find(operation="derive_arithmetic_model")[0], ledger.find(operation="read_source")[0]) if c.claim_id == "grade(55)"][0]
promote_claim(ledger, claim, [ledger.find(operation="derive_switch_model")[0]["evidence_id"]])
```

La llamada se eleva .`ClaimError`y nombra el ID derivado. Ahora pasa el ID de ejecución:

```python
run = run_records_for(ledger, "grade", [55])[0]
promote_claim(ledger, claim, [run["evidence_id"]])["status"]
```

El estado es `pass`, y los regresados `limitations`llevar la sentencia de construcción de anfitrión.

### Laboratorio C: rechazar una instantánea obsoleta

```python
import json
snapshot = json.loads(REA_SNAPSHOT_PATH.read_text())
target, binding = rea_snapshot_binding(snapshot)
AnswerCache(target, binding).import_snapshot(snapshot)
stale = dict(binding["analysis_profile"]["parameters"], jump_table_evidence="typed-case-default-blocks-v2")
AnswerCache(target, {"provider": binding["provider"], "analysis_profile": make_profile(binding["provider"], stale)}).import_snapshot(snapshot)
```

El primer importe de la exportación 2. El segundo aumento `CacheError`con`profile_mismatch`Una cadena de versiones cambió y la instantánea entera se rechaza.

### Lab D: rompe la cadena

```python
unknown_id = [u for u, r in ledger.unknowns.items() if r[-1]["status"] == "investigating"][0]
ledger.verify_unknown_chain(unknown_id)
ledger.unknowns[unknown_id][0]["severity"] = "critical"
ledger.verify_unknown_chain(unknown_id)
```

La primera llamada regresa 2. La segunda se eleva.`EvidenceIntegrityError`, porque la revisión 1 ya no coincide con su propio digesto y la revisión 2 pone en un digesto que ya no existe.

## Laboratorio de práctica

Cierra el abierto desconocido y agrega un segundo proveedor.

Requisitos:

- Extenderse`run_grid()`Así que las nueve entradas faltantes de la demanda de rango se ejecutan, luego confirmen los movimientos desconocidos a `resolved`en la revisión 3.
- Añadir`llvm-objdump -d`como segundo proveedor con su propio `provider`objetar y registrar su lista de `grade`¿ Cómo ?`observed`¿ Qué ?
- Escriba un`derived`registro que indica las dos listas, Ghidra y llvm-objdump, decodificar las mismas 14 instrucciones, y vincular ambas.
- Abre un desconocido para `grade_label(105)`con severidad `high`, ya que el índice de la tabla 4 dice `labels[4]`, y resolverlo con una carrera.
- Exporta el libro mayor con `export_bundle()`, importarlo en un nuevo libro mayor, y demostrar que cada cadena todavía verifica.

El éxito significa un informe en el que cada oración cita un ID del paquete y el uno `unknown`El reclamo se ha ido.

## Laboratorio de calificación

Cinco preguntas sobre los accesorios de esta lección.`code/lab_answers.json`Escribir números como números o como cuerdas hexáticas como `"0x40"`, y listas como listas JSON.

1. `t1`: el ID de pruebas que da la computación_evidencia_id para el grado de pruebas reales.
2. `t2`: el número de limitaciones en el registro de calificaciones.
3. `t3`: el estado de relación_cubrición del registro de calificaciones.
4. `t4`: la confianza y la autoridad del registro de calificaciones, como una lista de dos cadenas.
5. `t5`: el SHA-256 del objeto del registro de calificación.

Compruebe sus respuestas:

```bash
cd phases/13-tools-and-protocols/35-evidence-confidence-and-unknowns
python3 code/lab.py
```

El verificador calcula cada valor esperado desde `code/main.py`En los accesorios comprometidos.`PASS`¿ Qué ?`FAIL`o `TODO`para cada tarea y una puntuación, y sale de 0 sólo cuando pasan los cinco. `python3 code/lab.py my-answers.json`- ¿Qué? - Vea otro archivo.`code/tests/test_lab.py`prueba que el comprobador acepta las respuestas correctas y rechaza las incorrectas.

### Ve cómo se ejecuta

El [Inside a Decompiler](https://aiengineeringfromscratch.com/inside-a-decompiler?symbols=off)La máquina mantiene un libro mayor de pruebas con cuatro bandejas: observado, derivado, inferido e desconocido. Este enlace abre la ejecución con la tabla de símbolos apagada, por lo que el nombre y el final del archivo de función son desconocidos y los archivos de inicio como inferidos.

## Lección Artefacto

Esta lección produce`outputs/skill-evidence-ledger.md`. Le dice a un agente cómo escribir cada hallazgo como un registro con confianza, autoridad, limitaciones y enlaces.`pass`¿ Qué ?`fail`¿ Qué ?`unknown`, o`unsupported`¿ Qué ?

## Verifique el hecho

Ejecutar la demostración y la suite determinista:

```bash
cd phases/13-tools-and-protocols/35-evidence-confidence-and-unknowns
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

Las 28 pruebas demuestran:

- ambas rea `evidence_id`valores, el perfil de la digestión, y ambos instantáneos `query_id`Los valores se recomputan a partir del contenido
- una ruta cambiada mantiene la identificación y una limitación cambiada la cambia
- se rechazan un registro manipulado, un proveedor de perfil descoincidente, un float y un valor enum desconocido
- un registro duplicado de los resultados `duplicate`, un récord contradictorio se eleva, y una importación de paquete es atómica
- las importaciones de paquetes de explotación registradas con 18 registros y sin vínculos colgantes
- Un desconocido se abre en la revisión 1, se niega a una obsoleta `expected_revision`, y se resuelve sólo con observado `controlled-replay`pruebas
- una revisión editada rompe la cadena
- La promoción rechaza las pruebas inferidas, derivadas, vacías y no relacionadas, y pasa con un resultado de acuerdo
- cobertura parcial es `unknown`y una carrera en desacuerdo es `fail`
- la caché se golpea en la clave exacta y se pierde en otros parámetros
- la caché rechaza una versión de perfil golpeado, un objetivo equivocado, una versión de formato equivocado, y una clave manipulada
- El lector del ELF está de acuerdo con `llvm-objdump`sobre los desplazamientos, tamaños, bytes y reubicaciones, y la tabla se lee `1, 2, 3, 4, 4`
- Las predicciones utilizan la división C, y las carreras registradas coinciden con las 17 afirmaciones de entrada única
- con un compilador presente, las carreras en vivo coinciden también

## Modos de falla de producción

| Failure | What the weak report says | What the ledger requires |
|---|---|---|
| Pseudocode read as behaviour | "grade returns 0 for failing scores" | an `execute_function` record for that input, or `unsupported` |
| Decompiler signature trusted | "percent_change returns 16 bytes" | the signature marked as the provider's inference with its `low` confidence |
| Empty list read as absence | "the function has no callers" | an `unknown` status, or a coverage field when the tool provides one |
| Path hashed into the id | "the record changed" after a copy | the semantic projection without `name` and `local_path` |
| Limitation dropped from a copy | a record that looks stronger than the source | id mismatch and rejection on import |
| Partial grid rounded up | "all scores from 90 pass" | `unknown` with `covered` 2 of 11 |
| Host run presented as the object's run | "the x86-64 object returned 25" | the host-build limitation carried on the claim |
| Unknown closed by a sentence | "resolved, see above" | a `verified` resolution with observed ids of the required authority |
| Stale revision overwritten | two analysts' edits collide | `revision-conflict` and a reread |
| Old snapshot replayed after a release | answers from a changed contract | `profile_mismatch` on import |
| Cache kept after a rename | a replay that hides a metadata write | invalidation before dispatch and a failed export |

## Regla de funcionamiento

Registra lo que has leído, lo que has calculado y lo que esperas como tres tipos de registro con tres palabras. Cierra un desconocido con un id, nunca con una oración. Repite una respuesta solo bajo la clave exacta con la que se guardó, y rechace cuando una cadena de versiones difiere.

## Leer más

- [rea 6.5.0 source, `src/domain/evidence.ts` and `src/domain/residualUnknown.ts`](https://github.com/morluto/rea)
- [RFC 8785, JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785)
- [Ghidra 12.1.4 release](https://github.com/NationalSecurityAgency/ghidra/releases)
- [Agent Failure Patterns in CTF Reverse Engineering, NDSS BAR 2026](https://www.ndss-symposium.org/wp-content/uploads/bar2026-58.pdf)
- [When Binaries Talk Back: Representation-Confusion Attacks on LLM-Assisted Reverse Engineering, arXiv 2607.12507](https://arxiv.org/abs/2607.12507)
- [Agentic decompilation study, arXiv 2606.06838](https://arxiv.org/abs/2606.06838)
- [When LLM Decompilers Recompile More and Preserve Less, arXiv 2609.05370](https://arxiv.org/abs/2609.05370)
- rea 6.5.0 fuente en etiqueta `3b79af2`que esta lección describe: `src/domain/evidence.ts`¿ Qué ?`.agents/skills/reverse-engineer-anything/SKILL.md`¿ Qué ?`docs/cli.md`¿ Qué ?`src/domain/reconstructionVerification.ts`¿ Qué ?`src/application/ReconstructionObligationCandidates.ts`¿ Qué ?`docs/tool-design.md`¿ Qué ?`src/domain/evidenceCompletionLedger.ts`¿ Qué ?`src/domain/residualUnknown.ts`¿ Qué ?`src/application/investigation/UnknownEvidence.ts`¿ Qué ?`.agents/skills/reverse-engineer-anything/references/evidence-workflows.md`¿ Qué ?`src/application/binary/AnalysisSnapshotCache.ts`¿ Qué ?`src/domain/analysisSnapshot.ts`¿ Qué ?`src/application/binary/BinarySession.ts`¿ Qué ?`src/application/binary/BinarySessionRecords.ts`¿ Qué ?`docs/mcp-contracts.md`¿ Qué ?
