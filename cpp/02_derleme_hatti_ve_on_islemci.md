# Derleme Hattı ve Ön İşlemci

### Ön İşleme

Kaynak kod derleyiciye girmeden önce `#` ile başlayan tüm ön işlemci direktifleri metin düzeyinde işlenir. Başlık dosyalarının içerikleri koda yapıştırılır, makrolar genişletilir ve koşullu derleme blokları süzülerek saf C++ metni (`.i` / `.ii`) üretilir. Dosya bulunamama (`file not found`), eksik kapatılmış makro veya yinelenen başlık sorunları bu aşamada tetkik edilir; sözdizimi ve tip denetimi henüz yapılmaz.

```cpp
// g++ -E main.cpp -o main.i
#include <iostream>
#define SURUM 2

#if SURUM > 1
    int surum_no = SURUM;
#endif
```

Uç durum: Başlık dosyasında include guard (`#pragma once` veya `#ifndef`) unutulursa döngüsel başlık eklemelerinde ön işlemci sonsuz özyinelemeye girerek dosya sınırını aşar ve hata verir.

### Derleme

Ön işlemden geçen saf metin, derleyici ön yüzü tarafından sözdizimsel ve anlamsal olarak incelenerek hedef mimariye ait Assembly koduna (`.s`) dönüştürülür. Şablonlar açılır (instantiation), tür uyumluluğu doğrulanır ve kapsam (scope) tetkiki tamamlanır. Sözdizimi hataları (`syntax error`), tür uyuşmazlığı (`type mismatch`), bildirilmemiş tanımlayıcılar ve `const` ihlalleri bu aşamada yakalanır.

```cpp
// g++ -S main.i -o main.s
template <typename T>
T topla(T a, T b) {
    return a + b;  // T tipi '+' işlecini desteklemezse derleme hatası üretir
}
```

Uç durum: `template` tanımlarındaki anlamsal ve tür hataları, şablon somutlaştırılana (instantiation) kadar derleyici tarafından tetkik edilmeyip gizli kalabilir.

### Çevirme

Assembler, önceki aşamada üretilen metin tabanlı Assembly komutlarını işlemcinin doğrudan yürütebileceği amaç koduna (`.o` / `.obj`) dönüştürür. Makine talimatları ikili düzende kodlanır ve yerel sembol tabloları inşa edilir; ancak başka dosyalara ait fonksiyon ve veri adresleri henüz çözümlenmediğinden yer tutucu (offset) olarak bırakılır. Yalnızca geçersiz Assembly sözdizimi veya hedef mimaride desteklenmeyen makine komutları bu aşamada hata üretir.

```cpp
// g++ -c main.s -o main.o
// C++ kodundan derleyicinin ürettiği satır içi assembly veya saf çeviri:
__asm__("movl $10, %eax");  // geçerli işlemci komutu makine koduna dönüştürülür
```

Uç durum: Satır içi Assembly (`inline assembly`) yazarken hedef mimarinin desteklemediği bir yazmaç (register) veya geçersiz buyruk kullanılırsa hata ancak bu çevirme aşamasında yakalanır.

### Bağlama

Bağlayıcı (linker), bağımsız derlenmiş amaç dosyalarını (`.o`) ve harici kütüphaneleri birleştirerek tek bir çalıştırılabilir ikili (`.exe` / ELF) inşa eder. Dosyalar arasındaki harici sembol referansları kesin bellek adresleriyle eşlenir. Tanımsız referans (`undefined reference`) veya mükerrer sembol tanımları (`multiple definition`) bu safhada yakalanan temel bağlayıcı hatalarıdır.

```cpp
// g++ main.o diger.o -o main.exe
extern void harici_islem();  // bildirildi fakat gövdesi hiçbir .o içinde yoksa:

int main() {
    harici_islem();          // linker 'undefined reference to harici_islem' hatası verir
    return 0;
}
```

Uç durum: Bir fonksiyon gövdesi başlık dosyasında `inline` anahtar kelimesi olmaksızın tanımlanıp birden fazla kaynak dosyaya dahil edilirse, bağlama aşamasında "multiple definition" çakışma hatası üretir.

### #ifdef ve Koşullu Derleme

