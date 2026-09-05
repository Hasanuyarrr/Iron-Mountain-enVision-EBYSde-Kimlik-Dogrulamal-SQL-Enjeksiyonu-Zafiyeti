# Iron Mountain enVision EBYS'de Kimlik Doğrulamalı SQL Enjeksiyonu Zafiyeti

## Genel Bakış

**enVision EBYS**, Türkiye'de kamu kurumları, üniversiteler ve özel sektör tarafından kullanılan, resmî yazışma, evrak yönetimi, iş akışı ve elektronik imza süreçlerini yürüten ASP.NET WebForms tabanlı bir Elektronik Belge Yönetim Sistemi (EBYS) ürünüdür. Ürün **IRON MOUNTAIN ARŞİVLEME HİZMETLERİ A.Ş.** tarafından geliştirilmekte ve dağıtılmaktadır (ürün kod tabanı `CBK.enVision` ad uzayını kullanmaktadır).

Ürünün evrak listeleme/detaylı arama ekranında, istemciden gelen **sıralama sütunu parametresi** herhangi bir beyaz liste doğrulamasından geçirilmeden ve parametrik sorgu kullanılmadan doğrudan SQL sorgusunun `ORDER BY` cümlesine birleştirilmektedir.

Bu durum, sistemde **herhangi bir geçerli hesaba sahip en düşük yetkili kullanıcının** — herhangi bir yönetici yetkisi olmaksızın — veritabanı sorgusunu değiştirmesine ve kör (blind) boolean tabanlı çıkarımla veritabanının tamamını okumasına imkân tanımaktadır. Uygulamanın veritabanına bağlandığı hesabın veritabanı üzerinde **`db_owner`** rolüne sahip olması nedeniyle etki okuma ile sınırlı kalmamakta; yazma ve silme de mümkün hâle gelmektedir.

Zafiyet, **31.08.2026 – 04.09.2026** tarihleri arasında yetkili bir sızma testi kapsamında bir kurum kurulumunda **üç ayrı oturumda, üç kez bağımsız olarak** doğrulanmıştır. Bu süre zarfında kurulumun uygulama bileşenleri güncellenmiş (gömülü Telerik UI bileşen sürümü 2023.3.1010.45 → 2025.2.609.462 olarak değişmiş) olmasına rağmen zafiyet **açık kalmaya devam etmiştir**; bu da sorunun bir bileşen/bağımlılık zafiyeti değil, **ürünün kendi uygulama kodundan kaynaklanan bir tedarikçi seviyesi zafiyeti** olduğunu göstermektedir.

## Etki

Sisteme giriş yapabilen herhangi bir kullanıcı (memur, akademisyen, stajyer, sözleşmeli personel — rol ayrımı gözetmeksizin) tek bir HTTP isteğiyle sorgu mantığını değiştirebilmekte ve doğrulanmış bir çıkarım orakılı (oracle) üzerinden aşağıdakileri gerçekleştirebilmektedir:

- **Kurumun tüm resmî yazışmasının okunması.** Enjeksiyon noktası, evrak veritabanının tamamına erişimi olan bir bağlantı üzerinde çalışmaktadır. Test sırasında veritabanı adı, sürüm bilgisi, bağlanan kullanıcı, sunucu adı ve şema büyüklüğü kesin biçimde doğrulanmıştır. Aynı teknikle evrak içerikleri, kullanıcı kayıtları ve iş akışı verileri karakter karakter çıkarılabilir.
- **Kişisel verilerin ifşası.** Veritabanı şemasında adında **T.C. kimlik numarası** ve **parola** geçen sütunların varlığı doğrulanmıştır. Bu, KVKK kapsamında özel nitelikli/kimlik verisi ihlali anlamına gelmektedir.
- **Veri bütünlüğünün ihlali (yazma yetkisi).** Uygulamanın veritabanı hesabının **`db_owner`** rolüne üye olduğu; veritabanı üzerinde `CONTROL`, `SELECT`, `INSERT`, `UPDATE` ve `DELETE` izinlerinin tamamına sahip olduğu doğrulanmıştır. Buna göre saldırgan resmî evrak kayıtlarını değiştirebilir, sahte evrak ekleyebilir veya kayıtları silebilir. Bir EBYS ürününde bu, hukuken delil niteliği taşıyan kayıtların tahrifi anlamına gelir.
- **Denetim izinin silinebilmesi.** Şemada adında `LOG`/`AUDIT` geçen tabloların varlığı doğrulanmıştır. `db_owner` yetkisiyle bu tablolar da yazılabilir olduğundan, saldırgan kendi izlerini silebilir; olay müdahalesi ve adli inceleme güvenilirliği ortadan kalkar.
- **Yanal hareket yüzeyi.** Veritabanı sunucusunda uygulamanın kendi veritabanı dışında birden fazla veritabanı bulunduğu görülmüştür.

