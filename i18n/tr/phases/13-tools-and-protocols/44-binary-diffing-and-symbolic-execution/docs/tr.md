# İkilik Diff ve Sembol İcra: Eşleşen Fonksiyonlar ve Girdiler için Çözüm

> İki fonksiyon arasındaki bir eşleşme, tek bir kaliteyle tek bir imza tarafından yapılan bir iddia ve çözülmüş bir giriş, tek bir yol koşulunun bir modelidir; bu ders her ikisini de oluşturur ve onları radiff2, Ghidra, angr ve z3 ile karşılaştırır.

**Type:** Build
**Languages:** Python, C, x86-64 assembly, Java (one Ghidra script)
**Prerequisites:** Phase 13 · 33 (disassembly and control flow), 34 (lifting to pseudocode), 36 (rebuild and prove equal)
**Time:** ~150 minutes

## Öğrenme Hedefleri

- Bir nesnenin ve listelenmesinden beş işlev imza oluşturun: tam baytlar, maskeli yerleşim alanları olan baytlar, mnemonik dizisi, akış-grafi şekli ve sembol adı.
- BinDiff tarzındaki açgözlülükle düzenlenen bir maç çalıştırın, burada bir imzanın sadece her iki tarafta eşsiz olduğu zaman eşleşir ve her maçın arkasındaki aşama ve kaliteyi kaydetin.
- İki byte-içinli fonksiyonun yüklenmiş görüntülerde byte korelatörü eksik olduğu durumunu yeniden üretin ve değiştirilen dört byteyi açıklayan yer değiştirme aritmetikası ile maskeliştirildikten sonra geçin.
- Radiff2'nin benzerlik oranlarını Levenshtein mesafesinden yeniden üretin ve bir oranın neden yalnızca 82 byte radare2 elde edildiğinde göründüğünü açıklayın.
- Küçük bir IR üzerinde bir fonksiyonun yollarını sayın, yol başına bir koşul taşıyin ve sınırlı bir arama ile girişler için her yaprakı çözün.
- Simvolik bir sınırlı bir döngü üzerinde yol patlama ölçüm ve sayılar, sınıflar ve modelleri angr 10.0.3 ve z3 5.1.0 ile karşılaştırın.

## Sorun

Yeniden inşa ettin .`ledger.c`- ...`-O2`Ve bir ajanı sormuştu , ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ... ...`-O1`Ajan bir farklılık yaptı ve rapor etti ki`grade`Bu programın altındaki 46 byet, mevcut olmayan bir değişimi açıklamak için bütçesini harcadı.`grade`Bu iki nesne dosyasında aynı görüntüleri kullanır.`.rodata`                                         `-O2`Program, yani bir yer değiştirme `grade`0x70 oranında büyümüş.

Aynı öğleden sonra hangi puanı istediğini sordun .`grade`Bu bir cevap. fonksiyon 90 ila 109 arasındaki her puan için 4'ü gönderir. Sorunun bir sınırlılığı yoktu, bu yüzden ajanın cevabı bir örnekti.

Bu ders, iki yöntemi standart kütüphane Python'da inşa eder ve bunların yanında üretim araçlarını çalışır. Araçlar ve kod anlaştığında iki türden çıkış vardır. Anlaşmaz olduklarında bir adres, bir bayt veya bakmak için bir bağ vardır.

```figure
re-44-matches-and-paths
```

## İşlev İmzası - Bir Kaliteli İddia

Flake 2004 yılında iki benzer yürütülebilir fonksiyon kümesi arasındaki izomorfizmin yapımı olarak ikili farklılık çerçevesini oluşturdu. Dullien ve Rolles 2005 yılında temel bloklara ve talimatlara kadar genişletti. BinDiff kavramlar sayfası, bu çalışmanın ortaya çıkardığı stratejiyi açıklar. Her iki tarafta da her fonksiyon için bir özelliği hesaplayın. Her tarafta tam olarak bir kez görünen bir değer eşleşir. Birden fazla kez görünen bir değer belirsizdir ve bu eşdeğerlik sınıfı içinde bir sonraki özelliği ile aşağıya çekilir. Eşdeğersiz bir değer eşsiz kalır. Her geçitten sonra, yeni eşleşmelerin çağrıcıları ve çağrıcıları, bir sonraki özelliğin benzersiz olması muhtemel olan küçük kümeler oluşturur ve işlem bir sonraki özelliği ile tekrarlanır.

Her eşleşen algoritmanın bir kalitesi vardır. Sayfalar çiğ fonksiyon baytlarının ve isimlerin eşleşmesi ile eşleşme ile çok iyi, akış-grafi MD indeks ve baş imza eşleşmesi (her bir mnemonik küçük bir baş, fonksiyon üzerinde ürün) iyi, dizileri referansları ortalama olarak ve döngü sayısı veya adres dizisi kötü olarak değerlendirir. Bir maçın güvenliği, onu üreten algoritmaların ortalama kalitesi, bir sigmoid ile ağırlaştırılır. Fonksiyon benzerliği yaklaşık %25 eşleşen akış grafik kenarları, %15 eşleşen temel bloklar, %10 eşleşen talimatlar ve %50 güvenilirlikle çarpılan akış grafik MD indeksindeki farkdır.

Bu ders, bu özelliklerden beşi ve önemli olan tek kuralı korur: her iki tarafta da benzersiz veya eşleşmez. MD indeksini düşürür, çünkü kavram sayfaları formül olmadan isimlendirir ve yerine basit bir derece dizisini kullanır. Kalite etiketleri kalır, çünkü eşleşme sadece onu üreten en zayıf gerçek kadar iyidir.

## Bir fonksiyon kaydından Beş İmza

Farklılık her nesne için iki şeyi okur. ELF bayt, sembol boyutları ve `.rela.*`Kayıtlar.`llvm-objdump -d`listing talimat sınırları ve mnemonik verir ve ders 33 bu nesnelerin her talimat üzerinde listede eşittir dekoder kanıtladı. `function_record()`listelenen talimatların baytlarının bu adresdeki nesnenin baytlarına eşit olduğunu kontrol eder ve başka bir şekilde yükselter.

Al .`grade`İçeride`ledger-x86_64-O1.o`Simbolde 0x10 ve boyut 46 oranında sıfırlama yazıyor . Bu yüzden kayıt 0x10 ile 0x3e arasında yer alır ve iki baytı hariç tutulur .`nop`Bu da bir sonraki fonksiyonu düzeltir.

