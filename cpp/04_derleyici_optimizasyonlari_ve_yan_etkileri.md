# Derleyici Optimizasyonu ve Yan Etkileri

Optimizasyon seviyesini `-O2`'den `-O3`'e çıkarmak her zaman daha hızlı bir program üretmez, bazı durumlarda programı yavaşlatır. Bunun nedeni `-O3`'ün kod boyutunu gözetmeden çalışmasıdır: döngüleri açar, fonksiyonları satır içine alır ve vektör komutları üretir. Kazanç saydığı bu dönüşümlerin hepsinin bir bedeli vardır ve bedel kazançtan büyük olabilir.

| Bayrak | Ne yapar | Bedeli |
| :--- | :--- | :--- |
| `-O2` | Kod boyutunu makul tutar, dengeli hızlandırır | Döngüleri tam açmadığı ve vektörleştirmediği için tepe verime ulaşamaz |
| `-O3` | Kod boyutunu umursamadan döngü ve hesaplama hızını kovalar | I-cache taşması, register spilling, küçük veride kurulum maliyeti |

### Instruction Cache (I-Cache)

İşlemcinin çalıştıracağı makine kodu komutlarını ana bellek yerine çekirdeğe en yakın L1 seviyesinde tutan donanımsal önbellektir. L1 önbellek veri (D-Cache) ve komut (I-Cache) olarak ikiye ayrılır; aranan komut önbellekteyse hit oluşur ve döngü gecikmesi yaşanmaz, bulunamadığında ise miss meydana gelir ve işlemci komut L2, L3 veya RAM'den getirilene kadar duraklar (stall). C++ doğrudan makine koduna derlendiği için aşırı inlining, kontrolsüz loop unrolling ve sanal fonksiyonların vtable üzerinden dolaylı atlamaları kod boyutunu şişirerek veya dağınık yerleşim oluşturarak I-Cache verimini düşürür. `[[likely]]` / `[[unlikely]]` nitelikleri, PGO ve LTO optimizasyonları sık çalışan sıcak kod bloklarını bellekte ardışık dizerek I-Cache yerelliğini korur.

```cpp
// Sıcak kod yolunu düz hatta tutup soğuk hata bloğunu uzağa taşımak I-cache'i korur
bool process_data(const char *buffer, size_t len) {
    if (buffer == nullptr) [[unlikely]] {
        log_critical_error();  // soğuk blok: uzağa taşınır, I-cache'i gereksiz işgal etmez
        return false;
    }
    return fast_path_parse(buffer, len);  // sıcak blok: ardışık dizilir
}
```

Uç durum: Aşırı satır içi genişletme (inlining) ve kontrolsüz döngü açma (loop unrolling), çağrı ve dallanma maliyetini düşürmeyi hedeflerken üretilen ikili dosya boyutunu (code bloat) I-Cache kapasitesinin üzerine çıkarıp sık miss gecikmelerine yol açar.

### Instruction Cache Baskısı ve Döngü Açma

Döngü açma (loop unrolling), döngü gövdesini birden çok kez arka arkaya yazarak sayaç artırma ve dallanma maliyetinden kurtulmayı amaçlar. Bunun karşılığında üretilen makine kodu birkaç katına çıkar ve işlemcinin L1 komut önbelleğinde (Instruction Cache) daha fazla yer kaplar. Sık çağrılan küçük bir fonksiyonun döngüsü tümüyle açıldığında komutlar bu önbelleğe sığmaz; her çağrıda önbellekten atılıp yeniden yüklenirler (cache thrashing) ve işlemci komutun gelmesini bekler.

```cpp
// Küçük veri üzerinde çalışan, fakat sık çağrılan bir fonksiyon
void process_small_block(int *__restrict out, const int *__restrict in) {
    for (int i = 0; i < 64; ++i) {
        out[i] = in[i] * 3 + (in[i] >> 2);
    }
}
// -O2: 2 veya 4 iterasyonu açıp derli toplu bir döngü üretir (~15-20 bayt); L1 I-cache'te rahat durur.
// -O3: 64 adımı ardışık satırlara açar, kod yüzlerce bayta çıkar ve I-cache'i zorlar.
```

