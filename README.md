<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/5dca15b9-7495-4dca-8b54-8f9fbd40ee90" />

# DDD (Domain-Driven Design) Nedir ve Core Prensipleri Nelerdir?

DDD, karmaşık yazılım sistemlerini yönetmek için ortaya çıkmış bir mimari metodoloji. Temelde yaptığı şey şu: teknik tarafta yazılan kodu, iş tarafının (business) kullandığı ortak dille (ubiquitous language) uyumlu hale getirmek. Günümüzde genellikle Event-Driven Architecture ile birlikte anılıyor, mikroservis dünyasında sık karşımıza çıkıyor. Temel motivasyon, devasa ve tek parça bir veri şeması yerine net sınırlar çizip karmaşıklığı izole etmek.

Core prensipler:

1. **Ubiquitous Language:** Kod tabanı, iş mantığı ve dokümantasyon hepsi aynı kelimelerle yazılır. Kod ile dokümanlar arasında kopukluk oluşmaz, projeye yeni katılan biri iş mantığını daha hızlı kavrar.

2. **Bounded Context:** Karmaşıklığı yönetmenin en kritik aracı. Bir modelin ve terimlerinin geçerli olduğu sınırı çizer. Bu sınırın dışına çıkıldığında aynı terim tamamen farklı bir şey ifade edebilir. Mesela "Product" kavramı Catalog tarafında fiyat, stok, kategori bilgisiyle dolu bir nesneyken, Order tarafında sadece sipariş satırına referans veren basit bir kimliğe dönüşebilir.

3. **Context Mapping:** İki Bounded Context'in birbiriyle nasıl ilişkilendiğini, hangi yöntemle (shared kernel, customer supplier, conformist vb.) eşlendiğini gösteren teknik.

4. **Strategic / Tactical Ayrımı:** DDD iki ana aşamadan oluşur. Sistemin büyük resmi, sınırları, high level mimarisi çizilirken Strategic araçlar (Bounded Context, Context Mapping) kullanılır. Sınırlar netleştikten sonra koda inip domain logic yazmak gerektiğinde Tactical araçlar (Aggregate, Entity, Value Object, Repository) devreye girer.

5. **Domain-Centric Persistence:** Domain modeli ile persistence modeli birbirinden ayrılmalı. İş kurallarını taşıyan sınıflar ile veritabanına yazılan sınıflar aynı sınıf olmak zorunda değil.

6. **Continuous Refinement:** Domain model bir kere çizilip kilitlenen statik bir yapı değil, yaşayan bir yapı (living artifact). Business kuralları değiştikçe, domain expert ile yapılan her görüşme, production'da çıkan her edge case, yeni gelen bir regülasyon  bunların hepsi modelin eksik veya hatalı olduğunu fark ettiren birer an olabilir. O noktada kod, yeni anlayışı yansıtacak şekilde refactor edilir. Model ve kod birlikte evrilir.

## Tactical Pattern

Sistemin business detaylarına indiğimiz kısım.

**1. Entities**

Identity'si olan, eşitlik karşılaştırması bu identity üzerinden yapılan nesneler. Durumları zaman içinde değişse de kimlikleri sabit kalır, bu yüzden genelde mutable tasarlanırlar. Identity için UUID, sıralı bir ID veya TC kimlik no gibi natural key kullanılabilir. Son zamanlarda UUIDv7 kullanımı yaygınlaştı, zaman sıralı olduğu için index performansı açısından avantaj sağlıyor  ama bu DDD'nin bir kuralı değil, pratikte sık rastlanan bir mühendislik tercihi sadece.

**2. Value Objects**

Entity'den farkı: identity'leri yok, immutable'lar. Eşitlikleri kimlik üzerinden değil, taşıdıkları değer üzerinden belirlenir.

**3. Aggregates**

İş kurallarını (invariant) ve veri bütünlüğünü korumak için bir araya gelmiş Entity ve Value Object grubu. Bu grup her zaman tutarlı (consistent) kalır. Dış dünyayla tek bir giriş noktası vardır, buna Aggregate Root denir. Sistemdeki tüm durum değişiklikleri ve veritabanı işlemleri bu transactional boundary gözetilerek grup halinde yapılır.