Bu sayede, bu sayede, bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binanın altındaki bir binin altının altındaki bir binin altının altındaki bir binbinte bir binin altındaki bir binin altının altındaki bir binbinte bir binin altındaki bir binin altının altındaki bir binbinte bir binin altının altındaki bir binbin altındaki bir binbinte bir bin bin bin bin binin altının altının altındaki bir bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin bin`R_X86_64_PC32 .rodata+0x10`0x36'da, dört byte genişliğinde, `leaq (%rip), %rcx`RELA nesnesinde bu dört bayt zaten sıfır tutar, bu yüzden iki imza burada her nesnenin her fonksiyonuna katılır.

Mnemonik imzası 14 mnemonik sekansını hash eder:

```text
movslq imulq movq sarq shrq addl addl xorl cmpl ja movl leaq movl retq
```

Akış grafik şekli imzası, girişin ulaşabileceği bloklara sınırlı olan ders 33'ün temel bloklarından ve tiplenmiş kenarlarından gelir. `grade`- ...`-O1`0x10 blokları, 3 blok ve 3 kenarı ile tamamlanır.`ja 0x3d`Alınan kenar ve düşme kenarıyla, blok 0x31 0x3d'ye düşer ve 0x3d geri gelir. Derece dizisi blok başına sıralanmış bir sırada (girin, çık) listeler yapar, böylece imza okur `3b 3e [0/2,1/1,2/0]`- Evet .`grade`- ...`-O0`Kayıt , beşinci sıçrama yoluyla indirek sıçramaları çözüyor .`.rela.rodata`33 dersinde olduğu gibi girişi yapar. Bu da sekiz blok ve dört ile on bir kenar verir.`table`Kenarları.

İsim imzası, sembol adı, var olduğu zaman en güçlü gerçek ve yapı çıkarıldığında hiçbir şey.

Bir fonksiyonun bir `lea``.rodata`, bir sınır kontrolü ve bir indeks yükü,`referenced_table()`Tablo 36 dersinde olduğu gibi okunuyor:`cmpl $0x4`Daha önce`ja`Beş tane giriş ve yük verildi.`movl (%rcx,%rax,4), %eax`Her biri 4 byte verir.`grade`- ...`-O1``.rodata+0x14`okur `1, 2, 3, 4, 4`- Evet .

## Açgözlü Eşyacı

`match_functions()`Bu sayede, her aşamada tam olarak bir işlevi olan her değer kabul edilir. maç aşama, aşamanın kalitesi, aynı zamanda aynı olan diğer her imza ve benzerlik oranını kaydeder.

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

Ders sırası, ilk olarak dört yapısal aşamayı ve son olarak isimleri yürütür, böylece yapısal aşamalar test edilir ve isimler onları daha sonra sınıflandırır.`-O1``-O2`- ...

| Function | Stage | Agreeing signatures | Similarity |
|---|---|---|---|
| `percent_change` | exact bytes | all five | 1.000 |
| `grade` | exact bytes | all five | 1.000 |
| `total` | name | name only | 0.171 |
| `grade_label` | exact bytes | all five | 1.000 |

`total`Clang, 31 byte ve 13 talimattan 140 byte ve 41 talimatlara yükseldi, çünkü clang, bu döngüyü SSE2 ile vektörleştirdi ve bir skalar kalıntı döngüsü korudu.

- Evet .`-O0``-O1`Aynı sırada bir yanlış eşleşme oluşur:

| Function at -O0 | Matched at -O1 | Stage | Names agree |
|---|---|---|---|
| `percent_change` | `grade_label` | flow-graph shape | no |
| `grade` | `grade` | name | yes |
| `total` | `total` | name | yes |
| `grade_label` | unmatched | | |

`percent_change`- ...`-O0`Dört bloklu bir elmas: bir giriş ki dalları `before == 0`Bu da bir arıza, iki kol ve bir birleşim.`0/2, 1/1, 1/1, 2/0`- Evet .`grade_label`- ...`-O1`- Evet .`grade`Bu nedenle, bir diğer değerli değerli değerler, yani değerli değerler, değerli değerler ve değerler, değerli değerler ve değerler ile ilgili olarak, değerlendirilmiş değerler ve değerler ile ilgili olarak, değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler ile değerlendirilmiş değerler arasında bağlantılıdır.`grade_label`Sağda, yani gerçek`percent_change`ve `grade_label`Çiftler eşsiz kalır.

BinDiff siparişi gerçek isimlerin var olduğu ikinci aşama olarak isim eşleşimi yürütür ve `BINDIFF_ORDER`Bu şekilde dörtü de aynı şekilde.`-O0`fonksiyonları eşleşir.`-O1`Ghidra'nın aşağıdaki raporları, Ghidra'nın aşağıdaki sonuçları olan isimle versiyonları belirler. Ders, her iki siparişini de demo'da tutar, böylece başarısızlık görünür.`if`ve `else`Geri dönmeden önce bir araya gelir.

## Aynı baytlar, yer değiştirilmiş operatörler

Ghidra bir nesneyi ithal ettiğinde yer değiştirme uyguluyor. Bu nedenle byte korelatörü yüklenmiş görüntüleri karşılaştırır ve ders 36 maskeli yer değiştirme alanları bu nedenle. `load_image()`Üretimleri, bölgeleri başlık sırasıyla gösterir, her bir bölümün kendi başlıklarına uyum sağlayarak `sh_addralign`, ve 0x100000'den sonra her kayıt uygulanır:`S + A - P`32 bitlik bir değer olarak yazılmış .`R_X86_64_PC32`ve `R_X86_64_PLT32`- Evet .`S + A`64 bit olarak `R_X86_64_64`- Evet .

Düzenleme, her iki programda her bölümün başlamasındaki Ghidra 12.1.4 ile uyumludur. Test süiti, alınan Versiyon Takip çıkışına karşı kontrol eder:

| Section | `-O1` program | `-O2` program |
|---|---|---|
| `.text` | 0x100000, 159 bytes | 0x100000, 271 bytes |
| `.rodata` | 0x1000a0 | 0x100110 |
| `.rodata.str1.1` | 0x1000f0 | 0x100160 |

`.text`112 byte daha uzun `-O2`, `total`- Evet .`.rodata`0x70 bayt daha ileriye başlar. `grade`Her iki programda 0x100010'da oturuyor.`leaq`0x33'te, 0x36'da ve 0x3a'da sona erer.`.rodata+0x14`- Evet .

- Evet .`-O1`Resim: `0x1000a0 + 0x14 - 0x10003a = 0x7a = 122`- İçeride .`-O2`Resim: `0x100110 + 0x14 - 0x10003a = 0xea = 234`- Alan baytları `7a 00 00 00`Bir görüntüde ve `ea 00 00 00`Bu da bir diğerinden farklıdır. 46'ın bir baytı farklıdır, Levenshtein oranı 0.978, ve tam bayt aşaması başarısız olur.`grade_label`Bu sayede, masa ile aynı 0x70 ile hareket eder, böylece yer değiştirme oranı eşit kalır ve her iki bakışta da tam baytlar ile geçer.