**Sınırlayıcı etken (dürüstlük notu):** Veritabanı hesabı `sysadmin` **değildir** ve `xp_cmdshell` üzerinde çalıştırma izni **bulunmamaktadır**; bağlı sunucu (linked server) tanımlı değildir. Dolayısıyla bu zafiyetten tek adımda işletim sistemi seviyesinde kod çalıştırmaya geçiş **gösterilememiştir**. Etki, veritabanı sınırı içinde tam gizlilik ve tam bütünlük kaybıdır.

Zafiyetin istismarı için özel bir araç, ayrıcalıklı hesap veya kullanıcı etkileşimi gerekmemektedir.

## Teknik Ayrıntı

Zafiyetli uç nokta, evrak listeleme ve detaylı arama ekranının arama gönderimidir:

```http
POST /enVision/DocumentModule/DOC_Documents.aspx?value=<kodlanmış sayfa parametreleri> HTTP/1.1
Host: <kurum-ebys-adresi>
Content-Type: application/x-www-form-urlencoded
Cookie: ASP.NET_SessionId=<oturum>; envision.AUTH=<oturum>

__EVENTTARGET=<arama düğmesi>&...&grdMainSORT_ID=<SIRALAMA SÜTUNU>&...
```

İstek gövdesindeki **`grdMainSORT_ID`** alanı, sonuç ızgarasının hangi sütuna göre sıralanacağını belirtir ve sunucu tarafında oluşturulan sorgunun `ORDER BY` cümlesine **doğrudan dizgi birleştirmesiyle** eklenmektedir. `ORDER BY` bağlamı parametrik sorgu ile bağlanamayan bir bağlam olduğundan, bu alanın **yalnızca bilinen sütun adlarına/indekslerine karşı beyaz listeye tabi tutulması** gerekirdi; ürün bunu yapmamaktadır.

Enjeksiyonun varlığı, alanın davranış farklarıyla parmak izi düzeyinde kanıtlanmıştır:

| Gönderilen değer | Sunucu davranışı | Anlamı |
|---|---|---|
| Geçerli sütun indeksi | Arama çalışır, sonuç ızgarası döner | Temel (baseline) davranış |
| Bir değer döndüren alt sorgu | Arama çalışır, sonuç ızgarası döner | **Alan SQL olarak yorumlanıyor** |
| Geçersiz/dizgi değer | Genel hata mesajı | Sorgu derlenmiyor |
| Sıralama yönü eki içeren değer | Genel hata mesajı | Kodun ifadenin sonuna ayrıca yön eklediğini gösterir |

Koşula bağlı olarak çalışma zamanı hatası üreten bir ifade kullanıldığında, sunucu iki **kararlı ve birbirinden net biçimde ayrılabilir** yanıt üretmektedir:

- **Koşul doğru:** Arama gerçekten çalışır, güncel sonuç kümesi render edilir.
- **Koşul yanlış:** Yanıt gövdesine, aramanın başarısız olduğunu bildiren sabit bir hata mesajı bloğu enjekte edilir ve eski/varsayılan liste gösterilir.

