# Standart Öznitelikler

C++11 ile standartlaşan `[[...]]` sözdizimi, derleyiciye kodun mantıksal davranışını bozmadan ek bilgi (metadata) iletmek, statik analiz kurallarını sıkılaştırmak ve uyarıları yönetmek için kullanılır.

| Öznitelik | Standart | Temel Amacı |
| :--- | :--- | :--- |
| `[[noreturn]]` | C++11 | Fonksiyonun normal yoldan asla geri dönmeyeceğini bildirir |
| `[[carries_dependency]]` | C++11 | Çok iş parçacıklı işlemlerde bellek bağımlılık zincirini taşır |
| `[[deprecated]]` | C++14 | Kullanım ömrü dolan varlıklar için derleme uyarısı üretir |
| `[[fallthrough]]` | C++17 | `switch-case` içinde kasıtlı geçişi bildirerek uyarıyı kapatır |
| `[[nodiscard]]` | C++17 | Fonksiyon dönüş değerinin veya nesnenin göz ardı edilmesini engeller |
| `[[maybe_unused]]` | C++17 | Kullanılmayan varlıklar için derleyicinin uyarısını bastırır |
| `[[likely]]` / `[[unlikely]]` | C++20 | Dal tahmincisi ve kod yerleşimi için olasılık ipucu verir (Bkz. [04_derleyici_optimizasyonlari_ve_yan_etkileri.md](./04_derleyici_optimizasyonlari_ve_yan_etkileri.md)) |
| `[[no_unique_address]]` | C++20 | Boş üyenin 0 bayt kaplamasını sağlayarak nesne boyutunu küçültür (Bkz. [01_bellek_yonetimi.md](./01_bellek_yonetimi.md)) |
| `[[assume(koşul)]]` | C++23 | Mantıksal koşulun doğruluğunu derleyiciye varsayım olarak bildirir (Bkz. [04_derleyici_optimizasyonlari_ve_yan_etkileri.md](./04_derleyici_optimizasyonlari_ve_yan_etkileri.md)) |
| `[[indeterminate]]` | C++26 | Başlatılmamış değişkenlerin bilerek bırakıldığını bildirerek uyarıyı susturur |

### Öznitelik Mekanizması

C++11 ile standartlaşan `[[...]]` sözdizimi, derleyiciye kodun mantıksal davranışını değiştirmeden ek bilgi iletmek, statik analizi sıkılaştırmak ve gereksiz uyarıları bastırmak için kullanılır. Ön işlemci (`#define`, `#pragma`) derleme öncesinde çalışan saf bir metin manipülasyon aracı iken, öznitelikler doğrudan dil gramerinin ve soyut sözdizimi ağacının bir parçasıdır. Tamamen derleme zamanında değerlendirilirler ve doğrudan bir çalışma zamanı maliyeti üretmezler.

```cpp
// Doğrudan AST seviyesinde derleyiciye iletilen öznitelikler
[[nodiscard]] int veri_oku();

void parametre_testi([[maybe_unused]] int bayrak) {
    // ön işlemci metin süzmesi yapmaz; semantik analiz doğrudan çalışır
}
```

Uç durum: Standart güvencesi uyarınca bir derleyici tanımadığı standart veya özel öznitelikle karşılaştığında derlemeyi durduramaz; sözdizimini geçerli kabul edip özniteliği görmezden gelerek derlemeye devam etmek zorundadır.

### Derleyiciye Özel Öznitelikler

Derleyiciler ISO C++ standardında yer almayan platforma özgü optimizasyon ve davranışları kendi isim alanları üzerinden öznitelik olarak sunar. Clang (`[[clang::...]]`), GCC (`[[gnu::...]]`) ve MSVC (`[[msvc::...]]`) gibi ön ekler taşınabilirliği bozmadan hedef derleyiciye doğrudan yönlendirici buyruklar iletir. Standartlara uyumlu her derleyici, tanımadığı bir isim alanına veya ada sahip öznitelikle karşılaştığında hata vermez; sözdizimini geçerli sayıp özniteliği yok sayarak derlemeyi sürdürür.

```cpp
// Derleyiciye özel optimizasyon ve davranış bildirimleri
[[gnu::always_inline]] inline void zorunlu_inline() {
    // GCC için satır içi genişletmeyi garanti eder
}

[[clang::optnone]] void optimizasyonsuz_calis() {
    // Clang üzerinde yerel optimizasyonları devre dışı bırakır
}
```

Uç durum: Farklı derleyicilerin aynı isimli öznitelikler için farklı semantikler benimsemesi durumunda çapraz derleme ortamlarında sessiz davranış farkları oluşabilir.

### [[nodiscard]]

Fonksiyonun dönüş değerinin veya tanımlanan bir türün çağrıcı tarafından göz ardı edilmesini engeller; üretilen değer tüketilmediğinde derleyici uyarısı tetikler. Hata kodlarının, kaynak tutan sarmalayıcıların veya `std::expected` benzeri nesnelerin tetkik edilmeden unutulmasını önlemek amacıyla kullanılır. C++20 standardıyla birlikte uyarının nedenini ve yönlendirmesini açıklayan isteğe bağlı bir metin desteği eklenmiştir.

```cpp
[[nodiscard("Hata kodunun tetkik edilmesi zorunludur")]]
int dosya_ac() {
    return -1; // başarısız işlem
}

void calistir() {
    dosya_ac(); // Derleyici uyarısı: dönüş değeri göz ardı edildi
}
```

Uç durum: Dönüş değeri `(void)dosya_ac();` biçiminde açıkça `void` türüne dönüştürüldüğünde derleyici uyarısı kasten susturulur ve değer tetkik edilmeden göz ardı edilebilir.