Ghidra aynı bölünmeyi rapor ediyor.`-O1``-O2`, Tam fonksiyon byte eşleşimi bulundu `percent_change`ve `grade_label`Tam fonksiyon talimatları eşleşirken bu iki kişiyi bulduk .`grade`, ve Tam Fonksiyon Mnemonik Uyumları aynı üçü bulmuştur. Görev korelatörü, yer değiştirmeyi kapsar. Dersin görüntü görüntüsü, Ghidra'nın bayt korelatörü bulduğu iki fonksiyon için tam-bayt eşleşmelerini ve talimat korelatörü bulduğu tam üç için maske-bayt anlaşmasını verir. Test kümesi her iki eşitliği de doğruluyor.

6.5.0 rea'lar `compare_functions`Bu dersdeki yükleme cihazı ve maske bu normallaşmayı sağlar.

## Hangi tedbirler

6.2.4 radar gemileri`radiff2`- Evet .`-AC`Dosyaları analiz eder ve her fonksiyona bir satır yazdırır: sol boyut ve adres, MATCH veya UNMATCH, bir oran ve sağ boyut ve adres.

| Pair | `percent_change` | `grade` | `total` | `grade_label` |
|---|---|---|---|---|
| `-O0` vs `-O1` | UNMATCH 0.130435 | UNMATCH 0.426829 | UNMATCH 0.106061 | UNMATCH 0.269841 |
| `-O1` vs `-O2` | MATCH 1.000000 | MATCH 1.000000 | UNMATCH 0.171429 | MATCH 1.000000 |
| `-O1` vs `-O1` | MATCH 1.000000 | MATCH 1.000000 | MATCH 1.000000 | MATCH 1.000000 |

Radar2 kaynağının oranı `1 - distance / max(len_a, len_b)`, burada iki fonksiyonun parmak izi ile parmak izi arasındaki Levenshtein mesafesi fonksiyonun temel bloklarının adres sırasındaki bağlanmasıdır.`radare2_ratio()`İçeride`code/main.py`çiğ fonksiyon baytları üzerinde aynı formülü uyguluyor ve 0.130435, 0.106061, 0.269841, ve 0.171429'u altı onluklara kadar yeniden üretir.

0.426829' u çoğaltmıyor.`grade`- ...`-O0`. Fonksiyonun 118 bayt üzerinde oran 0.296610. radar 2 baskılarıdır `118`Bu nedenle, analiz giriş blokuna ulaşır, blok sonunda `jmpq *%rax`, öntanımlı durum ve geri dönüş: 82 byte, ders 33'ün düz geri dönüşsel düşüşü iddia ettiği aynı 82 .`-O1`47'e kadar.`1 - 47/82 = 0.426829`- Evet .`-e bin.relocs.apply=true`radar2 tüm 118 baytı bulur ve 0.288136 yazdırır. Bu, dört yer değiştirme baytının nesnenin sıfır alanlarından farklı olduğu yer değiştirilen görüntüler üzerindeki ders oranına eşittir.

`-O`Parmak izi opkod baytlarına sınırlandırır ve operandları sıfırlandırır.`-O0`karşılaştırıldığında`-O1`Bu oranlar 0.239130, 0.512195, 0.303030 ve 0.523810'a yükseldi.`total`- ...`-O1`karşılaştırıldığında`-O2`Tüm dosya mesafeleri de var: `-s`(Myers) benzerlik 0.956 ve iki nesne arasındaki mesafe 162 rapor ediyor.`-ss`Bu sayılara başlıklar, sembol tabloları ve yer değiştirme bölümleri dahil.

## Ghidra Versiyon Takip Korellatörleri

Ghidra'nın Versiyon Takip aracı, kaynak programını bir hedef programına ilişkili bir programla eşleştirir. Fonksiyon korelörleri, Tam Fonksiyon Bytes Düzeneği, Tam Fonksiyon talimatları Düzeneği (maskeli operandlar), Tam Fonksiyon Mnemonik Düzeneği ve Duplikasyon Fonksiyon talimatları Düzeneği (1:N ve N:M). Veriler ilişkilileri Tam, Duplikat ve Benzer Veriler Düzleşir. Sembol ilişkileri isimlerle aynı şekilde, kopya olarak veya benzerlik ile eşleşir. Referans korelatörleri daha fazla bilgi bulmak için kabul edilen eşleşmeleri kullanır. İşlevler hedef programda tanımlanmalıdır ve en az işlev uzunluğu 10 bayt'a varır. Otomatik Versiyon Takip, Tam Simbol Adı, Tam Veriler, Tam Fonksiyon Bytes, Tam Fonksiyon talimatları, Tam Fonksiyon Mnemonikleri, Çift Fonksiyon talimatları, sonra bir referans korelatörü çalıştırır ve kabul edilenler için içten eşleşmeler oluşturur. 11.2'de BSim korelatörü eklendi.

Ders, ilişkileri başsızca yakaladı.`capture_ghidra_vt.sh`Üç nesneyi bir projeye aktarıp çalıştırır.`ListVersionTrackingMatches.java`, kısa bir senaryo .`AbstractGhidraVersionTrackingScript`, kaynak ve hedef programlarını yolla açar, Otomatik Versiyon Takip sırasındaki altı referans olmayan korelatörleri çalıştırır ve blok, işlev, korelatör ve eşleşme başına bir satır yazdırır. Bu bölümdeki her sayı bu iki metin dosyasından gelir.

| Correlator | `-O0` to `-O1` | `-O1` to `-O2` |
|---|---|---|
| Exact Symbol Name Match | 4 functions | 4 functions, 5 data items (`labels.rel`, `.L.str.1` to `.L.str.4`) |
| Exact Data Match | 0 | 0 |
| Exact Function Bytes Match | 0 | 2 (`percent_change`, `grade_label`) |
| Exact Function Instructions Match | 0 | 3 (adds `grade`) |
| Exact Function Mnemonics Match | 0 | 3 |
| Duplicate Function Instructions Match | 0 | 0 |

Arada her fonksiyon değişmiş .`-O0`ve `-O1`Bu nedenle, bu sayfa, tüm benzerlikleri 1.0 ve güven 1.0 olarak tanımlar. Bu nedenle, basılan güven ölçeği, bu dersin kaydettiği açık bir sorudur ve çözülmez.