**4. Domain Events**

Bounded Context'lerin veya modüllerin birbirine sıkı sıkıya bağımlı olmadan haberleşmesini sağlar. Domain'de gerçekleşen kritik bir state değişikliğini sisteme bildirmek için kullanılır. Zaten gerçekleşmiş bir olayı temsil ettiği için isimlendirme her zaman geçmiş zamanla yapılır: `OrderCreatedEvent`, `ProductCreatedEvent`, `AccountCreatedEvent` gibi. Suffix konvansiyonu projeden projeye değişebilir, `Event` eki olmadan da kullanılabilir  önemli olan tutarlı bir isimlendirme seçip ona sadık kalmak.

```java
@Component
public class EventPublisher {
    private final ApplicationEventPublisher publisher;

    public EventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void publish(DomainEvent event) {
        publisher.publishEvent(event);
    }
}

public abstract class DomainEvent {
    private final LocalDateTime occurredOn;

    protected DomainEvent() {
        this.occurredOn = LocalDateTime.now();
    }

    public LocalDateTime getOccurredOn() {
        return occurredOn;
    }
}
```

Dikkat edilmesi gereken bir nokta var: Spring'in `ApplicationEventPublisher`'ını yukarıdaki gibi kullanırsan, event publish işlemi varsayılan olarak senkron çalışır  aynı thread'de, aynı transaction içinde. Event'i fırlatan kod, listener bitene kadar bloklanır. Gerçek anlamda asenkron istiyorsan listener metoduna `@Async` ekleyip `@EnableAsync`'i açman gerekir, ya da `@TransactionalEventListener(phase = AFTER_COMMIT)` ile event'i transaction commit'inden sonraya erteleyebilirsin. Farklı bir servise veya bounded context'e gerçek anlamda asenkron ulaşmak gerekiyorsa tipik çözüm Outbox Pattern + mesaj kuyruğu (Kafka gibi): event önce bir outbox tablosuna yazılır, sonra bir scheduler bunu kuyruğa publish eder.

**5. Repositories**

Domain ile persistence katmanı arasında bir abstraction kurar. Domain layer, verinin SQL mi NoSQL mi olduğunu, hangi ORM kullanıldığını, query'nin nasıl yazıldığını bilmez  sadece "bana bu aggregate'i getir" veya "bunu kaydet" der. Repository, aggregate'leri sanki in-memory bir koleksiyonmuş gibi sunan bir interface. Gerçek implementasyon (SQL, NoSQL document mapping, cache vb.) domain'in arkasında, infrastructure katmanında saklanır. (Bkz. Eric Evans, DDD, Repository Pattern.)

**6. Domain Services**

Stateless operasyonlar. Tek bir Entity veya Value Object'e doğal olarak ait olmayan, genelde birden fazla Aggregate arasında koordinasyon gerektiren domain logic'i temsil eder. Application Service'ten farkı: Application Service transaction, security, use case orchestration gibi konularla ilgilenir; Domain Service gerçek business rule içerir ve ubiquitous language ile ifade edilir.

## Domain Model ve Data Model Farkı

Domain Model, sistemin iş kurallarını ve davranışlarını temsil eder. Data Model ise bu verinin veritabanında nasıl saklanacağını gösteren, sadece veri taşıyan bir yapı. Aradaki temel fark: davranış ile veri birbirinden ayrılmış.

**Domain Model:**

```java
public class Order extends AggregateRoot<OrderId> {
    private final CustomerId customerId;
    private final Money price;
    private final List<OrderItem> items;
    private OrderStatus orderStatus;

    public void pay() {
        if (orderStatus != OrderStatus.PENDING) {
            throw new OrderDomainException("Order is not in correct state for pay operation!");
        }
        orderStatus = OrderStatus.PAID;
    }

    public void validateOrder() {
        validateInitialOrder();
        validateTotalPrice();
        validateItemsPrice(); // toplam fiyat ile satır toplamları eşleşmeli
    }
    // ... approve(), cancel(), initCancel() hepsi state geçişini kendi içinde kontrol ediyor
}
```

