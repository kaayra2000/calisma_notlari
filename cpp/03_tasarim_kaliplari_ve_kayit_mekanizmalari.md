# Tasarım Kalıpları

## Registry Pattern

### Explicit Registration

Bileşen veya sınıfların merkezi bir registry yapısına program akışı içinde doğrudan çağrılan bir fonksiyon ile açıkça kaydedilmesidir. Kayıt noktası belirgin ve çağrılar garantili olduğundan bağlayıcı optimizasyonları veya başlatma sırası belirsizlikleri kaydı engellemez. Ancak her yeni sınıf eklendiğinde merkezi başlatma kodunun güncellenmesi gerekmesi modülerliği ve Open-Closed prensibini zedeler.

```cpp
// main.cpp veya initialize() içinde doğrudan kayıt
Registry::get().kaydet("JSONAyristirici", []() {
    return std::make_unique<JSONAyristirici>();
});
Registry::get().kaydet("XMLAyristirici", []() {
    return std::make_unique<XMLAyristirici>();
});
```

Uç durum: Açık kayıt çağrısı program akışında unutulursa veya ilgili modülün başlatma fonksiyonu çağrılmazsa sınıf registry içinde bulunamaz.

### Self-Registration

Sınıfların merkezi bir başlatma fonksiyonuna dokunmadan, kendi kaynak dosyalarındaki global veya statik bir değişkenin başlatılması anında registry'ye kendisini eklemesidir. Yeni bir sınıfın derleme hedefine eklenmesiyle otomatik devreye giren tak-çıkar (plug-and-play) mimarisi sunar. Kayıt işlemini tetikleyen statik değişkenlerin kaynak dosyalar arasındaki başlatılma sırası garanti edilemez ve statik kütüphanelerle bağlandığında ölü kod elemesine (dead-code elimination) uğrayabilir.

```cpp
// JSONAyristirici.cpp
namespace {
    bool kaydedildi = []() {
        Registry::get().kaydet("JSONAyristirici", []() {
            return std::make_unique<JSONAyristirici>();
        });
        return true;
    }();
}
```

Uç durum: Statik başlatma sırası fiyaskosu (static initialization order fiasco) nedeniyle registry nesnesi henüz hayata gelmeden statik kayıt değişkeni çalışırsa tanımsız davranış (undefined behavior) oluşur.

