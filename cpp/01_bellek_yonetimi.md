# Bellek Yönetimi

### Small String Optimization

Kısa metinlerde heap tahsisini önlemek için karakterler nesnenin kendi gövdesindeki 24 baytlık (MSVC için 32 bayt) alanda saklanır. 64-bit mimaride dinamik moddaki üç alan (`char* veri`, `size_t boyut`, `size_t kapasite`) toplam 24 bayt kapladığından bir union ile yerel tampona dönüştürülür. Çalışma modunun kararı derleyiciler arasında şu akışla verilir:

- **GCC (libstdc++):** Nesne içindeki `veri` işaretçisi varsayılan olarak doğrudan nesnenin kendi gövdesindeki yerel dizi adresini (`&yerel_tampon[0]`) gösterir; boyut 15 karakteri aştığında işaretçi heap adresine yönlendirilir, böylece mod kararı için ek bir bayrak tetkiki yapılmaz (branchless).
- **Clang (libc++):** 24 baytın tamamı union içinde tutulur ve son baytın en anlamsız biti (LSB) bayrak olarak ayrılır; bu bit `0` ise kalan baytlar doğrudan karakter dizisi ve uzunluk olarak yorumlanır (22 karaktere kadar sığar), bit `1` ise alanlar işaretçi, boyut ve kapasite olarak okunur.
- **MSVC STL:** Büyüklükten tasarruf yerine basitliği seçerek 16 baytlık yerel tamponu boyut ve kapasite üyelerinden ayırır (toplam 32 bayt); `kapasite < 16` ise veri yerel tamponda, aksi halde işaretçinin gösterdiği heap alanında işlem görür.

```cpp
std::string kisa = "merhaba";   // 7 bayt: heap tahsisi yapılmaz, nesne içine yazılır
std::string uzun(50, 'x');      // tampon aşıldı: heap tahsisi yapılır

// libstdc++ ve MSVC için 15, libc++ için 22 karaktere kadar SSO geçerlidir
const char* p = kisa.data();    // doğrudan nesnenin yığın (stack) adresini gösterir
```

Uç durum: Taşıma işleminde heap tabanlı büyük string'ler yalnızca işaretçi devrederek O(1) sürede taşınırken, SSO modundaki kısa string'lerin karakter dizisi bayt bayt kopyalanmak zorundadır ve taşıma sonrası işaretçiler eski adrese bağlı kalıp geçersizleşebilir.

### [[no_unique_address]]

C++ dilinde her nesne bellekte en az 1 bayt yer kaplamak ve benzersiz bir adrese sahip olmak zorundadır. Durumsuz boş sınıflar veya ayırıcılar kompozisyon yoluyla sınıf içine üye eklendiğinde gereksiz bellek kullanımına ve dolgu baytlarına yol açar. `[[no_unique_address]]` özniteliği, ilgili boş üyenin ayrı bir adrese ihtiyaç duymadığını derleyiciye bildirerek üyenin 0 bayt yer kaplamasını sağlar. Böylece sınıf yerleşimi sıkılaşır ve boş taban sınıf optimizasyonu (EBO) kalıtıma ihtiyaç duyulmadan kompozisyon ile uygulanır.

```cpp
struct BosKural {}; // boyutu en az 1 bayt

struct Hizmet {
    int veri;                             // 4 bayt
    [[no_unique_address]] BosKural kural; // 0 bayt olarak yerleşir
}; // toplam nesne boyutu 8 bayt yerine 4 bayt olur
```

Uç durum: Aynı boş sınıf türünden iki ardışık üye `[[no_unique_address]]` ile işaretlendiğinde, dil kuralları gereği aynı türden iki alt nesne aynı adresi paylaşamayacağından derleyici ikinci üyeye en az 1 bayt yer ayırmak zorunda kalır.