Bu iki durum arasındaki fark kararlı olduğundan, kör boolean tabanlı çıkarım için güvenilir bir orakıl oluşmaktadır. *(Not: yanıt boyutu oturum süresince değiştiği için sınıflandırmanın sabit bir bayt eşiğine göre değil, hata mesajı bloğunun varlığına göre yapılması gerekir; bu, testi yapan taraf için bir metodoloji notudur.)*

Bu orakıl kullanılarak, bilinen değerlerin **doğru/yanlış eşleştirmesiyle** (harf harf çıkarım yapılmadan) aşağıdaki bilgiler doğrulanmıştır:

| Doğrulanan bilgi | Değer |
|---|---|
| Veritabanı yönetim sistemi | Microsoft SQL Server 2017 (14.0.1000.169) |
| Yama seviyesi | **RTM — hiçbir CU/GDR uygulanmamış** |
| Sürüm | Standard Edition (64-bit) |
| Veritabanı adı | Uygulamaya ait ürün veritabanı |
| Bağlanan hesap | Uygulamaya özel hesap (`sa` değil) |
| Veritabanı rolü | **`db_owner` (üye)** |
| Sunucu rolleri | `sysadmin`, `dbcreator`, `securityadmin` — üye **değil** |
| Kullanıcı tablosu sayısı | 100'ün üzerinde |

Her doğrulama için hem olumlu hem olumsuz kontrol ayrı ayrı sorulmuş, `NULL` dönen ifadelerin yanlış pozitif üretmesi elenmiştir. Ayrıca yanlış sunucu adı, yanlış veritabanı adı ve yanlış rol üyeliği sorguları beklendiği gibi "yanlış" yanıtı üretmiş, orakılın geçerliliği negatif kontrollerle teyit edilmiştir.

Zafiyet hem **senkron postback** hem de **asenkron (AJAX/delta) postback** akışında aynı biçimde tetiklenebilmektedir; yani sorunu istemci tarafındaki tek bir akışı engelleyerek kapatmak mümkün değildir.

### Aynı İstekte Denenip Zafiyetli Bulunmayan Parametreler

Kapsamın doğru belirlenebilmesi için, aynı istek gövdesindeki **235 parametrenin 227'si** hem tırnak enjeksiyonu hem de boolean orakıl yöntemiyle tek tek taranmış; kalan 8 alan elle incelenmiştir. Kullanıcı kimliği, sayfa damgası, sayfalama/limit alanları, serbest metin filtre alanları ve şifreli sorgu blob'ları dâhil hiçbirinde SQL enjeksiyonu tespit edilmemiştir.

**Zafiyet münhasıran sıralama sütunu parametresindedir.** Ancak ürünün diğer modüllerindeki liste/ızgara ekranlarının aynı ortak sıralama kod yolunu kullanması kuvvetle muhtemel olduğundan, düzeltmenin tek bir sayfayla sınırlı tutulmaması gerekmektedir.

## İlişkili Zayıflıklar

- **CWE-89** — SQL Komutunda Kullanılan Özel Elemanların Uygun Olmayan Biçimde Etkisizleştirilmesi ("SQL Injection")
- **CWE-1236 / CWE-20** — Girdinin beyaz listeye tabi tutulmaması *(`ORDER BY` bağlamında tek geçerli savunma budur)*
- **CWE-250** — Gereğinden Fazla Yetkiyle Çalıştırma *(uygulamanın veritabanına `db_owner` rolüyle bağlanması)*
- **CWE-285** — Uygun Olmayan Yetkilendirme *(en düşük yetkili kullanıcının veri katmanına sınırsız erişebilmesi)*

## Şiddet

**Yüksek — CVSS 3.1 Temel Puan 8.8** (`AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`)

