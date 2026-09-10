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