Uç durum: Açılan döngünün kod boyutu L1 komut önbelleği kapasitesini aştığında, dallanmadan kazanılan nanosaniyeler I-cache miss gecikmeleriyle fazlasıyla geri ödenir.

### Küçük Veri Setlerinde SIMD ve Vektörizasyon Maliyeti

`-O3` seviyesinde derleyici döngüleri AVX veya SSE gibi SIMD komutlarına çevirmeye çalışır (auto-vectorization); tek komutta sekiz eleman işlemek skalar döngüden hızlıdır. Ancak bu komutları çalıştırabilmek için derleyici asıl döngünün etrafına üç ek parça koyar: adresleri vektör sınırına hizalayan ön döngü (peeling), dizilerin bellekte üst üste binmediğini doğrulayan çakışma tetkiki (aliasing check) ve vektör genişliğine bölünmeyen artık elemanları işleyen temizlik döngüsü (scalar epilogue). Bu kurulumun maliyeti sabittir, işlenen eleman sayısından bağımsızdır.

```cpp
// Boyutu çalışma zamanında belli olan, fakat pratikte çoğunlukla küçük (ör. n=8) bir dizi
void sum_arrays(float *__restrict c, const float *__restrict a, const float *__restrict b, int n) {
    for (int i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}
// -O2: Yalın bir skalar 'addss' döngüsü üretir; kurulum maliyeti yoktur.
// -O3: Hizalama ve çakışma tetkikleri, 8'li AVX komutları ve artık elemanlar için ek döngü üretir.
```

Uç durum: Dizi boyutu $n$ pratikte 8 veya daha küçük kaldığında, sabit kurulum maliyeti asıl hesaplamadan uzun sürer ve vektörleştirilmiş kod skalar koddan yavaş çalışır.

### Satır İçi Genişletme ve Register Spilling