Puanlama; zafiyetin ağ üzerinden (`AV:N`), düşük karmaşıklıkla (`AC:L`), **yalnızca sıradan bir kullanıcı hesabıyla** (`PR:L`) ve **kullanıcı etkileşimi olmadan** (`UI:N`) istismar edilebilmesi esas alınarak yapılmıştır. Gizlilik, bütünlük ve erişilebilirlik etkilerinin tamamı **yüksek** kabul edilmiştir; bu, uygulamanın veritabanına `db_owner` rolüyle bağlandığının fiilen doğrulanmış olmasına dayanmaktadır (yalnızca varsayım değildir).

EBYS ürünlerinin kurum içinde geniş kullanıcı tabanına açık olması, "düşük ayrıcalık" ön koşulunun pratikte kurumdaki **her personel** anlamına gelmesi ve işlenen verinin niteliği (resmî yazışma, kimlik verisi, imza/iş akışı kayıtları) göz önüne alındığında, işletmeci kurumlar açısından bulgunun **kritik** olarak ele alınması önerilir.

## Etkilenen Sürümler

- **Ürün:** enVision EBYS (Elektronik Belge Yönetim Sistemi)
- **Üretici:** IRON MOUNTAIN ARŞİVLEME HİZMETLERİ A.Ş.
- **Doğrulanan sürüm:** `v8.2.d1606.b260264.c0.u0`
- **Ayrıca gözlenen sürüm:** `8.1.0.25833` (aynı kurulumda, güncelleme öncesi)

Zafiyet, kurulumun **8.1.x sürümünde tespit edilmiş ve 8.2.x sürümüne güncellendikten sonra da tetiklenebilir kalmıştır.** Sıralama parametresinin işlendiği kod yolunda değişiklik yapıldığına dair bir gösterge bulunmadığından, **8.2.d1606.b260264.c0.u0 ve öncesi tüm sürümler etkilenmiş kabul edilmelidir.** Kesin alt sınır, yalnızca üreticinin kaynak kod geçmişiyle belirlenebilir.

## Etkilenen Bileşen

Evrak listeleme / detaylı arama modülünde (`DocumentModule`), sonuç ızgarasının sıralama sütunu parametresinin (`grdMainSORT_ID`) SQL `ORDER BY` cümlesine dönüştürüldüğü sunucu tarafı kod yolu. Hem senkron hem asenkron postback akışı etkilenmektedir.

## Azaltıcı Önlemler

Üretici düzeltmesi yayımlanana kadar işletmecilerin şunları uygulaması önerilir:

1. **Veritabanı bağlantı hesabının yetkisi derhal düşürülmelidir.** Uygulama `db_owner` yerine, yalnızca ihtiyaç duyduğu tablolar üzerinde `SELECT`/`INSERT`/`UPDATE`/`DELETE` yetkisine sahip ayrılmış bir hesapla bağlanmalıdır. Bu tek adım, bütünlük ve denetim izi silme etkilerini büyük ölçüde ortadan kaldırır ve şiddeti belirgin biçimde düşürür. *(Uygulamanın şema değişikliği gerektiren güncelleme adımları varsa, bunlar ayrı ve yalnızca güncelleme sırasında kullanılan bir hesapla yürütülmelidir.)*
2. Ters vekil sunucu (reverse proxy) veya WAF katmanında, ilgili isteğin gövdesindeki sıralama parametresi denetlenmeli; **yalnızca sayısal değerlere izin verilmeli**, diğer tüm değerler reddedilmelidir. Bu, kalıcı çözümün yerine geçmez ancak beyaz liste tabanlı olduğu için etkili bir geçici önlemdir.
3. Veritabanı sunucusu güncel yama seviyesine getirilmelidir; tespit edilen kurulumda **SQL Server 2017 RTM** kullanılmakta ve yayımlanmasından bu yana hiçbir kümülatif güncelleme uygulanmamış görünmektedir.
4. Uygulama ve web sunucusu günlükleri, ilgili uç noktaya gelen olağandışı yoğunluktaki arama istekleri açısından geriye dönük incelenmelidir; kör çıkarım saldırıları yüzlerce-binlerce benzer istek üretir ve belirgin iz bırakır. Aynı incelemede arama hata mesajının anormal sıklıkta üretilip üretilmediğine bakılmalıdır.
5. İstismar izine rastlanması hâlinde, uygulama veritabanının tamamı ihlal edilmiş kabul edilmeli; evrak bütünlüğü doğrulanmalı ve KVKK kapsamında değerlendirme yapılmalıdır.
6. Veritabanı sunucusu yalnızca uygulama sunucusundan erişilebilir olmalı, ağ seviyesinde yalıtılmalıdır.

