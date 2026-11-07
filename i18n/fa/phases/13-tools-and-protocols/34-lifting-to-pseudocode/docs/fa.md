# رفع به کد پشو: IR، SSA و ساخت از ابتدا

> محصول Decompiler یک ادعای با یک مشتق پنهان است. وقتی که شما مشتق خود را بسازید، هر خط از آن محصول تبدیل به چیزی می شود که می توانید بررسی کنید.

**Type:** Build
**Languages:** Python, C
**Prerequisites:** Phase 13 · 32 (ELF from scratch), Phase 13 · 33 (disassembly and control flow)
**Time:** ~150 minutes

## اهداف یادگیری

- لیفت دستورالعمل های x86-64 را به یک IR کوچک با پرچم های صریح، بار، فروشگاه ها، شاخه ها، تماس ها و بازپرداخت ها رمزگذاری کرد.
- به سمت فاتر های فریم تبدیل کنید به متغیرهای محلی و به پارامترهای سیستم V ثبت کنید.
- SSA حداقل با عملکردهای phi در مرزهای تسلط بسازید، سپس جایگزین، ساده سازی و حذف کد مرده کنید.
- بهبود`if`،`if`با`else`،`while`و`do while`از نمودار جریان کنترل با پس از کنترل کننده ها و لبه های عقب.
- ضرب را با توضیح بده`0x66666667`به عنوان تقسیم به 10 و اثبات هویت.
- کد جیدرا را برای یک تابع خط به خط بخوانید و دستورالعمل یا حدس را که پشت هر تفاوت است نام دهید.

## مشکل

تو از " ريا " از رمز مخففي "`percent_change`در`ledger-x86_64-O1.o`. گيدرا 12.1.4 در حدود 10 ثانیه پاسخ داد:

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

منبع چهار خط طول داره.`int`ارزش ها و بازده ها 1 `int`.Ghidra یک مقدار بازگشت 16 بایت و یک پارامتر سوم را گزارش می دهد که منبع ندارد. شصت بایت کد ماشین درست است و Ghidra هر یک از آنها را به درستی رمزگذاری کرده است. امضا حدس است و هیچ چیز در متن آن را به عنوان یک نشان می دهد.

rea این متن را به عنوان `procedure_pseudo_code`در این نسخه، یک محدودیت به ثبت متصل می شود. محدودیت می گوید: "Pseudocode Ghidra decompiler output است، نه منبع اصلی و نه متن معادل output Hopper".`compare_functions`ابزار فرقی pseudocode به عنوان خطوط متن دارد. ثبت ابعاد خود می گوید "اجمع متن ارائه دهنده غیر شفاف است" و عادی سازی نقل مکان در دسترس نیست. rea نمی تواند به داخل متن نگاه کند و همچنین یک عامل که فقط آن را می خواند نمی تواند.

گزارش DX-Ball از rea نشان دهنده هزینه است.`FUN_00406400(void)`تماس گرفتن`__ftol()`. در نظرسنجی دستورالعمل یک ورودی ستک و یک عبارت x87 نشان داده شد. عامل فقط به دلیل اختلاف نظر دو دیدگاه ادامه داد.

این درس مشتق را که تجزیه کننده پنهان می کند، می سازد. شما از 56 دستورالعمل که درس 33 رمزگذاری کرد شروع می کنید، آنها را به یک IR بالا می بردید، به SSA نام می برد، ساده سازی و ساختار می کنید. سپس کد خود را در کنار Ghidra قرار می دهید.`percent_change`،`grade`و`total`تست ها محصول شما را با clang جمع آوری می کنند و آن را با منبع در یک شبکه ورودی اجرا می کنند.

```figure
lifting-stages-percent-change
```

## "بند و کلام"

`code/fixtures/ledger.c`این بخش از دوره، یک برنامه مشترک است:

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

`grade_label`در این قسمت، این دو تابع در دوگانه و در شی وجود دارد و در نمایش نمایش نیز آن را بالا می برد.

این شی با`clang -target x86_64-unknown-linux-gnu -O1 -fno-asynchronous-unwind-tables -c ledger.c`.`-O0`این موضوع در کنار آن برای بخش سلاکت های ستاک قرار دارد. `code/fixtures/README.md`این دستور به عنوان یک دستور برای هر یک از این دو دستورات، SHA-256 هر شی و دستور REA که وسایل Ghidra را تولید کرده است، ذکر می شود. ساخت باایت به باایت در زنجیره ابزار متصل شده بازتولید می شود.

. . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .`llvm-objdump -d -r`متن آن برای هر دو موضوع به عنوان`ledger-x86_64-O1.objdump.txt`و`ledger-x86_64-O0.objdump.txt`یک آزمایش این فایل ها را تجزیه و تحلیل می کند و هر دستورالعمل را با آدرس، با بیت و با متن AT&T مقایسه می کند. این شامل تمام 56 دستورالعمل در `-O1`و هر گونه آموزش در`-O0`. چه وقت`llvm-objdump`در مسیر بسته نصب شده، آزمون دوم آن را زنده اجرا می کند و دوباره مقایسه می کند.

اينه`percent_change`همونطور که کتاب مقدس ميگه:

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

## مرحله اول: بایت ها به دستورالعمل ها

درس 33 کد کدر را ساخت. این درس یک کپی کوچک را نگه می دارد تا کد به طور خودکار اجرا شود. کپی دقیقاً کدهای کدر را پوشش می دهد که برای آن پخش می شود.`ledger.c`در`-O0`و`-O1`. براي هر چيزي که بيشتر به وجود مياد`DecodeError`با آدرس و بايت کد آپکود