Program Farkları, Aletler altında farklı bir şeydir. Aynı adres alanı olan iki programı aynı adreslerde byte, etiket, kod birimleri, referanslar, yorumlar ve işlevlerle karşılaştırır.

## Diaphora ve Binary Ninja, Belgelerinden

Diaphora, ilk olarak 2015 yılında ve 2026-08-18 tarihli sürüm 3.4.2'de yayınlanan bir IDA Pro eklentisidir. 2.0'dan beri AGPL-3.0'dur ve IDAPython'a ihtiyaç duyar, bu yüzden IDA Home veya IDA Pro'da çalıştırılır ve IDA Free'da değil. Her veritabanını SQLite'ye ihraç eder ve heuristiklerini SQL sorguları olarak çalışır. Her heuristik bir isim, en iyi, kısmi veya güvenilir olmayan bir kategorinin, oran türünün ve SAME_CPU veya SLOW gibi seçmeli bayrakların vardır. Mevcut kaynaktaki isimler, bu dersin aşamaları gibi daha fazla okunuyor: Bytes hash, Aynı adres ve mnemonics, Aynı temizlenmiş birim, Mnemonics küçük prim ürün, Aynı düğümler, kenarlar, döngüler ve güçlü bağlantılı bileşenler, Topolojik sıralama hash, ve Hex-Rays dekompilerine ihtiyaç duyan pseudo-kod bulanık hashler. Yayın notları ayarlama tarihini gösterir: yavaş heuristikler sadece yaklaşık 2.000 fonksiyonun altında çalışır ve "Eşit grafik" güvenilir olmayan kategorilere taşındı. Bu hareket, bu dersin dört bloklu elmasıdır.

2026-09-03'te yayınlanan Binary Ninja 6.0, sağlayıcılar ve çözücülerle bir çerçeve olarak Binary Similarity'yi ekledi. BinDiff'i, yer değiştirülebilir operandlara dayanan tam fonksiyon eşleşmeleri için yapısal bir sağlayıcı ve WARP olarak gönderir ve rehberinin, yer değiştirülebilir talimatlardan ayrı fonksiyonlar eşleştiklerinde WARP'yi ve yeniden birleştirildikten sonra yapısal olarak ilişkili kaldığında BinDiff'i kullanmalarını söyler. Özellik sadece Ultimate baskısında. Binary Ninja'nın burada paket yöneticisi bulunmuyor ve çalıştırılmamıştır. Bu paragraf, onun belgelerinden ve başka hiçbir şeyden alınmıştır.

BinDiff, Apache 2.0'da 2023-09-25'ten beri açık kaynaklı bir uygulama olarak kullanılıyor. Son sürümle v8. Bir sökme cihazı ve IDA Pro 9.4 veya daha sonraki, Binary Ninja ve Ghidra'yı destekleyen BinExport eklentisine ihtiyaç duyuyor. Bir öğrenci için ücretsiz yol Ghidra artı BinExport artı BinDiff komut satırı. Ders onu çalıştırmadı ve pratik laboratuvarı bunu sizden istiyor.

## Aynı Kod, Farklı Sürekli

Ders 36 bir aday yeniden inşa etti `grade`Tablosunun son girişinde 3 olduğu, orijinalinde ise 4 olduğu. `grade_case10_wrong.o`Aynı 46 kod bytesine sahip `grade`- ...`-O1`.Matcher, her imzanın onayladığı tam baytlara eşleştiriyor.`1, 2, 3, 4, 4`Tek bir yer değiştirme yoluyla ve`1, 2, 3, 4, 3`Diğerleri ve raporlar `dataEqual: false`Diğer üçü .`-O1`sol tarafta eşsiz fonksiyonlar vardır, çünkü aday nesne bir fonksiyonu tutar.

Bu, yama analiz şeklidir. Byte'lerde durur ve hiçbir şey değişmediğini söyler. Ghidra'nın Tam Veri Düzeneği ve BinDiff'in İpucu-Stream Reference Düzeneği aynı nedenden dolayı var. Dersin versiyonu küçüktür: her eşleşen çift için, her fonksiyonun yer değiştirme noktası olan byte'leri, sınırların sayısını ve yükün genişliğini kontrol ederek karşılaştırın.

## Simvolik İdam: Yollar, Koşullar, Çözücüler

King, 1976'da sembolik yürütmeyi tanımladı. Girişler semboller haline gelir. Her ifade sembolik bir depoya güncellenir. Her dal yoluna bir koşul ekler ve çözücü yolun uygulanabilir olup olmadığını belirler ve ardından bir beton giriş üretir. Dalları olan bir işlev yol ağacı olur ve bir yaprak bir koşulla bir dönüştür.

Baldoni, Coppa, D'Elia, Demetrescu ve Finocchi 2018 yılında bu alanı araştırmışlardı. Bölüm 5'te, yol patlaması merkezi maliyet olarak adlandırılır: bir yürütücü her dalda çatlak yapabilir, bu nedenle devlet sayısı dal sayısında eksponansiel olarak büyüyebilir ve bir döngü, sınırları sembolik bir girişten bağlı olan bir döngü sınırsız bir yol kümesi verir. Listelerindeki hafiflemeler, her dalda çözülmeyen yolları kesmek, fonksiyon ve döngüler için özetler, devlet birleşimi ve az yaklaşım. Konkoloş yürütme, programı bir anda, konkret ve sembolik olarak çalıştırır, beton çalışması boyunca koşulları kaydeder, birini reddeder ve çözücüden bir sonraki giriş için sorar. DART (2005), CUTE (2005), ve SAGE (2008) isimlendirilmiş sistemlerdir.

KLEE (Cadar, Dunbar, Engler, OSDI 2008) LLVM bitkod üzerinde sembolik bir sanal makine, bu nedenle girişleri `clang -emit-llvm`KLEE 3.2'nin, NCSA lisansı altında 2025-12-23'te yayımlandı ve LLVM 16 önerildi.

Angr ikili işlemlerde çalışır.`angr.Project`nesneyi haritalayan ve yer değiştirme uygulayan bir CLE yüklemeci, bir bloku VEX IR'ye dönüştüren bir kaldırıcı (`block.vex`) ve devletler için bir fabrika ile alternatif olarak pypcode ile.`SimState`Kayıtlar, hafıza ve çözücü.`SimulationManager`Adlı devletlerin adımları `active`- Evet .`deadended`- Evet .`found`- Evet .`avoided`ve`errored`ve`explore(find=..., avoid=...)`Çelişki, Z3 üzerinde çözücü ön ucudur ve Angr 10.0'da ise ithalat edilen bir Rust uygulamasıdır.`from angr import claripy`. angr 10.0.3 2026-10-10'da yüklendi, BSD-2-Clause'dir ve Python 3.12 veya daha sonraki bir program gerektirir. MIT'in 2026-08-16 altında yayınlanan Z3 5.1.0, angr, Triton ve KLEE'nin seçmeli arka uçlarının arkasındaki çözücüdür.

