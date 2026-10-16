# Differenciamento binário e execução simbólica: funções de correspondência e solução de entradas

> Uma correspondência entre duas funções é uma afirmação feita por uma assinatura com uma qualidade, e uma entrada resolvida é um modelo de uma condição de caminho; esta lição constrói ambas e as verifica contra radiff2, Ghidra, angr e z3.

**Type:** Build
**Languages:** Python, C, x86-64 assembly, Java (one Ghidra script)
**Prerequisites:** Phase 13 · 33 (disassembly and control flow), 34 (lifting to pseudocode), 36 (rebuild and prove equal)
**Time:** ~150 minutes

## Objetivos de aprendizagem

- Construa cinco assinaturas de funções a partir de um objeto e sua listagem: bytes exatos, bytes com campos de realocação mascarados, sequência mnemônica, forma de fluxo-grafo e nome do símbolo.
- Execute um jogo de jogo em cena com ganância no estilo BinDiff, onde uma assinatura coincide apenas quando é única em ambos os lados, e grave o estágio e a qualidade por trás de cada jogo.
- Reproduzir o caso em que duas funções idênticas a byte falham um correlador de byte em imagens carregadas e passar após a mascaragem, com a aritmética de realocação que explica os quatro bytes alterados.
- Reproduzir as proporções de similaridade do radiff2 a partir da distância de Levenshtein e explicar por que uma proporção só aparece quando a comparação usa os 82 bytes de radar2 alcançados.
- Enumere os caminhos de uma função em um pequeno IR, carregue uma condição por caminho e resolva cada folha para entradas com uma pesquisa limitada.
- Meter a explosão de um caminho em um ciclo com um limite simbólico e comparar as contagens, classes e modelos com angr 10.0.3 e z3 5.1.0.

## O problema

Reconstruiu .`ledger.c`- Não .`-O2`E perguntou a um agente se alguma coisa mudou desde o`-O1`O agente fez uma diferença e informou que...`grade`O programa foi modificado, porque os seus bytes diferem nos dois programas carregados.`grade`As imagens são idênticas nos dois arquivos objetos.`.rodata`fica 0x70 bytes mais adiante no `-O2`Programa, por isso um deslocamento em `grade`cresceu em 0x70.

Na mesma tarde que me perguntaste qual é o resultado .`grade`a função retorna 4 para cada pontuação de 90 a 109, e a pergunta não tinha limite, então a resposta do agente era uma amostra onde você precisava de um conjunto.

As duas falhas vêm da mesma lacuna. Uma correspondência e uma entrada resolvida são cada uma reivindicação com um método por trás dela. Esta lição constrói os dois métodos em Python e executa as ferramentas de produção ao lado delas. Onde as ferramentas e o código concordam, você tem duas derivações. Onde eles discordam, você tem um endereço, um byte ou um bound para olhar.

```figure
re-44-matches-and-paths
```

## Uma assinatura de função é uma reivindicação com qualidade

Flake enquadrou a diferença binária em 2004 como a construção de um isomorfismo entre os conjuntos de funções de dois executáveis semelhantes. Dullien e Rolles ampliaram-na em 2005 para blocos e instruções básicas. A página de conceitos do BinDiff descreve a estratégia que surgiu desse trabalho. Compute um atributo para cada função em ambos os lados. Um valor que aparece exatamente uma vez em cada lado é um coincidência. Um valor que aparece mais de uma vez é ambíguo e é forrado com o próximo atributo dentro dessa classe de equivalência. Um valor sem contraparte permanece inigualável. Após cada passagem, os chamadores e chamados de novas partidas formam pequenos conjuntos onde o próximo atributo é provável que seja único, e o processo se repete com o próximo atributo.

Cada algoritmo de correspondência tem uma qualidade. A página classifica o hash de correspondência de bytes de função crua e o hash de nome como muito bom, o índice MD de fluxo-grafo e a assinatura primária (cada mnemônica um pequeno primário, produto sobre a função) como boa, as referências de cadeia como meio e a contagem de loop ou sequência de endereços como pobre. A confiança de uma correspondência é a qualidade média dos algoritmos que a produziram, ponderada por um sigmoide. A semelhança da função é de cerca de 25% de bordas de fluxo-grafico correspondentes, 15% de blocos básicos correspondentes, 10% de instruções correspondentes e 50% da diferença no índice MD do fluxo-grafico, multiplicado pela confiança.

Esta lição mantém cinco desses atributos e a única regra que importa: único em ambos os lados ou sem coincidência. Ele deixa cair o índice MD, porque as páginas de conceitos nomeam-no sem uma fórmula, e usa uma sequência de graus simples em seu lugar. Os rótulos de qualidade permanecem, porque uma coincidência é apenas tão boa quanto o fato mais fraco que a produziu.

## Cinco assinaturas de um registro de função

O diferencial lê duas coisas por objeto.`.rela.*`Os registos.`llvm-objdump -d`A listagem dá limites de instrução e mnemônica, e a lição 33 provou um decodificador igual à listagem em cada instrução desses objetos. `function_record()`verifica que os bytes de cada instrução listada são iguais aos bytes do objeto nesse endereço e aumenta de outra forma.

- Toma .`grade`em `ledger-x86_64-O1.o`O símbolo diz " offset 0x10 " e " size 46 " , por isso o registro cobre 0x10 a 0x3e e exclui os dois bytes `nop`que alinha a próxima função.

A assinatura de bytes exatos é o SHA-256 dos 46 bytes. A assinatura de bytes mascarados zero primeiro cada campo de realocação. Há um campo: o registro`R_X86_64_PC32 .rodata+0x10`No deslocamento 0x36, de 4 bytes de largura, o deslocamento do `leaq (%rip), %rcx`Em um objeto RELA esses quatro bytes já têm zero, então as duas assinaturas concordam sobre cada função de cada objeto aqui.

A assinatura mnemônica hashes a sequência de 14 mnemônicos:

```text
movslq imulq movq sarq shrq addl addl xorl cmpl ja movl leaq movl retq
```

