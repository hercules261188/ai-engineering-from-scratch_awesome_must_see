# Levantamento para Pseudocode: IR, SSA e Estruturação a partir do zero

> A saída do decompiler é uma reivindicação com uma derivação oculta. Quando você constrói a derivação você mesmo, cada linha dessa saída se torna uma coisa que você pode verificar.

**Type:** Build
**Languages:** Python, C
**Prerequisites:** Phase 13 · 32 (ELF from scratch), Phase 13 · 33 (disassembly and control flow)
**Time:** ~150 minutes

## Objetivos de aprendizagem

- Elevador decodificou instruções x86-64 para uma pequena IR com bandeiras explícitas, cargas, lojas, ramos, chamadas e retornos.
- Transforme os espaços de quadro em variáveis locais e os registros de argumentos do Sistema V em parâmetros.
- Construir SSA mínimo com funções phi nas fronteiras de domínio, em seguida, substituir, simplificar e remover código morto.
- Recuperar`if`- Não .`if`com`else`- Não .`while`, e `do while`do gráfico de fluxo de controlo com pós-dominatores e bordas traseiras.
- Explica o multiplicar por `0x66666667`como uma divisão por 10 e provar a identidade.
- Leia o pseudcódigo Ghidra para as mesmas funções linha por linha e nomeie a instrução, ou a suposição, por trás de cada diferença.

## O problema

Perguntaste à Rea o pseudônimo de `percent_change`em `ledger-x86_64-O1.o`Ghidra 12.1.4 respondeu em cerca de dez segundos:

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

A fonte tem quatro linhas de comprimento.`int`valores e retornos um `int`Ghidra relata um valor de retorno de 16 bytes e um terceiro parâmetro que a fonte não tem. Os dezesseis bytes de código de máquina são corretos, e Ghidra decodificou cada um deles corretamente. A assinatura é uma adivinhação, e nada no texto marca-o como um.

rea retorna este texto como o `procedure_pseudo_code`O resultado e liga uma limitação ao registro. A limitação diz: "Pseudocode é Ghidra decompiler saída, não fonte original e não texto equivalente à saída Hopper".`compare_functions`O instrumento difere pseudocód como linhas de texto. Seu próprio registro de dimensões diz "Assembly é texto opaco do provedor" e que a normalização de relocação não está disponível. rea não pode olhar para dentro do texto, e nem um agente que apenas lê-lo.

O registro da rea DX-Ball mostra o custo.`FUN_00406400(void)`- O que é ?`__ftol()`A visão de instrução mostrava uma entrada de pilha e uma expressão x87.

Esta lição constrói a derivação que o decompiler esconde. Você começa com as 56 instruções que a lição 33 decodificou, levanta-as para um IR, renomeia em SSA, simplifica e estrutura. Então você coloca seu pseudocód next Ghidra's para`percent_change`- Não .`grade`, e `total`Os testes compilam a sua saída com clang e executam-na contra a fonte através de uma grade de entrada.

```figure
lifting-stages-percent-change
```

## A Fixtura e o Oráculo

`code/fixtures/ledger.c`é a fixação compartilhada desta parte do curso:

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

`grade_label`As três funções acima são as que esta lição testa.

O objeto foi construído com `clang -target x86_64-unknown-linux-gnu -O1 -fno-asynchronous-unwind-tables -c ledger.c`- O .`-O0`O objeto fica ao lado dele para a seção de slots de pilha. `code/fixtures/README.md`lista ambos os comandos, o SHA-256 de cada objeto, e o comando rea que produziu os fixadores Ghidra.

O oráculo para o decodificador é `llvm-objdump -d -r`O texto para ambos os objetos é comprometido como`ledger-x86_64-O1.objdump.txt`E ...`ledger-x86_64-O0.objdump.txt`Um teste analisa esses ficheiros e compara cada instrução por endereço, por bytes e por texto da AT&T.`-O1`e todas as instruções em `-O0`Quando ?`llvm-objdump`O teste é feito no caminho fixado, um segundo teste é feito ao vivo e compara novamente.

Aqui está .`percent_change`Como o oráculo imprime:

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

## Fase 1: Bytes para instruções

A lição 33 construiu o decodificador. Esta lição mantém uma pequena cópia para que o código funcione por conta própria. A cópia cobre exatamente os códigos opcodes que são emitidos para`ledger.c`- Não .`-O0`E ...`-O1`Para qualquer outra coisa que eleve .`DecodeError`com o endereço e o byte do código op.

