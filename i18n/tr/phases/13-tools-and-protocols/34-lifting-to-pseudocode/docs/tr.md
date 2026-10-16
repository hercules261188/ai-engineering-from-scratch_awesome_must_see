# Pseudocode'ye yükseltme: IR, SSA ve sıfırdan yapılandırma

> Decompiler çıkışı gizli bir çıkışla bir iddia. Eğer çıkışın kendiniz oluşturduğunuzda, çıkışın her satırı kontrol edebileceğiniz bir şey haline gelir.

**Type:** Build
**Languages:** Python, C
**Prerequisites:** Phase 13 · 32 (ELF from scratch), Phase 13 · 33 (disassembly and control flow)
**Time:** ~150 minutes

## Öğrenme Hedefleri

- Lift, açık bayraklar, yükler, depolar, şubeler, aramalar ve geri dönüşler ile küçük bir IR'ye x86-64 talimatlarını çözmüştür.
- Çerçeve boşluklarını yerel değişkenlere ve Sistem V argüman kayıtlarını parametrelere dönüştürün.
- Yönetim sınırlarında phi fonksiyonlarıyla minimal SSA oluşturun, sonra ölü kodu değiştirin, basitleştirin ve kaldırın.
- İyileşmek`if`- Evet .`if`- Evet .`else`- Evet .`while`ve`do while`Kontrol akışı grafikinden, post-dominatörler ve arka kenarları ile.
-  çarpımı açıkla`0x66666667`10'a bölünerek kimliği kanıtlayın.
- Aynı fonksiyon için Ghidra pseudokodu'nu satır satır okuyun ve her farkın arkasındaki talimatın veya tahminin adını verin.

## Sorun

Rea ' nın pseudokodu için sordun .`percent_change`İçeride`ledger-x86_64-O1.o`Ghidra 12.1.4 yaklaşık on saniye sonra cevap verdi:

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

Kaynak dört satır uzunluğunda.`int`değerler ve dönüşler 1 `int`Ghidra, 16 baytlı bir geri dönüş değeri ve kaynakta bulunmayan üçüncü bir parametresi bildirir. Makine kodunun on altı baytı doğru ve Ghidra hepsini doğru şekilde çözmüştür. İmza bir tahmintir ve metinde hiçbir şey onu bir olarak işaret etmez.

rea bu metni  olarak gönderir.`procedure_pseudo_code`Bu kısıtlama "Pseudocode Ghidra dekompiler çıkışı, orijinal kaynak değil ve Hopper çıkışı için metin eşdeğer değil" diyor.`compare_functions`araç metin satırları olarak pseudokod farklılık gösterir. Kendi boyut kaydı "Yüzelim açık olmayan sunucu metni" diyor ve yer değiştirme normalleşmesi kullanılamıyor. rea metin içine bakamaz ve sadece okuyan bir ajan da bunu yapamaz.

Rea'nın DX-Ball kaydı maliyetini gösteriyor.`FUN_00406400(void)`Aramak`__ftol()`Bu görüntüde bir yığın giriş ve x87 ifadesi gösterilmişti.

Bu ders, dekompilerin sakladığı derivasyonu oluşturur. Ders 33'ün çözdüğü 56 talimattan başlayarak onları IR'ye yükseltir, SSA'ya yeniden adlandırır, basitleştirir ve yapılandırır. Sonra pseudokodunu Ghidra'nın yanında yerleştirir.`percent_change`- Evet .`grade`ve`total`Testler çıkışınızı clang ile bir giriş şebekesi üzerinden kaynağa karşı çalıştırır.

```figure
lifting-stages-percent-change
```

## Çatırma ve Orakel

`code/fixtures/ledger.c`Bu kursun ortak bir parçasıdır:

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

`grade_label`Bu ders, yukarıdaki üç işlevi test eder.

Nesne , `clang -target x86_64-unknown-linux-gnu -O1 -fno-asynchronous-unwind-tables -c ledger.c`- Ne ?`-O0`Nesne, yığın yuvası bölümünde yanındadır. `code/fixtures/README.md`Bu, her nesnenin SHA-256 ve Ghidra fixtures'i üreten rea komutlarını listeler.

Dekodörün oracüleri `llvm-objdump -d -r`. Her iki nesne için de metni ,`ledger-x86_64-O1.objdump.txt`ve `ledger-x86_64-O0.objdump.txt`. Bir test bu dosyaları analiz eder ve her talimatı adres, byte ve AT&T metni ile karşılaştırır.`-O1`Ve her talimatın .`-O0`Ne zaman ?`llvm-objdump`sabit yolda kurulur, ikinci bir test canlı olarak çalışır ve tekrar karşılaştırılır.

İşte burası .`percent_change`Orakel yazdığı gibi:

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

## 1. aşama: talimatlara ait baytlar

Ders 33'te dekodör oluşturuldu. bu ders küçük bir kopyayı tutar böylece kod kendiliğinden çalışır. kopyası tam olarak yayınlanan opkodları kapsar.`ledger.c`- ...`-O0`ve `-O1`Başka bir şey için de .`DecodeError`Adres ve opkod bayt ile.