A assinatura de forma do flow-graph vem dos blocos básicos e bordas digitais da lição 33, restritos aos blocos que a entrada pode alcançar. `grade`- Não .`-O1`Tem três blocos e três bordas. Bloco 0x10 termina em`ja 0x3d`Com uma borda tomada e uma borda de queda, o bloco 0x31 cai para 0x3d, e 0x3d retorna. A sequência de graus lista (in, out) por bloco em ordem ordenada, de modo que a assinatura lê `3b 3e [0/2,1/1,2/0]`- Para o ...`grade`- Não .`-O0`O registro resolve o salto indireto através dos cinco `.rela.rodata`Entradas exatamente como a lição 33 fez, que dá oito blocos e onze bordas com quatro `table`- As bordas.

A assinatura do nome é o nome do símbolo, é o fato mais forte quando existe e nada quando a construção é despojada.

Mais um campo vai para o caso de correção mais tarde.`lea`realocação em`.rodata`, um controlo de limites, e uma carga indexada,`referenced_table()`O texto diz a tabela da maneira como a lição 36 fez:`cmpl $0x4`Antes de`ja`dá cinco entradas e a carga `movl (%rcx,%rax,4), %eax`dá quatro bytes cada.`grade`- Não .`-O1`A mesa em `.rodata+0x14`lê `1, 2, 3, 4, 4`- Não .

## O ganancioso empate

`match_functions()`O jogo registra o estágio, a qualidade do estágio, todas as outras assinaturas que também concordam e uma relação de semelhança.

```python
for stage in order:
    for signature, candidates in left_groups.items():
        partners = right_groups.get(signature, [])
        if len(candidates) != 1 or len(partners) != 1:
            continue
        a, b = candidates[0], partners[0]
        agreeing = [s.id for s in STAGES if a.signatures()[s.id] == b.signatures()[s.id]]
        matches.append(Match(a, b, stage, agreeing, radare2_ratio(a.raw, b.raw), data_equal))
```

A ordem das lições é a de executar primeiro os quatro estágios estruturais e os nomes são os últimos, de modo que os estágios estruturais são testados e os nomes os classificam depois.`-O1`contra`-O2`- Não .

| Function | Stage | Agreeing signatures | Similarity |
|---|---|---|---|
| `percent_change` | exact bytes | all five | 1.000 |
| `grade` | exact bytes | all five | 1.000 |
| `total` | name | name only | 0.171 |
| `grade_label` | exact bytes | all five | 1.000 |

`total`O clang vectorizou o loop com SSE2 e manteve um loop escalário restante. Sua forma passou de cinco blocos e cinco bordas para dez blocos e treze bordas. Nada estrutural sobrevive, e apenas o símbolo une os dois.

Para o`-O0`contra`-O1`A mesma ordem produz uma correspondência errada:

| Function at -O0 | Matched at -O1 | Stage | Names agree |
|---|---|---|---|
| `percent_change` | `grade_label` | flow-graph shape | no |
| `grade` | `grade` | name | yes |
| `total` | `total` | name | yes |
| `grade_label` | unmatched | | |

`percent_change`- Não .`-O0`é um diamante de quatro blocos: uma entrada que se ramifica em `before == 0`, dois braços, e uma articulação que retorna.`0/2, 1/1, 1/1, 2/0`- Não .`grade_label`- Não .`-O1`- Não .`grade`O formato é único em ambos os lados e a regra aceita. O rótulo de qualidade média é o aviso, e o estágio de nome, executado depois, contradiz isso. A coincidência falsa também consome `grade_label`à direita, então o real `percent_change`E ...`grade_label`Os pares permanecem inigualáveis.

A ordem BinDiff executa a correspondência de nomes como a segunda etapa em que existem nomes reais, e `BINDIFF_ORDER`O código faz o mesmo.`-O0`funções correspondem às suas `-O1`A lição mantém ambas as ordens na demonstração para que a falha seja visível.`if`E ...`else`que se juntar antes do retorno.

## Bytes idênticos, operação realocada

Ghidra aplica relocalizações quando importa um objeto. o seu correlador de byte compara imagens carregadas e lição 36 campos de relocalização mascarados por esse motivo. `load_image()`Reproduz a importação.`sh_addralign`, e coloca-os a partir de 0x100000. Então aplica cada registro:`S + A - P`escrito como um valor assinado de 32 bits para `R_X86_64_PC32`E ...`R_X86_64_PLT32`- Não .`S + A`como 64 bits para `R_X86_64_64`- Não .

O layout concorda com o Ghidra 12.1.4 em cada seção de início em ambos os programas, que o conjunto de testes verifica contra a saída de rastreamento de versão capturada:

| Section | `-O1` program | `-O2` program |
|---|---|---|
| `.text` | 0x100000, 159 bytes | 0x100000, 271 bytes |
| `.rodata` | 0x1000a0 | 0x100110 |
| `.rodata.str1.1` | 0x1000f0 | 0x100160 |

`.text`é 112 bytes mais longo em `-O2`, o crescimento de`total`- Então ...`.rodata`Começa 0x70 bytes mais adiante. `grade`Está em 0x100010 em ambos os programas.`leaq`O campo de deslocamento 0x33 tem o campo de deslocamento 0x36 e termina em 0x3a.`.rodata+0x14`- Não .

No`-O1`Imagem: `0x1000a0 + 0x14 - 0x10003a = 0x7a = 122`- No ...`-O2`Imagem: `0x100110 + 0x14 - 0x10003a = 0xea = 234`Os bytes de campo são`7a 00 00 00`em uma imagem e `ea 00 00 00`O campo de variação de 46 é 0,978, e o estágio de bytes exatos falha.`grade_label`movido pelo mesmo 0x70 que a sua tabela, então os seus deslocamentos permaneceram iguais e ele passa em bytes exatos em ambos os pontos de vista.