**Data Model (Spring Data JPA örneği, infrastructure/persistence katmanı):**

```java
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
@Table(name = "orders")
@Entity
public class OrderEntity {
    @Id
    private UUID id;
    private UUID customerId;
    private BigDecimal price;
    @Enumerated(EnumType.STRING)
    private OrderStatus orderStatus;
    private String failureMessages;

    @OneToOne(mappedBy = "order", cascade = CascadeType.ALL)
    private OrderAddressEntity address;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItemEntity> items;
}
```

İlk sınıfta `pay()` metodu kendi kuralını kendi içinde kontrol ediyor, ikincide tüm alanlar setter ile dışarıdan serbestçe değiştirilebiliyor. Bu fark bizi Anemic ve Rich Model ayrımına götürüyor.

## Anemic ve Rich Model Nedir?

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/4b3fcd52-08c8-410e-aa0f-7d09a105f771" />

Rich Domain Model, iş kurallarını kendi içinde koruyan, davranış barındıran model. Nesne üzerindeki her durum değişikliği kendi metotları üzerinden gerçekleşir, bu metotlar çalışmadan önce ilgili kuralı kontrol eder. Yukarıdaki `Order` sınıfı tam olarak bunu yapıyor: `pay()` metodu, sipariş PENDING durumda değilse ödemeyi reddediyor.

Anemic Model davranışsız  sadece get/set içerir, hiçbir iş kuralı taşımaz. Genelde persistence katmanında (JPA Entity, DTO) görülür, bilinçli olarak bu şekilde tasarlanır. Yukarıdaki `OrderEntity` buna örnek; setter'ları var ama hiçbir kontrolü yok.

Burada bir ayrım yapmak lazım: Anemic Model'in kendisi anti-pattern değil. Martin Fowler'ın eleştirdiği şey, iş kuralının hiçbir yerde rich bir şekilde yaşamaması, kuralın servis sınıflarına dağılmış if-else yığınlarına dönüşmesi. Persistence katmanının anemic olması (`OrderEntity`) ile domain katmanının rich olması (`Order`) gayet rahat bir arada bulunabilir. DDD'nin "Domain-Centric Persistence" prensibi zaten tam olarak bunu söylüyor.

| | Rich Model | Anemic Model |
|---|---|---|
| Davranış | Var (`pay()`, `approve()`) | Yok (sadece get/set) |
| Kural koruması | Nesnenin kendi içinde | Yok |
| Tipik olarak bulunduğu yer | Domain Layer | Persistence / Infrastructure Layer |

### Martin Fowler ve Eric Evans Anemic Domain Model'i Neden Anti Pattern Olarak Tanımlıyor?

Fowler bu konuyu Eric Evans ile konuştuktan sonra, anemic model'in giderek yaygınlaştığını ve bunun gerçek bir nesne modeli savunan biri için iyi bir gelişme olmadığını yazmıştı. Sebepleri:

**1. Encapsulation İhlali**

OOP'nin temel fikri, veriyi ve o veri üzerinde çalışan davranışı bir arada tutmak (encapsulation). Fowler, Anemic Model'in bunu tam tersine çevirdiğini, ortaya prosedürel bir tasarım çıktığını söylüyor  nesne yönelimli programlamanın savunucularının uzun süredir karşı durmaya çalıştığı yaklaşım tam olarak bu. Anemic Domain Model kullanan bir sistem görünüşte sınıflardan oluşur ama gerçekte C'deki struct + fonksiyon mantığından farksız. 1970'lerin prosedürel tasarım sorunları, 2000'lerin nesne yönelimli kod tabanında yeniden ortaya çıkıyor.

**2. Duplication**

İş mantığı entity'den ayrılıp service katmanına dağıldığında, aynı kuralın birden fazla yerde tekrar yazılması riski artar. Kuralın doğal sahibi olan entity bu mantığı barındırmadığı için her servis aynı kontrolü kendi içinde yeniden yazmak zorunda kalabilir.