کد کدگر به ترتیب بایت های SDM اینتل، جلد 2 بخش 2.1 را دنبال می کند. پیشگویی ها اول، سپس کد opcode، سپس ModR/M، سپس SIB، سپس جابجایی، سپس فوری هستند. بخش 2.2.1 پیشگویی REX را اضافه می کند، یک بایت از `0x40`به`0x4F`REX.W یک اپراند ۶۴ بیت را انتخاب می کند، REX.R `reg`در این زمینه، REX.X شاخص SIB را گسترش می دهد و REX.B را گسترش می دهد `r/m`بخش 2.1.3 سه میدان ModR/M را تعریف می کند.`mod`،`reg`و`r/m`، و سه میدان SIB`scale`،`index`و`base`. در حالت 64 بت`mod = 00`با`r/m = 101`یک تغییر 32 بیت نسبت به دستورالعمل بعدی (قسم 2.2.1.6) است.

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

.`leaq (%rip), %rcx`در`grade+0x23`هفت بايت با صفر جابجا شدن است.`R_X86_64_PC32.rodata+0x10`در تعویض`0x36`، میدان حرکت را بالا می برد و در مرحله سوم حلش می کند

## مرحله دوم: بلوک ها و حاکمان

یک بلوک پایه یک ردیف اظهارات با یک ورودی در بالا و یک خروج در پایین است. این تعریف از آلن (1970) است و هر کاغذ تجزیه کننده آن را دوباره استفاده می کند. `build_blocks()`به عنوان یک بلاک، هر یک از کلک ها به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به عنوان یک کلک به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور کامل به طور قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل قابل اعتماد است.`jmp`. یکتا داره`ret`هیچ کدام از آنها وجود ندارد و یک بلوک که با یک دستورالعمل معمولی به پایان می رسد به رهبر بعدی می رسد.

`total`در`-O1`پنج بلوک دارد:

| Block | Instructions | Successors |
|---|---|---|
| `0x40` | `testl %esi, %esi` then `jle 0x5c` | `0x5c`, `0x44` |
| `0x44` | `movl %esi, %ecx`, `xorl %edx, %edx`, `xorl %eax, %eax`, `nopw` | `0x50` |
| `0x50` | `addl (%rdi,%rdx,4), %eax`, `incq %rdx`, `cmpq %rdx, %rcx`, `jne 0x50` | `0x50`, `0x5b` |
| `0x5b` | `retq` | exit |
| `0x5c` | `xorl %eax, %eax` then `retq` | exit |

`immediate_dominators()`الگوریتم تکراری کوپر، هاروی و کندی (2001) است. این بلوک ها را در بعد از ترتیب معکوس حرکت می دهد و حاکمان پیشروهای پردازش شده را تا زمانی که هیچ چیز تغییر نمی کند، متقاطع می کند. در نمودار های کوچک این اندازه، آن را در دو گذر انجام می دهد. برای`total`میگه`0x44`تسلط داره`0x50`. همچنین میگه که`0x50`خود را از طریق لبه عقب تسلط می دهد، که این همانطوری است که مرحله 8 حلقه را پیدا می کند.

همون تابع دو برابر بیشتر اجرا میشه`dominance_frontiers()`در مرحله 5 از آن برای SSA استفاده می کند.`post_dominators()`این را روی نمودار معکوس با یک گره خروج مجازی اجرا می کند تا در مرحله 8 ترکیب ها را پیدا کند.

## مرحله سوم: بالا بردن به یک IR کوچک