Ghidra relata a mesma divisão.`-O1`contra`-O2`, Função exacta Bytes Match encontrada `percent_change`E ...`grade_label`, Exact Function Instructions Match encontrou esses dois mais `grade`A correlação de instrução enmasca as operandes, que cobrem o deslocamento. A visão de imagem da lição dá correspondências de bytes exatos para exatamente as duas funções que o correlador de bytes de Ghidra encontrou e concordância de bytes mascaradas para exatamente os três que seu correlador de instrução encontrou.

- 6,5,0 reais `compare_functions`O carregador e a máscara nesta lição fornecem essa normalização.

## Que medidas radiff2

Radare2 6.2.4 navios `radiff2`- Com o`-AC`Ele analisa os arquivos e imprime uma linha por função: o tamanho esquerdo e endereço, MATCH ou UNMATCH, uma proporção, e o tamanho certo e endereço.

| Pair | `percent_change` | `grade` | `total` | `grade_label` |
|---|---|---|---|---|
| `-O0` vs `-O1` | UNMATCH 0.130435 | UNMATCH 0.426829 | UNMATCH 0.106061 | UNMATCH 0.269841 |
| `-O1` vs `-O2` | MATCH 1.000000 | MATCH 1.000000 | UNMATCH 0.171429 | MATCH 1.000000 |
| `-O1` vs `-O1` | MATCH 1.000000 | MATCH 1.000000 | MATCH 1.000000 | MATCH 1.000000 |

A relação na fonte de radares2 é `1 - distance / max(len_a, len_b)`, onde a distância é a distância de Levenshtein entre as duas impressões digitais da função e uma impressão digital é a concatenation dos blocos básicos da função em ordem de endereço.`radare2_ratio()`em `code/main.py`Implementa a mesma fórmula sobre os bytes da função bruta e reproduz 0,130435, 0,106061, 0,269841, e 0,171429 a seis décimas.

Não se reproduz 0,426829 para `grade`- Não .`-O0`. Sobre os 118 bytes da função a relação é de 0,296610.`118`A sua impressão digital é mais curta. sem realocações aplicadas não pode resolver a tabela de salto, então sua análise chega ao bloco de entrada, o bloco que termina em`jmpq *%rax`O Levenshtein distância entre esses 82 bytes e os 46 bytes em `-O1`é de 47, e`1 - 47/82 = 0.426829`- Com o`-e bin.relocs.apply=true`O radar2 encontra todos os 118 bytes e imprime 0,288136, o que equivale à relação da lição sobre as imagens relocadas, onde os quatro bytes de deslocamento diferem dos campos zero do objeto.

`-O`O sistema de controlo de dados é um sistema de controlo de dados que permite a utilização de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de dados de`-O0`contra `-O1`As relações aumentam para 0,239130, 0,512195, 0,303030 e 0,523810, e `total`- Não .`-O1`contra `-O2`As distâncias de arquivo inteiro também existem: `-s`(Myers) relata semelhança de 0,956 e distância de 162 entre os dois objetos.`-ss`Os números incluem cabeçalhos, tabelas de símbolos e seções de realocação e não dizem nada sobre funções.

## Correlatores de rastreamento de versões Ghidra

A ferramenta de localização de versões da Ghidra combina um programa fonte com um programa de destino com correlatores. Os correlatores de função são: Exact Function Bytes Match, Exact Function Instructions Match (operands mascarados), Exact Function Mnemonics Match e Duplicate Function Instructions Match (1:N e N:M). Os correlatores de dados são Exact, Duplicate e Similar Data Match. Os correlatores de símbolos correspondem exatamente aos nomes, como duplicados ou por semelhança. Os correladores de referência utilizam correspondências aceitas para encontrar mais. As funções devem ser definidas no programa de destino e o comprimento mínimo da função é definido por defeito para 10 bytes. Auto Version Tracking executa Nome do Símbolo Exact, Dados Exact, Bytes da Função Exact, Instruções da Função Exact, Mnemônica da Função Exact, Instruções de Função Duplicada, depois um correlador de referência, e cria correspondências implícitas para as aceitas. Foi adicionado um correlador BSim no ponto 11.2.

A lição capturou os correlatores sem cabeça.`capture_ghidra_vt.sh`importa os três objetos num projecto e executa `ListVersionTrackingMatches.java`, um breve guião que se estende .`AbstractGhidraVersionTrackingScript`O sistema de correlação de código aberto (SCR) abre os programas de origem e destino por caminho, executa os seis correlatores não-referência na ordem de rastreamento de versão automática e imprime uma linha por bloco, função, correlador e correspondência.

| Correlator | `-O0` to `-O1` | `-O1` to `-O2` |
|---|---|---|
| Exact Symbol Name Match | 4 functions | 4 functions, 5 data items (`labels.rel`, `.L.str.1` to `.L.str.4`) |
| Exact Data Match | 0 | 0 |
| Exact Function Bytes Match | 0 | 2 (`percent_change`, `grade_label`) |
| Exact Function Instructions Match | 0 | 3 (adds `grade`) |
| Exact Function Mnemonics Match | 0 | 3 |
| Duplicate Function Instructions Match | 0 | 0 |

Cada função mudou entre `-O0`E ...`-O1`A página de ajuda descreve os correlatores exatos como semelhança 1.0 e confiança 1.0, então a escala da confiança impressa é uma pergunta aberta que esta lição registra e não resolve.

Programas Diferenças, sob Ferramentas, é uma coisa diferente. Comparar dois programas com o mesmo espaço de endereço em bytes, rótulos, unidades de código, referências, comentários e funções em endereços iguais. É um endereço alinhado diferença e não corresponde nada que se moveu.

## Diafora e Ninja Binário, de suas documentações