**3. Yanıltıcı Abstraction**

Anemic Model dışarıdan OOP gibi görünür ama özünde prosedürel. DDD'de bir domain'e baktığımızda business kuralları ve davranış bekleriz, oysa burada içi boş bir abstraction var. Sınıflar var ama davranış yok.

**4. Domain Model'in Anlamını Kaybetmesi**

Eric Evans'ın asıl kaygısı biraz daha derin. DDD'de modelin amacı, iş alanının gerçek kurallarını ve davranışlarını kod içinde temsil etmek. `Order` sınıfı sadece veri tutuyorsa kod artık iş alanını temsil etmiyor, sadece bir veri şeması temsil ediyor. O zaman "domain model" terimi anlamsızlaşır  ortada gerçek bir model yok, veritabanı tablolarının nesnelere bire bir yansıması var.

**5. Test Edilebilirlik ve Bakım Kolaylığı Yanılsaması**

Bazıları Anemic Model'i "servis katmanını test etmek daha kolay" diye savunur ama Fowler'a göre bu yanlış bir kazanç algısı. Mantık merkezi bir yerde toplanmadığı için, değişen bir iş kuralının etkilediği tüm servisleri bulup güncellemek gerekir. Bu bakım maliyetini düşürmez, servisler büyüdükçe tam tersine artırır.

## Bounded Context Nedir ve Neden Önemli?

Bounded Context, domain modellerimizin semantik sınırını çizer. Bir Bounded Context içinde kullanılan terimler, kurallar ve modeller tutarlı ve net bir anlam taşır (ubiquitous language). Aynı terim farklı bir Bounded Context'e geçildiğinde tamamen farklı bir anlam veya yapı taşıyabilir. Bu bir hata değil  DDD'nin bilinçli olarak kabul ettiği ve yönettiği bir gerçek.

**1. Aynı Terim, Farklı Context'te Farklı Anlam**

Büyük bir sistemde tek bir "evrensel model" oluşturmaya çalışmak genelde başarısız olur, çünkü aynı kavram farklı departmanlar için farklı şeyler ifade eder. Mesela bir e-ticaret sisteminde `Customer`:

- Sales Context'te kredi limiti, sipariş geçmişi, indirim seviyesi taşır.
- Support Context'te açık ticket'ları ve iletişim tercihlerini taşır.
- Shipping Context'te sadece adres ve teslimat tercihleri önemlidir.

Bu üç `Customer` aynı sınıf gibi görünse de aslında birbirinden bağımsız modeller. Bounded Context bu modellerin birbirine karışmasını önler.

**2. God Model Riskini Engeller**

Tüm sistemi tek bir model ile temsil etmeye çalışmak zamanla devasa, herkesin her şeyi eklediği, hiçbir context'e tam uymayan bir "tanrı model" doğurur. Bounded Context her modelin kendi sınırları içinde sade ve anlamlı kalmasını sağlar.

**3. Microservice / Modül Sınırlarını Belirler**

Pratikte Bounded Context, mikroservis mimarisinde servis sınırlarını çizmek için en sık kullanılan yöntem. Her Bounded Context bağımsız bir servise (veya en azından bağımsız bir modüle) dönüşebilir, bu da gevşek bağlılık (loose coupling) sağlar.

**4. Takım Otonomisi**

Her Bounded Context'in kendi modeli, kendi veritabanı şeması, kendi terminolojisi olduğu için farklı takımlar birbirine müdahale etmeden paralel çalışabilir. Organizasyon yapısı (takımlar) ile yazılım mimarisi (context'ler) arasındaki bu örtüşme Conway's Law'ın doğal bir sonucu.

**5. Context Mapping ile Yönetilen İlişkiler**

Bounded Context'ler birbirinden tamamen kopuk değil, aralarında ilişki kurulması gerekiyor. DDD bunun için bazı pattern'ler tanımlar:

- **Shared Kernel:** İki context'in ortak, küçük bir model parçasını paylaşması.
- **Customer-Supplier:** Bir context'in çıktısının başka bir context'e girdi olması.
- **Conformist:** Bir context'in başka bir context'in modeline uyum sağlaması.
- **Anticorruption Layer (ACL):** Dış veya legacy bir sistemin modelinin kendi context'ine bozulmadan sızmasını önleyen çeviri katmanı.

**6. Belirsizliği ve İletişim Hatalarını Azaltır**

Geliştiriciler ve domain uzmanları arasındaki konuşmalarda "hangi `Order`'dan bahsediyoruz?" gibi belirsizlikler ortadan kalkar, çünkü her terim hangi context'te konuşulduğuna bağlı olarak net bir anlam taşır.

Bounded Context kısaca: "tek ve mükemmel bir model" hayalinden vazgeçip her alanın kendi gerçeğine sahip olmasına izin veren bir sınır çizme stratejisi.

## Event Storming

Event Storming, Alberto Brandolini'nin geliştirdiği, karmaşık bir iş sürecini geliştiriciler, domain expert'ler ve diğer ilgili taraflar bir arada otururken post-it'ler kullanarak kronolojik bir akış halinde modellediğimiz bir workshop tekniği. DDD'de en sık kullanılan "keşif" aracı; Bounded Context ve Aggregate sınırlarını çizmeden önce, sürecin gerçekte ne olduğunu ortaya çıkarmak için kullanılır. Mantık şu: sistemi baştan koda dökmeye çalışmak yerine, önce "sistemde zaman içinde ne oldu?" sorusuna odaklanılır. Modellemenin merkezinde Domain Event'ler (geçmiş zamanlı olaylar) olduğu için isim de buradan geliyor.