Dekoder Intel SDM'nin bayt sırasını izler, 2. Cilt, bölüm 2.1. Önlükler önce gelir, sonra opcode, sonra ModR/M, sonra SIB, sonra yer değiştirme, sonra hemen. 2.2.1 Bölüm REX önlüğünü ekler, bir bayt `0x40`- ...`0x4F`Rex.W 64 bitli bir operand seçer, Rex.R `reg`alanı, REX.X SIB endeksini genişletiyor ve REX.B genişletiyor `r/m`2.1.3 bölümünde üç ModR/M alanı tanımlanır.`mod`- Evet .`reg`ve`r/m`, ve üç SIB alanı `scale`- Evet .`index`ve`base`64 bit modunda.`mod = 00`- Evet .`r/m = 101`"Bit" (BIT) ifadesi, "BIT" ifadesi anlamına gelir.

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

- Evet .`leaq (%rip), %rcx`- ...`grade+0x23`Bu nesne, yer değiştirme ile birlikte yer değiştirme ile birlikte.`R_X86_64_PC32.rodata+0x10`- Yapabilir miyim ?`0x36`Asansör 3. aşamada çözüyor.

## İkinci aşama: Bloklar ve Yöneticiler

Temel blok, bir giriş üstte ve bir çıkış altta olan bir ifade dizisidir. Allen'ın 1970'deki tanımlamasıdır ve her dekompiler kağıdı onu tekrar kullanır. `build_blocks()`Bu sayede, bir blokun bir sonraki atlamasından sonra iki tane ve bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir sonraki atlamasından sonra bir atlaması vardır.`jmp`- Bir tane var.`ret`Hiç bir tane yok ve sıradan bir talimatla biten bir blok bir sonraki liderin eline geçiyor.

`total`- ...`-O1`Beş bloklu:

| Block | Instructions | Successors |
|---|---|---|
| `0x40` | `testl %esi, %esi` then `jle 0x5c` | `0x5c`, `0x44` |
| `0x44` | `movl %esi, %ecx`, `xorl %edx, %edx`, `xorl %eax, %eax`, `nopw` | `0x50` |
| `0x50` | `addl (%rdi,%rdx,4), %eax`, `incq %rdx`, `cmpq %rdx, %rcx`, `jne 0x50` | `0x50`, `0x5b` |
| `0x5b` | `retq` | exit |
| `0x5c` | `xorl %eax, %eax` then `retq` | exit |

`immediate_dominators()`Bu, Cooper, Harvey ve Kennedy'nin (2001) tekrarlayıcı algoritmasıdır. Bu blokları ters post-sıra ile yürür ve hiçbir şey değişene kadar işlenmiş öncekilerin baskıcılarını kesiyor.`total`- Öyle diyor.`0x44`Yöneticisi .`0x50`Ayrıca şöyle yazıyor:`0x50`Arka kenarından kendini kontrol eder, bu da 8. aşamada döngü bulma şeklini gösterir.

Aynı işlev iki kat daha fazla çalışır.`dominance_frontiers()`5. aşamada SSA için kullanıyor.`post_dominators()`8. aşamada birleştirmeler bulmak için sanal çıkış düğmesi ile ters çizilen grafikte çalışır.

## 3. aşama: Küçük IR'ye yükseltme

