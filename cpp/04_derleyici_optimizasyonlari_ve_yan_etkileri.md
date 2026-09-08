# Derleyici Optimizasyonu ve Yan Etkileri

Optimizasyon seviyesini `-O2`'den `-O3`'e çıkarmak her zaman daha hızlı bir program üretmez, bazı durumlarda programı yavaşlatır. Bunun nedeni `-O3`'ün kod boyutunu gözetmeden çalışmasıdır: döngüleri açar, fonksiyonları satır içine alır ve vektör komutları üretir. Kazanç saydığı bu dönüşümlerin hepsinin bir bedeli vardır ve bedel kazançtan büyük olabilir.

| Bayrak | Ne yapar | Bedeli |
| :--- | :--- | :--- |
| `-O2` | Kod boyutunu makul tutar, dengeli hızlandırır | Döngüleri tam açmadığı ve vektörleştirmediği için tepe verime ulaşamaz |
| `-O3` | Kod boyutunu umursamadan döngü ve hesaplama hızını kovalar | I-cache taşması, register spilling, küçük veride kurulum maliyeti |

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