Diaphora é um plugin IDA Pro, lançado pela primeira vez em 2015 e na versão 3.4.2 a partir de 2026-08-18. É AGPL-3.0 desde 2.0 e precisa de IDAPython, então ele é executado em IDA Home ou IDA Pro e não em IDA Free. Exporta cada banco de dados para SQLite e executa as suas heurísticas como consultas SQL. Cada heurística tem um nome, uma categoria de Melhor, Parcial ou Não Confiável, um tipo de relação e bandeiras opcionais como SAME_CPU ou SLOW. Os nomes na fonte atual são como os estágios desta lição com mais deles: hash de bytes, o mesmo endereço e mnemonics, a mesma montagem limpa, Mnemonics pequeno-primes-produto, os mesmos nós, bordas, loops e componentes fortemente conectados, hash de classificação topológica, e pseudo-código hashes confusos que precisam do decompilador Hex-Rays. Suas notas de lançamento mostram o histórico de sintonização: a heurística lenta só corre abaixo de cerca de 2.000 funções, e "O mesmo gráfico" passou para a categoria não confiável. Esse movimento é o diamante de quatro blocos desta lição, visto em escala.

Binary Ninja 6.0, lançado em 2026-09-03, adicionou a Similaridade Binária como um framework com provedores e resolvedores. Ele envia BinDiff como um provedor estrutural e WARP para correspondências de funções exatas que tolerem operands relocaveis, e seu guia diz para usar WARP quando funções coincidem além de instruções relocaveis e BinDiff quando permanecem estruturalmente relacionadas após a recompilação. O recurso está na edição Ultimate apenas. Binary Ninja não tem gerenciador de pacotes instalado aqui e não foi executado, por isso este parágrafo vem de sua documentação e nada mais.

O BinDiff em si é open source desde 2023-09-25 sob Apache-2.0, com a versão 8 da versão mais recente. Ele precisa de um desmonteador e o plugin BinExport, que suporta IDA Pro 9.4 ou posterior, Binary Ninja e Ghidra. O caminho livre para um aprendiz é Ghidra mais BinExport mais a linha de comando BinDiff. A lição não o executou, e seu Practice Lab pede que você o faça.

## O mesmo código, constante diferente

Lição 36 manteve um candidato reconstruir de `grade`cuja tabela tem 3 como última entrada, onde o original tem 4. `grade_case10_wrong.o`tem os mesmos 46 bytes de código que `grade`- Não .`-O1`O matcher as combina em bytes exatos com cada assinatura concordando.`1, 2, 3, 4, 4`através de uma realocação e `1, 2, 3, 4, 3`através do outro e dos relatórios `dataEqual: false`Os outros três .`-O1`As funções são incomparáveis à esquerda, porque o objeto candidato possui uma função.

Esta é a forma de patch-análise. Uma função diferente que termina em bytes diz que nada mudou. Ghidra's Exact Data Match e BinDiff's string-reference matching existem pela mesma razão. A versão da lição é pequena: para cada par combinado, compare os bytes em que cada função se realoca, com a contagem dos limites e a largura da carga.

## Execução simbólica: caminhos, condições, soluções

King descreveu a execução simbólica em 1976. As entradas se tornam símbolos. Cada declaração atualiza um armazém simbólico. Cada ramo adiciona uma condição ao caminho, e um solvente decide se o caminho é viável e produz uma entrada concreta que o segue. Uma função com ramos se torna uma árvore de caminhos, e uma folha é um retorno com uma condição anexada.

Baldoni, Coppa, D'Elia, Demetrescu e Finocchi pesquisaram o campo em 2018. Sua seção 5 chama a explosão de caminho como o custo central: um executor pode forjar em cada ramo, de modo que a contagem do estado pode crescer exponencialmente no número de ramos, e um loop cujo limite depende de uma entrada simbólica dá um conjunto ilimitado de caminhos. As mitigações que listam são poda de caminhos irrealizáveis resolvendo em cada ramo, resumos para funções e circuitos, fusão de estado e sub-aproximação. A execução concólica executa o programa de forma concreta e simbólica ao mesmo tempo, registra as condições ao longo da execução concreta, nega uma e pede ao solvente a próxima entrada. DART (2005), CUTE (2005), e SAGE (2008) são os sistemas nomeados.

KLEE (Cadar, Dunbar, Engler, OSDI 2008) é uma máquina virtual simbólica sobre bitcode LLVM, por isso sua entrada é `clang -emit-llvm`O KLEE 3.2 foi lançado em 2025-12-23 sob a licença NCSA, com LLVM 16 recomendado.