Ön işlemci seviyesinde çalışan `#ifdef`, belirtilen makro tanımlıysa ilgili kod bloğunu derleme hattına dahil eder, aksi halde metin seviyesinde tamamen ayıklar. Henüz C++ derleyicisi ve tip denetimi devreye girmeden çalıştığından elenen bloklar ikili dosyada yer kaplamaz ve çalışma zamanı maliyeti oluşturmaz. Platform bağımlı API çağrılarını ayırmak veya hata ayıklama kodlarını derlemeden soyutlamak için kullanılır.

```cpp
#ifdef _WIN32
    windows_ozel_api_cagrisi();
#elif defined(__linux__)
    posix_ozel_sistem_cagrisi();
#endif
```

Uç durum: `#ifdef` blokları içinde kalan sözdizimi hataları, ilgili makro tanımlanmadığı sürece ön işlemci tarafından atlandığı için derleyici tarafından hiçbir zaman tetkik edilemez.

### if constexpr ve Derleme Zamanı Mantığı

C++17 ile gelen `if constexpr`, şablonlarda veya derleme zamanı sabitlerinde dallanma yapılmasını sağlarken kodun tip güvenliğini ve sözdizimi geçerliliğini korur. Sağlanmayan dal derlenmiş ikiliden çıkarılır ancak ön işlemcinin aksine derleyici tarafından bütünüyle ayrıştırılıp tetkik edilmeye devam eder. Bu sayede `#ifdef` yönergelerinin yol açtığı tip güvensizliği ve kapsam dışı metin karmaşası önlenir.

```cpp
constexpr bool HizliAlgoritmaKullan = true;

if constexpr (HizliAlgoritmaKullan) {
    hizli_hesapla();
} else {
    standart_hesapla();
}
```

Uç durum: `if constexpr` bloğunun çalışmayan dalındaki kod derlenmese bile sözdizimsel olarak geçerli olmak zorundadır; geçersiz bir ifade derleme hatasına yol açar.

### --whole-archive

Bağlayıcıya, belirtilen statik kütüphanedeki tüm nesne dosyalarını (`.o`) dışarıdan doğrudan sembol referansı olmasa dahi nihai ikiliye zorla dahil etmesini bildirir. Standart bağlayıcı davranışı, statik kütüphaneden yalnızca doğrudan çağrılan sembollerin dosyalarını çekip referans verilmeyen dosyaları ölü kod elemesiyle (dead-code elimination) tamamen ayıklar. Dosya içi anonim alanda statik değişkenle çalışan self-registration mekanizmaları bu bayrak olmadan statik kütüphaneye alındığında elenir.

```cpp
// CMakeLists.txt veya bağlayıcı komutu:
// g++ main.o -Wl,--whole-archive -lEklentiler -Wl,--no-whole-archive -o uygulama
target_link_libraries(uygulama PRIVATE
    -Wl,--whole-archive Eklentiler -Wl,--no-whole-archive
)
// Modern CMake (3.24+): target_link_libraries(uygulama PRIVATE "$<LINK_LIBRARY:WHOLE_ARCHIVE,Eklentiler>")
```

Uç durum: `--whole-archive` sonrasında `--no-whole-archive` bayrağı ile normal tarama moduna dönülmezse sonraki tüm kütüphaneler de gereksiz yere bütünüyle ikiliye gömülerek dosya boyutunu ve çakışma riskini artırır.

### Derleme Zamanı Hata Tetkiki

Kaynak koda yazılan anlamsız metinler, sözdizimi kurallarına aykırı yapılar ve bildirilmemiş tanımlayıcılar doğrudan derleme aşamasında sözdizimsel ve anlamsal analiz ön yüzü tarafından yakalanır. Hata tespit edildiğinde derleme hattı anında kesilir ve ikili dosya üretimi reddedilerek hatalı kodun çalışma zamanına geçmesi engellenir. Ön işlemci yalnızca metin manipülasyonu yaptığından bu anlamsal tetkikleri gerçekleştiremez; dil kurallarının geçerliliği derleyicinin soyut sözdizimi ağacı inşası sırasında tetkik edilir.

```cpp
void islem() {
    asdfkkjlasdlkjsaf;      // Hata: bildirilmemiş tanımlayıcı (undeclared identifier)
    int sayi = "metin";     // Hata: geçersiz tür dönüşümü
}
```

Uç durum: `#ifdef` ile elenen kod blokları veya somutlaştırılmayan şablon gövdelerindeki anlamsal hatalar derleme hattına girmediği için derleme zamanında yakalanamaz.