IR شش نوع اعلامیه دارد:`Assign`،`Store`،`Branch`،`Jump`،`IndirectJump`و`Return`.عبارات ثابت، متغیر، عملیات دوگانه با عرض، کاست (`sext`،`zext`،`trunc`پنج شکل دیگر وجود دارد که بعداً ساده تر می شود:`Flags`،`Cond`،`Concat`،`SignFill`و`Symbolic`. .

ثبت نامها متغیرهای نامگذاری شده توسط خانواده 64 بت هستند.`%eax`.`Assign`به`rax`با عرض 32 و خواندن`%eax`در مورد "`rax`در حالت 64 بت، نوشتن 32 بیت صفر را به داخل کل ثبت می کند، بنابراین یک خواندن 64 بیت بعدی با همان مقدار یک `zext`. بلند کننده تعریف رسیدن را نمی داند، بنابراین عرض خواندن را ثبت می کند و اجازه می دهد SSA در مرحله 5 کاست را وارد کند.

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

### پرچم ها متغیر هستند

هر دستور که کد شرط را تنظیم می کند یک متغیر به نام  را می نویسد`flags`، و هر پرش مشروط آن را می خواند.`cmp a, b`نوشته شده`cmp(a, b)`، یک`test a, b`نوشته شده`test(a, b)`و`sub`همش رو مي نويسه`cmp`شکل به عنوان یک اثر جانبی.`jcc`می شه`Branch(Cond(cc, flags), target, fallthrough)`کد شرط، وقتی که این دو در مرحله 6 ملاقات کنند، عامل مقایسه را تعیین می کند:

| Code | After `cmp(a, b)` | After `test(a, a)` |
|---|---|---|
| `je` | `a == b` | `a == 0` |
| `jne` | `a != b` | `a != 0` |
| `jl`, `jge`, `jle`, `jg` | signed `<`, `>=`, `<=`, `>` | signed compare with 0 |
| `jb`, `jae`, `jbe`, `ja` | unsigned `<`, `>=`, `<=`, `>` | unsigned compare with 0 |

`xorl %eax, %eax`به سمت راست بالا می رود`eax = 0`پرچم هاي که مي بندازن نوشته مي شه:`add`پرچم هاي سبکي که هيچ پرش در دستگاهي نميخواد`xor`زبان مهمه`grade`، جایی که گیدرا از دستش داد

### کلتید و آی دی وی

`cltd`پر کردن`%edx`با علامت "بيت"`%eax`.`idivl %edi`دوتا 64 بت را تقسیم می کند `%edx:%eax`از طرف`%edi`، مقدار را وارد می کند`%eax`، و بقيه در`%edx`. بلند کننده جفت رو به نام موقت مي نويسه`dividend`اول، بقيه جمله بعد از اون نوشته هاي قديمي`%eax`در SSA نصف بالا به عنوان`signfill`ساده کننده جفت را به نیمه پایین کاهش می دهد و تقسیم به 32 بیت می شود`/`. .

### سوراخ های جمع در -O0

در`-O0`کلنگ همه ی محلی ها رو در قاب نگه می داره`percent_change`فروشگاه ها`%edi`به`-0x8(%rbp)`و`%esi`به`-0xc(%rbp)`بعدش براي هر استفاده ايشون رو ميخواد`pushq %rbp`بعدش`movq %rsp, %rbp`بعد از اون مقدمه، يه اپراند خاطره با پايه`%rbp`, هیچ شاخصی نیست و یک تغییر مکان تبدیل به یک متغیر به نام می شود `local_<hex>`.`-0x8(%rbp)`.`local_8`،`-0xc(%rbp)`.`local_c`. .

دستورالعمل های فریم خود هیچ IR تولید نمی کنند و نه هم`subq`و`addq`در`%rsp`هر اپراند حافظه ديگه اي به عنوان`Load`یا`Store`. .

### نقل مکان

.`leaq (%rip), %rcx`در`grade`.در پرونده نقل مکان صفر هست`0x36`میگه`R_X86_64_PC32.rodata+0x10`. لینکر ميخواد بنويسه`S + A - P`به میدان.`S`آدرس این`.rodata`،`A`اضافه شده است`0x10`و`P`این آدرس میدان است. سپس دستورالعمل محاسبه می کند.`next_ip + disp = (P + 4) + S + A - P = S + A + 4`. .

بلند کننده بدون لینکر همان حساب را اعمال می کند. هدف تعویض اضافه شده به علاوه فاصله از میدان تا پایان دستورالعمل است.`0x10 + 4 = 0x14`. گزارش هاي IR ميگن`rcx = &.rodata+0x14`و اون ميز پنج تا رو نگه داره`int`ارزش ها`1, 2, 3, 4, 4`. .

## مرحله چهارم: پارامترهای کنوانسیون دعوت

سیستم V AMD64 psABI، بخش 3.2.3، استدلال های کلاس INTEGER را در `%rdi`،`%rsi`،`%rdx`،`%rcx`،`%r8`،`%r9`، به این ترتیب. نتیجه در`%rax`و بعدش`%rdx`بخش 3.2.1 میگه`%rbp`،`%rbx`و`%r12`از طریق`%r15`به طرف دعوت کننده تعلق دارد و باید حفظ شود.

`collect_params()`یک قانون اعمال می شود: یک ثبت استدلال که قبل از هر نوشتن در تابع خوانده می شود یک پارامتر است. بالاترین چنین ثبت حساب را تنظیم می کند، بنابراین یک تابع که می خواند `%rsi`و هرگز`%rdi`هنوز دو پارامتر داره`percent_change`میخواد`%edi`و`%esi`، پس اون داره`param_1`و`param_2`.`grade`فقط میخواد`%edi`.`total`میخواد`%rdi`به عنوان یک پایه 64 بایت و `%esi`به عنوان تعداد 32 بت

حالا گيدرا`param_3`يه توضیح داره`idivl`نوشته شده`%rdx`، و psABI اجازه می دهد تا یک 128-بیت INTEGER بازگشت در `%rax:%rdx`در یک فایل شی وجود ندارد که به آن ها بگوید مقدار بازگشت چقدر گسترده است.

در مسیر صفر،`xorl %eax, %eax`فقط می نویسد`%eax`. گيدرا مي خواد`%rdx`در آنجا به عنوان یک ورودی از تابع، و این ورودی به `param_3`دستورات درست است، امضا حدس زدن درباره قسمت هایی است که دستورات باز می گذارند

## مرحله 5: SSA، یک نام در هر تعریف

بعد از بلند شدن`grade`نوشته شده`rax`چهار بار و`rcx`سه بار سیترون، فرانت، روزن، وگمن و زادک (1991) شکل واحد اختصاصی ثابت را تعریف می کنند. هر متغیر دقیقا یک بار اختصاص داده می شود و تابع های phi در نقاط پیوستن از بین تعاریف که به آنها می رسد انتخاب می کنند. ساخت آنها دو مرحله ای دارد. بخش 5.1 تابع های phi را با استفاده از مرزهای تسلط قرار می دهد و بخش 5.2 هر تعریف و استفاده را با پیاده روی روی روی درخت تسلط تغییر نام می دهد.`insert_phis()`و`rename()`از اين دو قسمت پيروى کنيد

لیست SSA از`total`نشان می دهد که ساخت و ساز چه چیزی را تولید می کند:

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

نسخه 0 از یک ثبت نام ارزش آن در زمان ورود است. `esi_0`و`rdi_0`. پارامترها هستن . سرگوشن حلقه`0x50`دو پیشگامش را دارد، بلوک تنظیم و خودش، بنابراین هر متغیر با یک تعریف در حلقه یک phi را در آنجا می گیرد.`flags_2`وقتي به اينجا رسيدم فاي مرده`flags`در هر دو پیشگام تعریف دارد و مرحله 6 آن را حذف می کند چون هیچ چیزی آن را نمی خواند.

مرحله تغییر نام هم عرض ها را اصلاح می کند.`edx_1 = 0`یک تعریف 32 بیت است و`rdx_3 = rdx_2 + 1`64 بایت است، پس فیه`rdx_2`در طول طول طول طولش بیشتر میشه`%rcx`در`cmpq %rdx, %rcx`64 بيت است در حالی که تعریف آن`ecx_1 = esi_0`32 بت است، بنابراین استفاده می شود`zext64(ecx_1)`هر بازیگری که در کد آخر قرار گرفته از این قانون است

قسمت "لایت" راه خروج از SSA است.`coalesce_webs()`این کار برای اشکال که یک کامپایلر برای این عملکردها منتشر می کند کار می کند. این می تواند بر روی نسخه های موازی شکسته شود، جایی که دو phis در یک با ارزش های مبادله متحد می شوند و درس این مورد را اداره نمی کند. بخش 7 سایترون خروج عمومی را از طریق رنگگذاری توصیف می کند.

## مرحله 6: ساده سازی، جایگزینی، حذف کد مرده

`optimize()`سه بار عبور می کند تا هیچ چیز تغییر نکند. ساده کننده عبارت ها را با این قوانین دوباره می نویسد:

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

تعویض پیش رو یک تعریف را به تنها استفاده اش منتقل می کند. قاعده سه شرط دارد. استفاده در همان بلوک است. استفاده یک بیانیه است و نه یک phi. هنگامی که تعریف حاوی بار است، هیچ ذخیره یا تماس بین تعریف و استفاده قرار نمی گیرد. یک ثابت یا یک آدرس بخش در هر جای جز به یک phi جایگزین می شود، زیرا یک استدلال phi باید یک متغیر برای وب باقی بماند.

حذف کد مرده یک وظیفه یا یک phi را حذف می کند که مقصد آن به کار نمی رود و بیان آن به هیچ عنوان تماس ندارد.`percent_change`بقيه`edx_2 = dividend_1 % edi_0`مرده، پس اول مياد.`edx_1 = signfill(eax_1)`با یک استفاده، بنابراین جایگزین می شود`dividend_1 = edx_1:eax_1`. .

قانون جفت بعدش کاهش می یابد`signfill(eax_1):eax_1`به`eax_1`، و تقسیم شدنش تبدیل میشه`eax_1 / edi_0`حالا`eax_1 = esi_1 * 100`یک کاربرد دارد و`esi_1 = esi_0 - edi_0`یک استفاده دارد و هر دو به بیان بازگشت حرکت می کنند.`flags`تعریف ها به شرایط شاخه جایگزین شده و ناپدید می شوند. چهار دستورالعمل به یک خط تبدیل می شوند:

```c
return (param_2 - param_1) * 100 / param_1;
```

### ضرب 0x66666667

`grade`نمره رو به 10 ميزنه و کلنگ نمره اي ميده`idiv`. از اون خارج ميشه`imulq $0x66666667, %rax, %rax`، پس`sarq $0x22, %rcx`پس از اينکه يه کپي داشته باشين`shrq $0x3f, %rax`در کتاب "قسمت با اعداد کامل غیر متغیر با استفاده از ضرب" (PLDI 1994) ، روش را توصیف می کند.`m`و یک تغییر`s`. براي هر`x`در محدوده`(x * m) >> s`باید برابر باشه`x / d`. .

اینجای`m = 0x66666667 = 1717986919`و`s = 34`. حسابي ها درست ميکنن:`2^34 / 10 = 1717986918.4`، و سقف هم هست`1717986919`اشتباه اينه که`m * 10 - 2^34 = 6`. تا هویت هر 32 بت رو نگه داره`x`، اوقات خطا`2^31`بايد پايين باشه`2^34`، پس خطای باید حداکثر 8 باشد. 6 حداکثر 8 است.`(x * m) >> 34`برابر است`floor(x / 10)`برای هر منفی نیست`int`. .

برای منفی`x`کف از صفر دور می شود و C به سمت صفر دور می شود.`shrq $0x3f`از قطعه علامت محصول خارج می شود که دقیقاً زمانی که 1 است`x`منفی است. اضافه کردن آن گرد گردی را اصلاح می کند.

`magic_divisor(m, s)`يه جفت کانديد را آزمايش ميکنه.`d`به عنوان مقدار تمام عدد از `2^s`از طرف`m`و تلاش می کنه`d`و`d + 1`. اين قبول ميکنه`d`فقط وقتی`m`برابر با سقف `2^s / d`و خطای محدود باقی می ماند. همان ضرب با `s = 33`می دهد`d = 5`، که درست است، چون`0x66666667`همش سقف`2^33 / 5`. آزموني که هویت رو با`c_div(x, 10)`در ۲۴۰۶ مقدار از جمله هر دو پای`int`فاصله

با اعتراف به اين`grade`میخواد`ecx_1 = param_1 / 10 - 6;`. با قطع کردن تشخیص ، دیمو شکل خام را چاپ می کند:

```c
rax_1 = (long)param_1 * 0x66666667;
ecx_1 = (int)(rax_1 >> 34) + (int)((unsigned long)rax_1 >> 63) - 6;
```

هر دو مرتب و هر دو محاسبه همان ارزش ها. کلید را نگه دارید. وقتی کد را از یک مرتب کننده که شما مطالعه نکرده اید می خوانید، فرم خام صادقانه است.

## مرحله هفتم: انواع استفاده

کد ماشین دارای عرض و بدون نوع است.`infer_types()`این شبکه که 32 بیت عرض دارد،`int`، و یک وب 64 بایت است`long`. یک وب که پایه ی دسترسی عنصر است`base[idx]`با عناصر 4 بايت`int *`. نوع اشاره به طول کپی ها و phis پخش می شود، بنابراین `local_8 = param_1`در`-O0`هر دوتا رو ميکنه`int *`. نوع بازگشت عرض ارزش در `return`اظهارات

نتیجه ی`total`.`int total(int *param_1, int param_2)`. گيدرا نوشته`int total(long param_1,uint param_2)`هر دو با دستورالعمل ها مطابقت دارند. بلند کننده از بار مقیاس شده به عنوان مدرک اشاره استفاده کرد. گیدرا عرض را نگه داشت و دسترسی را به عنوان`*(int *)(param_1 + uVar2 * 4)`. نه مي تونم ببينم`const`و هيچکدوم نمي تونن بفهمن`int *`از`unsigned *`. .

لی، آویگرینوس و بروملی (TIE، NDSS 2011) بازیافت نوع فریم به عنوان حل محدودیت در یک شبکه. نتیجه یک مرز بالا و پایین به جای یک حدس است. نونان، لوگینوف و کوک (Retypd، PLDI 2016) افزودن پلیمورفیسم و تکرار. آنها گزارش 98 درصد به یاد آوردن در .`const`این درس از دو قانون ارزان ترین از آن خط کار استفاده می کند و نام هایی را که نمی توانند ببینند.

حسابداری بدون امضا در عبارت ها و نه در انواع متغیر نگهداری می شود. `ja`بعد از`cmpl $0x4, %ecx`چاپ شده`(unsigned int)ecx_1 > 4`و`shrq`چاپ شده`(unsigned long)rax_1 >> 63`متغیرها امضا می شوند تا خواننده ارزش ثبت را و تنها جایی که در آن دستورالعمل به عنوان امضا نشده دیده شود.

## مرحله ۸: ساخت از نمودار

سیفوانتز (1994) ، فصل 6، مناطق نمودار را با شکل های یک زبان سطح بالا مطابقت می دهد. حلقه ها اول (6.6.1) ، سپس شرایط دو طرفه (6.6.2) ، سپس شرایط n طرفه (6.6.3) هستند. یاکدان، اِشویلر، گِراردز-پادیلا و اسمیت (NDSS 2015) از جاده ی دیگر استفاده می کنند. آنها شرایطی را که هر گره تحت آن اجرا می کند، بدست می آورند و ساختار را از این شرایط بدون اشکال ثابت بسازند. انگیزه شان اندازه گیری است. هیکس ریز در یک نمونه تولید شد 1,571 `goto`بیانات در 49514 خط، یک در هر 32 خط، و ساختار آنها هیچ یک از 1530 تابع اصلی را تولید نکرد.

`structure()`یک عضو کوچک از خانواده اول است. به دو واقعیت از مرحله 2 نیاز دارد.`Branch`یک لوله طبیعی مجموعه بلوک هایی است که بدون عبور از سرگویی به منبع لوله عقب می رسند.

برای یک`Branch`با پیوستن`j`:

- اگه`j`هدف پرش هست، فرست`if (!cond) { fallthrough region }`و ادامه بده`j`. .
- اگه`j`در حال سقوط، پخش`if (cond) { target region }`و ادامه بده`j`. .
- اگه`j`بازي کوچک تر را به عنوان`if (cond) {... }`و بازوی دیگر پس از آن، بدون یک`else`اين بازگشت زودرس است`percent_change`و`total`. .
- در غیر اینصورت پخش کنید`if (cond) { target } else { fallthrough }`و ادامه بده`j`اينم`percent_change`در`-O0`، جایی که هر دو اسلحه ذخیره می شوند`local_4`و در پيالوگ ملاقات ميکنيم

برای یک سر و سر حلقه:

- اگر سرش به یک`Branch`با یک جانشین خارج از حلقه، پخش کنید`while (cond) { body }`.`total`در`-O0`این شکل داره: سرش مقایسه می کنه`local_14`با`local_c`و برگ ها رو ادامه بده`jge`. .
- اگر سر و کله هم بلوک با لبه عقب باشد، یا در عوض از لچ خارج شود، emit `do { body } while (cond);`.`total`در`-O1`شکلش اینه: بلوک`0x50`اضافه می کند، افزایش می دهد، مقایسه می کند و به خودش می پرشد.

یک`IndirectJump`بدون جانشین، یک بلوک را به پایان می رساند.`grade`در`-O0`يه ميز پرنده واقعي هست`jmpq *%rax`از طریق پنج تعویض نسبی در `.rodata+0x0`دروس 33 اين اهداف رو حل ميکنه اينجا بلندگر يه يادداشت چاپ ميکنه`0x76`،`0x7f`،`0x88`و`0x91`در حال حاضر، این دو مورد از این دو مورد در مورد استفاده از یک دستگاه استفاده می شود.`switch`. .

### -O1 با سوئیچ چیکار کرد؟

در`-O1`کلنگ يه ميز پرش رو نگه نداشت`grade`. حسابش رو انجام داد`score / 10 - 6`و نتیجه را با 4 به عنوان یک عدد غیرموقعه مقایسه کرد. سپس ارزش بازگشت را از جدول پنج ورودی از `int`در`.rodata+0x14`. مقایسه بدون امضا يه دستوريه که هر دو رو پوشش ميده`score / 10 < 6`و`score / 10 > 10`تفاوت منفی یک مقدار بزرگ بدون امضا است، بنابراین همان چک را شکست می دهد.

میز نگه داره`1, 2, 3, 4, 4`برای کوتیوینت های 6 تا 10`switch`با پنج تا`return`اعلامیه ها تبدیل به یک`if`و یک صف خوانده شده. بلند کننده دقیقاً همین را منتشر می کند و نام میز را می دهد`lookup_14`پس از تعویض بخش آن

طول جدول یک هوریستیک است.`table_values()`از جدول شروع به جدول بعدی`.rodata`از هر چیزی تعویض می کنه`.text`در این شی، آفسیت بعدی که به آن اشاره می شود `0x28`، جدول`grade_label`، پس`lookup_14`.`(0x28 - 0x14) / 4 = 5`.از نظر نميشه با 4 مقایسه کرد .گيدرا طولش رو هم نميدونه`*(uint *)(&DAT_001000b4 + (ulong)uVar2 * 4)`، یک آدرس بائیت به علاوه یک تعویض، و پنج مقدار را برای خواننده برای نگاه کردن به بالا.

## کنار هم با گیدرا

هر دو تا از نسخه های دمو چاپ شده متن گیدرا از`code/fixtures/ghidra/*.json`، شواهد ثبت کرده که ریا در سال 2026-10-14 برگشت`npx -y rea-agents@6.5.0 decompile ledger-x86_64-O1.o <function> --provider ghidra --format json`. هر ريكوردي همراهش داره`confidence: observed`و`authority: shipped-artifact`. متن مشاهده می شود. ادعاهای درون متن حاصل شده و اسناد نمی گویند کدامین.

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

نسخه اصلاح شده در `code/fixtures/ghidra/ledger-x86_64-O1.ghidra-repaired.c`فقط این تابع را ویرایش می کند و آزمایش آن را با منبع بیش از 81 جفت ورودی مقایسه می کند.

### درجه

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

### کل

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

آزمون رفتار متن Ghidra را برای `grade`و`total`بدون تغییر، دو تا اضافه میشه`typedef`خط ها و تعریف`DAT_001000b4`از بیست بايت در `.rodata+0x14`هر دو تابع با منبع شبكه مطابقت دارند

## آنچه تحقیقات درباره خواندن سئودوکوید می گویند

LLM4Decompile (EMNLP 2024) تنها Ghidra را با 20.12 درصد قابلیت اجرای مجدد در HumanEval-Decompile اندازه گیری کرد. حدود یک نفر از هر پنج تابع، ادعاهای همراه با معیار را دوباره جمع آوری و از آن عبور می کند. با اصلاح تولید گیدرا با مدل 6.7B، این رقم به 52.74 درصد رسید. DecompileBench (Findings of ACL 2025) شش decompiler صنعتی و شش رویکرد LLM را در 23،400 عملکرد از 130 برنامه اجرا کرد. نتایج LLM بهتر بود و 52.2 درصد کمتر از عملکردی درست بود.

لیو، راف و میسینسکی (arXiv 2609.05370, سپتامبر 2026) این شکاف را توضیح می دهند. آنها از ۳۰۰ تابع کتابخانه گیت هاب و ۲۸۷ تابع مبتنی بر CVE استفاده کردند. قوی ترین مدل اصلاحات، میزان ساخت گیدرا را از 75 تا 90 درصد افزایش داد در حالی که میزان مطابقت رفتار از 74 تا 62 درصد کاهش یافت. کاندیداهای که در هر آزمون در معیار معیار عبور کردند هنوز هم در مورد ورودی های گیج شده متفاوت بودند. این اتفاق در ۴.۹ درصد موارد در هشت سیستم و تا ۱۳ درصد برای یک سیستم رخ داد. تا یک دهم آسیب پذیری های آشکار شده از کد دقیق ناپدید شده اند.

علت آن ها ساده است. زمینه ها، انواع، کلیه ها و نگهبانان اختراع شده، جای نامعلوم های قابل مشاهده ای را که در متن آن ها باقی مانده بود، جایگزین می کنند.

به همین دلیل است که اون پرینت بلند کننده`(unsigned long)(unsigned int)ecx_1`به جای یه اسم زیبا تر یه یادداشت چاپ می کنه`switch`برای پرش حل نشده و`lookup_14`به جای برچسب یک مدل اختراع می کند. هر خط یک مشتق دارد که می توانید در آن بخوانید`code/main.py`. گيدرا`undefined1 [16]`. زشت و صادقانه است . مدل رو پاک و تمیز ميکنه`int percent_change(int, int)`درست است به دلیل اشتباه. همان مدل یک امضا اشتباه در تابع بعدی تولید خواهد کرد.

## آن را بسازید

نمایش نمایش را اجرا کن:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode
python3 code/main.py
```

دمو تمام چهار تابع را رمزنگاري مي کند`-O1`هدف و مقایسه هر دستورالعمل با انجام شده است`llvm-objdump`متن`percent_change`،`grade`و`total`این IR را قبل از SSA، لیست SSA را با تابع های phi و کد پتوکو در کنار متن Ghidra چاپ می کند. سپس چاپ می کند `grade`با قطع کردن تشخیص تقسیم،`-O0`نسخه های `percent_change`،`total`و`grade_label`و خلاصه ای:

```json
{
  "functions": ["grade", "grade_label", "percent_change", "total"],
  "instructions": 56,
  "blocks": {"percent_change": 3, "grade": 3, "total": 5, "grade_label": 4},
  "phis": {"percent_change": 0, "grade": 1, "total": 2, "grade_label": 1},
  "objdumpDifferences": 0
}
```

بخون`code/main.py`در این ترتیب:

1. `read_object()`سرپرستی بخش ها، جدول نماد ها و `.rela`بخش هایی از یک شی که می تواند در ELF64 تغییر مکان کند.
2. `decode_one()`و`render_att()`یک دستور رو رمزگشایی کن و آن را به این ترتیب چاپ کن`llvm-objdump`آره
3. `build_blocks()`،`immediate_dominators()`و`dominance_frontiers()`و اطلاعات را به صورت نمودار بسازید.
4. `lift_insn()`تبدیل یک دستورالعمل به بیانات IR و حل نقل مکان ها.
5. `insert_phis()`و`rename()`ساخت SSA و وارد کردن قطعات عرض
6. `make_simplifier()`،`substitute_once()`و`remove_dead_code()`داخل رو ببري`optimize()`. .
7. `collect_params()`،`coalesce_webs()`و`infer_types()`نام و تایپ شبکه ها
8. `structure()`با`post_dominators()`و`natural_loops()`بدن C رو پخش ميکنه
9. `lift_function()`مرحله ها رو با هم ببندد و`lift_all()`تمام اشیا را با امضا کالی بلند می کند.

## ازش استفاده کن

برای بلند کردن یک تابع از جسم خود، آن را با `clang -target x86_64-unknown-linux-gnu -c`و تماس بگیرید`lift_all(read_object(path))`. وقتي که ديكودر بلند بشه`DecodeError`، باط opcode را از پیام بخوانید و آن را در نقشه opcode SDM پیدا کنید. سپس یک ورودی را به جدول های بالای فایل اضافه کنید. هنگامی که بلند کننده روی یک mnemonic بالا می رود، یک قانون را به `lift_insn()`وقتی سازنده یک`goto`، گراف رو با  چاپ کن`IrPrinter`و تصمیم بگیرید که کدام قانون شکل گم شده است. هر اضافه کوچک است و هر یک با آن بررسی می شود`llvm-objdump`یا از طریق آزمایشات موجود.

هنگامی که Ghidra نصب شده است، rea را روی همان تابع اجرا کنید و دو متن را کنار هم قرار دهید. قبل از اینکه به هر یک از متن ها باور کنید، جدول تفاوت را پر کنید. تفاوت با یک دستورالعمل پشت آن انتخاب ارائه است. تفاوت بدون دستورالعمل پشت آن حدس است و حدس می تواند از هر دو طرف باشد.

## آزمایشگاه تعاملی

### آزمایشگاه A: یک تابع از طریق پنج مرحله

از`code`دایرکتوری، پیتون باز:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode/code
python3 -q
```

راه رفتن:

```python
from main import *
obj = read_object(FIXTURES / "ledger-x86_64-O1.o")
lifted = lift_function(obj, "percent_change")
for insn in lifted.decoded: print(f"0x{insn.address:x}: {insn.raw.hex(' ')}  {insn.att()}")
print(lifted.ir_text)
print(lifted.ssa_text)
print(lifted.pseudocode)
```

دنبالش کن`%esi`از`subl %edi, %esi`به`esi_1 = esi_0 - edi_0`به`(param_2 - param_1)`بايت`fe`ModR/M از `subl`، با`mod = 11`،`reg = edi`و`r/m = esi`. .

### آزمایشگاه بی: زبان تقسیم

```python
print(lift_function(obj, "grade", recognize_division=False).pseudocode)
print(magic_divisor(0x66666667, 34), magic_divisor(0x66666667, 33), magic_divisor(0x66666667, 31))
```

خط اول ضرب خام و تغییرات را چاپ می کند. خط دوم چاپ می کند.`10 5 None`.درسی که چرا همان ضرب به 5 تقسیم می شود با یک تغییر کمتر و چرا تغییر 31 خطای خطایی را شکست می دهد.

### آزمایشگاه C: یک شی جدید کد کش را متوقف می کند

دستگاه را در `-O2`و بلندش کنم:

```bash
clang -target x86_64-unknown-linux-gnu -O2 -fno-asynchronous-unwind-tables -c fixtures/ledger.c -o "$TMPDIR/ledger-O2.o"
python3 -c 'from main import *; from pathlib import Path; import os; lift_all(read_object(Path(os.environ["TMPDIR"]) / "ledger-O2.o"))'
```

کدگر در  توقف می کنه`0x56`با`opcode 81 is outside the fixture subset`. کد آپکود`81`این فرم گروه 1 با یک 32 بت فوری است.`andl $0x7ffffff8, %edx`، و کلنگ ازش در حلقه متراکي استفاده کرد`total`. اضافهش کن کنارش`0x83`در`decode_one()`، تکرار کنید و توجه کنید که کد آپکد بعدی شما را متوقف می کند. زیر مجموعه یک ورودی در یک زمان رشد می کند و هر ورودی با `llvm-objdump`. .

### آزمایشگاه D: ارزش پیش فرض Ghidra در درجه

```python
print(ghidra_pseudocode("grade"))
for score in (-25, 0, 59, 60, 100, 115):
    sign = -(score >> 31)
    product_sign = ((score * 0x66666667) & ((1 << 64) - 1)) >> 63
    print(score, sign ^ product_sign)
```

هر خطي 0 ميده`uVar1`عبارت درست و غیر قابل خواندن است و دستورالعمل`xorl %eax, %eax`در دو بايت ميگه

## آزمایشگاه تمرین

اضافه کردن`switch`بازپرداخت برای`grade`در`-O0`. .

الزامات:

- پنج نقل مکان رو در سال`.rela.rodata`از`ledger-x86_64-O0.o`، هر یک`R_X86_64_PC32.text+offset`، و پنج هدف مورد را از جدول در `.rodata+0x0`. .
- اضافه کنید`Switch`در این حالت، این گزینه به عنوان یک "مجموعه" در IR با عبارت شاخص، لیست هدف و پیش فرض از `ja`قبل از بارگذاری میز
- به بلوک سوئیچ یک جانشین در هر هدف بده تا هراسنده ها، هراسنده ها و SSA همچنان کار کنند.
- پخش کن`switch (local_10) { case 0:... }`با يک نفر`case`هر هدف و اپيلوگ مشترک بعد از پیوستن
- اون نوتی رو برای هر پرشی غیر مستقیم که نمیتونی میزش رو ببندی نگه داری
- نتیجه را با clang جمع آوری کنید و آن را با منبع در شبکه امتیاز در آزمون ها مقایسه کنید.

موفقیت به معنی`grade`در`-O0`و`grade`در`-O1`تولید همان خروجی در شبکه، با یک `switch`و نه`goto`. .

## آزمایشگاه درجه بندی شده

پنج سوال در مورد نصبات این درس. هر جواب را با کد ساختی پیدا کن، سپس آن را به `code/lab_answers.json`. اعداد رو به عنوان اعداد یا به عنوان رشته های ششگانه بنویس`"0x40"`, و لیست ها به عنوان لیست JSON

1. `t1`: پارامتر هایی که کنوانسیون تماس %_change را در -O1 می دهد.
2. `t2`: پف افعال در شکل SSA کل در -O1.
3. `t3`: تقسیم کننده ای که magic_divisor برای 0x66666667 با شیفت 34 پیدا می کند.
4. `t4`: آخرین عبارت از کد پشو %_change در -O1 به عنوان یک خط متن.
5. `t5`: نوع C بازیابی شده از param_1 کل در -O1.

جواباتتون رو چک کنيد:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode
python3 code/lab.py
```

چکگر هر ارزش انتظار شده را از  محاسبه می کند`code/main.py`بر روی وسایل بسته شده چاپ می کند`PASS`،`FAIL`یا`TODO`برای هر کار و یک امتیاز، و فقط وقتی که پنج مورد از آن عبور می کنند از صفر خارج می شود.`python3 code/lab.py my-answers.json`پرونده ي ديگه رو چک مي کنه`code/tests/test_lab.py`ثابت می کند که چک کننده پاسخ های درست را قبول کرده و پاسخ های اشتباه را رد کرده است.

### ببینیم که میره

.[Inside a Decompiler](https://aiengineeringfromscratch.com/inside-a-decompiler?opt=O0)ماشین این درس را به عنوان مطبوعات بلند کننده، نوارها، ساخت و ساز و چاپگر، با IR، SSA و کد پتو`code/main.py`اين لينک بازيگر`-O0`هدف: سه سوراخ سقف تبدیل به محل و jig هر دو چنگل را برای اگر / غیر بسته می کند.

## آثار هنری

این درس به ما کمک می کند`outputs/skill-lifting-to-pseudocode.md`. از آن برای رفع یک تابع با یک مشتق برای هر خط و برای پر کردن جدول تفاوت با متن یک decompiler استفاده کنید. این آرتیفکت نیاز به تجزیه مرجع، نقل مکان، کنوانسیون تماس، بررسی جمع آوری، مقایسه رفتار و حکم برای هر خط متفاوت دارد.

## بررسی کنید

نمایش و مجموعه تعیین کننده را اجرا کنید:

```bash
cd phases/13-tools-and-protocols/34-lifting-to-pseudocode
python3 code/main.py
python3 -m unittest discover -s code/tests -v
```

این مجموعه 25 تست داره.

- دیکوتر با تعهدات توافق می کند`llvm-objdump`متن در هر دستورالعمل هر دو شی و با یک زنده `llvm-objdump`وقتی نصب شده است
- REX، ModR/M، SIB و جابجایی میدان `leaq (%rip), %rcx`به گونه ای که در SDM توصیف شده، رمزگذاری می شوند
- یک کد opcode خارج از زیر مجموعه را افزایش می دهد `DecodeError`بجاي حدس زدن
- خواننده ELF چهار تابع و سه تابع را فهرست می کند `.text`نقل مکان
- `total`پنج بلوک داره`0x44`تسلط داره`0x50`و`0x50`یک سر و سر حلقه است که به عنوان بدن خود
- `%rdi`و`%rsi`تبدیل شدن`param_1`و`param_2`و`param_1`در`total`.`int *`
- `test`و`cmp`با`je`،`jle`،`ja`و`jge`تبدیل شدن`==`،`<=`، امضا نشده`>`و`<`
- SSA دو تابع Phi زنده را در سر سر حلقه قرار می دهد`total`و هيچکس در`percent_change`
- کد پشویدوکد `percent_change`،`grade`و`total`برابر با متن انتظار می رود
- `grade`میخواد`param_1 / 10 - 6`با تشخیص و ضرب خام با تشخیص غیر
- `0x66666667`با شیفت 34 به 10 تقسیم می شود، با شیفت 33 به 5 تقسیم می شود و هویت در شبکه باقی می ماند
- در قسمت " پکن "`-O0`تبدیل شدن`local_4`،`local_8`،`local_c`، و نشانگر قاب هرگز ظاهر نمیشه
- میز در `.rodata+0x14`میخواد`1, 2, 3, 4, 4`از طریق انتقال
- اون پرش غیر مستقیم`grade`در`-O0`گزارش شده با چهار بلوک که بلند نشده اند
- چراغ های Ghidra حاوی 16 بایت بازگشت،`param_3`، ساده نکردن`xor`و`long param_1`
- شش واحد منتشر شده با `clang -std=c11 -Wall -Werror`برای`x86_64-unknown-linux-gnu`
- عملکردهای بلند شده، متن Ghidra اصلاح شده و منبع همان 232 خط را بر روی شبکه ورودی چاپ می کنند

## روش های شکست تولید

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

## قاعده عملیاتی

کد فرضی را به عنوان مشتق که ندیدید، نگاه کنید. آن را با یک آسانسور که می توانید بخوانید، از دستورالعمل ها بازتولید کنید. دو متن را کنار هم قرار دهید و دستورالعمل یا حدس را پشت هر خط متفاوت بنویسید. هر دو را جمع آوری کنید و هر دو را در برابر منبع اجرا کنید. یک خط بدون دستورالعمل پشت آن یک حدس باقی می ماند، مهم نیست که چقدر تمیز خوانده شود.

## خواندن بیشتر

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
- rea 6.5.0 منبع در برچسب `3b79af2`که این درس شرح می دهد:`src/domain/functionComparisonDimensions.ts`،`website/evidence/dx-ball-sound-pan.md`. .