## Ders Üzerinde Küçük Bir İnkârcı 34 IR

İdamcı ,`code/main.py`Dersin ifade şekilleri ile küçük bir IR kullanıyor:`Assign`- Evet .`Branch`- Evet .`Jump`- Evet .`Return`, artı `Switch`Ders 34'ün Pratik Laboratuvarı'nın istediği terminatör.`Const`- Evet .`Var`- Evet .`Bin`genişliği ile, `Cast`- Evet .`Load`Adlı bir beton masadan, ve `Not`. Dört sabit işlev basitleştirilmesinden sonra ders 34'ün ulaştığı seviyede el ile kaldırılır.`0x66666667`katılaştırmak oldu `score / 10`, ve her IR blokundan gelen talimat adresleri listelenir.`check_ir_against_listing()`adreslerin fonksiyonun listesini tam olarak bir kez kapsadığını belirtir.

`grade`- ...`-O0`Bu IR'de:

```text
0x30: local_8 = score; local_10 = score / 10 - 6; if (local_10 >u 4) goto 0x9a else 0x62
0x62: switch local_10 -> 0x91, 0x88, 0x7f, 0x76, 0x76
0x76: local_4 = 4; goto 0xa1        0x7f: local_4 = 3; goto 0xa1
0x88: local_4 = 2; goto 0xa1        0x91: local_4 = 1; goto 0xa1
0x9a: local_4 = 0                    0xa1: return local_4
```

Değişken hedefler `jump_table_targets()`, 33 ders ile aynı yer değiştirme aritmetikası, IR'nin sabitlemeyi okuduğu gibi.

`execute()`Bir durum, mevcut blok, blokların izleri, koşul listesi ve her değişkeni girişlerin üzerinde bir ifadede haritası yapan bir ortamı içerir. `Assign`Çevreyi kaynağa değiştirir ve sonucu saklar. `Branch`Bu durumda, solver, solver'in yerine getirmesini ve solver'e iki kez sormasını ister: bir kez koşul eklenmiş, bir kez ise negatifleştirilmiştir.`Switch`Tablo girişlerini hedef ve çatallar olarak bir kez ayrı bir hedef için giriş eşitliklerinin ayrılması ile gruplar, Angr simgesel atlama hedefine de böyle davranır. `Return`Adım sınırı'ndan uzun bir iz, kısaltılmış yolu işaretler ve durdurur.

Yolumuzu takip edin .`return 4`Giriş, görevlendirme yapar.`local_10 = score / 10 - 6`- Şubenin durumu`local_10 >u 4`Bu yolda yanlış, bu yüzden durum listesi geçerli.`!(score / 10 - 6 >u 4)`.Switch girişleri 3 ve 4 her ikisi de 0x76'da belirtilir, bu yüzden çatal 0x76'a eklenir `score / 10 - 6 == 3 || score / 10 - 6 == 4`. 0x76 blokları .`local_4 = 4`, 0xa1'e atlar ve geri dönüş yerine `4`Yaprak, bu durum ve bu değerin çiftidir.

Çözücü `BoundedSearch`. Giriş başına sınırlı bir alanı tutar, alanların ürünü sayılır ve her kısıtlamayı C semantik ile değerlendirir: 32 bitlik sarmalama, kısaltma bölümü, imzalanan ve imzalanmamış karşılaştırmalar ve aritmetik veya mantıksal değişimler.`score`-200'den 1200'e kadar, `grade`Bu değerlendirme, 44460 değerlendirme ve birkaç milisaniye sürer.

| Leaf | Returns | Condition | Scores in the bound |
|---|---|---|---|
| 0x76 | 4 | `!(q >u 4) && (q == 3 \|\| q == 4)` | 90 to 109, 20 inputs |
| 0x7f | 3 | `!(q >u 4) && q == 2` | 80 to 89, 10 inputs |
| 0x88 | 2 | `!(q >u 4) && q == 1` | 70 to 79, 10 inputs |
| 0x91 | 1 | `!(q >u 4) && q == 0` | 60 to 69, 10 inputs |
| 0x9a | 0 | `q >u 4` | 1,351 inputs, the rest of the bound |

`q``score / 10 - 6`Aynı sınıflar çıkıyor .`grade`- ...`-O1`, IR'nin iki yolu olduğu yerde, masa yolu geri döner `lookup_14[q]`ve varsayılan yol 0'u geri verir ve soru geri gönderilen değere bir kısıtlama olur, çünkü ulaşılması gereken bir durum blok yoktur.`lookup_14`nesnenin görüntüsünden okunuyor.`.rodata`0x14'te.

Sınır, yöntemin sınırıdır. Sınırlı bir arama alanının içinde tamamlanır ve dışındaki sessiz olur. -200 ila 1200 arasında 60 ila 109 arasında hiçbir puan 0 göndermez ve 110 veya daha fazla puan sıfır dışı bir değer göndermez. z3 tüm 2^32 puanlar için her ikisini de kanıtlar ve sonraki bölüm iki yöntemin farklı tanıklar gönderdiğini gösterir.

## Yol Patlaması Ölçülmüş

`total`- ...`-O1`Dersin 33'ü: bir test`count`, ekleyen bir döngü vücudu `values[rdx]`ve artışlar `rdx`, ve bir `jne`Vücuduna dönerken`rdx != rcx`- IR'de .`rcx`- Evet .`zext64(count)`ve `rdx`0'dan başlıyor. Arka kenarından geçen her yol, çatalların üzerinde.`rdx != count`- Evet .`rdx`Bir sabit, bu yüzden yol için `count = c`Vücudunu yönetir.`c`Zaman ve çıkışlar.`count`0 ile 2 arasında sınırlı olan uygulayıcı 3 yol bulur. 0 ile 4 arasında 5 ve 0 ile 8 arasında 9 bulunur. Her yol sayısının üçgen toplamını gönderir: (0, 0), (1, 1), (2, 3), (8, 36). angr, 3, 5 ve 9 bitiş durumunu ve aynı dokuz sayım ve toplam çiftini üç optimizasyon düzeyinde gönderir.`-O2`, vectör döngüsünün en az 8 sayısına ihtiyacı vardır ve bu nedenle bu aralığın yol sayısını asla değiştirmez.