`-O3`, fonksiyon çağrı maliyetini ortadan kaldırmak için satır içine almayı (inlining) derinlemesine uygular ve bunu döngü açmayla birleştirir. İki dönüşüm birlikte çalıştığında aynı anda canlı (live) tutulması gereken ara değişken sayısı katlanır. İşlemcinin fiziksel yazmaç (register) sayısı sabit olduğundan (x86-64'te 16 genel amaçlı yazmaç) bu sayı bir noktada sınırı aşar.

```cpp
static inline void complex_math(float *a, float *b, float *c) {
    float t1 = a[0] * b[0] + c[0];
    float t2 = a[1] * b[1] - c[1];
    c[0] = t1 + t2;
}

void caller_loop(float *data, int count) {
    for (int i = 0; i < count; ++i) {
        // -O3 hem bu çağrıyı satır içine alır hem de döngüyü açar:
        // her açılmış adım kendi t1 ve t2 değerini canlı tutar
        complex_math(&data[i], &data[i + 1], &data[i + 2]);
    }
}
```

Uç durum: Canlı değişken sayısı yazmaç sınırını aştığında derleyici bir kısmını yığına yazmak (`mov %rax, -8(%rsp)`) ve kullanacağı an geri okumak zorunda kalır; register spill denen bu gidiş gelişler, kaçınılan çağrı maliyetinden pahalıya gelebilir.

### Bellek Bant Genişliği ve Pre-fetching Uyuşmazlığı

Agresif optimizasyon seviyelerinde derleyici bellek erişim düzenini de değiştirir: iç içe döngülerin sırasını takas eder (loop interchange) veya veriyi kullanılmadan önce önbelleğe çeken yazılımsal ön yükleme (pre-fetch) komutları yerleştirir. İşlemcinin kendi donanımsal ön yükleyicisi (hardware prefetcher) zaten erişim örüntüsünü izleyip aynı işi yapmaktadır. İki mekanizmanın tahminleri uyuşmadığında aynı veri için iki kez bellek trafiği doğar.

```cpp
// Sütun sütun gezildiği için her erişim farklı bir cache line'a düşer;
// derleyici bu örüntüde döngü takası veya pre-fetch uygulamaya çalışır
for (int j = 0; j < sutun; ++j) {
    for (int i = 0; i < satir; ++i) {
        matris[i][j] = matris[i][j] * 2;
    }
}
```

Uç durum: Yazılımsal pre-fetch komutları donanımsal ön yükleyicinin tahminiyle uyuşmadığında boşuna bellek bant genişliği harcanır ve hâlâ kullanımda olan faydalı veriler önbellekten atılır.

### `-O0`

Varsayılan seviyedir ve hiçbir optimizasyon açmaz. Her değişken yığında tutulur, her ifade kaynaktaki sırayla çalışır, hiçbir fonksiyon inline edilmez. Bu yüzden derleme hızlıdır ve debugger'da görülen durum kaynak kodla birebir örtüşür.

Uç durum: `-O0`'da çalışıp `-O2`'de bozulan kod optimizasyon hatası değildir; neredeyse her zaman undefined behavior'ın gizlenmiş olmasıdır.

### `-O1`

Derleme süresini fazla uzatmayan, tek fonksiyon içinde biten ucuz dönüşümleri açar. Amaç tepe verim değil, `-O0`'ın ürettiği gereksiz yükü temizlemektir.

| Bayrak | Ne yapar |
| :--- | :--- |
| `-fdce`, `-fdse` | dead code ve dead store elimination: sonucu kullanılmayan hesap ve yazma silinir |
| `-ftree-ccp` | constant propagation: derleme anında bilinen değerler yerine konur |
| `-fmerge-constants` | aynı sabit ve string literal tek kopyaya indirilir |
| `-fif-conversion` | kısa `if` blokları dallanma yerine `cmov` komutuna çevrilir |
| `-fomit-frame-pointer` | `rbp` yığın çerçevesi için ayrılmaz, genel amaçlı yazmaç olarak kullanılır |
| `-finline-functions-called-once` | tek yerden çağrılan `static` fonksiyonlar inline edilir |

Uç durum: `-fomit-frame-pointer` açık olduğunda `perf` ve benzeri profiler'lar stack'i geriye doğru çözemez; profil alınacak build'lerde `-fno-omit-frame-pointer` ile geri kapatılır.

### `-O2`

`-O1` kümesine, tüm fonksiyonu birden gören pahalı analizleri ekler. Üretim build'lerinin varsayılanıdır: kod boyutunu ciddi biçimde büyütmeden en geniş dönüşüm setini uygular.

| Bayrak | Ne yapar |
| :--- | :--- |
| `-finline-functions` | küçük fonksiyonlar çağrı yerine gövdeleriyle değiştirilir |
| `-fgcse`, `-ftree-pre` | common subexpression ve partial redundancy elimination: aynı hesap bir kez yapılır |
| `-fstrict-aliasing` | farklı türden iki işaretçinin aynı adresi göstermediği varsayılır, yükleme tekrarı elenir |
| `-fdevirtualize` | gerçek tür derleme anında biliniyorsa sanal çağrı doğrudan çağrıya döner |
| `-foptimize-sibling-calls` | tail call'lar `call` yerine `jmp` ile yapılır, stack büyümez |
| `-fschedule-insns2` | komutlar pipeline duraklamasını azaltacak sırayla dizilir |
| `-freorder-blocks` | sık çalışan bloklar arka arkaya konur, soğuk bloklar sona atılır |

Uç durum: `-fstrict-aliasing` bir varsayımdır, tetkik değildir; `float`'ı `int*` ile okumak gibi tür sınırını aşan `reinterpret_cast` kullanımları `-O2`'de uyarı vermeden yanlış sonuç üretir, doğru yol `std::memcpy` veya `std::bit_cast`'tir.

### `-O3`

`-O2` kümesine, kod boyutunu büyütme pahasına hız kovalayan döngü dönüşümlerini ekler. Kazanç büyük dizilerde gerçektir; küçük ve sık çağrılan kodda I-cache baskısı ve kurulum maliyeti yüzünden ters tepebilir.

| Bayrak | Ne yapar |
| :--- | :--- |
| `-ftree-loop-vectorize` | döngü SIMD komutlarına çevrilir, bir komutta birden çok eleman işlenir |
| `-floop-unroll-and-jam` | dış döngü açılır, iç döngü gövdeleri birleştirilir |
| `-fpeel-loops`, `-fsplit-loops` | ilk veya düzensiz iterasyonlar döngüden ayrılır, kalan kısım düzgün hâle gelir |
| `-funswitch-loops` | döngü içindeki değişmeyen `if` dışarı çıkarılır, döngü iki kopyaya bölünür |
| `-fpredictive-commoning` | ardışık iterasyonların tekrar okuduğu değerler yazmaçta taşınır |
| `-fipa-cp-clone` | sabit argümanla çağrılan fonksiyonun o argümana özel kopyası üretilir |

Uç durum: `-O3` üretilen kodu birkaç katına çıkarabilir; hız kazancı ölçülmeden varsayılmaz, `-O2` ile karşılaştırmalı benchmark almadan `-O3`'e geçmek çoğu kod tabanında kayıptır.

### `-Os` ve `-Oz`

`-Os`, `-O2` kümesinden kod boyutunu büyüten dönüşümleri çıkarır: inline eşiği düşer, loop unrolling ve vectorization kapanır, fonksiyon hizalama dolgusu yapılmaz. `-Oz` aynı hedefi daha sert uygular ve boyut için hızdan taviz vermeyi kabul eder. Küçük kod tümüyle I-cache'e sığdığı için bu seviyeler bazı gerçek uygulamalarda `-O3`'ten hızlı çıkar.

Uç durum: `-Os` embedded ve boyut kısıtlı hedefler için düşünülür, fakat cache'e sığma etkisi yüzünden sunucu tarafında da denenmeye değer; hangi seviyenin kazandığı ancak ölçümle belli olur.

### `-Og`

Hata ayıklamayı bozmayan optimizasyonları açar; `-O1`'e yakın bir küme uygular fakat değişkenlerin debugger'da görünürlüğünü ve satır bilgisini koruyanları seçer. Debug build'lerde `-O0` yerine önerilen seviyedir, çünkü kod makul hızda kalırken breakpoint ve değişken izleme çalışmaya devam eder.

Uç durum: `-Og` bile bazı değişkenleri yazmaca aldığı için debugger `<optimized out>` gösterebilir; o değişken kritikse ilgili çeviri birimi `-O0` ile derlenir.

### `-Ofast`

`-O3` kümesine standart uyumunu bozan gevşetmeleri ekler: `-ffast-math`, `-fno-protect-parens` ve `-fallow-store-data-races`. Kayan nokta ifadelerinin yeniden gruplanmasına izin verdiği için `-O3`'ün vektörleştiremediği `float` indirgeme döngüleri burada SIMD'e çevrilir.

Uç durum: `-ffast-math` `-ffinite-math-only` içerdiğinden derleyici `NaN` oluşmayacağını varsayar ve `x != x` tetkiki her zaman `false` döner; ayrıca bayrak, denormal sayıları sıfırlayan MXCSR bitini program başlangıcında açtığı için bu bayrakla derlenmemiş kütüphanelerin davranışını da değiştirir.

### Seviye Dışı Bayraklar

Seviye bayrakları hangi dönüşümlerin uygulanacağını seçer; bu grup ise dönüşümlerin hangi bilgiye dayanacağını belirler ve seviyeden bağımsız eklenir.

| Bayrak | Ne yapar | Bedeli |
| :--- | :--- | :--- |
| `-march=native` | derleyen makinenin tüm komut kümesini (AVX2, AVX-512) kullanır | ikili başka işlemcide `SIGILL` ile çöker |
| `-mtune=...` | komut kümesini değiştirmeden komut sırasını hedef işlemciye göre ayarlar | yanlış hedefte küçük verim kaybı |
| `-flto` | optimizasyonu çeviri birimleri arasına taşır, farklı `.cpp` dosyaları birbirine inline olur | bağlama süresi ve bellek tüketimi artar |
| `-fprofile-generate` / `-fprofile-use` | gerçek çalışma verisiyle hangi dalın sıcak olduğu ölçülür, yerleşim ona göre yapılır | iki aşamalı build; profil eskirse kod kötüleşir |

Bir seviyenin o derleyici sürümünde tam olarak neyi açtığı `gcc -Q --help=optimizers -O2` ile listelenir; iki seviyenin farkını görmek için iki çıktının `diff`'i alınır.

Uç durum: `-march=native` derleme yapılan makineye göre kod üretir; build sunucusu ile dağıtım sunucusunun işlemcisi farklıysa program ilk çalıştırmada `Illegal instruction` verir, taşınabilirlik gerektiğinde `-march=x86-64-v2` gibi taban bir seviye sabitlenip yalnızca `-mtune` serbest bırakılır.

### [[assume]]

C++23 ile gelen `[[assume(koşul)]]` özniteliği, derleyiciye belirtilen ifadenin çalışma zamanında her zaman doğru olduğunu bildirir. Koşul için çalışma zamanında herhangi bir denetim kodu üretilmez; derleyici bu bilgiyi değer aralıklarını daraltmak, gereksiz sınır tetkiklerini kaldırmak ve döngüleri açmak için doğrudan bir optimizasyon girdisi olarak kullanır. Bildirilen varsayım çalışma zamanında yanlış çıkarsa program tanımsız davranışa (undefined behavior) sürüklenir.

```cpp
int bol(int x) {
    [[assume(x > 0)]]; // x'in pozitif olduğu garanti edilir
    return x / 2;      // derleyici negatif tetkiki ve işaret düzeltmesi üretmez
}

int dizi_erisim(const int* ptr, int i) {
    [[assume(i >= 0 && i < 16)]];
    return ptr[i];     // sınır tetkiki ve taşma koruması elenir
}
```

Uç durum: `[[assume]]` içine yazılan koşul ifadesi derleyici tarafından yürütülmeyip sadece analiz edildiğinden ifade içindeki olası yan etkiler çöpe atılır ve asla çalışmaz; varsayımın ihlal edilmesi ise derleyiciyi yanıltarak tanımsız davranış üretir.

### [[likely]] ve [[unlikely]]

C++20 ile gelen `[[likely]]` ve `[[unlikely]]` öznitelikleri, koşullu ifadelerde (if, switch) hangi dallanmanın daha olası olduğunu derleyiciye bildirir. Derleyici bu ipuçlarını kullanarak sıcak (hot) kod yollarını bellekte ardışık dizer ve işlemcinin dal tahmincisini (branch predictor) yönlendirir. Soğuk (cold) kalan hata veya istisna blokları döngü gövdesinden uzağa taşınarak komut önbelleği (I-Cache) yerelliği korunur.

```cpp
void paket_isle(const char* veri, int uzunluk) {
    if (veri != nullptr) [[likely]] {
        hizli_ayristir(veri, uzunluk); // sık çalışan sıcak yol
    } else [[unlikely]] {
        hata_kaydi_olustur();          // nadir çalışan soğuk yol uzağa ötelenir
    }
}
```

Uç durum: Yanlış işaretlenen dallanmalar işlemcinin donanımsal dal tahmincisini yanıltarak önbellek ıskalamalarına (branch misprediction) ve soğuk blokların I-Cache'i gereksiz doldurmasına yol açar.

