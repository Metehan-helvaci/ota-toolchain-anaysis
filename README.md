# OTA Toolchain Analizi — Contiki-NG UDP Firmware'leri (Sky & Z1)

Bu çalışma, [ismailhakkituran/ota-toolchain-anaysis](https://github.com/ismailhakkituran/ota-toolchain-anaysis) şablon reposundan forklanmıştır. Şablondaki kontrol listesi başlıkları altına, iki farklı platform için derlenmiş firmware imajlarının MSP430 araç zinciri (`msp430-*`) ile analizleri eklenmiştir.

| | Firmware A | Firmware B |
|---|---|---|
| Dosya | `udp-server.sky` | `udp-client.z1` |
| Platform | Tmote Sky / TelosB | Zolertia Z1 |
| Hedef MCU | MSP430F1611 (48 KB Flash / 10 KB RAM) | MSP430F2617 (92 KB Flash / 8 KB RAM) |
| Rol | UDP **sunucu** (RPL kök) | UDP **istemci** |
| Derleyici | mspgcc GCC 4.7.2 | mspgcc GCC 4.7.2 |

> **Kapsam notu:** Bu teslim, ödev şartnamesinin **ELF analizi** odağına uygun olarak şablonun şu başlıklarını gerçek araç çıktılarıyla doldurur: **1, 2, 3, 4, 7, 8, 17, 22**. Assembly/networking/profiling gibi dinamik/derin başlıklar (5, 6, 9–16, 18–21, 23) bu ELF-odaklı çalışmanın kapsamı dışındadır. Bellek/disk yerleşimi `bellek-haritasi.svg` dosyasında görselleştirilmiştir.

---

# 1. Binary Kimlik Analizi

```
$ file udp-server.sky
ELF 32-bit LSB executable, TI msp430, version 1 (embedded),
statically linked, with debug_info, not stripped
```

| Özellik | Server (.sky) | Client (.z1) | Anlamı |
|---|---|---|---|
| ELF format | ELF32, EXEC | ELF32, EXEC | Çalıştırılabilir, mutlak-adresli imaj (relocatable/shared değil) |
| Mimari | TI msp430 | TI msp430 | `e_machine` alanı; MSP430 araç zinciri gerektirir |
| Endianness | little-endian (LSB) | little-endian | Çok baytlı değerler düşük bayt önce; vektör adresleri buna göre okunur (`00 40`→`0x4000`) |
| Entry point | `0x4000` | `0x3100` | Reset sonrası ilk komut adresi = MCU'nun Flash başlangıcı |
| ABI | Standalone App | Standalone App | Bare-metal; işletim sistemi yok |
| Toolchain | mspgcc 4.7.2 (`.comment`) | mspgcc 4.7.2 | `$ msp430-readelf -p .comment` ile doğrulandı |
| Debug sembolü | var (not stripped) | var (not stripped) | DWARF + sembol tablosu korunmuş → `addr2line` mümkün |

**Endianness nedir:** Belleğe çok baytlı bir sayının hangi sırada yazıldığıdır. Little-endian'da en düşük anlamlı bayt en düşük adrese gelir. **ABI nedir:** Uygulama İkili Arayüzü — fonksiyon çağrı kuralları, register kullanımı, veri hizalama gibi derleyici/donanım sözleşmesidir; burada "Standalone App" çıplak donanım ABI'sini işaret eder.

**Optimization tahmini:** Kod yoğunluğu ve `-ffunction-sections` benzeri ayrı fonksiyon yerleşimi, boyut-odaklı (`-Os`) tipik bir Contiki-NG derlemesine işaret eder.

Araçlar: `file`, `msp430-readelf -h`, `msp430-readelf -p .comment`, `msp430-objdump`, `msp430-strings`

---

# 2. Bellek Kullanım Analizi

**Flash / RAM / Stack / Heap anlamları:** *Flash* kalıcı, programlanabilir, güç kesilince silinmeyen bellektir (kod burada durur). *RAM* uçucu çalışma belleğidir (değişkenler, stack). *Stack* fonksiyon çağrıları ve yerel değişkenler için yukarıdan aşağı büyüyen alandır. *Heap* dinamik tahsis (`malloc`) alanıdır — Contiki-NG genelde statik bellek kullandığı için pratikte yok denecek kadar azdır.

```
$ msp430-size udp-server.sky                 $ msp430-size udp-client.z1
   text    data    bss    dec               text   data   bss    dec
  43000     330   6968  50298               42542   336  5888  48766
```

| | .text (Flash) | .data (Flash+RAM) | .bss (RAM) | RAM toplam (data+bss) | RAM doluluk |
|---|---|---|---|---|---|
| Server | 43.000 B | 330 B | 6.968 B | 7.298 B | %71 (10 KB içinde) |
| Client | 42.542 B | 336 B | 5.888 B | 6.224 B | %76 (8 KB içinde) |

- **.text boyutu:** Flash'a yazılacak kod. Server'da Flash doluluğu ~%90 (48 KB), Client'ta ~%46 (92 KB) — OTA için yeni imajın bu bütçeye sığması şarttır.
- **.data:** Hem Flash'ta (ilk değer) hem RAM'de yer kaplar; RAM bütçesinden de düşülür.
- **.bss:** Sadece RAM tüketir, Flash'a yazılmaz (`NOBITS`).
- **Stack kullanım tahmini:** SP `0x3900`'den (Sky) başlar; statik veriden artan ~3 KB stack+heap'e kalır. **Heap:** Contiki-NG statik bellek kullandığından ayrı bir heap bölümü görülmez.
- **Büyük veri yapıları:** `.bss`'in büyüklüğü ağ yığını tamponlarından gelir; Server'da RPL kök komşu tablosu nedeniyle `.bss` daha büyüktür.

**Section dağılımı / memory map:** Ayrıntılı yerleşim için bkz. `bellek-haritasi.svg`.

Araçlar: `msp430-size`, `msp430-readelf -S`, `msp430-nm`

---

# 3. Symbol / Function Analizi

```
$ msp430-readelf -s ... (tür dağılımı)
Server: 457 FUNC, 218 OBJECT, toplam 919 sembol
Client: 482 FUNC, 228 OBJECT, toplam 1024 sembol
```

nm harf kodları: **T/t** = fonksiyon (global/static), **D/d** = `.data` değişkeni, **B/b** = `.bss` değişkeni, **A** = linker mutlak sembolü, **U** = tanımsız, **W** = weak.

| Kategori | Örnek semboller |
|---|---|
| C giriş & runtime | `main` (0x403e), `_reset_vector__`, `autostart_start` |
| Contiki process entry | `process_thread_etimer_process`, `process_thread_ctimer_process`, `process_thread_cc2420_process` |
| ISR (kesme) fonksiyonları | `cc2420_interrupt`, `cc2420_port1_interrupt`, `cc2420_timerb1_interrupt` |
| Radio driver | `cc2420_process`, `cc2420_*` ailesi (CC2420 802.15.4 radyo) |
| Timer callback | `etimer_*`, `ctimer_set_with_process`, `ctimer_reset` |
| Networking | `tcpip_process`, `simple_udp_process`, `uip_process` |
| Linker sınır sembolleri | `__data_start`, `__bss_start/end`, `__stack`, `__bss_size` |
| **Server'a özgü** | `udp_server_process`, `rpl-dag-root` sembolleri |
| **Client'a özgü** | `udp_client_process`, `udp_rx_callback`, `node_id_z1_restore` |

**Yorum:** Yüzlerce isimli sembol, imajın stripped olmadığını ve tam Contiki-NG ağ yığını içerdiğini gösterir. Semboller tek başına firmware'in bir **RPL/6LoWPAN UDP istemci–sunucu çifti** olduğunu reverse engineering'e gerek kalmadan ele verir.

Araçlar: `msp430-nm -n`, `msp430-readelf -s`, `msp430-objdump`

---

# 4. String ve Metadata Analizi

```
$ msp430-strings udp-server.sky | grep -iE "..."
Starting Contiki-NG-develop/v4.9-639-g6ac4608cd     ← sürüm izi
UDP server / Simple UDP process                      ← rol
RPL Lite, 6lowpan, coap                              ← protokol yığını
Node ID: %u                                          ← log format
Check failed: %u vs. %u                              ← assert/debug kalıntısı
rpl-dag-root.c, rpl-icmp6.c, rpl-mrhof.c             ← kaynak dosya izleri

$ msp430-strings udp-client.z1
UDP client
Sending request %lu to                               ← istemci davranışı
node-id-z1.c, udp-client.c, udp_rx_callback          ← Z1'e özgü
```

**Yorum:**
- **Debug/printf mesajları:** `Node ID: %u`, `Sending request %lu to`, `Check failed: %u vs. %u` — çalışma anı log/assert izleri.
- **Sürüm metadata'sı:** Her ikisi de `Contiki-NG v4.9-639-g6ac4608cd` (client `-dirty` → commit sonrası yerel değişiklik).
- **Protokol izleri:** RPL, 6LoWPAN, CoAP stringleri ağ yığınını doğrular.
- **Rol ayrımı:** "UDP server" vs "UDP client" + "Sending request" stringi istemci/sunucu rolünü kesinleştirir.
- **Hardcoded gizli bilgi/credential:** Tespit edilmedi.

Araçlar: `msp430-strings`

---

# 7. ELF Yapısı Analizi

**ELF header** (bkz. Başlık 1) + **Section header** + **Program header** + **Symbol table** + **Vektör tablosu** + **Başlangıç rutinleri**:

```
$ msp430-readelf -S udp-server.sky   (özet)
 [ 1] .text     PROGBITS  0x4000   0xa2f6  AX     ← kod
 [ 2] .rodata   PROGBITS  0xe2f8   0x04e2   A     ← salt-okunur veri
 [ 3] .data     PROGBITS  0x1100   0x014a  WA     ← ilk-değerli (RAM)
 [ 4] .bss      NOBITS    0x124a   0x1b36  WA     ← sıfır-değerli (RAM, dosyada yok)
 [ 5] .noinit   NOBITS    0x2d80   0x0002  WA     ← reset'te korunan
 [ 6] .vectors  PROGBITS  0xffe0   0x0020  AX     ← kesme vektör tablosu
 [ 8-15] .debug_*  ← DWARF hata ayıklama bölümleri (aranges/info/abbrev/line/frame/str/loc/ranges)
 [17] .symtab + [18] .strtab  ← sembol tablosu
```

| Bölüm | Tip | Bayrak | Konum | Rol |
|---|---|---|---|---|
| `.text` | PROGBITS | A,X | Flash | Makine kodu |
| `.rodata` | PROGBITS | A | Flash | Sabitler, stringler |
| `.data` | PROGBITS | W,A | LMA Flash → VMA RAM | İlk-değerli değişkenler |
| `.bss` | NOBITS | W,A | RAM | Sıfır-değerli değişkenler (dosyada yer tutmaz) |
| `.vectors` | PROGBITS | A,X | Flash en üstü | Kesme/reset vektörleri |
| `.debug_*` | — | — | (yüklenmez) | DWARF; `addr2line` için |

**PROGBITS vs NOBITS:** `.bss` `NOBITS` olduğundan 6968 baytı dosyada fiziksel yer tutmaz — sadece "şu kadar RAM ayır" bilgisidir.

**DWARF / debug:** `.debug_info`, `.debug_line` vb. bölümler adres↔kaynak satır eşlemesini sağlar (`msp430-addr2line`). Bunlar cihaza yüklenmez, yalnızca analiz/hata ayıklama içindir.

Araçlar: `msp430-readelf -h/-S/-l/-s`, `msp430-elfedit`

---

# 8. Interrupt ve Donanım Analizi

```
$ msp430-objdump -d -j .vectors udp-server.sky
0000ffe0 <__ivtbl_16>:
 fff0: ... 7c 42 7c 42 00 40    ← son word = 0x4000 (RESET vektörü = entry)

$ msp430-objdump -d -j .vectors udp-client.z1
0000ffc0 <__ivtbl_32>:
 fff0: ... 7c 33 7c 33 00 31    ← son word = 0x3100 (RESET = entry)
```

- **Vektör tablosu konumu:** MSP430'da Flash'ın en üstünde; en son giriş (`0xFFFE`) daima reset vektörüdür.
- **Tablo boyutu:** Server 16 giriş (`0xFFE0`), Client 32 giriş (`0xFFC0`) → F2617'nin daha fazla çevrebirim/kesme kaynağı olduğunu kanıtlar.
- **Kullanılan vs kullanılmayan kesmeler:** Tekrar eden `0x7c42`/`0x337c` değerleri kullanılmayan vektörlerin ortak default handler'a yönlendiğini; farklı değerler (radyo, timer) gerçek ISR'ları gösterir.
- **Donanım kesme handler'ları:** `cc2420_interrupt`, `cc2420_port1_interrupt` (GPIO/SFD), `cc2420_timerb1_interrupt` (Timer_B) sembolleri radyo ve zamanlayıcı kesmelerini işaret eder.

Araçlar: `msp430-objdump`, `msp430-readelf`

---

# 17. Linker ve Build Sistemi Analizi

**LMA ≠ VMA (kopyalama mekanizması):**
```
$ msp430-readelf -l udp-server.sky   (.data segmenti)
LOAD  Offset 0xa8ae  VirtAddr 0x1100  PhysAddr 0xe7da  RW
                     └ çalışma (RAM)   └ yükleme (Flash)
```
`.data`'nın ilk değerleri Flash'ta (PhysAddr/LMA `0xe7da`) saklanır, çalışma adresi RAM'dedir (VirtAddr/VMA `0x1100`). Aradaki kopyalamayı başlangıç kodu yapar.

**Section placement / vector placement:** Linker script `.text`'i Flash başına (`0x4000`/`0x3100`), `.vectors`'ı Flash en üstüne (`0xFFE0`/`0xFFC0`), `.data/.bss`'i RAM başına (`0x1100`) yerleştirir.

**Startup code (reset rutini):**
```
4000: mov.b &0x0120,r5            ; WDTCTL (watchdog) kurulumu
400c: <__init_stack> mov #0x3900,r1   ; SP = stack tepesi (0x3900)
4010: <__do_copy_data> mov #330,r15   ; .data boyutu = 330 (size çıktısıyla birebir)
      ...copy Flash→RAM... ; ...clear .bss... ; call main
```
Akış: **stack kur → .data kopyala → .bss sıfırla → main**. `__do_copy_data`'daki `#330` sabiti `msp430-size`'daki `data=330` ile tutarlı → linker analizinin doğrulaması.

**Symbol resolution:** `statically linked` → tüm semboller imaj içinde çözülmüş, tanımsız (`U`) sembol yok.

Araçlar: `msp430-ld`, `msp430-readelf -l`, `msp430-nm`, `msp430-objdump -d`

---

# 22. Karşılaştırmalı Firmware Analizi

| Ölçüt | udp-server.sky (F1611) | udp-client.z1 (F2617) | Fark / Yorum |
|---|---|---|---|
| Entry point | 0x4000 | 0x3100 | MCU Flash başlangıcına eşit (RAM boyutu farkı) |
| Vektör tablosu | 0xFFE0, 16 giriş | 0xFFC0, 32 giriş | F2617 daha çok çevrebirim |
| Code size (.text) | 43.000 B | 42.542 B | ~%1 fark; aynı kod tabanı |
| RAM (.data+.bss) | 7.298 B | 6.224 B | Server RPL kök tablosu → +1 KB |
| Function count | 457 FUNC | 482 FUNC | Benzer karmaşıklık |
| ISR yoğunluğu | cc2420 + timer | cc2420 + timer | Aynı radyo (CC2420) |
| Networking complexity | RPL kök + sunucu | RPL istemci | Server daha ağır |
| Symbol farkı | `udp_server`, `rpl-dag-root` | `udp_client`, `udp_rx_callback`, `node_id_z1` | Rol ayrımı |
| Optimization / derleyici | mspgcc 4.7.2, ~`-Os` | aynı | Fark yok |

**Sonuç:** İki imaj, aynı Contiki-NG kaynağından aynı derleyiciyle **farklı kart hedeflerine** derilen bir UDP istemci–sunucu ikilisidir. Tüm adres farkları hedef MCU'nun bellek haritasından türer: RAM boyutu → Flash başlangıcı (entry) → vektör tablosu konumu.

---

## Görsel

<img width="1734" height="1424" alt="image" src="https://github.com/user-attachments/assets/da78196a-2f56-45fa-a2f8-c419315ffab3" />


*Sky/F1611 (10 KB RAM, Flash 0x4000) ve Z1/F2617 (8 KB RAM, Flash 0x3100) için firmware'in disk (Flash) ve bellek (RAM) yerleşimi.*

---