Kalıcı çözüm için üreticinin şunları yapması gerekmektedir:

- Sıralama parametresi **beyaz listeye** tabi tutulmalıdır. Kabul edilen değerler, sunucu tarafında tanımlı sütun listesindeki bir indeks veya sabit bir eşleme anahtarı olmalı; istemciden gelen dizgi hiçbir koşulda sorguya birleştirilmemelidir. `ORDER BY` bağlamı parametrik sorguyla korunamaz; kaçış (escaping) tabanlı savunma bu bağlamda **yetersizdir**.
- Sıralama yönü de aynı biçimde iki sabit değerden birine eşlenmelidir.
- Ürün genelinde **aynı ızgara/sıralama altyapısını kullanan tüm modüller** gözden geçirilmeli, düzeltme tek bir sayfayla sınırlı tutulmamalıdır.
- Kurulum dokümantasyonu ve kurulum betikleri, uygulamanın veritabanına `db_owner` rolüyle bağlanmasını öngörmeyecek biçimde güncellenmeli; en az ayrıcalık ilkesine uygun ayrılmış bir veritabanı hesabı oluşturulmalıdır.
- Sunucu tarafı hata yakalama korunmalıdır: mevcut kurulumda yığın izi (stack trace) sızdırılmaması olumlu bir uygulamadır; ancak hata/başarı ayrımının kendisi bir orakıl oluşturduğundan, asıl çözümün girdi doğrulaması olduğu unutulmamalıdır.

## Sorumlu Açıklama Notu

Bu doküman, zafiyetin tetiklenmesine ait somut yük (payload) ve istismar kodlarını **bilinçli olarak içermemektedir**; yalnızca zafiyetin sınıfı, konumu ve etkisi tanımlanmıştır. Ham istek/yanıt çiftleri ve doğrulama betikleri, koordineli açıklama süreci kapsamında üretici ve ilgili ulusal siber olaylara müdahale ekibiyle (USOM/TR-CERT) doğrudan paylaşılmıştır.

Testler, ilgili kurumun yazılı izniyle yürütülen yetkili bir sızma testi kapsamında gerçekleştirilmiştir. Veri çıkarımı, zafiyetin varlığını ve yetki seviyesini kanıtlamaya yetecek asgari düzeyde (sürüm, veritabanı adı, bağlanan hesap, rol üyeliği, sayaç bilgileri) tutulmuş; **kişisel veri içeren tabloların içeriği çekilmemiş, hiçbir kayıt değiştirilmemiş veya silinmemiştir.** Yazma yetkisi, veri değiştirilerek değil yalnızca izin sorgulanarak doğrulanmıştır.

## CVE Kimliği

Henüz atanmadı — CVE talebi beklemede

## Bulan

Hasan Hüseyin UYAR – Netlore Security

## Açıklama Takvimi

| Tarih | Olay |
|---|---|
| 2026-08-28 | Hedef uygulamada ilk keşif ve kimlik doğrulama yüzeyinin incelenmesi |
| 2026-08-31 | Sıralama parametresindeki SQL enjeksiyonu tespit edildi ve doğrulandı; veritabanı yetki seviyesi (`db_owner`) teyit edildi |
| 2026-09-01 | Zafiyet yeni bir oturumda ve asenkron postback akışında yeniden doğrulandı |
| 2026-09-04 | Ürün sürümü `v8.2.d1606.b260264.c0.u0` üzerinde üçüncü kez doğrulandı; zafiyet hâlâ açık |