Hiçbir bağlanmadan uygulayıcı asla bitmez, çünkü her tekrar sayımı bir daha uygulanabilir bir yoldur.`count`Bu rakamlar, 64'e kadar, 22 yol geri verir, 2'yi kısaltır ve var olan 65'in diğer 41'ini asla başlatmaz. Rapor bu üç rakamı verir.

## Angr ve z3 Aynı Sorular

`capture_angr.py`CLE, her yer değiştirilebilir nesneyi 0x400000'de ELF arka uçla yükledi.`-O0`Ve diğerleri için 8 ve dört sembolü çözmüş.`grade`Simvolik bir 64 bit argüman ile,`run()`5 ülkenin sonu gelmişti .`-O0`Ve 2 tane .`-O1`ve `-O2`, 0.475, 0.408 ve 0.318 saniye içinde.`eax == v`Bu durumun sonucunda, her bir döngüden önce belirlenen ve çözücüye imzaladığı minimum ve maksimum miktarı soran kişi 60 ila 69, 70 ila 79, 80 ila 89, 90 ila 109 arasında bir miktar ödeme yaptı.`v`Her seviyede 1 ila 4 arasında ve 0'da -2147483648 ila 2147483647 arasında iki çapraz kontrol de yerine getirilmedi.

CFGFast' ın kullanıldığı Reach-a-Block formunda `grade`- ...`-O0`, 33 ders bulduğumuz sekiz blok listesi, o zaman `explore(find=0x400076)`4. depolayan blok için 0.47 saniye içinde üç hali olan bir durum buldu. Argument ve örnekler 91, 93, 94, 97 ve 101'de minimum 90 ve maksimum 109'u buldu.

- Evet .`percent_change`, iki sınırlı devlet .`before[31:0] != 0`ve `before[31:0] == 0`- Zıfır dışı durumu sormak .`eax == 50`Verdi .`before = 21055744, after = 332315210`- ...`-O0`- Evet .`40566804, 834296877`- ...`-O1`ve`-24484867, 993934694`- ...`-O2`-16 ' den 16 ' a kadar sınırlı arama cevapları`before = -10, after = -15`Z3, Ders 34 listesi ile ilgili aynı soruyu sordu, cevaplar `before = -25092599, after = 305712160`. Test kümesi, IR değerlendirici ile beş çiftin hepsini değerlendirir ve her biri 50 verir. Bir model bir giriş varlığını kanıtlar. Giriş değil ve üç doğru araç üç farklı bir giriş verdi.

`capture_z3.py`modellerini`-O1`listesi `grade`32 bit bit vektörlerinde çarpma, iki şifte ve tablo ile ve daha dört soru sorar.`switch`32 bit skor için: 0.068 saniye içinde ayarlanmamış, böylece oluşturulan işlev tüm 2^32 girişleri için kaynağa eşit.`score / 10`Bu fark 0 ile 109 arasında sınırlıdır: Satılamıyor, bu yüzden ders 36'nın şebekesi onu göremiyor.`percent_change`- Evet .`before == 3`-133 ile oturdu.`after = 1073741823`36 dersindeki aşırı hücre, sınırlı aramalar, alanında ikinci bir tanık buldu.`after = -1`, nerede `(-1 - 3) * 100 / 3`-133'e kadar kesilmiş, hiç bir aşırı akış yok.

## Yapın

Demo çalıştır:

```bash
cd phases/13-tools-and-protocols/44-binary-diffing-and-symbolic-execution
python3 code/main.py
```

Çıktı bir JSON raporu. `diffs`nesne dosyası ve yüklenmiş görüntü eşleşmelerini tutmaktadır `-O0``-O1`ve `-O1``-O2`, artı `-O0`BinDiff sırasındaki çift.`toolComparison`her fonksiyon için radiff2 hüküm ve oran ve Ghidra korelatörlerinin yanında ders aşamasını koyar. `patch`- Bu ...`grade_case10_wrong.o`- Davayı.`radare2Ratios`Radiff2 sayısını yeniden üretir.`gradeRelocationCase`düzenini ve iki yer değiştirmesini gösterir ve `symbolic`Yolları, sınıfları ve patlama sayısını tutar.

Oku `code/main.py`Bu sırada:

1. `ElfObject`Bölümleri, sembolleri ve `.rela.*`Kayıtlar.`parse_listing()`okur `llvm-objdump`Mesaj.
2. `jump_table_targets()`ve `build_cfg()`ulaşılabilir bloklar ve tiplenmiş kenarlar. `FunctionRecord.signatures()`Beş imza oluşturur.
3. `load_image()`Bölümlerin belirlenmesi ve uygulanması `S + A - P`ve `S + A`- Evet .
4. `match_functions()`- O zaman sen de oyunculuğun teklifi yap.`levenshtein()`ve `radare2_ratio()`Benzerliği gösterir.
5. `parse_radiff2()`ve `parse_ghidra_vt()`karşılaştırma için kaydedilen araç çıkışlarını okuyun.
6. `evaluate()`- Evet .`substitute()`ve`BoundedSearch`Bu ifade semantik ve çözücüdür.
7. `execute()`Yolları sayıyor.`solve_returns()`her yaprak için bir giriş kümesi istiyor.
8. `grade_ir_o0()`- Evet .`grade_ir_o1()`- Evet .`percent_change_ir_o1()`ve`total_ir_o1()`- El kaldırmaları ve talimat adresleri.

Üç emir daha var.`python3 code/main.py diff A.o B.o`listeleri yanlarında bulunan iki nesneyi ayırt eder ve `--image`yüklenmiş görüntülerini farklılaştırır. `python3 code/main.py paths grade O0`Bir fonksiyonun yollarını ve sınıflarını yazdırır `paths total 8`Bir sınırlı döngü.`python3 code/main.py figure-data`İnteraktif figürün yerleştirdiği verileri yazdırır.

## Kullan

Kendi programınızın iki yapısında bu sırada farklı ve uygulayıcı kullanın.

1. Her nesne için bir listeyi yakalayın ve bırakın `function_record()`Bayeye karşı doğrulayın.
2. Önce nesne dosyalarını ayırt edin. Her maçın aşamasını okuyun ve ikinci bir imzanın kabul edilene kadar akış grafik şekli eşleşmelerini öneriler olarak değerlendirin.
3. Yüklü görüntüleri ayırt edin. Tam baytlardan maskeli baytlara düşen bir fonksiyon verilerine göre hareket eder ve yer değiştirme aritmetikası hareket eden bölüme isim verir.
4. Her maç için referans tabloları karşılaştırın. Aynı kod farklı sabitlerle değiştirilmiş bir fonksiyon.
5. Adlar aşamasını çalıştırın ve karşılaştırın. Yapılama sembollere sahip olduğunda, BinDiff sırasını çalıştırın ve yapısal sonucu sadece aşamaları sınıflandırmak için kullanın.
6. Bir araç bir oran rapor ettiğinde, kendi ile karşılaştırmadan önce hangi baytların içine girdiğini öğrenin.
7. İsteğiniz girişleri olan fonksiyon için, IR'yi her ifade için bir adresle yazın, çalıştır `execute()`Her simgesel döngü sayıcısı üzerinde bir çizgi ile, ve her yaprak çöz.
8. Aynı soruyu angr veya z3 üzerinden kullanın ve sınıf sınırlarını karşılaştırın.