### [[maybe_unused]]

Tanımlanmış ancak kodun belirli dallarında veya konfigürasyonlarında kullanılmayan değişken, parametre veya fonksiyonlar için derleyicinin ürettiği kullanılmayan varlık uyarısını bastırır. Yalnızca hata ayıklama modunda çalışan ya da platforma bağlı kodlarda uyarıları temiz tutmak için tercih edilir. Ön işlemci blokları arasına `#ifdef` koyma ihtiyacını ortadan kaldırarak kodun okunabilirliğini artırır.

```cpp
void kayit_dus([[maybe_unused]] int hata_kodu) {
#ifdef DEBUG
    hata_yazdir(hata_kodu);
#endif
    // DEBUG kapalıyken 'unused parameter' derleyici uyarısı engellenir
}
```

Uç durum: Gerçekten değer atanıp kullanılması gereken bir değişkenin unutulması durumunda `[[maybe_unused]]` tüm uyarıları susturacağı için mantıksal hatalar statik analizden kaçabilir.

### [[fallthrough]]

`switch-case` bloklarında bir durumdan diğerine bilerek `break` koyulmadan geçildiğini derleyiciye bildirir. Derleyicilerin eksik `break` durumunda ürettiği kaza eseri düşme (fall-through) uyarılarını kapatır. Akışın bilinçli bir tasarım tercihi olduğunu açıkça belgeleyerek kod incelemelerinde olası belirsizlikleri giderir.

```cpp
switch (durum) {
    case 1:
        on_hazirlik_yap();
        [[fallthrough]]; // sonraki case'e bilinçli geçiş; uyarıyı engeller
    case 2:
        asıl_islemi_yurut();
        break;
}
```

Uç durum: `[[fallthrough]]` özniteliği yalnızca bir `case` veya `default` etiketinden hemen önceki son ifade olarak konumlandırılabilir; aksi takdirde derleme hatası verir.

### [[deprecated]]

Bir fonksiyonun, sınıfın, türün veya şablonun kullanım ömrünün dolduğunu ve gelecekte kaldırılacağını belirtir. İlgili varlık kod içinde çağrıldığında veya referans verildiğinde derleyici geliştiriciye uyarı verir. C++14 ile standartlaşan bu yapı, isteğe bağlı bir açıklama metni alarak kullanıcının hangi yeni API'ye yönelmesi gerektiğini bildirebilir.

```cpp
[[deprecated("Eski arayüz; guvenli_oku() fonksiyonunu kullanın")]]
void eski_oku();

void calistir() {
    eski_oku(); // Derleyici uyarısı: 'eski_oku' kullanımdan kaldırıldı
}
```

Uç durum: Şablon uzmanlaşmalarında ana şablon `[[deprecated]]` ile işaretlenmiş olsa bile özelleştirilmiş şablonlar bu özniteliği otomatik miras almayabilir.

### [[noreturn]]

Fonksiyonun normal yürütme akışıyla çağırıcıya asla geri dönmeyeceğini derleyiciye bildirir. `std::terminate()`, `exit()` çağıran veya sonsuz döngüde çalışan fonksiyonlarda yığın çerçevesi (stack frame) ve dönüş kodu temizliği üretilmesini önleyerek optimizasyon sağlar. Bu fonksiyonların çağrıldığı noktalardan sonra gelen ulaşılamaz kod uyarılarını susturur.

```cpp
[[noreturn]] void olumcul_hata_ver() {
    sistem_gunlugu_yaz();
    std::terminate(); // fonksiyon asla normal geri dönüş yapmaz
}
```

Uç durum: `[[noreturn]]` ile işaretlenen bir fonksiyon `return` işletir veya gövdenin sonuna ulaşırsa tanımsız davranış oluşur.

### [[carries_dependency]]

Çok iş parçacıklı bellek modelinde `std::memory_order_consume` ile okunan bağımlılık ağacının fonksiyon sınırları üzerinden taşındığını derleyiciye bildirir. Zayıf bellek modellerine sahip mimarilerde fonksiyon çağrısı sırasında gereksiz bellek bariyerleri üretilmesini engeller. Bağımlılık zincirinin veri yolu üzerinden korunduğunu garanti ederek eşzamanlı okuma başarımını artırır.

```cpp
[[carries_dependency]] int* bagimlilik_aktar([[carries_dependency]] int* p) {
    return p; // işaretçiye bağlı bellek sırası korunur
}
```

Uç durum: Güncel derleyicilerin büyük bölümü `std::memory_order_consume` yönergesini içsel olarak `memory_order_acquire` seviyesine terfi ettirdiği için bu özniteliğin donanım seviyesindeki başarım etkisi sınırlı kalabilir.

### [[indeterminate]]

C++26 ile dile eklenen bu öznitelik, başlatılmamış (uninitialized) yerel değişkenlerin kasıtlı olarak o durumda bırakıldığını derleyiciye bildirir. Güvenlik odaklı statik analiz araçlarının veya derleyicilerin tanımsız değer okumalarına yönelik ürettiği uyarıları susturur. Harici bir donanım kesmesiyle veya işletim sistemi çağrısıyla doldurulacak büyük bellek tamponlarında gereksiz sıfırlama maliyetini önler.

```cpp
void tampon_isleme() {
    [[indeterminate]] unsigned char tampon[1024]; // bilerek başlatılmadı
    donanimdan_tampon_oku(tampon, 1024);
}
```

Uç durum: `[[indeterminate]]` ile işaretlenmiş bellek alanı değer atanmadan okunursa doğrudan tanımsız davranış üretir; öznitelik yalnızca statik uyarıları susturur, çalışma zamanı güvenliği sağlamaz.