O decodificador segue a ordem de bytes do Intel SDM, Volume 2, secção 2.1. Os prefixos vêm primeiro, depois o opcode, depois o ModR/M, depois o SIB, depois o deslocamento, depois o imediato.`0x40`- Não .`0x4F`O REX.W seleciona um operando de 64 bits, o REX.R estende o `reg`O campo REX.X estende o índice SIB e o REX.B estende `r/m`A secção 2.1.3 define os três campos ModR/M `mod`- Não .`reg`, e `r/m`, e os três campos SIB `scale`- Não .`index`, e `base`- Em modo de 64 bits.`mod = 00`com`r/m = 101`significa um deslocamento de 32 bits em relação à próxima instrução (seção 2.2.1.6).

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

O `leaq (%rip), %rcx`- Não .`grade+0x23`O objeto carrega uma realocação.`R_X86_64_PC32.rodata+0x10`em compensação`0x36`O elevador resolve-o na fase 3.

## Fase 2: Blocos e dominadores

Um bloco básico é uma sequência de declarações com uma entrada no topo e uma saída na parte inferior.`build_blocks()`Reúne líderes: a entrada da função, cada alvo de salto dentro da função e cada endereço após um salto ou retorno. Cada bloco recebe sucessores de sua última instrução.`jmp`- Não, não.`ret`Não tem nenhum, e um bloco que termina numa instrução ordinária passa para o próximo líder.

`total`- Não .`-O1`tem cinco blocos:

| Block | Instructions | Successors |
|---|---|---|
| `0x40` | `testl %esi, %esi` then `jle 0x5c` | `0x5c`, `0x44` |
| `0x44` | `movl %esi, %ecx`, `xorl %edx, %edx`, `xorl %eax, %eax`, `nopw` | `0x50` |
| `0x50` | `addl (%rdi,%rdx,4), %eax`, `incq %rdx`, `cmpq %rdx, %rcx`, `jne 0x50` | `0x50`, `0x5b` |
| `0x5b` | `retq` | exit |
| `0x5c` | `xorl %eax, %eax` then `retq` | exit |

`immediate_dominators()`O algoritmo iterativo de Cooper, Harvey e Kennedy (2001) percorre os blocos em pós-ordem inversa e intersecta os dominadores dos predecessores processados até que nada mude.`total`Diz-se que`0x44`Domina .`0x50`Também diz que`0x50`domina-se através da borda traseira, que é como estágio 8 encontra o ciclo.

A mesma função funciona duas vezes mais.`dominance_frontiers()`utiliza- o para a SSA na fase 5. `post_dominators()`corre no gráfico invertido com um nó de saída virtual para encontrar conjuntos na etapa 8.

## Fase 3: Levantamento para uma pequena IR