## İnteraktif Laboratuvar

### Laboratuvar A: El ile beş imza

- Evet .`code`Dizin, açık Python:

```bash
cd phases/13-tools-and-protocols/44-binary-diffing-and-symbolic-execution/code
python3 -q
```

Çık:

```python
from main import *
grade = {r.name: r for r in fixture_records("ledger-x86_64-O1.o")}["grade"]
grade.raw.hex(" ")
grade.masked_bytes() == grade.raw
grade.mnemonics
grade.cfg_shape()
grade.signatures()
```

Yazdırmadan önce fonksiyonun 0x26 oranında dört bayt yazın `grade.raw`- O zaman aynı çizgileri kullan .`grade`-`ledger-x86_64-O0.o`Ve sayın `table` kenarları`grade.edges`- Evet .

### Laboratuvar B: Yer değiştirme bayt hashini kırmaya bak

```python
case = grade_relocation_case()
case["layout"]
case["displacements"], case["rodataShift"]
case["objectBytesEqual"], case["imageBytesEqual"], case["maskedImageBytesEqual"]
```

Hesaplama`0x100110 + 0x14 - 0x10003a`okumadan önce kağıt üzerinde .`displacements`Sonra tabanı değiştir .`load_image()`0x400000'e kadar ve iki operandın birlikte hareket ettiği için yer değiştirmelerin değişmediğini onaylayın.

### C laboratuvarı: yanlış eşleşme ve onu önleyen sırayla

```python
report = diff_fixtures("ledger-x86_64-O1.o", "ledger-x86_64-O2.o")
[(m.left.name, m.right.name, m.stage) for m in report.matches]
report = diff_fixtures("ledger-x86_64-O0.o", "ledger-x86_64-O1.o")
[(m.left.name, m.right.name, m.stage, m.names_agree) for m in report.matches]
report = diff_fixtures("ledger-x86_64-O0.o", "ledger-x86_64-O1.o", stages=BINDIFF_ORDER)
[(m.left.name, m.right.name, m.stage) for m in report.matches]
```

Yazdır `records(...)["percent_change"].cfg_shape()`için`-O0`ve `grade_label`için`-O1`Sonra sıralamayı değiştiren 6. aşamayı yazın. Böylece sıralama artık önemli değil ve onları ayırıp ayırmayacağını görelim.

### Laboratuvar D: Yollar ve sınırlar

```python
report = grade_paths("O0")
[(p["trace"], p["condition"], p["returns"]) for p in report["paths"]]
report["returns"]["4"]
total_paths(2), total_paths(4), total_paths(8)
total_paths(64, step_limit=24)["truncated"]
```

Değişiklik`SCORE_DOMAIN`- ...`range(0, 110)`Ve tekrar çalıştır .`grade_paths("O0")`0 için sınıf 60 giriş olarak küçülür ve iki çapraz kontrol hala geçerlidir.`range(90, 100)`Ve hangi yaprakların hiç modeli olmadığını okuyun.

## Pratik Laboratuvar

BinDiff'i oyunlara koy ve üç sonuçla karşılaştır.

Gereksinimler:

- BinExport uzantısını Ghidra 12.1.4'e yükleyin ve üç defter programını ihraç edin. BinExport yayın etiketini ve Ghidra sürümünü kaydetin.
- BinDiff 8 komut satırını yükle ve `-O0`karşılaştırıldığında`-O1`ve `-O1`karşılaştırıldığında`-O2`Sadece benzerlik seçeneği ile sonuçlar.
- Fonksiyon başına benzerliği ve güvenini yakalayın `code/fixtures/`Tam emirleri ile.`build_fixtures.sh`- Evet .
- Uzaklaştırma`tool_comparison()`bir `bindiff`BinDiff'in eşleşen seti ile karşılaştırılan bir test ekle.`-O1`karşılaştırıldığında`-O2`Dersin maskeli baytları ile.
- BinDiff algoritmasının adı verilen bir paragraf yazın .`grade`ve yüklü görüntülerin karşılaştırıldığını.

Başarılı olmak, demo'nun her fonksiyona dört araç sütunu yazdırdığını ve yeni testin bir eşitlik veya bir açıklanmış fark belirttiğini gösterir.

## Ders Artifakti

Bu ders de içerir .`outputs/skill-binary-diffing-and-symbolic-execution.md`. İki yapılandırmayı ve bir fonksiyonu, her maç için belirtilen bir kalitede, her eşleşmeyen veya değiştirilmiş fonksiyonun açıklaması ve her yaprak için çözülmüş bir giriş seti ve onu üreten sınır ile bir yol raporu için kullanın.

## Kontrol et

Demo ve süiti çalıştır:

```bash
cd phases/13-tools-and-protocols/44-binary-diffing-and-symbolic-execution
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

Verifikasyon, şunu kanıtlamalıdır:

- Simgolu tablolar, 46 , 118, 66, 39 boyutları olan dört işlevi gösterir.`-O0`16, 46, 31, 63 ' te`-O1`, ve 16, 46, 140, 63'te`-O2`
- `.rela.text``-O1`nesneyi şöyle tanımlayın:`llvm-objdump -r`basıyor.
- listelenen her talimat simgesel boyutunda ve baytları nesneyle eşit ve değiştirilmiş bayt yükseltilmektedir
- `total`- ...`-O1`Beş blok, beş kenar ve bir kendi döngüsü vardır ve şekil dizilisi belirleyici
- `grade`- ...`-O0`5 atlama masası hedefini kurtarır .`.rela.rodata`Sekiz blok ve on bir kenarı vardır.
- Masası olmadan,`grade`- ...`-O0`4 blokta 82 byte'ye ulaşır.
- `percent_change`- ...`-O0`ve `grade_label`- ...`-O1`Aynı dört blok şekli
- Tam ve maskeli hashler yer değiştirilebilir bir nesnenin her işlevi üzerinde anlaşılır
- `-O1`ve `-O2`Üç fonksiyon için bayt paylaşıyor, `-O0`ve `-O1`payı yok
- Referanslı tablo `grade`okur `1, 2, 3, 4, 4`ve adayın okuyuşları `1, 2, 3, 4, 3`
- yükleme alanı `.text`- Evet .`.rodata`ve`.rodata.str1.1`Ghidra onları her iki programda da yerleştirdi.
- `R_X86_64_PC32`yazıyor:`S + A - P`ve 122 yer değiştirme sağlar.`R_X86_64_64``.data`İletiş dizini adresleri
- `grade`Byte nesnelerde uyumlu ve görüntülerde 0x70 bir değişim ile farklıdır ve maskeleme eşleşimi geri getirir
- `-O1`karşılaştırıldığında`-O2`Tam baytlarda üç işlevi eşleştiriyor ve `total`Adıyla; görüntü görünümü eşleşir `grade`1 - 1/46 benzerliği olan maskeli baytlar
- Ders sırası bir yanlış akış grafik şekli eşleşmesi üretir .`-O0`karşılaştırıldığında`-O1`BinDiff siparişleri de dörtüyle aynı .
- Yapıştırma adayı byte ile eşleşir ve referans verilerde başarısız olur
- Levenshtein oranı, 0.130435, 0.106061, 0.269841, 0.171429, 0.426829'u 82 byte üzerinde yeniden üretir ve yer değiştirilen bayt oranları, yer değiştirme uygulanması ile radiff2 baskıları
- radiff2'nin MATCH seti, exact-byte aşamasına eşit, ve Ghidra'nın byte ve talimat korelörleri görüntü görünümünün tam ve maskeli setlerine eşit
- Ghidra sadece isimleri buldu .`-O0`karşılaştırıldığında`-O1`, her maçta 1.0 benzerlik ve 10.0 güven ile
- El kaldırılmış IR, listelenen her talimatı bir kez kapsar ve değerlendirici C semantikasını takip eder.
- `grade`5 yollu bir yol var.`-O0`- Evet. - Evet.`-O1`Aynı sınıflar
- sınırlı sınıflar 60 ila 69, 70 ila 79, 80 ila 89, 90 ila 109'da her üç seviyede ve z3'lerde, 5 ve 2 sınırlı durumlarla ve 90 ila 109'da ulaşma blok aralığı ile
- İki çapraz kontrolün sınırda, angr'de veya z3'te bir modeli yoktur ve z3 listenin kaynağı anahtarına eşit olduğunu kanıtlar
- Her zaman`percent_change`Sınırlı arama, angr ve z3'den model, istenen değere değerlendirir
- `total`3, 5, 9 yolları verir ve üçgen toplamları angr ile aynıdır ve adım kapı 22'yi geri verir ve 2'yi keser.
- Resim verileri, her iki görüntü, yer değiştirme numaraları ve giriş setleri ile birlikte beş yaprak içerir ve sitedeki modül aynı verileri içerir.

## Üretim Başarısızlık Modları

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

## İşlem Kuralı

Bir imza ile aynı anda tek tek değerleri kabul edin ve her iki tarafta da benzersiz değerler kabul edin ve sahneyi ve kalitesini maçla tutun. Nesneleri ve yüklü görüntüleri ayırt edin ve iki arasında aşama değiştiren her fonksiyonu açıklayın. Her dalda bir çözücü ile yolları sayın, her sembolik döngüyü bağlayın ve giriş seti sınırlarıyla bildirin.

## Daha Fazla Okumak

- [BinDiff concepts: matching strategy, algorithms, and quality ratings](https://github.com/google/bindiff/blob/main/docs/concepts.md)
- [BinDiff repository and the v8 open-source release](https://github.com/google/bindiff/releases/tag/v8)
- [BinExport README: IDA Pro, Binary Ninja, and Ghidra exporters](https://github.com/google/binexport)
- [Flake, "Structural Comparison of Executable Objects" (DIMVA 2004) and Dullien, Rolles, "Graph-based comparison of Executable Objects" (SSTIC 2005), shipped in the BinDiff repository](https://github.com/google/bindiff/tree/main/docs/papers)
- [Diaphora README and heuristics source](https://github.com/joxeankoret/diaphora)
- [Ghidra Version Tracking correlators help page](https://github.com/NationalSecurityAgency/ghidra/blob/master/Ghidra/Features/VersionTracking/src/main/help/help/topics/VersionTrackingPlugin/VT_Correlators.html)
- [Ghidra Auto Version Tracking help page](https://www.ghidradocs.com/11.0_PUBLIC/help/VersionTracking/help/topics/VersionTrackingPlugin/VT_AutoVT.html)
- [Ghidra 12.1.4 release](https://github.com/NationalSecurityAgency/ghidra/releases)
- [Binary Ninja 6.0 "Krypton" release notes](https://binary.ninja/2026/09/03/binary-ninja-6.0-krypton.html)ve [Binary Similarity user guide](https://docs.binary.ninja/guide/similarity.html)
- [radare2 book: radiff2](https://book.rada.re/tools/radiff2/intro.html)ve [radare2 6.2.4 source](https://github.com/radareorg/radare2/releases/tag/6.2.4)
- [King, "Symbolic execution and program testing" (CACM 1976)](https://dl.acm.org/doi/10.1145/360248.360252)
- [Baldoni et al., "A Survey of Symbolic Execution Techniques" (ACM Computing Surveys 2018)](https://dl.acm.org/doi/10.1145/3182657)
- [Cadar, Sen, "Symbolic execution for software testing: three decades later" (CACM 2013)](https://dl.acm.org/doi/10.1145/2408776.2408795)
- [Cadar, Dunbar, Engler, "KLEE: Unassisted and Automatic Generation of High-Coverage Tests for Complex Systems Programs" (OSDI 2008)](https://www.usenix.org/legacy/events/osdi08/tech/full_papers/cadar/cadar.pdf)ve [KLEE 3.2 release](https://github.com/klee/klee/releases/tag/v3.2)
- [Shoshitaishvili et al., "SoK: (State of) The Art of War: Offensive Techniques in Binary Analysis" (IEEE S&P 2016)](https://oaklandsok.github.io/papers/shoshitaishvili2016.pdf)
- [angr documentation: loading, simulation managers, and the simulation pipeline](https://docs.angr.io/en/latest/core-concepts/toplevel.html)ve [angr 10.0.3 on PyPI](https://pypi.org/project/angr/)
- [Z3 5.1.0 release](https://github.com/Z3Prover/z3/releases/tag/z3-5.1.0)
- [Triton dynamic binary analysis library](https://github.com/JonathanSalwan/Triton)
- [System V Application Binary Interface, AMD64 Architecture Processor Supplement, Table 4.9: Relocation Types](https://gitlab.com/x86-psABIs/x86-64-ABI)