IR altı ifade türüne sahiptir: `Assign`- Evet .`Store`- Evet .`Branch`- Evet .`Jump`- Evet .`IndirectJump`ve`Return`. İfadeler sabitler, değişkenler, genişliğe sahip ikili işlemler, atışlar (`sext`- Evet .`zext`- Evet .`trunc`5 şekil daha var ve daha sonra basitleştirilmiştir:`Flags`- Evet .`Cond`- Evet .`Concat`- Evet .`SignFill`ve`Symbolic`- Evet .

Registerlar 64 bit ailesi tarafından adlandırılan değişkenlerdir.`%eax`- Evet .`Assign`- ...`rax`32 genişliğinde ve bir okuma ile `%eax`- Bu bir okuma.`rax`32 bit yazma biçiminde 32 bit yazma sıfır genişlemeyi tam kayıtlara uzatır, böylece aynı değerli daha sonraki 64 bit okuma bir `zext`. Asansör ulaşma tanımını bilmiyor, bu yüzden okuma genişliğini kaydeder ve SSA'nın 5. aşamada döküm yerleştirmesine izin verir.

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

### Bayraklar değişken

Şart kodlarını belirleyen her talimat bir değişkenin adı yazılır `flags`, ve her şartlı sıçrama onu okuyor.`cmp a, b`Yazar `cmp(a, b)`, a `test a, b`Yazar `test(a, b)`ve`sub`Aynı şeyi yazıyor.`cmp`Bu, bir yan etkisi olarak şekillendirildi.`jcc``Branch(Cond(cc, flags), target, fallthrough)`Şart kodu, 6 aşamada iki bölünme karşısında karşılaştırma operatörünü belirler:

| Code | After `cmp(a, b)` | After `test(a, a)` |
|---|---|---|
| `je` | `a == b` | `a == 0` |
| `jne` | `a != b` | `a != 0` |
| `jl`, `jge`, `jle`, `jg` | signed `<`, `>=`, `<=`, `>` | signed compare with 0 |
| `jb`, `jae`, `jbe`, `ja` | unsigned `<`, `>=`, `<=`, `>` | unsigned compare with 0 |

`xorl %eax, %eax``eax = 0`Bayraklar şöyle yazıyor:`add`Bu şekilde bir atlama yapmadan bayraklar çizer.`xor`İdman meselesi `grade`Ghidra'nın kaybettiği yer.

### Ctd ve IDEV

`cltd`dolduruyor .`%edx` işaretli bir parça ile`%eax`- Evet .`idivl %edi`64 bit çiftini bölüyor `%edx:%eax`- ...`%edi`, oranı içeriyor `%eax`, ve kalanı `%edx`Asan çiftleri geçici bir isimle yazıyor .`dividend`Önce kalan ifade, sonra eski`%eax`SSA'da yüksek yarısı,`signfill`Basitleştiricisi çiftin aşağı yarısına düşürülmesini sağlar ve bölünme 32 bit olur.`/`- Evet .

### -O0'da yığın boşlukları

- Evet .`-O0`Clang, tüm yerelleri çerçeveye sokar.`percent_change`mağazalar `%edi`- ...`-0x8(%rbp)`ve `%esi`- ...`-0xc(%rbp)`Sonra her kullanımı için tekrar okuyor.`pushq %rbp`Ardından `movq %rsp, %rbp`O prologun ardından, bir hafıza operandı ve bir taban.`%rbp`, indeks yok ve bir yer değiştirme adı verilen değişken haline gelir.`local_<hex>`- Evet .`-0x8(%rbp)`- Evet .`local_8`- Evet .`-0xc(%rbp)`- Evet .`local_c`- Evet .

Çerçeve talimatları bile IR üretmez ve ne de `subq`ve `addq`- Evet .`%rsp`Başka bir hafıza operandı bir `Load`veya `Store`- Evet .

### Yer değiştirme

- Evet .`leaq (%rip), %rcx`İçeride`grade`Dosyada sıfır yer değiştirme var.`0x36`diyor ki`R_X86_64_PC32.rodata+0x10`Bağlantı yazar .`S + A - P`Sahaya çıkın.`S`adresidir `.rodata`- Evet .`A`Ek olarak `0x10`ve`P`Bu, alanın adresidir.`next_ip + disp = (P + 4) + S + A - P = S + A + 4`- Evet .

Yükselticisi aynı aritmetikleri bir bağlantı olmadan uyguluyor.`0x10 + 4 = 0x14`- İN diyor ki:`rcx = &.rodata+0x14`Ve orada masa beş kişiyi tutuyor .`int`değerler`1, 2, 3, 4, 4`- Evet .

## 4. Adım: Çağırış Anlaşmasından Parametre

Sistem V AMD64 psABI, bölüm 3.2.3, INTEGER sınıf argümanlarını `%rdi`- Evet .`%rsi`- Evet .`%rdx`- Evet .`%rcx`- Evet .`%r8`- Evet .`%r9`Sonuç şu sırada gelir.`%rax`Ve sonra ...`%rdx`3.2.1 bölümünde şöyle yazıyor:`%rbp`- Evet .`%rbx`ve`%r12`- Evet .`%r15`Çağırana ait ve korunmalıdır.

`collect_params()`Bu kuralın bir kısmı vardır: fonksiyonda herhangi bir yazmadan önce okunur bir argüman kaydı bir parametre.`%rsi`Ve asla .`%rdi`Hâlâ iki parametre var.`percent_change`okur `%edi`ve `%esi`- Evet , öyle .`param_1`ve `param_2`- Evet .`grade`Sadece okur.`%edi`- Evet .`total`okur `%rdi`64 bitlik bir taban olarak ve `%esi`32 bit sayımı olarak.

Şimdi Ghidra'nın `param_3`Bir açıklaması var.`idivl`Yazar `%rdx`, ve psABI 128 bitli INTEGER geri dönüşünü sağlar `%rax:%rdx`Bir nesne dosyasında, geri dönüş değerinin ne kadar geniş olduğunu söyleyen bir aramacı yoktur. Ghidra geri dönüş değer olarak her iki kayıtı da tutmuştur.

Zira yol üzerinde,`xorl %eax, %eax`Sadece yazıyor.`%eax`Ghidra okudu .`%rdx`Bu işlevden bir giriş olarak, bu giriş `param_3`Bu imza, talimatların açık bıraktığı parçaları tahmin ediyor.

## 5. aşama: SSA, tanım başına bir isim

- Yukarı kaldırıldıktan sonra.`grade`Yazar `rax`4 kez ve`rcx`Üç kez. Cytron, Ferrante, Rosen, Wegman ve Zadeck (1991) statik tek görev biçimini tanımlar. Her değişken tam olarak bir kez atfedilir ve phi fonksiyonları, onlara ulaşan tanımlar arasında seçilir. İnşaatları iki aşamalı. Bölüm 5.1'de, pi fonksiyonları egemenlik sınırlarını kullanarak yerleştirir ve bölüm 5.2'de her tanım ve kullanımı egemenlik ağacının üzerinde bir yürüyüş ile yeniden adlandırır.`insert_phis()`ve `rename()`Bu iki bölümü takip edin.

SSA'nın listesi `total`İnşaatın ne ürettiğini gösterir:

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

Bir rejistrin 0 versiyonu, giriş sırasında değeridir. `esi_0`ve `rdi_0`- Bu da bir simge.`0x50`Bu da bir değişkenin bir fi olduğunu gösterir.`flags_2`Phi, geldiğinde ölü.`flags`Her iki önceki aşamada da tanımları var. 6. aşamada da hiçbir şey okumayı nedeniyle kaldırılır.

Ad değiştirme adımı genişlikleri de düzeltir. `edx_1 = 0`32 bit tanımlı bir `rdx_3 = rdx_2 + 1`64 bit, bu yüzden phi `rdx_2`Daha geniş bir alanı alır.`%rcx`İçeride`cmpq %rdx, %rcx`64 bitken tanımlaması `ecx_1 = esi_0`32 bit, bu yüzden kullanımı `zext64(ecx_1)`Son pseudokoddaki her oyuncu bu tek kuralı takip ediyor.

"Lite" kısmı SSA'dan çıkış yolu.`coalesce_webs()`Bu, bir kompiler tarafından bu işlevler için yayılan şekiller için çalışır. Birlikte iki phis değişim değerlerini birleştirirken paralel kopyalara bölünebilir ve ders bu durumu ele almıyor. Cytron'un 7. bölümü renkleme yoluyla genel çıkışı tanımlar.

## 6. Adım: Basitleştir, Değiştir, Ölü Kodları Kaldır

`optimize()`Basitleştiricisi ifadeleri aşağı yukarı aşağıdaki kurallarla yeniden yazar:

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

Önceki değiştirme bir tanımı tek kullanımına taşıyor. Kural üç şartı vardır. Kullanım aynı blokta. Kullanım bir ifade ve bir phi değil. Defini bir yük içerdiğinde, tanım ve kullanım arasında hiçbir depo veya çağrı bulunmaz. Bir sabit veya bir bölüm adresi phi dışında her yerde değiştirilir, çünkü bir phi argümanı web için bir değişken kalmalıdır.

Ölü kod kaldırmak, bir görev veya bir phi'yi, hedefinin kullanılmaz olduğu ve ifade edilmesi çağrıda bulunmayan bir görev silmektedir.`percent_change`Geri kalanı .`edx_2 = dividend_1 % edi_0`Ölüdür, yani önce gidiyor.`edx_1 = signfill(eax_1)`bir kullanım için kullanılır, böylece `dividend_1 = edx_1:eax_1`- Evet .

Çift kuralı sonra azaltır `signfill(eax_1):eax_1`- ...`eax_1`, ve bölünme `eax_1 / edi_0`- Şimdi .`eax_1 = esi_1 * 100`tek bir kullanım ve `esi_1 = esi_0 - edi_0`Bir kullanım var ve her ikisi de geri dönüş bildirimine geçiyor.`flags`Bu nedenle, bu kısımlar, bir dizi metin olarak kullanılır.

```c
return (param_2 - param_1) * 100 / param_1;
```

### 0x66666667 çarpması

`grade`Notu 10 ile bölüyor ve clang hayır yayıyor.`idiv`- Çıkıyor .`imulq $0x66666667, %rax, %rax`O zaman ...`sarq $0x22, %rcx`Bir kopyasına, o zaman `shrq $0x3f, %rax`Granlund ve Montgomery, "Multiplikasyon kullanarak değişmez tam sayılarla bölme" (PLDI 1994), yöntemi tanımlar.`m`Ve bir vardiya.`s`Her biri için .`x`Aralıkta,`(x * m) >> s``x / d`- Evet .

İşte .`m = 0x66666667 = 1717986919`ve `s = 34`- Matematik doğrular:`2^34 / 10 = 1717986918.4`Ve tavan `1717986919`Hata şu .`m * 10 - 2^34 = 6`Kimlik her 32 bit üzerinde kalsın .`x`, hata zamanları `2^31`Aşağıda kalmak zorunda `2^34`Bu yüzden hata en fazla 8 olmalıdır.`(x * m) >> 34`eşit `floor(x / 10)`- Nefi olmayan her şey için .`int`- Evet .

Negatif için`x`zemin sıfırdan uzaklaşırken C sıfıra doğru döner. `shrq $0x3f`ürünün işaret bitini çıkarır, bu da tam olarak 1 olduğunda `x`- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

`magic_divisor(m, s)`- Bir aday çiftini test eder.`d`'nin tam sayı oranı olarak`2^s`- ...`m`Ve dener.`d`ve `d + 1`- Bir `d`Sadece`m`                                            `2^s / d`ve hatalı bağlama geçerli.`s = 33`ver .`d = 5`, bu doğru, çünkü `0x66666667`Aynı zamanda `2^33 / 5`- Kimliği kontrol eden bir test .`c_div(x, 10)`                       `int`Aralık.

Tanınma devam eder.`grade`okur `ecx_1 = param_1 / 10 - 6;`Tanımayı kapatırken demo çiğ şekli basıyor:

```c
rax_1 = (long)param_1 * 0x66666667;
ecx_1 = (int)(rax_1 >> 34) + (int)((unsigned long)rax_1 >> 63) - 6;
```

Her ikisi de aynı değerleri oluşturur ve hesaplar. Şalter tutun. İncelemediğiniz bir kompiliörden kod okuduğunuzda, ham biçim dürüst bir biçimdir.

## 7. Adım: Kullanım Tipleri

Makine kodunun genişliği var ve türleri yok.`infer_types()`Bu, üç tür kanıtı C türüne dönüştürür.`int`, ve 64 bitli bir web `long`Bir element erişimi için temel olan bir web .`base[idx]`4 baytlı elementler ile `int *`. işaretçi tipi kopyalar ve fişler boyunca yayılır, bu yüzden `local_8 = param_1`- ...`-O0`İkisini de yapar.`int *`. Geri dönüş tipi değerin genişliğini `return`Açıklamalar.

Sonuç:`total`- Evet .`int total(int *param_1, int param_2)`Ghidra yazdı .`int total(long param_1,uint param_2)`İkisi de talimatlara uymuş.Lifter, ölçülü yükü işaretçi olarak kullandı.Ghidra kayıt genişliğini korudu ve girişini şöyle yazdı:`*(int *)(param_1 + uVar2 * 4)`- Göremiyorum .`const`, ve hiçbirisi bilemez .`int *`-`unsigned *`- Evet .

Lee, Avgerinos ve Brumley (TIE, NDSS 2011) çerçeve tipi kurtarma bir red üzerinde kısıtlama çözümü olarak. Sonuç bir tahmin yerine üst ve alt sınırdır. Noonan, Loginov ve Cok (Retypd, PLDI 2016) polimorfizm ve geri dönüşü ekler.`const`Bu ders, bu iş çizgisinden en ucuz iki kural kullanıyor ve göremediği şeyleri adlandırabiliyor.

İmza edilmemiş aritmetik, değişken türlerde değil ifadelerde tutulur. `ja`Sonra .`cmpl $0x4, %ecx` gibi izler`(unsigned int)ecx_1 > 4`ve`shrq` gibi izler`(unsigned long)rax_1 >> 63`Değişkenler imzalanmış kalır, böylece bir okuyucu kayıt değerini ve talimatın imzalanmamış olarak ele aldığı tek yeri görür.

## 8. Adım: Grafikten Yapılandırma

Cifuentes (1994), 6. bölüm, grafikin bölgelerini yüksek düzeyde bir dil şekillerine karşı eşleştirir. İlk olarak döngüler (6.6.1), sonra iki yönlü koşullar (6.6.2), sonra n yönlü koşullar (6.6.3). Yakdan, Eschweiler, Gerhards-Padilla ve Smith (NDSS 2015) diğer yoldan geçiyorlar. Her düğümün uygulandığı ve yapıdaki yapıdaki koşulları, sabit şekiller olmadan oluşturduğu koşulları çıkarırlar. Onların motivasyonu bir ölçümdür. Bir örnek üzerinde 1571 adet Hex-Rays üretildi.`goto`49.514 satırdaki açıklamalar, 32 satırda bir, ve yapılandırmaları 1.530 temel fonksiyonlarda hiçbirini üretmedi.

`structure()`Bir blokun hemen sonrası dominantoyu, bir blokun sonuna kadar iki gerçek gerektirir.`Branch`Bir arka kenar, kaynağını ele alan bir blokun kenarıdır. Doğal döngü, başlığı geçmeden arka kenar kaynağına ulaşan blokların bir kümesidir.

Bir .`Branch`katlanmakla`j`- ...

- - Eğer`j`Atlama hedefi, yayın`if (!cond) { fallthrough region }`Ve devam et .`j`- Evet .
- - Eğer`j`Bu, düşüşün, yayımlamanın bir yolu.`if (cond) { target region }`Ve devam et .`j`- Evet .
- - Eğer`j`Bu da bir işlev çıkışı, her iki kol geri döner.`if (cond) {... }`Ve diğer kolun arkasından, bir `else`Bu erken dönüş .`percent_change`ve `total`- Evet .
- Aksi takdirde yayın .`if (cond) { target } else { fallthrough }`Ve devam et .`j`- Bu ...`percent_change`- ...`-O0`, her iki silahın da depoladığı yer .`local_4`Ve epilogta buluşuruz.

Bir döngü başlığı için:

- Başlık bir  ile sona erirse`Branch`Bir halefiyle, döngü dışında, yayın`while (cond) { body }`- Evet .`total`- ...`-O0`Şöyle şekillidir: başlık karşılaştırır `local_14`- Evet .`local_c`ve yapraklar devam ediyor.`jge`- Evet .
- Eğer başlık aynı zamanda arka kenarlı bloksa veya kilit çıkışları ise, yayın `do { body } while (cond);`- Evet .`total`- ...`-O1`Bu şekli: blok`0x50`ekler, artır, karşılaştırır ve kendi kendine atlar.

Bir `IndirectJump`Bir blokta hiçbir halefi yok.`grade`- ...`-O0`Gerçek bir atlama masası.`jmpq *%rax`5 nispeten karşılaştırma yoluyla `.rodata+0x0`Bu yüzden, bu noktada, bir tane daha yazıyor.`0x76`- Evet .`0x7f`- Evet .`0x88`ve`0x91`Bu, sadece dolaylı atlama yoluyla ulaşılabilir ve kaldırılmamıştır.`switch`- Evet .

### -O1 anahtarı ne yaptı?

- Evet .`-O1`Clang için bir atlama masası tutmadı .`grade`- Bilgisayarı .`score / 10 - 6`Sonra 5 girişli bir tabloyla geri dönüş değerini yükledi.`int`- ...`.rodata+0x14`İmza edilmemiş bir karşılaştırma , her ikisini de kapsayan bir talimat .`score / 10 < 6`ve `score / 10 > 10`Negatif bir fark büyük bir imzalanmamış değerdir, bu yüzden aynı kontrolden geçerli değildir.

Masada kalır .`1, 2, 3, 4, 4`6 ila 10 oranları için `switch`Beş kişiyle.`return`Açıklamalar bir `if`Asansör tam olarak bunu yayıyor ve masanın adını veriyor.`lookup_14`Bölümünün ödenmesinden sonra.

Masanın uzunluğu heuristiktir.`table_values()`Masa başından sonraki sayfaya kadar okuruyor `.rodata`Bu kadarını karşılaştır .`.text`Bu nesneye göre, sonraki referanslı ofset `0x28`, tablo`grade_label`- Evet .`lookup_14`- Evet .`(0x28 - 0x14) / 4 = 5`Ghidra'nın uzunluğu da bilmiyor.`*(uint *)(&DAT_001000b4 + (ulong)uVar2 * 4)`, bir byte adresi artı bir ofset, ve okuyucu için bakmak için beş değer bırakın.

## Ghidra'yla Birlikte

Demo her çiftin baskılarını yapar.`code/fixtures/ghidra/*.json`... 2026-10-14'te geri döndüğünü gösteren kanıt kayıtları.`npx -y rea-agents@6.5.0 decompile ledger-x86_64-O1.o <function> --provider ghidra --format json`Her kayıtta bir tane var .`confidence: observed`ve `authority: shipped-artifact`Metin gözlemlenir. metin içindeki iddialar çıkarılır ve kayıtlarda hangileri belirtilmez.

### %_değişim

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

Düzeltilen versiyon .`code/fixtures/ghidra/ledger-x86_64-O1.ghidra-repaired.c`Bu test sadece bu işlevi düzenler ve test 81 giriş çiftinin kaynağı ile karşılaştırılır.

### sınıfı

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

### toplam

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

Davranış testi Ghidra metnini oluşturur.`grade`ve `total`Değişmemiş.`typedef`Sıkıntılı bir şekilde`DAT_001000b4`20 byte'den `.rodata+0x14`Her iki fonksiyon da kaynakla eşleşir.

## Araştırmalar Pseudocode Okuması Hakkında Ne Söyler

LLM4Decompile (EMNLP 2024) Ghidra'yı tek başına HumanEval-Decompile'de yüzde 20.12 yeniden uygulanabilirliğe değerlendirdi. Beş işlevden biri, referans değerinin yanında gelen iddiaları yeniden birleştirdi ve geçti. Ghidra'nın 6.7B modelleriyle üretimini iyileştirmek bu oranı yüzde 52.74'e çıkardı. DecompileBench (Findings of ACL 2025) 130 programdan 23.400 işlevi üzerinde altı endüstriyel dekompiliyer ve altı LLM yaklaşımı yürütmüştür. LLM sonuçları daha iyi okundu ve %52,2 daha düşük bir işlevsel doğruluk gösterdi.

Liu, Raff ve Micinski (arXiv 2609.05370, Eylül 2026) boşluğu açıklar. GitHub kütüphanesi fonksiyonları 300 ve CVE tabanlı fonksiyonlar 287 kullanmışlar. En güçlü rafine modelinde Ghidra'nın inşa oranı yüzde 75'ten yüzde 90'a yükseldi. Davranışlı eşleşme oranı ise yüzde 74'ten yüzde 62'ye düştü. Benchmark'daki her sınavı geçen adaylar hala karışık girişler konusunda farklılık gösterdi. Bu, sekiz sistemdeki vakaların yüzde 4,9'unda ve bir sistem için yüzde 13'e kadar oldu. Açıklanan güvenlik açığıların onundan biri kadar, daha iyi kodlardan kayboldu.

Bu tür türde yeni yapılmış alanlar, türler, kalleler ve koruyucular, yazıda dekompilerci tarafından bırakılan görünen bilinmeyen şeyleri değiştirdi.

Bu yüzden asansör izleri .`(unsigned long)(unsigned int)ecx_1`Daha güzel bir isim yerine bir not yazdırıyor.`switch`çözülmemiş atlama için ve`lookup_14`Bir model etiket yerine bir etiket icat eder. her satırın okuyabileceğiniz bir türev vardır.`code/main.py`Ghidra'nın.`undefined1 [16]`- Güzel bir model temiz.`int percent_change(int, int)`Aynı model bir sonraki işlevi temiz yanlış imza üretir.

## Yapın

Demo çalıştır:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode
python3 code/main.py
```

Demo , `-O1`Oturma ve her talimatı , yapılmış olanlarla karşılaştırır .`llvm-objdump`- Mesaj için.`percent_change`- Evet .`grade`ve`total`SSA'nın önüne IR'yi, phi fonksiyonları olan SSA listesini ve Ghidra metninin yanındaki pseudokodu yazdırır.`grade`Bölüm tanıma kapatıldığında, `-O0``percent_change`- Evet .`total`ve`grade_label`, ve bir özet:

```json
{
  "functions": ["grade", "grade_label", "percent_change", "total"],
  "instructions": 56,
  "blocks": {"percent_change": 3, "grade": 3, "total": 5, "grade_label": 4},
  "phis": {"percent_change": 0, "grade": 1, "total": 2, "grade_label": 1},
  "objdumpDifferences": 0
}
```

Oku `code/main.py`Bu sırada:

1. `read_object()`Bölüm başlıklarını, sembol tablosunu ve `.rela`ELF64 yer değiştirilebilir bir nesnenin bölümleri.
2. `decode_one()`ve `render_att()`Bir talimatı çözüp , doğru şekilde yazdır .`llvm-objdump`- Evet.
3. `build_blocks()`- Evet .`immediate_dominators()`ve`dominance_frontiers()`Grafik gerçekleri oluşturun.
4. `lift_insn()`Bir talimatı IR açıklamalarına dönüştürür ve yer değiştirmeleri çözür.
5. `insert_phis()`ve `rename()`SSA'yı oluşturup genişlik dökümlerini ekle.
6. `make_simplifier()`- Evet .`substitute_once()`ve`remove_dead_code()`İçeri koş .`optimize()`- Evet .
7. `collect_params()`- Evet .`coalesce_webs()`ve`infer_types()`Ağların adını ve yazısını.
8. `structure()`- Evet .`post_dominators()`ve `natural_loops()`C vücudu yayıyor.
9. `lift_function()`aşamaları bir araya getirir ve`lift_all()`Bir nesneyi Callee imzalarıyla kaldırır.

## Kullan

Bir fonksiyonu kendi nesnenin üzerinden kaldırmak için, onu kullanın.`clang -target x86_64-unknown-linux-gnu -c`Ve aramak .`lift_all(read_object(path))`- Dekodör yüklendiğinde .`DecodeError`, mesajdan opcode byte okuyun ve SDM opcode haritasında bul. Sonra dosyanın üstündeki tablolara bir giriş ekleyin.`lift_insn()`- Yapımcı bir `goto`, grafikleri basın `IrPrinter`Her eklem küçük ve her birine karşı kontrol edilir.`llvm-objdump`Ya da mevcut testler tarafından çığlık atmak.

Ghidra yüklendiğinde, aynı işlevi çalıştırın ve iki metni yan yana koyun. Herhangi bir metneye inanmadan önce fark tablosunu doldurun. Arkasında bir talimat olan bir fark sunum seçeneğidir. Arkasında bir talimat olmayan bir fark bir tahmindir ve tahmin her iki taraftan da olabilir.

## İnteraktif Laboratuvar

### Laboratuvar A: beş aşamada bir fonksiyon

- Evet .`code`Dizin, açık Python:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode/code
python3 -q
```

Çık:

```python
from main import *
obj = read_object(FIXTURES / "ledger-x86_64-O1.o")
lifted = lift_function(obj, "percent_change")
for insn in lifted.decoded: print(f"0x{insn.address:x}: {insn.raw.hex(' ')}  {insn.att()}")
print(lifted.ir_text)
print(lifted.ssa_text)
print(lifted.pseudocode)
```

Takip edin .`%esi`-`subl %edi, %esi`- ...`esi_1 = esi_0 - edi_0`- ...`(param_2 - param_1)`- Bayt .`fe`  modr/m`subl`, ile`mod = 11`- Evet .`reg = edi`ve`r/m = esi`- Evet .

### Laboratuvar B: Bölüm dilini

```python
print(lift_function(obj, "grade", recognize_division=False).pseudocode)
print(magic_divisor(0x66666667, 34), magic_divisor(0x66666667, 33), magic_divisor(0x66666667, 31))
```

İlk satır çiğ çarpımı ve değişimleri basar.`10 5 None`Aynı çarpıcının neden bir şifte az olan 5 ile bölündüğünü ve neden 31 şifte hata sınırını çiğnediğini öğrenin.

### Laboratuvar C: Yeni bir nesne dekodörü durdurur

`-O2`Ve kaldır:

```bash
clang -target x86_64-unknown-linux-gnu -O2 -fno-asynchronous-unwind-tables -c fixtures/ledger.c -o "$TMPDIR/ledger-O2.o"
python3 -c 'from main import *; from pathlib import Path; import os; lift_all(read_object(Path(os.environ["TMPDIR"]) / "ledger-O2.o"))'
```

Dekodörün durması `0x56`- Evet .`opcode 81 is outside the fixture subset`- Opcode .`81`32 bitli bir anlık form olan grup 1 formudur.`andl $0x7ffffff8, %edx`, ve Clang onu vektörlü döngüde kullandı `total`- Yanına ekle .`0x83`İçeride`decode_one()`Alt kümesi bir seferde bir giriş büyütür ve her giriş karşılaştırılır `llvm-objdump`- Evet .

### Laboratuvar D: Ghidra'nın standart değeri derecede

```python
print(ghidra_pseudocode("grade"))
for score in (-25, 0, 59, 60, 100, 115):
    sign = -(score >> 31)
    product_sign = ((score * 0x66666667) & ((1 << 64) - 1)) >> 63
    print(score, sign ^ product_sign)
```

Her satırda 0'luk izler. Ghidra'nın `uVar1`ifade doğru ve okunamaz ve talimat `xorl %eax, %eax`İki baytta yazıyor.

## Pratik Laboratuvar

Ekle`switch``grade`- ...`-O0`- Evet .

Gereksinimler:

- Beş yer değiştirmeyi okuyun .`.rela.rodata``ledger-x86_64-O0.o`, her biri`R_X86_64_PC32.text+offset`, ve 5 dava hedefini hesaplayın .`.rodata+0x0`- Evet .
- Bir ekle`Switch`Endeks ifadesi, hedef listesi ve öntanımlı `ja`Masanın yüklenmesinden önce.
- Sıkıştırıcı blokuna hedef başına bir halefi verin böylece dominanterler, post-dominatorlar ve SSA çalışmaya devam etsin.
- Çıkar .`switch (local_10) { case 0:... }`Bir taneyle.`case`Birleştirilmesinden sonra hedef ve ortak epilog.
- Masasını bağlayamayan herhangi bir indirek at için not tut.
- Sonuçları clang ile oluşturun ve testlerde puan çizelgesinde kaynağı ile karşılaştırın.

Başarılı olmak demektir .`grade`- ...`-O0`ve `grade`- ...`-O1`Aynı çıkışın ağ üzerinde üretilmesi, bir `switch`Ve hayır .`goto`- Evet .

## Sanatlı

Bu ders bize çok yararlı .`outputs/skill-lifting-to-pseudocode.md`. Her satır için bir türev ile bir işlevi kaldırmak ve bir dekompiler metnini karşı fark tablosunu doldurmak için kullanın.

## Kontrol et

Demo ve Deterministik Suite'i çalıştır:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

- Suit 25 testle oluşuyor.

- Dekodör, sözleşmeyi kabul eder.`llvm-objdump`Her iki nesneyi de bir canlı ile yazıyor.`llvm-objdump`kurulduğunda
- REX, ModR/M, SIB ve `leaq (%rip), %rcx`SDM'de açıklandığı gibi kodlandırılır
- Alt kümeden dışarıda bir opkod yükseltir `DecodeError`Tahmin etmek yerine
- ELF okuyucuunda dört işlevi ve üç işlevi listelenir `.text`Yer değiştirme
- `total`Beş bloklu bir yer.`0x44`Yöneticisi .`0x50`ve`0x50`Kendisiyle bir döngü başlığı olan bir vücut
- `%rdi`ve `%rsi`Olmak`param_1`ve `param_2`ve`param_1`İçeride`total`- Evet .`int *`
- `test`ve `cmp`- Evet .`je`- Evet .`jle`- Evet .`ja`ve`jge`Olmak`==`- Evet .`<=`İmza edilmemiş .`>`ve`<`
- SSA iki canlı phi fonksiyonunu ın döngü başlığına yerleştirir .`total`Ve hiçbiri .`percent_change`
- `percent_change`- Evet .`grade`ve`total`beklenen metin ile eşittir
- `grade`okur `param_1 / 10 - 6`Tanınmış ve çiğ çarpı tanınmamış olarak
- `0x66666667`34'ün 10'a bölünmesi, 33'ün 5'e bölünmesi ve kimlik ağda kalması
- `-O0`Olmak`local_4`- Evet .`local_8`- Evet .`local_c`, ve çerçeve işaretçisi asla görünmez
- `.rodata+0x14`okur `1, 2, 3, 4, 4`Yer değiştirme yoluyla
- indirekt sıçrayış .`grade`- ...`-O0`4 taşınmamış blokları ile rapor edilir.
- Ghidra cihazları 16 byet geri dönüşü içerir.`param_3`, basitleştirilmemiş .`xor`, ve `long param_1`
- 6 emisyon birimi ile oluşturulur.`clang -std=c11 -Wall -Werror`için`x86_64-unknown-linux-gnu`
- kaldırılan fonksiyonlar, onarılmış Ghidra metni ve kaynak, giriş şebekesi üzerinde aynı 232 satır yazdırıyor

## Üretim Başarısızlık Modları

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

## İşlem Kuralı

Bu nedenle, bu metinleri birbiriyle birlikte yerleştirin ve her farklı satırın arkasında talimat ya da tahmin yazın. Her ikisini de birleştirin ve kaynağa karşı çalıştırın.

## Daha Fazla Okumak

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
- rea 6.5.0 kaynağı tag `3b79af2`Bu ders şöyle anlatıyor:`src/domain/functionComparisonDimensions.ts`- Evet .`website/evidence/dx-ball-sound-pan.md`- Evet .
