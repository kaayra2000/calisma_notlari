# Not Yazım Kuralları

Bu belgenin tek amacı, depoya eklenen her notun aynı biçimde yazılmasını sağlamaktır. Kalıp sabit kaldığı sürece depo büyüdükçe okunabilirliğini kaybetmez ve bir kavramı aramak her zaman aynı hızda olur.

## Dizin düzeni

Her programlama dili kökte kendi dizinini alır ve dizin adı dilin yaygın kısa adıdır. Dizinin içinde konu başına bir Markdown dosyası bulunur; dosya adı iki haneli sıra numarasıyla başlar ve Türkçe karakter içermeyen küçük harflerle devam eder, örneğin `03_derleme_ve_baglama.md`.

Sıra numarası kavramların öğrenilme sırasını değil, dosyaların listelenme sırasını belirler. Yeni bir konu eklendiğinde numaralar yeniden düzenlenmez, dosya sona eklenir.

Her dil dizininde bir `README.md` bulunur ve o dizindeki konu dosyalarını tek satırlık açıklamalarla listeler. Kök `README.md` yalnızca dil dizinlerine bağlantı verir, konu ayrıntısına girmez.

## Not kalıbı

Her kavram üçüncü seviye başlıkla açılır ve dört parçadan oluşur. Başlık kavramın mülakatta geçen adıdır, özet iki ile dört cümlelik bir paragraftır, kod örneği kavramı gösteren en kısa parçadır ve tuzak satırı o kavramda en sık verilen yanlış cevabı söyler.

~~~markdown
### RAII

Kaynak ömrü nesne ömrüne bağlanır; yıkıcı çalıştığı an kaynak serbest kalır.
İstisna atılsa bile yığın çözülmesi yıkıcıyı çağırdığı için sızıntı olmaz.

```cpp
std::lock_guard<std::mutex> g(m);  // kilit alındı
// kapsam biterse kilit otomatik bırakılır
```

Tuzak: `new` ve `delete` çifti elle yazılırsa aradaki `throw` sızıntı üretir.
~~~

Özet paragrafı kavramın ne olduğunu ve neden var olduğunu söyler, tarihçesini anlatmaz. Dört cümleyi aşan bir özet, kavramın ikiye bölünmesi gerektiğinin işaretidir.

Kod örneği beş ile on beş satır arasında kalır ve kavramı anlatan satırlar dışında hiçbir şey içermez. Başlık dosyaları, `main` gövdesi ve hata kontrolü gibi parçalar örneği uzatıyorsa yazılmaz. Kavram kodla değil yalnızca sözle anlatılabiliyorsa kod bloğu tamamen atlanır.

Tuzak satırı zorunludur ve her zaman `Tuzak:` ile başlar. Tek cümledir, kavramın yanlış hatırlanan yüzünü söyler. Bu satır notun mülakat değerini taşıyan parçasıdır; özet unutulsa bile tuzak akılda kalır.

## Terim kullanımı

Standart İngilizce terimler doğrudan yazılır. `move semantics`, `vtable`, `undefined behavior` gibi ifadelerin karşısına parantez içinde Türkçe çeviri konmaz, aynı şekilde Türkçe bir terimin yanına parantezle İngilizcesi eklenmez. Anlatım Türkçedir, terimler İngilizce kaldığı yerde İngilizce kalır.

Anlatımda numaralandırılmış listelerden ve yoğun madde işaretlerinden kaçınılır. Paragraf akışı tercih edilir; sıralamanın kendisi bilgi taşıyorsa, örneğin derleme aşamaları anlatılıyorsa, liste kullanılabilir.

## Komit düzeni

Her komit tek bir konuya dokunur. Bir oturumda beş kavram eklendiyse ve bunlar farklı dosyalara dağılmışsa, dosya başına ayrı komit atılır. Komit mesajları Türkçe ve küçük harfle yazılır, `gcommit` aracının `docs` tipi kullanılır.

Yeni bir konu dosyası eklendiğinde dosyanın kendisi ve ilgili `README.md` güncellemesi aynı komitte yer alır, böylece indeks hiçbir zaman dosyaların gerisinde kalmaz.