O IR tem seis tipos de declarações: `Assign`- Não .`Store`- Não .`Branch`- Não .`Jump`- Não .`IndirectJump`, e `Return`. Expresões constantes, variáveis, operações binárias com largura, castes (`sext`- Não .`zext`- Não .`trunc`Os cinco outros tipos existem, mas serão simplificados mais tarde:`Flags`- Não .`Cond`- Não .`Concat`- Não .`SignFill`, e `Symbolic`- Não .

Registros são variáveis nomeadas pela sua família de 64 bits.`%eax`é um `Assign`- Não .`rax`com largura 32, e uma leitura de `%eax`é uma leitura de `rax`Em modo de 64 bits, uma escrita de 32 bits se estende a zero no registro completo, de modo que uma leitura posterior de 64 bits do mesmo valor é um `zext`O elevador não conhece a definição de alcance, por isso registra a largura de leitura e permite à SSA inserir o molde na fase 5.

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

### As bandeiras são uma variável

Cada instrução que define os códigos de condição escreve uma variável chamada `flags`, e cada salto condicional lê-lo.`cmp a, b`escreve`cmp(a, b)`, a `test a, b`escreve`test(a, b)`, e `sub`- É o mesmo .`cmp`A forma como efeito colateral.`jcc`torna-se`Branch(Cond(cc, flags), target, fallthrough)`O código de condição determina o operador de comparação quando os dois se encontram na fase 6:

| Code | After `cmp(a, b)` | After `test(a, a)` |
|---|---|---|
| `je` | `a == b` | `a == 0` |
| `jne` | `a != b` | `a != 0` |
| `jl`, `jge`, `jle`, `jg` | signed `<`, `>=`, `<=`, `>` | signed compare with 0 |
| `jb`, `jae`, `jbe`, `ja` | unsigned `<`, `>=`, `<=`, `>` | unsigned compare with 0 |

`xorl %eax, %eax`é levantado diretamente para `eax = 0`As bandeiras que ele coloca são escritas como:`add`As bandeiras de estilo que não saltam na fixação.`xor`Idioma é importante em `grade`, onde Ghidra perdeu a cabeça.

### CGT e IDEV

`cltd`- Não .`%edx`com o sinal de `%eax`- Não .`idivl %edi`Divide o par de 64 bits `%edx:%eax`Por`%edi`, coloca o coeficiente em `%eax`, e o resto em `%edx`O levantador escreve o par num nome temporário .`dividend`O restante da declaração, então, lê o antigo`%eax`No SSA, a metade superior aparece como`signfill`O simplificador reduz o par para a metade baixa, e a divisão se torna um 32-bit `/`- Não .

### Figura 1 - 0

- Não .`-O0`O Clang mantém todos os locais no quadro.`percent_change`Lojas`%edi`- Não .`-0x8(%rbp)`E ...`%esi`- Não .`-0xc(%rbp)`O levantador observa para que não haja uma armação.`pushq %rbp`seguida por `movq %rsp, %rbp`Depois do prólogo, uma ópera de memória com base.`%rbp`, sem índice, e um deslocamento se torna uma variável chamada `local_<hex>`- Não .`-0x8(%rbp)`É o que é`local_8`- Não .`-0xc(%rbp)`É o que é`local_c`- Não .

As próprias instruções de quadro não produzem IR, e nem o fazem.`subq`E ...`addq`- Não .`%rsp`Qualquer outro operando de memória permanece um`Load`ou `Store`- Não .

### Relocalizações

O `leaq (%rip), %rcx`em `grade`O deslocamento em compensação`0x36`Diz:`R_X86_64_PC32.rodata+0x10`O linker escreverá .`S + A - P`- Para o campo.`S`é o endereço de `.rodata`- Não .`A`é o adicionado`0x10`, e `P`O comando calcula o número de pontos de referência.`next_ip + disp = (P + 4) + S + A - P = S + A + 4`- Não .

O levantador aplica a mesma aritmética sem um linker. O objetivo de compensação é o adendo mais a distância do campo até o final da instrução, `0x10 + 4 = 0x14`A IR diz:`rcx = &.rodata+0x14`E a mesa ali tem os cinco .`int`Valores`1, 2, 3, 4, 4`- Não .

## Fase 4: Parâmetros da Convenção de Convocatória

O sistema V AMD64 psABI, secção 3.2.3, aprova os argumentos da classe INTEGER em `%rdi`- Não .`%rsi`- Não .`%rdx`- Não .`%rcx`- Não .`%r8`- Não .`%r9`O resultado retorna em`%rax`E depois ...`%rdx`O ponto 3.2.1 diz que `%rbp`- Não .`%rbx`, e `%r12`através de`%r15`pertencem ao chamador e devem ser preservados.

`collect_params()`aplica uma regra: um registro de argumentos que é lido antes de qualquer escrita na função é um parâmetro. O maior registro desse tipo fixa a contagem, então uma função que lê `%rsi`E nunca .`%rdi`Ainda tem dois parâmetros.`percent_change`lê `%edi`E ...`%esi`, então tem .`param_1`E ...`param_2`- Não .`grade`Só lê .`%edi`- Não .`total`lê `%rdi`como uma base de 64 bits e `%esi`como uma contagem de 32 bits.

Agora é a Ghidra.`param_3`Tem uma explicação.`idivl`escreve`%rdx`, e o psABI permite um retorno de 128 bits INTEGER em `%rax:%rdx`Em um arquivo objeto não há um chamador para dizer o tamanho do valor de devolução.

No caminho zero,`xorl %eax, %eax`Só escreve .`%eax`Ghidra leu .`%rdx`A entrada da função foi feita por um`param_3`A assinatura é uma suposição sobre as partes que as instruções deixam abertas.

## Fase 5: SSA, Um nome por definição

Depois de levantar,`grade`escreve`rax`Quatro vezes e `rcx`- Três vezes. Cytron, Ferrante, Rosen, Wegman e Zadeck (1991) definem a forma de atribuição estática única. Cada variável é atribuída exatamente uma vez, e as funções phi em pontos de junção selecionam entre as definições que as atingem. A construção deles tem duas etapas. A secção 5.1 coloca as funções phi usando fronteiras de domínio, e a secção 5.2 renomeia cada definição e uso com um passeio sobre a árvore dominadora.`insert_phis()`E ...`rename()`Segue as duas secções.

A lista da SSA de `total`Mostra o que a construção produz:

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

A versão 0 de um registo é o seu valor na entrada. `esi_0`E ...`rdi_0`O cabeçalho do loop.`0x50`tem dois predecessores, o bloco de configuração e ele mesmo, então cada variável com uma definição no loop obtém um phi lá.`flags_2`A SSA mínima coloca-o porque...`flags`O estágio 6 remove-o porque nada o lê.

O passo de renome também fixa as larguras. `edx_1 = 0`é uma definição de 32 bits e `rdx_3 = rdx_2 + 1`é de 64 bits, então o phi `rdx_2`O que é que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é o que é.`%rcx`em `cmpq %rdx, %rcx`é de 64 bits enquanto a sua definição `ecx_1 = esi_0`é de 32 bits, então o uso se torna `zext64(ecx_1)`Cada elenco no pseudocódio final vem desta única regra.

A parte "lite" é a saída da SSA.`coalesce_webs()`O sistema de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de computador de

## Etapa 6: Simplificar, Substituir, Remover Código Morto

`optimize()`O simplificador reescreve expressões de baixo para cima com estas regras:

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

A substituição avançada muda uma definição para seu único uso. A regra tem três condições. O uso está no mesmo bloco. O uso é uma declaração e não uma phi. Quando a definição contém uma carga, nenhuma armazenagem ou chamada fica entre a definição e o uso. Um endereço constante ou uma seção é substituído em qualquer lugar, exceto em uma phi, porque um argumento phi tem que permanecer uma variável para a web.

A eliminação de código morto elimina uma atribuição ou uma phi cujo destino não tem utilidade e cuja expressão não tem chamada.`percent_change`O restante`edx_2 = dividend_1 % edi_0`Está morto, então vai primeiro.`edx_1 = signfill(eax_1)`com uma única utilização, por isso é substituído por `dividend_1 = edx_1:eax_1`- Não .

A regra do par reduz então`signfill(eax_1):eax_1`- Não .`eax_1`, e a divisão se torna `eax_1 / edi_0`Agora .`eax_1 = esi_1 * 100`tem uma única utilização e `esi_1 = esi_0 - edi_0`O processo de reestruturação é um processo de reestruturação.`flags`As definições foram substituídas pelas condições de ramo e desaparecem.

```c
return (param_2 - param_1) * 100 / param_1;
```

### O multiplicador 0x66666667

`grade`Divide a pontuação por 10, e o clang emite não.`idiv`- Ele emite .`imulq $0x66666667, %rax, %rax`, então`sarq $0x22, %rcx`- Então, em uma cópia.`shrq $0x3f, %rax`O compilador escolhe um multiplicador.`m`E um turno .`s`- Para cada um .`x`na faixa, `(x * m) >> s`deve ser igual `x / d`- Não .

Aqui .`m = 0x66666667 = 1717986919`E ...`s = 34`A matemática confirma:`2^34 / 10 = 1717986918.4`, e o teto é`1717986919`O erro é:`m * 10 - 2^34 = 6`Para que a identidade seja mantida em cada 32 bits .`x`, os tempos de erro `2^31`Deve ficar abaixo.`2^34`, então o erro deve ser no máximo 8.`(x * m) >> 34`- É igual .`floor(x / 10)`para cada não negativo `int`- Não .

Para negativo `x`O piso gira para longe do zero enquanto o C gira para o zero. `shrq $0x3f`Extrae o bit de sinal do produto, que é 1 exatamente quando `x`Adicioná-lo corrige o arredondamento.

`magic_divisor(m, s)`Teste um par candidato.`d`como o coeficiente inteiro de `2^s`Por`m`E tenta .`d`E ...`d + 1`Aceita um .`d`Só quando`m`é igual ao limite máximo de `2^s / d`e o erro ligado mantém. O mesmo multiplicador com `s = 33`dá`d = 5`, o que é correto, porque `0x66666667`é também o teto de `2^33 / 5`Um teste verifica a identidade contra`c_div(x, 10)`Em 2.406 valores incluindo os dois extremos da`int`- O alcance.

Com reconhecimento,`grade`lê `ecx_1 = param_1 / 10 - 6;`Com o reconhecimento desligado, a demonstração imprime o formulário bruto:

```c
rax_1 = (long)param_1 * 0x66666667;
ecx_1 = (int)(rax_1 >> 34) + (int)((unsigned long)rax_1 >> 63) - 6;
```

Quando você lê código de um compilador que não estudou, a forma crua é a honesta.

## Fase 7: Tipos de uso

O código da máquina tem largura e não há tipos. `infer_types()`Uma rede cujas definições são de 32 bits de largura é`int`, e uma web de 64 bits é `long`Uma web que é a base de um acesso a elementos`base[idx]`com elementos de 4 bytes é `int *`O tipo de ponteiro se espalha ao longo de cópias e phis, então `local_8 = param_1`- Não .`-O0`Faz os dois .`int *`. O tipo de devolução é a largura do valor no `return`- Declarações.

O resultado para `total`É o que é`int total(int *param_1, int param_2)`Ghidra escreveu:`int total(long param_1,uint param_2)`O levantador usou a carga em escala como prova de ponteiro.`*(int *)(param_1 + uVar2 * 4)`Nem sequer conseguem ver .`const`E nem um nem outro sabe .`int *`de`unsigned *`- Não .

Lee, Avgerinos e Brumley (TIE, NDSS 2011) frame type recovery como solução de restrição sobre uma rede. O resultado é um limite superior e inferior em vez de uma conjectura. Noonan, Loginov e Cok (Retypd, PLDI 2016) adicionam polimorfismo e recorrência.`const`A lição de "Paperback" é uma lição que usa as duas regras mais baratas da linha de trabalho e chama o que não pode ver.

A aritmética não assinada é mantida nas expressões e não nos tipos de variáveis. `ja`Depois`cmpl $0x4, %ecx`impressões como `(unsigned int)ecx_1 > 4`, e `shrq`impressões como `(unsigned long)rax_1 >> 63`As variáveis permanecem assinadas para que o leitor veja o valor do registo e o único lugar onde a instrução o trata como não assinado.

## Fase 8: Estruturação a partir do gráfico

Cifuentes (1994), capítulo 6, corresponde regiões do gráfico às formas de uma linguagem de alto nível. Os loops vêm primeiro (6.6.1), depois os condicionalismos bidirecionais (6.6.2), depois os condicionalismos n-direita (6.6.3). Yakdan, Eschweiler, Gerhards-Padilla e Smith (NDSS 2015) tomam a outra estrada. Eles derivam a condição sob a qual cada nó executa e construem a estrutura a partir dessas condições, sem formas fixas. A motivação deles é uma medida. Hex-Rays em uma amostra produzida 1.571 `goto`As declarações em 49.514 linhas, uma por 32 linhas, e a sua estruturação não produziram nenhuma das 1.530 funções de núcleo.

`structure()`É um pequeno membro da primeira família. Precisa de dois fatos a partir da fase 2.`Branch`O limite de fundo é uma borda de um bloco que domina sua fonte. O ciclo natural é o conjunto de blocos que chegam à fonte de fundo sem passar o cabeçalho.

Para um `Branch`com joão `j`- Não .

- Se`j`é o alvo de salto, emitir`if (!cond) { fallthrough region }`e continuar em`j`- Não .
- Se`j`É a queda, emitir`if (cond) { target region }`e continuar em`j`- Não .
- Se`j`É a saída da função, ambos os braços retornam.`if (cond) {... }`E o outro braço depois dele, sem um `else`Este é o regresso precoce de`percent_change`E ...`total`- Não .
- Caso contrário , emitir .`if (cond) { target } else { fallthrough }`e continuar em`j`- Isto é ...`percent_change`- Não .`-O0`, onde ambas as armas armazenam`local_4`E encontramo-nos no epílogo.

Para um cabeçalho de loop:

- Se o cabeçalho termina em um `Branch`com um sucessor fora do circuito, emitir `while (cond) { body }`- Não .`total`- Não .`-O0`tem esta forma: o cabeçalho compara `local_14`com`local_c`e folhas em`jge`- Não .
- Se o cabeçalho é também o bloco com a borda traseira, ou o fechamento sai em vez disso, emitir `do { body } while (cond);`- Não .`total`- Não .`-O1`tem esta forma: bloco `0x50`Adiciona, aumenta, compara e salta para si mesmo.

Um `IndirectJump`Acaba um bloco sem sucessores.`grade`- Não .`-O0`É uma mesa de saltos.`jmpq *%rax`através de cinco compensações relativas em `.rodata+0x0`Aqui o elevador imprime uma nota em vez disso.`0x76`- Não .`0x7f`- Não .`0x88`, e `0x91`O laboratório de prática acrescenta que os dados de um grupo de dados de um grupo de dados são obtidos apenas através do salto indireto e não foram levantados.`switch`- Não .

### O que o -O1 fez ao interruptor

- Não .`-O1`O Clang não guardou uma mesa de salto para o`grade`- Ele calculou .`score / 10 - 6`E comparou o resultado com 4 como um número não assinado.`int`- Não .`.rodata+0x14`A comparação sem assinatura é uma instrução que abrange ambas as partes .`score / 10 < 6`E ...`score / 10 > 10`Uma diferença negativa é um grande valor não assinado, por isso falha no mesmo cheque.

A mesa aguenta .`1, 2, 3, 4, 4`para os coeficientes 6 a 10.`switch`com cinco `return`As declarações tornaram-se uma`if`O elevador emite exatamente isso e chama a mesa.`lookup_14`após a sua secção de compensação.

O comprimento da mesa é heurístico.`table_values()`Leia da tabela de início para a próxima `.rodata`O que é que é?`.text`Referências de realocação, ou ao final da seção. Neste objeto, o próximo offset referido é `0x28`, a tabela de `grade_label`- Então ...`lookup_14`- Não .`(0x28 - 0x14) / 4 = 5`A comparação sem assinatura com 4 concorda. Ghidra também não sabe o comprimento.`*(uint *)(&DAT_001000b4 + (ulong)uVar2 * 4)`, um endereço de byte mais um offset, e deixou os cinco valores para o leitor olhar para cima.

## Junto com Ghidra

A demonstração imprime cada par.`code/fixtures/ghidra/*.json`Os registos da prova mostram que a rea voltou em 2026-10-14.`npx -y rea-agents@6.5.0 decompile ledger-x86_64-O1.o <function> --provider ghidra --format json`Cada disco tem um`confidence: observed`E ...`authority: shipped-artifact`O texto é observado. As alegações dentro do texto são derivadas, e os registos não dizem quais.

### %_change

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

A versão reparada em `code/fixtures/ghidra/ledger-x86_64-O1.ghidra-repaired.c`O teste é feito com a fonte de 81 pares de entradas.

### grau

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

### Total

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

O teste de comportamento compilou o texto Ghidra para `grade`E ...`total`Não mudou, adiciona dois.`typedef`Linhas e definição de `DAT_001000b4`dos vinte bytes em `.rodata+0x14`Ambas as funções correspondem à fonte na grade.

## O que a pesquisa diz sobre a leitura de pseudocódios

LLM4Decompile (EMNLP 2024) avaliou Ghidra sozinho em 20,12% de re-executabilidade em HumanEval-Decompile. Cerca de uma em cada cinco funções recompilou e passou as afirmações que vêm com o índice de referência. Refinando a produção da Ghidra com o modelo 6.7B, elevou isso para 52,74%. DecompileBench (Findings of ACL 2025) executou seis decompiladores industriais e seis abordagens de LLM em 23.400 funções de 130 programas. Os resultados do LLM foram melhores e tiveram uma correcção de funcionalidade 52,2% menor.

Liu, Raff e Micinski (arXiv 2609.05370, setembro 2026) explicam a lacuna. Usaram 300 funções de biblioteca do GitHub e 287 funções baseadas em CVE. O modelo de refinamento mais forte aumentou a taxa de construção de Ghidra de 75 para 90 por cento, enquanto a taxa de correspondência comportamental caiu de 74 para 62 por cento. Os candidatos que passaram em todos os testes no índice de referência ainda divergiram em entradas confusas. Isso aconteceu em 4,9% dos casos em oito sistemas, e até em 13% para um sistema. Até um décimo das vulnerabilidades reveladas desapareceram do código refinado.

As causas que eles rastream são simples: campos, tipos, calles e guardas inventados substituíram os desconhecidos visíveis que o decompiler havia deixado no texto.

É por isso que o elevador imprime .`(unsigned long)(unsigned int)ecx_1`Imprime uma nota em vez de um "`switch`para o salto não resolvido, e `lookup_14`Em vez de uma etiqueta um modelo inventaria.`code/main.py`- Ghidra's.`undefined1 [16]`É feio e honesto.`int percent_change(int, int)`O mesmo modelo produzirá uma assinatura errada na próxima função.

## Construí-lo

Executa a demonstração:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode
python3 code/main.py
```

A demonstração decodifica as quatro funções do `-O1`Objeto e compara cada instrução com o cometido `llvm-objdump`- Não, não.`percent_change`- Não .`grade`, e `total`Imprime o IR antes do SSA, a listagem do SSA com funções phi e o pseudocódio ao lado do texto Ghidra.`grade`com o reconhecimento de divisão desligado, o `-O0`versões de `percent_change`- Não .`total`, e `grade_label`, e um resumo:

```json
{
  "functions": ["grade", "grade_label", "percent_change", "total"],
  "instructions": 56,
  "blocks": {"percent_change": 3, "grade": 3, "total": 5, "grade_label": 4},
  "phis": {"percent_change": 0, "grade": 1, "total": 2, "grade_label": 1},
  "objdumpDifferences": 0
}
```

Leia `code/main.py`na ordem seguinte:

1. `read_object()`Leia os cabeçalhos da secção, a tabela de símbolos e o `.rela`Seções de um objeto ELF64 realojável.
2. `decode_one()`E ...`render_att()`Descifrar uma instrução e imprimir-a no caminho `llvm-objdump`- Não, não.
3. `build_blocks()`- Não .`immediate_dominators()`, e `dominance_frontiers()`Construir os fatos gráficos.
4. `lift_insn()`transforma uma instrução em declarações de IR e resolve as realocações.
5. `insert_phis()`E ...`rename()`Construir SSA e inserir lançamentos de largura.
6. `make_simplifier()`- Não .`substitute_once()`, e `remove_dead_code()`Correr para dentro .`optimize()`- Não .
7. `collect_params()`- Não .`coalesce_webs()`, e `infer_types()`nome e digitar as telas.
8. `structure()`com`post_dominators()`E ...`natural_loops()`emite o corpo C.
9. `lift_function()`liga os estágios e `lift_all()`Levanta um objeto inteiro com assinaturas de callee.

## Usá-lo

Para levantar uma função de seu próprio objeto, construí-la com `clang -target x86_64-unknown-linux-gnu -c`e ligar .`lift_all(read_object(path))`Quando o decodificador levantar`DecodeError`, leia o byte de código optico da mensagem e encontre-o no mapa de código optico SDM. Adicione uma entrada às tabelas no topo do arquivo. Quando o elevador eleva em um mnemônico, adicione uma regra para `lift_insn()`Quando o estruturador emite um`goto`, imprimir o gráfico com `IrPrinter`Cada adição é pequena e cada uma é controlada contra`llvm-objdump`Ou clang pelos testes existentes.

Quando o Ghidra estiver instalado, execute rea na mesma função e coloque os dois textos lado a lado. Preencha a tabela de diferença antes de acreditar em qualquer texto. Uma diferença com uma instrução por trás dela é uma escolha de apresentação. Uma diferença sem instrução por trás dela é uma adivinhação, e a adivinhação pode ser de ambos os lados.

## Laboratório Interativo

### Laboratório A: uma função através de cinco etapas

- Do`code`diretório, Python aberto:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode/code
python3 -q
```

- Correr .

```python
from main import *
obj = read_object(FIXTURES / "ledger-x86_64-O1.o")
lifted = lift_function(obj, "percent_change")
for insn in lifted.decoded: print(f"0x{insn.address:x}: {insn.raw.hex(' ')}  {insn.att()}")
print(lifted.ir_text)
print(lifted.ssa_text)
print(lifted.pseudocode)
```

Segue-me .`%esi`de`subl %edi, %esi`- Não .`esi_1 = esi_0 - edi_0`- Não .`(param_2 - param_1)`O byte .`fe`é o ModR/M do `subl`, com `mod = 11`- Não .`reg = edi`, e `r/m = esi`- Não .

### Laboratório B: o idioma da divisão

```python
print(lift_function(obj, "grade", recognize_division=False).pseudocode)
print(magic_divisor(0x66666667, 34), magic_divisor(0x66666667, 33), magic_divisor(0x66666667, 31))
```

A primeira linha imprime o multiplicar bruto e as mudanças.`10 5 None`Descubra por que o mesmo multiplicador divide por 5 com um turno a menos e por que o turno 31 não consegue a linha de erro.

### Laboratório C: um novo objeto para o decodificador

Construir o dispositivo em `-O2`E levantá-lo:

```bash
clang -target x86_64-unknown-linux-gnu -O2 -fno-asynchronous-unwind-tables -c fixtures/ledger.c -o "$TMPDIR/ledger-O2.o"
python3 -c 'from main import *; from pathlib import Path; import os; lift_all(read_object(Path(os.environ["TMPDIR"]) / "ledger-O2.o"))'
```

O decodificador termina em `0x56`com`opcode 81 is outside the fixture subset`- Código de opção .`81`é o formulário do grupo 1 com um instantâneo de 32 bits.`andl $0x7ffffff8, %edx`, e Clang usou-o no ciclo vectorizado de `total`- Adicione-o ao lado .`0x83`em `decode_one()`O subconjunto aumenta uma entrada por vez e cada entrada é verificada contra `llvm-objdump`- Não .

### Laboratório D: Valor padrão de Ghidra em grau

```python
print(ghidra_pseudocode("grade"))
for score in (-25, 0, 59, 60, 100, 115):
    sign = -(score >> 31)
    product_sign = ((score * 0x66666667) & ((1 << 64) - 1)) >> 63
    print(score, sign ^ product_sign)
```

Cada linha imprime 0, Ghidra `uVar1`A expressão é correta e ilegível, e a instrução `xorl %eax, %eax`Diz-o em dois bytes.

## Laboratório de Prática

Adicionar`switch`recuperação para `grade`- Não .`-O0`- Não .

Requisitos:

- Leia as cinco mudanças em `.rela.rodata`de`ledger-x86_64-O0.o`, cada um `R_X86_64_PC32.text+offset`, e calcular os cinco objetivos do caso a partir da tabela em `.rodata+0x0`- Não .
- Adicionar um`Switch`O terminal para o IR com a expressão do índice, a lista de alvos e o padrão do `ja`antes da carga da mesa.
- Dê ao bloqueador um sucessor por alvo para que os dominadores, os pos-dominadores e o SSA continuem a trabalhar.
- Emite`switch (local_10) { case 0:... }`com um .`case`por alvo e o epílogo compartilhado após a união.
- Mantenha a nota para qualquer salto indireto cuja mesa não possa ser amarrada.
- Compile o resultado com clang e compare-o com a fonte sobre a grade de pontuação nos testes.

O sucesso significa`grade`- Não .`-O0`E ...`grade`- Não .`-O1`produzir a mesma saída sobre a rede, com um `switch`E não .`goto`- Não .

## Artifacto

Esta lição produz`outputs/skill-lifting-to-pseudocode.md`. Utilize-o para levantar uma função com uma derivação para cada linha e para preencher a tabela de diferença contra o texto de um decompilador.

## Verifique

Execute a demonstração e a suíte determinista:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

A suite tem 25 testes.

- O decodificador concorda com o comprometido `llvm-objdump`texto em cada instrução de ambos os objetos, e com um vivo `llvm-objdump`quando está instalado
- REX, ModR/M, SIB e o offset de campo de deslocamento de `leaq (%rip), %rcx`são decodificados como descrito no MDS
- um código op fora do subconjunto aumenta `DecodeError`Em vez de adivinhar
- O leitor do ELF lista as quatro funções e as três `.text`Realocações
- `total`Tem cinco quarteirões.`0x44`Domina .`0x50`, e `0x50`é um cabeçalho de loop com a si mesmo como o corpo
- `%rdi`E ...`%rsi`Tornar-se`param_1`E ...`param_2`, e `param_1`em `total`É o que é`int *`
- `test`E ...`cmp`com`je`- Não .`jle`- Não .`ja`, e `jge`Tornar-se`==`- Não .`<=`Não assinado .`>`, e `<`
- SSA coloca duas funções Phi ao vivo no cabeçalho do loop de `total`E nenhum em .`percent_change`
- O pseudcódigo de `percent_change`- Não .`grade`, e `total`é igual ao texto esperado
- `grade`lê `param_1 / 10 - 6`com reconhecimento e multiplicação bruta com reconhecimento
- `0x66666667`com o turno 34 dividido por 10, com o turno 33 dividido por 5, e a identidade mantém-se na grade
- - Esposas em pilha`-O0`Tornar-se`local_4`- Não .`local_8`- Não .`local_c`, e o marcador de quadro nunca aparece
- A mesa em `.rodata+0x14`lê `1, 2, 3, 4, 4`através da realocação
- O salto indireto em`grade`- Não .`-O0`é relatado com os seus quatro blocos não levantados
- Os aparelhos Ghidra contêm o retorno de 16 bytes, o `param_3`, o não simplificado .`xor`, e o `long param_1`
- As seis unidades emitidas compilam com `clang -std=c11 -Wall -Werror`Para`x86_64-unknown-linux-gnu`
- As funções levantadas, o texto Ghidra reparado e a fonte imprimem as mesmas 232 linhas sobre a grade de entrada

## Modos de falha de produção

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

## Regra de funcionamento

Trate o pseudocód como uma derivação que você não viu. Reproduzir a partir das instruções com um elevador que você pode ler. Coloque os dois textos lado a lado, e escreva a instrução ou a adivinha atrás de cada linha que difere. Compile ambos e corre ambos contra a fonte quando você tem. Uma linha sem instrução atrás dela permanece uma adivinha, não importa o quão limpo ele lê.

## Mais leitura

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
- rea 6.5.0 fonte em tag `3b79af2`que esta lição descreve: `src/domain/functionComparisonDimensions.ts`- Não .`website/evidence/dx-ball-sound-pan.md`- Não .