Angr trabalha em binários. `angr.Project`O sistema de carga de blocos de veículos (CLE loaders) que mapeam o objeto e aplicam as transferências, um elevador que transforma um bloco em VEX IR (`block.vex`O programa de trabalho foi desenvolvido em 1995 e foi desenvolvido em 1995 com o objectivo de promover a cooperação entre os Estados-Membros.`SimState`Contém registros, memória e um solvente.`SimulationManager`Passo escondidos de estados nomeados `active`- Não .`deadended`- Não .`found`- Não .`avoided`, e `errored`, e `explore(find=..., avoid=...)`A solução de solução é a solução frontal do solvente em relação ao Z3, e em angr 10.0 é uma implementação vendida Rust importada como `from angr import claripy`. angr 10.0.3 foi carregado em 2026-10-10, é BSD-2-Clause, e precisa de Python 3.12 ou posterior. Z3 5.1.0, lançado 2026-08-16 sob MIT, é o solvente por trás de angr, Triton e backend opcional de KLEE.

## Um pequeno executor sobre a lição 34 IR

O executor em `code/main.py`utiliza uma pequena IR com as formas de declaração da lição 34: `Assign`- Não .`Branch`- Não .`Jump`- Não .`Return`, mais o `Switch`O termo que o laboratório de prática da lição 34 pediu.`Const`- Não .`Var`- Não .`Bin`com largura, `Cast`- Não .`Load`de uma mesa de concreto com nome, e `Not`. As quatro funções fixas são levantadas à mão no nível que a lição 34 atinge após a simplificação, onde o `0x66666667`multiplicando tornou-se `score / 10`, e cada bloco de IR lista os endereços de instrução que veio de. `check_ir_against_listing()`afirma que os endereços cobrem a listagem da função exatamente uma vez.

`grade`- Não .`-O0`Nessa IR:

```text
0x30: local_8 = score; local_10 = score / 10 - 6; if (local_10 >u 4) goto 0x9a else 0x62
0x62: switch local_10 -> 0x91, 0x88, 0x7f, 0x76, 0x76
0x76: local_4 = 4; goto 0xa1        0x7f: local_4 = 3; goto 0xa1
0x88: local_4 = 2; goto 0xa1        0x91: local_4 = 1; goto 0xa1
0x9a: local_4 = 0                    0xa1: return local_4
```

Os alvos do switch vêm de`jump_table_targets()`, a mesma aritmética de realocação que a lição 33, então o IR lê a fixação.

`execute()`mantém uma lista de trabalho de estados de caminho. Um estado tem o bloco atual, o rastro de blocos, a lista de condições e um ambiente que mapeia cada variável para uma expressão sobre as entradas. `Assign`Substitui o ambiente na fonte e armazena o resultado. `Branch`O solvente não pode satisfazer um lado, que é a poda ansiosa que a pesquisa descreve. `Switch`Grupar as entradas da tabela por alvo e forcas uma vez por alvo distinto com a disyunção das igualidades de entrada, que é como Angr trata um alvo de salto simbólico também. `Return`Uma traça mais longa do que o limite de passo marca o caminho truncado e o impede.

Siga o caminho para o`return 4`A entrada atribui `local_10 = score / 10 - 6`- A condição do ramo .`local_10 >u 4`é falso neste caminho, então a lista de condições é válida `!(score / 10 - 6 >u 4)`As entradas 3 e 4 do interruptor apontam para 0x76, então o garfo para 0x76 adiciona `score / 10 - 6 == 3 || score / 10 - 6 == 4`. Bloco 0x76 atribui `local_4 = 4`, salta para 0xa1, e o retorno substitui para `4`A folha é o par daquela condição e daquele valor.

O solvente é`BoundedSearch`. Ele possui um domínio finito por entrada, enumera o produto dos domínios e avalia cada restrição com semântica C: envelope de 32 bits, divisão truncada, comparações assinadas e não assinadas, e mudanças aritméticas ou lógicas. Uma divisão por zero ou um índice de tabela fora da tabela faz com que essa atribuição falhe na restrição.`score`-200 a 1200 os cinco caminhos de `grade`A solução de cada folha para o conjunto de entrada dá:

| Leaf | Returns | Condition | Scores in the bound |
|---|---|---|---|
| 0x76 | 4 | `!(q >u 4) && (q == 3 \|\| q == 4)` | 90 to 109, 20 inputs |
| 0x7f | 3 | `!(q >u 4) && q == 2` | 80 to 89, 10 inputs |
| 0x88 | 2 | `!(q >u 4) && q == 1` | 70 to 79, 10 inputs |
| 0x91 | 1 | `!(q >u 4) && q == 0` | 60 to 69, 10 inputs |
| 0x9a | 0 | `q >u 4` | 1,351 inputs, the rest of the bound |

`q`significa`score / 10 - 6`As mesmas aulas são feitas em`grade`- Não .`-O1`, onde o IR tem dois caminhos, o caminho da tabela retorno `lookup_14[q]`e o caminho padrão retornar 0, e a questão se torna uma restrição no valor devolvido, porque não há um bloco de caso a alcançar.`lookup_14`é lido a partir do objeto `.rodata`A 0x14.

O limite é o limite do método. Uma busca limitada é completa dentro de seu domínio e silenciosa fora dele. Dentro de -200 a 1200 nenhuma pontuação em 60 a 109 retorna 0 e nenhuma pontuação de 110 ou mais retorna um valor não zero. z3 prova ambos para todos os 2^32 pontuações, e a próxima seção mostra onde os dois métodos retornam diferentes testemunhas.

## Explosão do caminho medida

`total`- Não .`-O1`é o ciclo da lição 33: um teste em`count`, um corpo de ciclo que adiciona `values[rdx]`e incrementos `rdx`, e um `jne`De volta ao corpo enquanto`rdx != rcx`- Na IR .`rcx`É o que é`zext64(count)`E ...`rdx`Começa em 0 e cada passagem através da lateral da garrafa.`rdx != count`com`rdx`uma constante, então o caminho para `count = c`- É o que faz com o corpo .`c`- O que é que é?`count`Limitado a 0 a 2, o executor encontra 3 caminhos. Com 0 a 4 encontra 5, e com 0 a 8 encontra 9. Cada caminho retorna a soma triangular de sua contagem: (0, 0), (1, 1), (2, 3), até (8, 36). angr relata estados 3, 5 e 9 delimitados e os mesmos nove pares de contagem e soma em todos os três níveis de otimização, incluindo `-O2`, cujo ciclo vetorial precisa de uma contagem de pelo menos 8 e, portanto, nunca altera a contagem de caminhos nesta faixa.

Sem limites o executor nunca termina, porque cada contagem de iterações é um caminho mais viável.`count`O relatório dá esses três números. O custo de solução também cresce: 33 avaliações na linha 2, 115 em 4, 531 em 8, e 6.900 para a corrida limitada.

## Angr e z3 sobre as mesmas questões

`capture_angr.py`O CLE carregou cada objeto realocavel com o backend ELF a 0x400000, aplicou 13 realocações para `-O0`E 8 para os outros, e resolveu os quatro símbolos.`grade`com um argumento simbólico de 64 bits, `run()`O processo de resolução do problema foi concluído com 5 estados em desfalque em`-O0`E 2 em `-O1`E ...`-O2`, em 0,475, 0,408 e 0,318 segundos.`eax == v`Para cada estado de cadastro e pedindo ao solvente o mínimo e o máximo assinados, deu 60 a 69, 70 a 79, 80 a 89, e 90 a 109 para `v`Os dois controles cruzados não foram realizados.

O formulário de alcance de um bloco utilizou CFGFast em `grade`- Não .`-O0`, que lista os oito blocos lição 33 encontrados, então `explore(find=0x400076)`Para o bloco que armazena 4. Encontrou um estado com três ainda ativos em 0,47 segundos, com um mínimo de 90 e máximo de 109 no argumento e amostras 91, 93, 94, 97 e 101.

Para o`percent_change`, dois estados em deserto com as restrições`before[31:0] != 0`E ...`before[31:0] == 0`- Pedindo o estado não-zero para`eax == 50`- Não .`before = 21055744, after = 332315210`- Não .`-O0`- Não .`40566804, 834296877`- Não .`-O1`, e `-24484867, 993934694`- Não .`-O2`A busca limitada em -16 a 16 respostas`before = -10, after = -15`. z3, fez a mesma pergunta sobre a lista da lição 34, respostas `before = -25092599, after = 305712160`. O conjunto de testes avalia todos os cinco pares com o avaliador IR e cada um dá 50. Um modelo prova que uma entrada existe.

`capture_z3.py`Modelagem do `-O1`lista de `grade`Em 32 bits bit-vectores com o multiplicador, os dois shifts, e a tabela, e faz mais quatro perguntas.`switch`Para algumas pontuações de 32 bits: não definido em 0,068 segundos, então a função compilada é igual à fonte em todas as entradas de 2^32.`score / 10`O mesmo diferença restringida a 0 a 109: não satisfeito, por isso a grade da lição 36 não o pode ver.`percent_change`- Não .`before == 3`com resultado -133 é sentado com `after = 1073741823`, a célula de sobre-abastecimento da lição 36, e a busca limitada encontra uma segunda testemunha dentro de seu domínio,`after = -1`, onde`(-1 - 3) * 100 / 3`Truncates para -133 sem sobreflow.

## Construí-lo

Executa a demonstração:

```bash
cd phases/13-tools-and-protocols/44-binary-diffing-and-symbolic-execution
python3 code/main.py
```

A saída é um relatório JSON. `diffs`mantém os fósseis de arquivo objeto e imagem carregada para `-O0`contra`-O1`E ...`-O1`contra`-O2`, mais o `-O0`par na ordem BinDiff. `toolComparison`coloca o estágio da lição ao lado do veredicto e da razão radiff2 e dos correlatores Ghidra para cada função. `patch`É o `grade_case10_wrong.o`- O caso.`radare2Ratios`Reproduz os números radiff2, `gradeRelocationCase`mostra o layout e os dois deslocamentos, e `symbolic`Detém os caminhos, as aulas e as contagens de explosões.

Leia `code/main.py`na ordem seguinte:

1. `ElfObject`lê seções, símbolos e `.rela.*`Registros.`parse_listing()`- O que é ?`llvm-objdump`- Não.
2. `jump_table_targets()`E ...`build_cfg()`dar blocos acessíveis e bordas de tipografia. `FunctionRecord.signatures()`Construirá as cinco assinaturas.
3. `load_image()`estabelece as secções atribuídas e aplica-se `S + A - P`E ...`S + A`- Não .
4. `match_functions()`É o melhor jogo.`levenshtein()`E ...`radare2_ratio()`- Não. - Não.
5. `parse_radiff2()`E ...`parse_ghidra_vt()`Leia as saídas da ferramenta capturadas para a comparação.
6. `evaluate()`- Não .`substitute()`, e `BoundedSearch`são a semântica das expressões e o solvente.
7. `execute()`Enumera os caminhos. `solve_returns()`pede a cada folha um conjunto de entradas.
8. `grade_ir_o0()`- Não .`grade_ir_o1()`- Não .`percent_change_ir_o1()`, e `total_ir_o1()`São os elevadores de mão com os seus endereços de instrução.

Existem mais três comandos.`python3 code/main.py diff A.o B.o`Diferencia dois objetos cujas listas estão ao lado deles, e `--image`Diferencia as imagens carregadas. `python3 code/main.py paths grade O0`Imprime os caminhos e classes de uma função, `paths total 8`O circuito com um limite. `python3 code/main.py figure-data`Imprime os dados que a figura interativa incorpora.

## Usá-lo

Use o difer e o executor nesta ordem em duas construções do seu próprio programa.

1. Capturar uma lista para cada objeto e deixar `function_record()`Verifique-o contra os bytes, corrige a listagem antes de qualquer outra coisa.
2. Difere os arquivos objetos primeiro. Leia o estágio de cada coincidência e trate as coincidências de forma do gráfico de fluxo como sugestões até que uma segunda assinatura concorde.
3. Diferencia as imagens carregadas. Uma função que desce de bytes exatos para bytes mascarados se move em relação aos seus dados, e a aritmética de deslocamento nomeia a seção que se moveu.
4. Para cada coincidência, compare as tabelas referenciadas. O mesmo código com constantes diferentes é uma função modificada.
5. Quando a construção tiver símbolos, execute a ordem BinDiff e use o resultado estrutural apenas para classificar os estágios.
6. Quando uma ferramenta relata uma relação, descubra quais bytes foram inseridos antes de compará-la com a sua.
7. Para a função cujas entradas você deseja, escrever o IR com um endereço por instrução, executar `execute()`com um limite em cada contador de ciclo simbólico, e resolver cada folha.
8. Execute a mesma consulta através do angr ou z3 quando estiverem disponíveis e compare os limites da classe.

## Laboratório Interativo

### Laboratório A: cinco assinaturas à mão

- Do`code`diretório, Python aberto:

```bash
cd phases/13-tools-and-protocols/44-binary-diffing-and-symbolic-execution/code
python3 -q
```

- Correr .

```python
from main import *
grade = {r.name: r for r in fixture_records("ledger-x86_64-O1.o")}["grade"]
grade.raw.hex(" ")
grade.masked_bytes() == grade.raw
grade.mnemonics
grade.cfg_shape()
grade.signatures()
```

Escreva os quatro bytes no offset 0x26 da função antes de imprimir `grade.raw`Então corre as mesmas linhas .`grade`de`ledger-x86_64-O0.o`e contar o `table`bordas em `grade.edges`- Não .

### Laboratório B: assistir a realocação quebrar o hash de byte

```python
case = grade_relocation_case()
case["layout"]
case["displacements"], case["rodataShift"]
case["objectBytesEqual"], case["imageBytesEqual"], case["maskedImageBytesEqual"]
```

Computação`0x100110 + 0x14 - 0x10003a`em papel antes de ler .`displacements`Então , muda a base em`load_image()`Para 0x400000 e confirmar que os deslocamentos não mudam, porque ambos os operandos se movem juntos.

### Laboratório C: a coincidência falsa e a ordem que a evita

```python
report = diff_fixtures("ledger-x86_64-O1.o", "ledger-x86_64-O2.o")
[(m.left.name, m.right.name, m.stage) for m in report.matches]
report = diff_fixtures("ledger-x86_64-O0.o", "ledger-x86_64-O1.o")
[(m.left.name, m.right.name, m.stage, m.names_agree) for m in report.matches]
report = diff_fixtures("ledger-x86_64-O0.o", "ledger-x86_64-O1.o", stages=BINDIFF_ORDER)
[(m.left.name, m.right.name, m.stage) for m in report.matches]
```

Impressão`records(...)["percent_change"].cfg_shape()`Para`-O0`E ...`grade_label`Para`-O1`Então escreva um sexto estágio que hashes os mnemônicos ordenados, então a ordem não importa mais, e ver se ele os separa.

### Laboratório D: caminhos e limites

```python
report = grade_paths("O0")
[(p["trace"], p["condition"], p["returns"]) for p in report["paths"]]
report["returns"]["4"]
total_paths(2), total_paths(4), total_paths(8)
total_paths(64, step_limit=24)["truncated"]
```

Mudança .`SCORE_DOMAIN`- Não .`range(0, 110)`e reiniciar .`grade_paths("O0")`A classe para 0 se encolhe para 60 entradas e as duas verificações cruzadas ainda mantêm.`range(90, 100)`E ler quais folhas não têm modelo.

## Laboratório de Prática

Exerce o BinDiff nos equipamentos e compare-o com os três resultados que já tem.

Requisitos:

- Instale a extensão BinExport no Ghidra 12.1.4 e exporta os três programas do livro de conta. Grave a tag de lançamento do BinExport e a versão Ghidra.
- Instale a linha de comando BinDiff 8 e produzir o `-O0`contra `-O1`E ...`-O1`contra `-O2`resultados com a opção de somente semelhança.
- Captar a semelhança e a confiança por função em `code/fixtures/`com os comandos exatos em `build_fixtures.sh`- Não .
- Extensão`tool_comparison()`com um `bindiff`coluna e adicionar um teste que compara o conjunto correspondente de BinDiff em `-O1`contra `-O2`com o conjunto de dados mascarados da lição.
- Escreva um parágrafo que nomeie o algoritmo BinDiff usado para `grade`e se comparou imagens carregadas.

O sucesso significa que a demonstração imprime quatro colunas de ferramentas por função e o novo teste afirma uma igualdade ou uma diferença explicada.

## Lição Artifacto

Esta lição inclui `outputs/skill-binary-diffing-and-symbolic-execution.md`. Utilizá-lo para transformar duas construções e uma função em uma diferença em fase com uma qualidade declarada por correspondência, uma explicação de cada função incomparável ou alterada e um relatório de caminho com um conjunto de entrada resolvido por folha e o limite que a produziu.

## Verifique

Execute a demonstração e a suite:

```bash
cd phases/13-tools-and-protocols/44-binary-diffing-and-symbolic-execution
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

A verificação deve provar:

- As tabelas de símbolos dão as quatro funções com tamanhos 46, 118, 66, 39 em `-O0`, 16, 46, 31, 63 em `-O1`, e 16, 46, 140, 63 em `-O2`
- O `.rela.text`registos dos `-O1`O objeto é descrito como `llvm-objdump -r`Imprima-os
- Cada instrução listada está na extensão do símbolo e seus bytes são iguais ao objeto, e um byte alterado aumenta
- `total`- Não .`-O1`tem cinco blocos, cinco bordas e um self loop, e sua cadeia de forma é determinista
- `grade`- Não .`-O0`Recuperar cinco alvos de mesa de salto de `.rela.rodata`E tem oito blocos e onze bordas.
- sem a mesa,`grade`- Não .`-O0`Atinge 82 bytes em quatro blocos
- `percent_change`- Não .`-O0`E ...`grade_label`- Não .`-O1`ter a mesma forma de quatro blocos
- hashes exatos e mascarados concordam em cada função de um objeto realojável
- `-O1`E ...`-O2`compartilhar bytes para três funções, `-O0`E ...`-O1`Não compartilhar
- o quadro de referência de `grade`lê `1, 2, 3, 4, 4`e as leituras do candidato.`1, 2, 3, 4, 3`
- os locais do carregador `.text`- Não .`.rodata`, e `.rodata.str1.1`onde Ghidra os colocou em ambos os programas
- `R_X86_64_PC32`É escrito como `S + A - P`e dá deslocamento 122, e `R_X86_64_64`- Notas em `.data`recebem endereços de cadeia
- `grade`bytes concordam nos objetos e diferem nas imagens por uma mudança de 0x70, e a mascaragem restaura a correspondência
- `-O1`contra `-O2`combina três funções em bytes exatos e `total`por nome; a visualização da imagem corresponde `grade`por bytes mascarados com semelhança 1 - 1/46
- A ordem das lições produz uma correspondência falsa de forma do gráfico de fluxo em `-O0`contra `-O1`, e a ordem BinDiff coincide com os quatro por nome
- O candidato do corrector coincide em bytes e falha em dados referenciados
- A relação Levenshtein reproduz 0,130435, 0,106061, 0,269841, 0,171429, 0,426829 sobre 82 bytes, e as relações de bytes realocados imprimem radiff2 com realocações aplicadas
- O conjunto MATCH do radiff2 é igual ao estágio de bytes exatos, e os correlatores de byte e instrução do Ghidra são iguais aos conjuntos exatos e enmascarados da visão da imagem
- Ghidra só encontrou nomes .`-O0`contra `-O1`, com similaridade de 1,0 e confiança de 10,0 em cada partida
- A IR levantada à mão cobre todas as instruções listadas uma vez e o avaliador segue a semântica C
- `grade`Tem cinco caminhos em `-O0`com as condições de troca e dois em `-O1`com as mesmas classes
- as classes de limites 60 a 69, 70 a 79, 80 a 89, 90 a 109 de angr igual a todos os três níveis e z3, com estados de 5 e 2 limites e a faixa de alcance de um bloco 90 a 109
- Os dois controles cruzados não têm modelo na linha, em angr ou em z3, e z3 demonstra que a lista é igual ao interruptor de fonte
- - Todos .`percent_change`Modelo da pesquisa limitada, angr e z3 avalia para o valor solicitado
- `total`dá 3, 5, 9 caminhos com as somas triangulares, o mesmo que angr, e o limite de passos retorna 22 e truncadas 2
- Os dados da figura contêm as duas visualizações, os números de realocação e cinco folhas com os seus conjuntos de entrada, e o módulo do site incorpora os mesmos dados

## Modos de falha de produção

| Failure | What the weak tool reports | What the analysis must prove |
|---|---|---|
| Byte hash on a loaded image | "grade modified" between two builds with identical code | The field offset, the two displacements, and the section shift; the masked hash matches |
| Masked hash on a relocatable object | "masking made no difference" | The fields were already zero; the relocation records and the referenced constants carry the change |
| Flow-graph shape alone | A confident pair of unrelated functions | A second signature agrees, or the names contradict and the match is dropped |
| Names before structure on a stripped build | Nothing matches | The structural stages with their qualities, and the unmatched list |
| A ratio compared across tools | "radiff2 says 0.43 and the lesson says 0.30" | Which bytes each side fingerprinted; 82 bytes without relocations, 118 with |
| Same code, different constant | "exact match" | The table read through the relocation and the bounds check |
| One model reported as the input | "the score is 90" | The class 90 to 109 with its count, and the bound it was found in |
| Bounded search reported as unsat | "no input gives -133" | Unsat inside the bound only; z3 finds 1073741823 outside it |
| Symbolic loop without a bound | A timeout | The bound or the cap, the paths returned, truncated, and never started |
| Closed tool described as run | Numbers with no capture | A capture file with the command and version, or a sentence saying documentation only |

## Regra de funcionamento

Compare em uma assinatura por vez, aceite apenas valores únicos em ambos os lados, e mantenha o estágio e sua qualidade com o jogo. Difere os objetos e as imagens carregadas e explique cada função que muda de estágio entre os dois. Enumere caminhos com um solvente em cada ramo, amarrar cada ciclo simbólico e relatar um conjunto de entrada com seu limite. Um único modelo sem limite é uma amostra.

## Mais leitura

- [BinDiff concepts: matching strategy, algorithms, and quality ratings](https://github.com/google/bindiff/blob/main/docs/concepts.md)
- [BinDiff repository and the v8 open-source release](https://github.com/google/bindiff/releases/tag/v8)
- [BinExport README: IDA Pro, Binary Ninja, and Ghidra exporters](https://github.com/google/binexport)
- [Flake, "Structural Comparison of Executable Objects" (DIMVA 2004) and Dullien, Rolles, "Graph-based comparison of Executable Objects" (SSTIC 2005), shipped in the BinDiff repository](https://github.com/google/bindiff/tree/main/docs/papers)
- [Diaphora README and heuristics source](https://github.com/joxeankoret/diaphora)
- [Ghidra Version Tracking correlators help page](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/VersionTracking/src/main/help/help/topics/VersionTrackingPlugin/VT_Correlators.html)
- [Ghidra Auto Version Tracking help page](https://www.ghidradocs.com/11.0_PUBLIC/help/VersionTracking/help/topics/VersionTrackingPlugin/VT_AutoVT.html)
- [Ghidra 12.1.4 release](https://github.com/NationalSecurityAgency/ghidra/releases)
- [Binary Ninja 6.0 "Krypton" release notes](https://binary.ninja/2026/09/03/binary-ninja-6.0-krypton.html)E ...[Binary Similarity user guide](https://docs.binary.ninja/guide/similarity.html)
- [radare2 book: radiff2](https://book.rada.re/tools/radiff2/intro.html)E ...[radare2 6.2.4 source](https://github.com/radareorg/radare2/releases/tag/6.2.4)
- [King, "Symbolic execution and program testing" (CACM 1976)](https://dl.acm.org/doi/10.1145/360248.360252)
- [Baldoni et al., "A Survey of Symbolic Execution Techniques" (ACM Computing Surveys 2018)](https://dl.acm.org/doi/10.1145/3182657)
- [Cadar, Sen, "Symbolic execution for software testing: three decades later" (CACM 2013)](https://dl.acm.org/doi/10.1145/2408776.2408795)
- [Cadar, Dunbar, Engler, "KLEE: Unassisted and Automatic Generation of High-Coverage Tests for Complex Systems Programs" (OSDI 2008)](https://www.usenix.org/legacy/events/osdi08/tech/full_papers/cadar/cadar.pdf)E ...[KLEE 3.2 release](https://github.com/klee/klee/releases/tag/v3.2)
- [Shoshitaishvili et al., "SoK: (State of) The Art of War: Offensive Techniques in Binary Analysis" (IEEE S&P 2016)](https://oaklandsok.github.io/papers/shoshitaishvili2016.pdf)
- [angr documentation: loading, simulation managers, and the simulation pipeline](https://docs.angr.io/en/latest/core-concepts/toplevel.html)E ...[angr 10.0.3 on PyPI](https://pypi.org/project/angr/)
- [Z3 5.1.0 release](https://github.com/Z3Prover/z3/releases/tag/z3-5.1.0)
- [Triton dynamic binary analysis library](https://github.com/JonathanSalwan/Triton)
- [System V Application Binary Interface, AMD64 Architecture Processor Supplement, Table 4.9: Relocation Types](https://gitlab.com/x86-psABIs/x86-64-ABI)
