# Senkronizasyon ve Eşzamanlılık

### Mutex ve Semaphore

Mutex, paylaşılan kritik bölgeyi tekil erişimle koruyan ve kilidi yalnızca alan thread'in bırakabildiği sahiplik tabanlı bir kilitleme mekanizmasıdır. Semaphore ise kaynak havuzunu yöneten veya iş parçacıkları arasında olay bildiren sayaç tabanlı bir sinyalleşme aracıdır. Binary semaphore sayacı 0 veya 1 olsa dahi sahiplik barındırmaz; bu sayede bir thread beklerken tamamen farklı bir thread veya kesme rutini izni serbest bırakabilir. Mutex'te ise sahiplik işletim sistemi tarafından takip edildiğinden öncelik terslenmesini önleyen priority inheritance mekanizması işletilebilir.

```cpp
std::mutex mtx;
std::binary_semaphore sem{0};

// Mutex: Yalnızca kilidi alan thread serbest bırakabilir (sahiplik kuralı)
mtx.lock();
mtx.unlock(); // Farklı thread açarsa tanımsız davranış üretir

// Binary semaphore: Sahiplik yoktur, sinyalleşme amacıyla kullanılır
sem.acquire(); // Thread B sayacı 0 yaparak olayı bekler
sem.release(); // Thread A veya kesme rutini sayacı 1 yapıp B'yi uyandırır
```

Uç durum: Binary semaphore sahiplik barındırmadığından yetkisiz bir thread veya kesme rutini tarafından serbest bırakılarak kritik bölge korumasını delebilir ve priority inheritance uygulanamadığı için öncelik terslenmesi kilitlenmesine yol açar.

### Counting Semaphore ile Yoğunluk Yönetimi

Sınırlı kapasiteye sahip kaynak havuzlarına aynı anda erişebilecek azami istek sayısını denetlemek ve aşırı yüklenmeyi önlemek için counting semaphore kullanılır. Sayaç toplam kaynak kotası kadar izinle başlatılır; her iş parçacığı `acquire` ile sayacı azaltarak kaynağı rezerve eder, kota tükendiğinde yeni çağrılar bloklanır. İşlemi biten iş parçacığı `release` çağırarak izni havuza iade eder ve sırada bekleyen iş parçacığının yürütülmesini sağlar.

```cpp
// Azami 3 eşzamanlı bağlantıya izin veren kaynak havuzu
std::counting_semaphore<3> havuz_kotasi{3};

void baglanti_kullan() {
    havuz_kotasi.acquire();   // Kota doluysa bekler, yer varsa sayacı azaltır
    // veritabanı veya ağ işlemleri yürütülür
    havuz_kotasi.release();   // İzni havuza iade eder, bekleyen thread'i uyandırır
}
```

Uç durum: İstisna fırlatıldığında `release` çağrısı işletilmezse izin kalıcı olarak tüketilir ve havuz zamanla kilitlenerek tüm yeni istekleri sonsuza kadar bloklar.

### Semaphore ile Sinyalleşme ve İletişim

İş parçacıkları arasında bir olayın tamamlandığını bildirmek veya veri aktarımını koordine etmek için başlangıç değeri sıfır olan semaphore kullanılır. Tüketici thread `acquire` çağırarak olayın gerçekleşmesini beklerken bloke olur. Üretici iş parçacığı veya bir kesme servis rutini (ISR) görevi tamamladığında `release` çağırarak sayacı artırır ve bekleyen tüketiciyi uyandırır.

```cpp
std::binary_semaphore veri_hazir{0};
std::string paylasilan_veri;

void uretici() {
    paylasilan_veri = "paket_icerigi";
    veri_hazir.release();     // Verinin hazır olduğunu tüketiciye bildirir
}

void tuketici() {
    veri_hazir.acquire();     // Veri üretilene kadar bekler
    // paylasilan_veri güvenle okunur
}
```

Uç durum: `acquire` çağrısından önce üretici tarafından art arda birden fazla `release` sinyali tetiklenirse binary semaphore sayacı en fazla 1 olacağından ara sinyaller kaybolur.