Klasik DDD pratiğinde genelde önce domain nesneleri (isimler/noun'lar) belirlenip yapı kurulmaya çalışılır. Bu bazen erken ve hatalı yapısal kararlara yol açabiliyor, çünkü domain'in gerçek süreçleri henüz tam anlaşılmadan modelleme yapılmış oluyor. Event Storming tam tersi bir yoldan gidip önce süreci ortaya çıkarmayı, yapısal kararları (Aggregate, Bounded Context) bu sürecin üzerine kurmayı öneriyor.

**Neden bu teknik kullanılır?**

Domain expert ile geliştirici arasındaki dil farkını kapatıyor. Herkes aynı renkli post-it'lere bakarak konuşuyor, teknik jargon zorunlu değil. İş süreci hakkında hızlıca ortak bir anlayış oluşturuyor, geleneksel modelleme yöntemlerinde gözden kaçabilecek gizli karmaşıklıkları ortaya çıkarıyor, teknik olmayan paydaşlarla teknik ekip arasındaki zihinsel modelleri hizalıyor.

Özellikle şu noktalarda işe yarıyor:

- Karmaşık bir domain içindeki Bounded Context'leri tespit etmek
- Sistemin yapı taşı olan Aggregate'leri keşfetmek
- Event-driven bir mimarinin temelini atmak

### Seviyeler

Event Storming tek seferlik bir oturum değil, üç seviyeden oluşan ve giderek detaylanan yinelemeli (iterative) bir süreç. Her seviye bir öncekinin üzerine inşa edilir, takım domain hakkında big-picture bir bakıştan implementasyona daha yakın bir modele doğru kademeli ilerler.

1. **Big Picture Level**  Amaç: iş domain'inin büyük resmini anlamak. Araç seti: Event, System, People, Hotspot.
2. **Process Level**  Amaç: süreçlerin nasıl işlediğine daha derin inmek. Araç seti: Event, Policy, Command, Read Model.
3. **Design Level (System/Software)**  Amaç: iş süreçleri ile yazılım tasarımı arasındaki köprüyü kurmak. Araç seti: Aggregate.

Seviyeler ilerledikçe modelleme dili de zenginleşiyor, her seviye bir öncekinin üstüne yeni kavramlar ekliyor. Son seviye (Design Level) doğrudan DDD'nin Tactical Pattern'leriyle, özellikle Aggregate ile kesişiyor  yani Event Storming doğru ilerletildiğinde sizi organik olarak tactical tasarıma taşıyabiliyor.

### Big Picture: Renkli Post-it'ler

Big Picture seviyesi domain keşfinin giriş noktası, bilinçli olarak detaydan kaçınıp sadece Domain Event'lere odaklanır. Nedeni şu: domain analizi yaparken ilginç ama core domain için kritik olmayan konulara fazla derinlemesine dalma riski var. Odağı dar tutmak, aşırı karmaşık ve domain'in en önemli yanlarını anlatamayan modeller üretilmesini engelliyor; bunun yerine kilit senaryolar hızlıca belirlenip önceliklendiriliyor.

Bu seviyede kullanılan elemanlar, farklı renkte post-it'lerle temsil edilir:

- **Domain Event (turuncu):** İş domain'inde gerçekleşmiş önemli bir olay veya durum değişikliği. Tamamlanmış bir eylem olduğunu vurgulamak için geçmiş zamanla yazılır (örn. "Sipariş Verildi", "Ödeme Alındı"). Soldan sağa bir zaman çizgisine dizilerek temel iş sürecinin ve akışın belirlenmesine yardımcı olur.
- **Hotspot (pembe/kırmızı):** Süreçteki çatışma, belirsizlik veya risk alanları. Teknik zorluk, iş kuralı çatışması veya netleştirilmesi gereken bir nokta olabilir. Genelde ilgili event'lerin yakınına veya çelişen elemanlar arasına konur, ileride tartışılması gereken konuları işaretler.
- **System (yeşil):** Ana süreçle etkileşime giren harici sistemler/servisler. Üçüncü parti API, legacy sistem, veritabanı veya başka bir mikroservis olabilir; ilişkili oldukları event ya da komutun yakınına yerleştirilir.
- **People / Actor (genelde çöp adam figürü):** Süreçteki farklı roller veya aktörler  müşteri, çalışan, sistem yöneticisi, dış paydaş. Her aşamada kimin dahil olduğunu gösterir.

### Process ve Design Level'da Eklenenler

Process ve Design Level'a geçildikçe modelleme diline yeni elemanlar eklenir:

- **Command (mavi):** Bir aktör ya da sistem tarafından tetiklenen, bir Aggregate üzerinde çalıştırılan eylem isteği  Aggregate üzerinde çağrılan bir metot gibi düşünülebilir. Aggregate'leri tespit etmek için Command'lar doğal bir başlangıç noktası.
- **Policy:** "Şu event gerçekleştiğinde şu komutu tetikle" mantığıyla çalışan otomatik ya da iş kuralı tabanlı tetikleyiciler. Genelde event'ler tarafından tetiklenir ama bir aktör tarafından da (mesela manuel bir onay süreciyle) tetiklenebilir.
- **Read Model:** Sürecin bir noktasında karar almak veya bir aktöre bilgi sunmak için ihtiyaç duyulan, mevcut durumu yansıtan veri görünümü.
- **Aggregate:** Design Level'da ortaya çıkan, domain invariant'larını ve veri bütünlüğünü koruyan yapı taşı  DDD'nin tactical pattern'lerindeki Aggregate ile birebir aynı kavram. Aynı Aggregate diyagramda farklı komutlarla (oluşturma, güncelleme, iptal gibi) birden fazla kez görünebilir.

### Örnek: Açık Artırma Sisteminde Teklif Verme

Senaryo şöyle: bir teklif veren (Bidder) bir açık artırmaya teklif eklemek istediğinde, sistem önce teklifin sahte/şüpheli olup olmadığını kontrol etmeli. Bu kontrolü, şüpheli teklif kalıplarını ve kullanıcıları tespit edebilen harici bir sistem yapıyor. Şüpheli işaretlenen teklifler reddediliyor; kontrolden geçen teklif açık artırmaya kaydediliyor ve teklif veren resmi olarak yarışa dahil oluyor. Başarılı bir teklifin ardından tüm katılımcılara e-posta ile bildirim gönderiliyor.

İlk iterasyonda sadece temel event'ler sırayla diziliyor  "Teklif İstendi", "Sahtecilik Kontrolü Yapıldı", "Teklif Reddedildi" / "Teklif Kaydedildi", "Bildirim Gönderildi" gibi  ve süreçteki belirsiz bir nokta hotspot olarak işaretleniyor. Bu aşama tamamen analitik; amaç takımın süreci kavraması ve olası sorunları erken görmesi.

İkinci iterasyonda model, kilit aktörler ve sistemler eklenerek zenginleşiyor: süreci başlatan Bidder, teklifleri analiz edip tek bir event üreten Sahtecilik Kontrol Sistemi, ve katılımcılara mesaj göndermekle sorumlu Bildirim Sistemi. Bu, sadece ne olduğunu değil, her adımda kimin ya da neyin dahil olduğunu da gösteren daha eksiksiz bir tablo.

Süreç Design Level'a taşındığında Policy ve Aggregate eklenir: bir Sahtecilik Kontrol Politikası teklifin sahte olup olmadığını kontrol etme sürecini başlatır; bir Teklif Kayıt Politikası sahtecilik kontrolünden geçen teklifin resmi kaydını tetikler; "Auction" bir Aggregate olarak teklif kaydına dair domain kurallarını korur; bir Sahte Teklif Politikası sahte işaretlenen bir teklifle ilgili sonraki adımları yönetir (burada bazen bir çalışanın manuel müdahalesi de gerekebilir); bir Read Model ise açık artırmanın güncel durumunu (mevcut en yüksek teklif gibi) sunar.

### DDD Workflow'undaki Yeri

Event Storming, DDD workflow'unda Strategic'ten Tactical'a geçişte bir köprü görevi görür: önce Big Picture'da domain keşfedilir, Bounded Context adayları ortaya çıkar; sonra Process Level'da süreçler, komutlar, politikalar netleştirilir; en sonunda Design Level'da Aggregate'ler belirlenip doğrudan Tactical Pattern'lere (Aggregate, Entity, Value Object, Repository) geçilir.

Bir nüans var: takım Big Picture seviyesinde domain hakkında kapsamlı bir anlayışa ulaştıktan sonra mutlaka Event Storming ile devam etmesi gerekmiyor. Sonraki adımda hangi modelleme tekniğinin kullanılacağı, özellikle DDD'nin Model-Driven Design prensibiyle uyumlu olarak, çeşitli faktörlere bağlı. Bu prensibe göre modelleme dili hem implementasyon için seçilen yazılım araçlarının kavramlarını yakından yansıtmalı hem de Aggregate, Entity gibi temel DDD yapı taşlarını ifade edebilecek şekilde DDD ile uyumlu olmalı.

### Mülakat İçin Notlar

- Event Storming bir DDD zorunluluğu değil, DDD ile çok iyi örtüşen bir keşif/workshop tekniği. Amacı hem "ne" (senaryolar, model) hem "nasıl" (insanları modelleme sürecine dahil etmek) sorularına birlikte cevap vermek.
- Üç seviye birbirinin yerine geçmiyor, kademeli detaylandırma sağlıyor. Hangi seviyede hangi post-it/araç kullanıldığını ayırt edebilmek önemli: Big Picture'da Event, System, People, Hotspot; Process'te Event, Policy, Command, Read Model; Design'da Aggregate.
- Command, Aggregate tespit etmenin doğal bir başlangıç noktası: bir Aggregate üzerinde çağrılabilecek komutları listelemek, o Aggregate'in sorumluluk sınırlarını netleştirir.
- Aynı Aggregate diyagramda farklı komutlarla birden çok kez görünebilir, bu normal  Aggregate'in birden fazla davranışı (create, update, cancel vb.) olduğunu gösterir.
- Policy'ler sadece event'lerle değil bazen bir aktörün manuel kararıyla da (örn. çalışanın şüpheli işlemi onaylaması/reddetmesi) tetiklenebilir; yani Event Storming tamamen otomatik süreçlerle sınırlı değil.
- Design Level "Software Design" aşamasına işaret eder ve doğrudan Tactical Pattern'lerle kesişir  Event Storming sadece bir analiz aracı değil, tasarım sürecine de rehberlik edebilir.

