# Bölüm 9 — Sınav soru bankası (birleşik senaryolar)

> Bölüm 1–7'de her konunun kendi soruları var (35 soru). Bu bölümdeki 20 soru **birden fazla servisi birlikte** test ediyor — gerçek SAP-C02 soruları böyle. Her soru için **neden diğer seçeneklerin yanlış olduğunu** ayrı ayrı okuyun; sınavda kazandıran budur.

**Nasıl kullanılmalı:** Cevabı açmadan önce kendi cevabınızı bir kâğıda yazın ve **neden diğerlerini elediğinizi** de yazın. Eleme gerekçeniz benim gerekçemle uyuşmuyorsa, o konuya geri dönün.

---

## Soru 1 — Çok katmanlı hibrit tasarım

Bir küresel şirketin `eu-central-1` ve `us-east-1`'de toplam 45 VPC'si var. Gereksinimler:
- Frankfurt ve Virginia'daki veri merkezleri her iki Region'daki prod VPC'lere erişmeli
- Prod ve non-prod VPC'ler birbirini görmemeli
- Tüm VPC'ler shared-services VPC'sindeki Active Directory'ye erişmeli
- On-prem ↔ AWS trafiği **şifreli** olmalı, hosted connection kullanılıyor
- Region'lar arası VPC trafiği mümkün olmalı

En uygun mimari nedir?

- **A.** Her Region'da bir TGW; TGW'ler arası peering + statik route; her Region'da DXGW + transit VIF; transit VIF üzerinden private IP VPN; TGW'de RT-PROD / RT-NONPROD / RT-SHARED route table'ları.
- **B.** Tek bir global TGW; DXGW ile bağla; MACsec ile şifrele; tek route table + security group izolasyonu.
- **C.** Cloud WAN core network; her Region'da segment; DXGW native attachment; public VIF üzerinden VPN.
- **D.** Her VPC çifti arasında inter-Region VPC peering; her VPC'de bir VGW; VGW'lere private VIF.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: A**

Her gereksinim tek tek karşılanıyor:
- **Çok Region:** TGW **bölgesel** bir kaynaktır — "global TGW" diye bir şey yoktur. Her Region'a bir TGW + aralarında **peering (statik route)**.
- **Prod/non-prod izolasyonu:** ayrı route table'lar (Bölüm 1.3, desen 2).
- **Shared services:** SHARED attachment'ı her iki tabloya **propagate**.
- **Şifreleme + hosted connection:** **MACsec hosted'da YOKTUR** → **private IP VPN over DX** (TGW private CIDR + DXGW + **transit VIF**).
- **On-prem → çok Region:** DXGW **global** bir kaynaktır; ama transit VIF → DXGW → TGW zinciri her Region için ayrı TGW association gerektirir (DXGW başına 6 TGW kotası rahat yeter).

- **B iki kez yanlış:** (i) "tek global TGW" **yoktur**, TGW bölgeseldir. (ii) **MACsec hosted connection'da desteklenmez.** Ayrıca izolasyonu SG ile yapmak 45 VPC'de sürdürülemez ve route table doğru katmandır.
- **C kısmen mantıklı ama iki hatası var:** Cloud WAN gerçekten güçlü bir seçenek (çok Region + segment), ama (i) **public VIF üzerinden VPN**, "tünel uç noktaları private olsun" demiyor ama şifreleme sağlar — sorun değil; asıl hata (ii) **DXGW bir Cloud WAN core network'e bağlıysa başka hiçbir association alamaz ve TEK segment'e eşlenir.** Prod/non-prod/shared üç segment senaryosunda tek DXGW ile bunu kuramazsınız. (Birden fazla DXGW gerekir; seçenek bunu söylemiyor.)
- **D yanlış:** 45 VPC × inter-Region peering = yüzlerce bağlantı, transitive değil, shared-services erişimi her VPC'den ayrı peering ister. "En uygun" olmaktan çok uzak.
</details>

---

## Soru 2 — Sessiz başarısızlık

Bir ekip Traffic Mirroring kurdu: kaynak ENI'ler VPC-A'da, hedef ise VPC-B'de bir NLB. Peering kuruldu, security group'lar UDP 4789'a izin veriyor, filtreler tanımlı. **Hiç paket gelmiyor ve hiç hata mesajı yok.** En olası iki sebep nedir? (İki cevap seçin)

- **A.** NLB'de UDP 4789 listener'ı yok.
- **B.** VPC-A ile VPC-B arasında **inter-Region** peering kullanılıyor.
- **C.** Mirror filtresinde outbound kural tanımlanmamış.
- **D.** Kaynak instance'lar Nitro değil.
- **E.** `PacketLength` 8500'den büyük ayarlanmış.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevaplar: A ve B**

- **A doğru — ve "hiç hata yok" ifadesinin imzası:** Doküman birebir: NLB hedefi için UDP 4789 listener'ı yoksa *"Traffic Mirroring **hatasız bir şekilde** başarısız olur"* (*"fails without an error indication"*). Bu, gerçek hayatta saatler kaybettiren en klasik hatadır ve sorudaki "hiç hata mesajı yok" ifadesi doğrudan buna işaret eder.
- **B doğru:** Cross-VPC mirror bağlantı seçenekleri **intra-Region VPC peering, transit gateway, GWLB endpoint** olarak listelenir. **Inter-Region peering listelenmemiştir** — çalışmaz.

- **C kısmen doğru ama "hiç paket gelmiyor"u tam açıklamaz:** Outbound kural yoksa **yalnızca outbound** mirror edilmez; inbound kurallar varsa inbound trafik yine gelir. Soru "hiç paket" diyor, dolayısıyla bu tek başına sebep değil. (Kural **hiç** yoksa hiçbir şey mirror edilmez — ama soru "filtreler tanımlı" diyor.)
- **D yanlış — ve yaygın bir yanlış inanç:** "Sadece Nitro" hatalı bir basitleştirmedir. 2021'de seçili non-Nitro (Xen) tipler eklendi (C4, D2, G3, H1, I3, M4, P2, P3, R4, X1, X1e) ve Ağu 2025'te tüm Nitro v4'e yayıldı. Bu bir **instance-tipi listesi** gereksinimidir.
- **E yanlış:** `PacketLength` 8500'den büyükse API **hata verir** — "hiç hata yok" ifadesiyle çelişir.
</details>

---

## Soru 3 — Maliyet optimizasyonu

Bir şirketin aylık ağ faturasında şu kalemler öne çıkıyor:
- 80 VPC'de her birinde S3 ve DynamoDB için interface endpoint
- 80 VPC'de her birinde 6 AWS servisi için interface endpoint
- Her VPC'de kendi resolver rule ilişkilendirmeleri
- 320 şube için 320 ayrı Site-to-Site VPN connection (her biri ~40 Mbps)

Hangi dört değişiklik maliyeti en çok azaltır? (Dört cevap seçin)

- **A.** S3 ve DynamoDB için interface endpoint'leri **gateway endpoint**'lere çevir.
- **B.** 6 AWS servisi için endpoint'leri **merkezî bir hub VPC'ye** taşı, private DNS'i kapat, PHZ + alias ile spoke'lara dağıt.
- **C.** Resolver rule ilişkilendirmelerini **Route 53 Profile** ile tek nesneye indir.
- **D.** 320 VPN connection'ı **VPN Concentrator**'lara topla (4 concentrator).
- **E.** Tüm VPC'leri tek bir VPC'de birleştir.
- **F.** Tüm endpoint'leri kaldırıp NAT gateway üzerinden internete çık.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevaplar: A, B, C ve D**

- **A — en büyük tek kazanç:** **Gateway endpoint tamamen ücretsizdir** (saatlik yok, GB yok). Interface endpoint ise ENI başına $0,01/saat + $0,01/GB. 80 VPC × 2 servis × ~3 AZ = 480 ENI ≈ aylık **$3.500** sadece saatlik ücret. Gateway'e çevirmek bunu sıfırlar. **Uyarı:** on-prem'den S3 erişimi gerekiyorsa interface endpoint kalmalı — o zaman **"private DNS only for inbound endpoint"** ile ikisi birlikte kullanılır.
- **B:** 80 VPC × 6 servis × 3 AZ = 1.440 ENI → merkezî hub'da 6 × 3 = 18 ENI. Ödünleşimleri kabul edersiniz: **tek endpoint policy → daha büyük blast radius**, **20.480 karakter** policy tavanı ve zonal servislerde **cross-AZ veri transfer ücreti**.
- **C:** Route 53 Profile, **100 ilişkilendirmeye kadar hesap başına $0,75/saat**. 80 VPC × N kural ilişkilendirmesi yerine 80 Profile ilişkilendirmesi. Ayrıca **rule↔VPC association kotası Region başına 2.000**'dir — bu yaklaşım kotayı da kurtarır.
- **D:** 320 VPN connection = 320 connection-saat + 320 TGW attachment ücreti. **VPN Concentrator** 100 siteyi **tek attachment**'ta toplar → 4 concentrator = 4 attachment. Şubeler 40 Mbps kullandığı için concentrator'ın **100 Mbps tünel** limiti uygundur. Kabul edilecek kısıtlar: **ECMP yok, private IP VPN yok, VGW yok, mevcut VPN'ler taşınamaz** (yeni connection kurulacak).

- **E yanlış:** 80 VPC'yi birleştirmek bir ağ optimizasyonu değil, çok yıllık bir kurumsal yeniden yapılandırma projesidir; blast radius'u ve hesap izolasyonunu yok eder.
- **F yanlış ve genellikle daha pahalı:** NAT gateway **$0,045/saat + $0,045/GB** (endpoint'in ~4,5 katı GB ücreti) ve trafiği internete taşır — hem güvenlik gerilemesi hem maliyet artışı.
</details>

---

## Soru 4 — Route öncelik zinciri

Bir TGW route table'da `10.50.0.0/16` prefix'i için şu kaynaklar var:
1. Bir VPC attachment'ından propagated `10.50.0.0/16`
2. Bir DXGW attachment'ından propagated `10.50.0.0/16`
3. Bir VPN attachment'ından propagated `10.50.0.0/16`
4. Bir VPN attachment'ına elle girilmiş **statik** `10.50.0.0/16`
5. Bir DXGW attachment'ından propagated `10.50.1.0/24`

`10.50.1.5` hedefli bir paket hangi attachment'a gider?

- **A.** VPC attachment (propagated `/16` — VPC en yüksek öncelikli tip)
- **B.** VPN attachment (statik route her şeyi yener)
- **C.** DXGW attachment (`10.50.1.0/24` — longest prefix match)
- **D.** DXGW attachment (propagated `/16` — DX VPN'i yener)

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: C**

**Longest prefix match her şeyin önündedir.** `10.50.1.5` hedefi için `10.50.1.0/24`, tüm `/16`'lardan daha spesifiktir — attachment tipi, statik/propagated ayrımı, hiçbiri devreye girmez.

Değerlendirme sırası:
1. **Longest prefix match** ← burada karar verildi
2. Aynı prefix ise: statik > propagated
3. Aynı prefix, farklı tip: sabit 10 basamaklı sıra
4. Aynı prefix, aynı tip: AS_PATH → MED → eBGP>iBGP

- **A yanlış:** VPC gerçekten propagated tipler arasında en yüksek önceliklidir (3. basamak), ama 3. basamağa **hiç gelinmiyor** — 1. basamakta karar veriliyor.
- **B yanlış:** Statik route, **aynı prefix** için propagated'ı yener (2. basamak). `/16` statik, `/24`'ten daha spesifik değildir.
- **D yanlış:** Aynı hata — DX gerçekten VPN'i yener ama yalnızca **aynı prefix** için.

**Takip sorusu refleksi:** `10.50.9.5` hedefli bir paket sorulsaydı? O zaman `/24` eşleşmez, dört adet `/16` yarışır → 2. basamak: **statik VPN route kazanır** (cevap B olurdu). Bu iki soruyu birlikte hatırlamak, tuzağı kalıcı olarak kapatır.
</details>

---

## Soru 5 — Endpoint + DNS kombinasyonu

Bir şirket, hub VPC'de merkezî interface endpoint'ler kurdu ve private DNS'i **açık** bıraktı. TGW ile bağlı spoke VPC'lerdeki EC2 instance'ları `secretsmanager.eu-central-1.amazonaws.com`'u çözemiyor ve internete çıkmaya çalışıyor. Sebep ve çözüm nedir?

- **A.** Spoke VPC'lerde `enableDnsSupport` kapalı; açılması gerekir.
- **B.** Private DNS etkinken oluşan gizli AWS-managed PHZ yalnızca endpoint'in VPC'sine hizmet eder. Private DNS'i kapat, kendi PHZ'ni oluşturup endpoint'e alias yaz ve spoke'lara ilişkilendir.
- **C.** TGW route table'da eksik rota var.
- **D.** Spoke VPC'lerde de aynı endpoint'ler oluşturulmalıdır.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: B**

Bu, merkezî endpoint deseninin en kritik detayı. Private DNS etkinleştirildiğinde AWS **gizli, yönetilen bir private hosted zone** oluşturur ve bunu **yalnızca endpoint'in bulunduğu VPC'ye** ilişkilendirir. Doküman: *"Route 53 Resolver'ı VPC'nizin dışından kullanamazsınız."* Spoke VPC'ler bu gizli PHZ'yi göremez, dolayısıyla servisin regional adını public IP'ye çözer ve internete gitmeye çalışır.

**Doğru desen:**
1. Endpoint'te **private DNS'i KAPAT**
2. Servis adıyla **kendi PHZ**'nizi oluştur, endpoint'e **ALIAS** kaydı yaz (`dkr.ecr` için **wildcard `*` alias**)
3. PHZ'yi spoke'lara ilişkilendir — ölçekte **Route 53 Profile** ile
4. On-prem için **inbound Resolver endpoint** + conditional forwarder

- **A yanlış:** `enableDnsSupport` kapalıysa VPC'de **hiçbir** DNS çözümlemesi olmaz — instance'lar internete de çıkamaz, hiçbir isim çözülmez. Semptom uyuşmuyor.
- **C yanlış:** Sorun **DNS çözümlemesi**, routing değil. Kanıt: instance internete çıkmaya çalışıyor, yani bir IP çözmüş — sadece **yanlış** (public) IP çözmüş. Routing çalışıyor.
- **D yanlış:** Çalışır ama merkezî desenin **tüm amacını** ortadan kaldırır — maliyeti azaltmak için merkezîleştirmişlerdi.

**2025 notu:** Route 53 Profiles artık **interface VPC endpoint'leri doğrudan tutabiliyor** (Nis 2025) ve AWS bu elle PHZ+alias desenini "artık önerilmiyor" diyor — ama mekanizmayı bilmek sınavda gerekli.
</details>

---

## Soru 6 — Inspection + appliance mode + GWLB

Bir mimar merkezî inspection kuruyor: spoke VPC'ler → TGW → inspection VPC → GWLB → üçüncü parti firewall'lar. Üç AZ kullanılıyor. Aşağıdaki hangi **dört** yapılandırma zorunludur? (Dört cevap seçin)

- **A.** Inspection VPC attachment'ında appliance mode etkinleştirilmeli.
- **B.** GWLB target group'ta 5-tuple flow stickiness kullanılmalı.
- **C.** Her AZ'de bir GWLB endpoint oluşturulmalı.
- **D.** İlgili TGW route table'ında route propagation açık olmalı.
- **E.** GWLB'de cross-zone load balancing etkinleştirilmeli.
- **F.** Firewall instance'larında source/destination check kapatılmalı.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevaplar: A, B, C ve D**

- **A:** Stateful inspection'da gidiş-dönüş simetrisi için **zorunlu**. Olmadan dönüş paketi başka AZ'deki firewall'a düşer → "unknown flow" → drop.
- **B:** Doküman: *"2-tuple ve 3-tuple flow stickiness, AWS Transit Gateway appliance mode açıkken **desteklenmez**."* Appliance mode kullanıyorsanız **5-tuple zorunlu**.
- **C:** GWLBE **tam olarak bir AZ'ye** bağlıdır ve subnet'i **sonradan değiştirilemez**. Cross-zone LB varsayılan **kapalı** olduğu için her AZ'de GWLBE + healthy appliance olmalı; yoksa o AZ trafiği **blackhole** eder.
- **D — en çok atlanan:** Appliance mode'un **gizli ön koşulu**: ilgili TGW route table'ında **route propagation açık olmalı**. Kapalıysa aynı-AZ trafiği bile hash'e düşer ve AZ izolasyonu bozulur.

- **E yanlış:** Cross-zone LB'yi açmak **AZ-yerelliğini bozar**, cross-AZ trafik yaratır ve appliance mode'un sağladığı simetriyi karmaşıklaştırır. Doğru tasarım **AZ başına GWLBE + appliance**'tır.
- **F yanlış:** GWLB **GENEVE kapsülleme** kullanır — paket appliance'a tünel içinde gelir, appliance'ın IP forwarding yapması gerekmez. Source/dest check, klasik "EC2'yi NAT/router yapma" deseninin gereğidir, GWLB mimarisinin değil.

**Bonus — appliance'ın uyması gereken sözleşme:** GENEVE ile yeniden kapsülle, **yalnızca dış IP'leri takas et (portları TAKAS ETME)**, **flow cookie'yi değiştirmeden** döndür, **aynı GWLB'ye** gönder.
</details>

---

## Soru 7 — Şifreleme seçim zinciri

Aşağıdaki dört senaryoyu doğru şifreleme çözümüyle eşleştirin:

1. 100 Gbps dedicated DX, yalnızca fiziksel linkte line-rate şifreleme yeterli
2. 1 Gbps hosted connection, DX üzerinde uçtan uca şifreleme + tünel uç noktaları private olmalı
3. TGW üzerinden geçen VPC↔VPC trafiği uyumluluk gereği şifreli olmalı
4. DX üzerinden şifreli erişim + AWS public servislerine (S3 public endpoint) de erişim

**Seçenekler:** MACsec · Private IP VPN over DX · TGW Encryption Support · Public VIF + klasik S2S VPN

<details>
<summary><b>Cevap ve analiz</b></summary>

**Eşleştirmeler:**

**1 → MACsec.** 10/100/400 Gbps **dedicated** bağlantılarda mevcut, ek ücretsiz. **Noktadan noktaya** (sizin router ↔ AWS DX cihazı) — doküman: *"birden fazla ardışık Ethernet veya diğer ağ segmenti boyunca uçtan uca şifreleme SAĞLAMAZ."* Senaryo "yalnızca fiziksel linkte" dediği için uygun. CKN/CAK: hex, **tam 64 karakter**, **yalnızca AES-256, yalnızca static CAK**. 100G/400G'de **GCM-AES-XPN-256 zorunlu**.

**2 → Private IP VPN over DX.** İki nedenle: (i) **MACsec hosted connection'da desteklenmez**, (ii) senaryo "tünel uç noktaları private" istiyor. Üç bileşen: **TGW (private IP CIDR atanmış) + DXGW + TRANSIT VIF**. VGW desteklenmez. Bonus: route ölçeği **5.000 çıkış / 1.000 giriş** (yalnız DX'te 200/100) ve **5 Gbps LBT** desteği.

**3 → TGW Encryption Support** (Kas 2025). VPC↔VPC trafiği için transit halinde şifrelemeyi zorunlu kılar. **Uyumlu:** VPC, DX, VPN attachment'ları. **Uyumsuz (önce kaldırılmalı):** Connect, **peering**, Network Firewall, VPN Concentrator, Client VPN attachment'ları, **security group referencing**, **multicast**. `modify-transit-gateway` ile açılır (oluşturmada değil), **14 güne kadar** sürebilir, `use1-az3`'te yok.

**4 → Public VIF + klasik Site-to-Site VPN.** Public VIF, AWS public endpoint'lerine erişim sağlar; şifreleme için üstüne klasik VPN kurulur. **Kritik: accelerated VPN, Direct Connect public VIF ile KULLANILAMAZ** — bu bir tuzak seçeneği olarak sunulur. Ayrıca public VIF MTU **yalnızca 1500**'dür ve SiteLink public VIF'te yoktur.
</details>

---

## Soru 8 — Bant genişliği matematiği

Bir şirket on-prem'den AWS'e **tek bir** veritabanı replikasyon akışıyla 4 Gbps sürekli throughput istiyor. Mevcut yapı: TGW'ye bağlı 3 adet standard Site-to-Site VPN connection, BGP, ECMP açık (toplam 6 tünel = 7,5 Gbps teorik). Ölçülen throughput **1,2 Gbps**'te tavan yapıyor. Kök neden ve çözüm nedir?

- **A.** ECMP düzgün çalışmıyor; TGW'de VPN ECMP support kapalı olabilir.
- **B.** ECMP 5-tuple hash kullandığı için tek bir akış tek tünele sabitlenir ve o tünelin 1,25 Gbps tavanını geçemez. Çözüm: LBT (5 Gbps) tünele geçmek veya transferi paralel akışlara bölmek veya Direct Connect'e geçmek.
- **C.** Üç VPN connection yerine bir VPN connection ile 6 tünel kurulmalı.
- **D.** MTU 1446'ya düşürülmeli.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: B**

Bu, VPN konusunun **en sevilen tuzağıdır**. TGW ECMP **5-tuple hash** kullanır: kaynak IP + kaynak port + hedef IP + hedef port + protokol. Tek bir TCP akışının 5-tuple'ı sabit olduğu için **her paketi aynı tünele** düşer. Toplam kapasite 7,5 Gbps olsa da **tek akış 1,25 Gbps'i geçemez** — ölçülen 1,2 Gbps bunu doğruluyor.

Üç çözüm:
1. **LBT (Large Bandwidth Tunnel)** → tünel başına **5 Gbps**. Ve May 2026'dan beri mevcut bağlantıyı **kesintisiz** çevirebilirsiniz (tünel IP'leri, CIDR'lar, PSK'lar korunur).
2. **Transferi paralel akışlara bölmek** (çoklu stream) → farklı 5-tuple'lar → farklı tüneller.
3. **Direct Connect'e geçmek** → tek akış için tünel tavanı yoktur.

- **A yanlış:** ECMP çalışıyor — kanıt: ölçüm tam olarak tek tünel tavanında. ECMP kapalı olsaydı da aynı sonuç görünürdü, ama soru "ECMP açık" diyor ve **ECMP açık olsa da tek akış problemi çözülmez**. Bu, ECMP'yi doğru sanan ama tek-akış kısıtını bilmeyen adayın düşeceği tuzak.
- **C yanlış:** Bir VPN connection **her zaman 2 tünel**tir — 6 tünel kurulamaz. Ayrıca tünel sayısını değiştirmek tek-akış problemini çözmez.
- **D yanlış:** MTU'yu düşürmek throughput'u **azaltır**, artırmaz. (VPN payload MTU'su zaten max 1446'dır.)
</details>

---

## Soru 9 — DNS + endpoint + on-prem

Bir şirket on-prem'den `s3.eu-central-1.amazonaws.com`'a **private** erişim kurdu (interface endpoint, private DNS açık). Bir ay sonra faturada beklenmeyen bir PrivateLink veri işleme kalemi görüyorlar. VPC içindeki uygulamalar zaten bir gateway endpoint'e sahipti. Ne oldu ve nasıl düzeltilir?

- **A.** Gateway endpoint çalışmıyor; route table'da prefix list rotası eksik.
- **B.** Private DNS açık olduğu için VPC içindeki uygulamalar da interface endpoint'e yönlendi. Interface endpoint'te "private DNS only for inbound endpoint" seçeneği etkinleştirilmeli.
- **C.** Gateway endpoint ve interface endpoint aynı VPC'de birlikte kullanılamaz.
- **D.** S3 bucket'ları farklı Region'da; cross-Region ücret oluştu.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: B**

Doküman bu senaryoyu **birebir bir uyarı** olarak veriyor: *"Eğer S3 için interface endpoint'te private DNS'i yapılandırırsanız ama sadece inbound Resolver endpoint için private DNS'i yapılandırmazsanız, hem on-prem ağınızdan hem VPC'nizden gelen istekler S3'e erişmek için interface endpoint'i kullanır. Dolayısıyla VPC'den gelen trafik için de — ücretsiz gateway endpoint yerine — interface endpoint ücreti ödersiniz."*

Mekanizma: private DNS açıkken `s3.eu-central-1.amazonaws.com` **VPC içinde de** endpoint ENI'sinin private IP'sine çözülür. Gateway endpoint'in prefix-list rotası devreye girmez çünkü hedef IP artık S3'ün public IP'si değil, ENI'nin private IP'sidir.

**Çözüm:** Konsolda **"Enable private DNS only for inbound endpoint"**. Doküman: *"Private DNS'i yalnızca inbound Resolver endpoint için, **sadece Amazon S3 interface endpoint'lerinde** yapılandırabilirsiniz."* — bu ayar **S3'e özeldir**, başka servis için yoktur.

- **A yanlış:** Gateway endpoint rotası duruyor; sorun routing değil **DNS çözümlemesi**. Trafik gateway'e hiç gelmiyor çünkü hedef IP değişti.
- **C yanlış:** Birlikte kullanılabilirler — zaten **önerilen optimal desen** budur.
- **D yanlış:** Senaryoda cross-Region bir ipucu yok ve fatura kalemi açıkça **PrivateLink veri işleme**.
</details>

---

## Soru 10 — Failover ve BGP

Bir şirket `10.0.0.0/8`'i on-prem'den hem DX (transit VIF → DXGW → TGW) hem VPN (TGW) üzerinden BGP ile duyuruyor. Şu problemler var:
1. DX kesildiğinde failover **90 saniye** sürüyor
2. On-prem stateful firewall'da ara ara "invalid state" düşüşleri var
3. Ekip, bakım penceresinde trafiği **kasıtlı olarak** VPN'e almak istiyor

Üç problemi çözen kombinasyon nedir? (Üç cevap seçin)

- **A.** DX VIF'inde on-prem router üzerinde BFD yapılandır.
- **B.** On-prem router'da DX için local preference'ı VPN'den yüksek ayarla.
- **C.** Bakım için TGW'de `10.0.0.0/8` → VPN attachment statik route ekle.
- **D.** VPN üzerinde AS_PATH prepend uygula.
- **E.** VPN'den `10.0.0.0/16` duyur.
- **F.** DX VIF'inde local preference community `7224:7100` kullan.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevaplar: A, B ve C**

- **A → problem 1.** BGP varsayılan hold time **90 saniye**. AWS tarafında asenkron BFD **her VIF'te otomatik etkin** ama **siz router'ınızda yapılandırmadan devreye girmez**. Yapılandırıldığında AWS min interval **300 ms × multiplier 3 ≈ 900 ms**. (Not: AWS graceful restart ile BFD'yi birlikte kullanmamanızı öneriyor.)
- **B → problem 2.** "Invalid state" düşüşleri asimetrik routing'in imzasıdır. AWS **çıkışta** DX'i zaten tercih eder (sabit attachment tipi sırası); **giriş** yönünü ancak on-prem router'ın local preference'ı belirler. İkisini hizalamak asimetriyi bitirir.
- **C → problem 3.** TGW route sırasında **statik route, propagated route'u yener** ve **VPN statik route'u listenin 1. basamağındadır** — DX-propagated'ın (4. basamak) önünde. Yani bakım penceresinde statik route ekleyip sonra silmek, trafiği deterministik olarak VPN'e alıp geri döndürmenin doğru yoludur.

- **D yanlış:** DX ve VPN **farklı attachment tipleri** olduğu için aralarındaki sıra **sabittir**; AS_PATH prepend bunu değiştirmez. Ayrıca farklı tipler arasında **ECMP de yapılmaz**.
- **E yanlış ve tehlikeli:** Daha spesifik prefix duyurmak, **longest prefix match** nedeniyle VPN'i **kalıcı olarak** kazandırır — pahalı DX hattı boşta kalır ve asimetri artar. Doğru pratik: VPN'den **aynı ya da daha geniş** prefix duyurmak.
- **F yanlış:** `7224:7100` = **LOW local preference**. Bunu DX'e uygulamak, AWS'in **çıkışta VPN'i** tercih etmesine yol açar — problemi tersine çevirir. (Ayrıca bu community'ler farklı **VIF'ler** arasında tercih belirler; farklı attachment tipleri arasındaki TGW sırasını değiştirmez.)
</details>

---

## Soru 11 — Çakışan CIDR + veritabanı

İki şirket birleşti. A şirketinin VPC'si `172.16.0.0/16`, B şirketinin VPC'si de `172.16.0.0/16`. A'daki uygulama sunucularının B'deki **Aurora PostgreSQL** cluster'ına erişmesi gerekiyor. IP değiştirmek mümkün değil. Ayrıca B'deki bir **on-prem** SFTP sunucusuna da (`10.200.5.10`) erişim gerekiyor. En az bileşenle çözüm?

- **A.** TGW + private NAT gateway + her iki VPC'ye ek routable CIDR + NLB.
- **B.** B'de resource gateway kur; Aurora için `ARN` tipi ve SFTP için `SINGLE` tipi resource configuration oluştur; RAM ile paylaş; A'da resource endpoint aç.
- **C.** B'de iki NLB kur (Aurora ve SFTP için), iki endpoint service oluştur, A'da iki interface endpoint aç.
- **D.** İki VPC arasında inter-Region peering kur.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: B**

VPC Lattice **resource configuration** tam bu iki ihtiyacı birlikte karşılıyor:
- **`ARN` tipi:** doküman *"Yalnızca Amazon RDS veritabanları desteklenir"* diyor — Aurora bir RDS motorudur, kapsam içinde. RDS failover'ında IP değişse bile ARN sabit kalır.
- **`SINGLE` tipi:** bir IP adresi veya domain adı hedefi — **on-prem `10.200.5.10`** için. Doküman: resource endpoint *"başka bir VPC'deki **veya ON-PREMISES bir ortamdaki** private subnet'te bulunabilecek"* kaynaklara erişim sağlar.
- **Resource endpoint load balancer GEREKTİRMEZ** → en az bileşen.
- **AWS RAM** ile cross-account paylaşılır.

Kısıtlar: resource gateway **load balancing yapmaz**; resource endpoint ile resource gateway arasında **en az 1 AZ örtüşmesi** gerekir; target'lar yalnızca private IP aralıklarından (public IP desteklenmez).

- **A yanlış (bu senaryo için):** Çalışır ama en fazla bileşen: TGW + 2 NAT gateway + 2 ek CIDR + NLB + statik route yönetimi. Ayrıca **VGW üzerinden VPN/DX trafiği NAT gateway'e yönlendirilemez** — on-prem SFTP erişimi için ekstra dikkat gerekir.
- **C çalışır ama daha fazla bileşen:** 2 NLB + 2 endpoint service + 2 interface endpoint + target/health yönetimi. Ayrıca on-prem SFTP'yi NLB'ye IP-target olarak eklemek B'nin VPC'sinden on-prem'e routable bağlantı gerektirir — mümkün ama ek iş.
- **D yanlış — iki hata:** (i) Çakışan CIDR'da **peering imkânsızdır**. (ii) "Inter-Region" ifadesi de yanlış — senaryoda Region farkı belirtilmemiş.
</details>

---

## Soru 12 — Reachability vs Access Analyzer

Bir güvenlik ekibi iki ayrı görev alıyor:
- **Görev 1:** "Yeni deploy edilen bir Lambda'nın VPC içindeki RDS'e erişemediği bildirildi, nedenini bul."
- **Görev 2:** "Tüm organizasyonda internete doğrudan açık hiçbir veritabanı ENI'si olmadığını her hafta doğrula."

Hangi araç eşlemesi doğrudur?

- **A.** Görev 1 → Network Access Analyzer, Görev 2 → Reachability Analyzer
- **B.** Görev 1 → Reachability Analyzer, Görev 2 → Network Access Analyzer
- **C.** İkisi de Reachability Analyzer
- **D.** Görev 1 → VPC Flow Logs, Görev 2 → AWS Config

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: B**

- **Görev 1 → Reachability Analyzer:** Belirli **bir kaynak ve bir hedef** verirsiniz; araç yolun açık olup olmadığını söyler ve kapalıysa **engelleyen bileşeni** (security group, NACL, route table, load balancer) belirtir. **$0,10/analiz.**
- **Görev 2 → Network Access Analyzer:** Kaynak-hedef çifti **vermezsiniz**; bir **Network Access Scope** tanımlarsınız (Amazon-provided **IGW-ingress** scope'u veya özel bir MatchPaths/ExcludePaths tanımı) ve araç tüm hesabı/Region'ı tarayıp **eşleşen tüm yolları** bulgu olarak döndürür. **ENI başına $0,002** olduğu için haftalık geniş tarama ekonomik.

- **A yanlış:** Araçlar ters. Access Analyzer tek bir yolun "neden kapalı" olduğunu söylemez; Reachability geniş tarama yapamaz (her çift için ayrı analiz + $0,10).
- **C yanlış:** Görev 2'yi Reachability ile yapmak binlerce analiz ve maliyet demektir; ayrıca "hangi çiftleri test edeceğinizi" önceden bilemezsiniz.
- **D yanlış:** Flow Logs **gerçekleşmiş trafiği** gösterir — Lambda hiç bağlanamadıysa kayıt olmayabilir ve "ne engelliyor" sorusunu cevaplamaz. AWS Config SG kurallarını denetler ama **route table + NACL + IGW + ENI kombinasyonunun** yarattığı gerçek yolu değerlendirmez.

**Her iki araç için ortak kısıtlar:** ikisi de **statik** (paket göndermez), ikisi de **aynı Region**. Reachability ayrıca **IPv4 only**, TGW Connect attachment'ı ve TGW policy table'ı desteklemez, NFW domain list/Suricata kurallarını değerlendirmez, GWLBE arkasını analizden hariç tutar, analizler **120 gün** sonra silinir.
</details>

---

## Soru 13 — Cloud WAN servis zinciri

Bir şirket Cloud WAN kullanıyor ve `prod` segment'i içindeki VPC'ler arası trafiğin inspection'dan geçmesini istiyor. Hangi yapılandırma gereklidir? (Üç cevap seçin)

- **A.** Segment `isolated` mode'a alınmalı.
- **B.** Inspection appliance'larının attachment'ı bir Network Function Group'a (NFG) atanmalı.
- **C.** Segment action olarak `send-via` tanımlanmalı.
- **D.** Segment action olarak `send-to` tanımlanmalı.
- **E.** Inspection VPC'de appliance mode etkinleştirilmeli.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevaplar: A, B ve E** (ve `send-via`, yani C de gerekli — aşağıya bakın)

> **Not:** Bu soru bilinçli olarak "dört doğru var, üç seç" gerginliği taşıyor. Gerçek sınavda böyle bir soru gelirse **en spesifik ve en çok atlanan** üçünü seçin. Burada tam doğru set **A, B, C, E**'dir; `send-to` (D) yanlış olan tek seçenektir.

- **A doğru ve en çok atlanan:** **Isolated mode zorunludur** — aynı segment içindeki attachment'lar arası inspection için. Isolated olmayan bir segment'te VPC'ler doğrudan konuşur ve inspection'a uğramaz.
- **B doğru:** NFG, inspection appliance attachment'larının kümesidir; attachment policy'de **tag key-value** ile eşlenir. **Region başına NFG'de en fazla 1 attachment.** Desteklenen tipler: Connect, DX Gateway, **Transit Gateway Route Table**, VPC, VPN.
- **C doğru:** **`send-via`** = **east-west** inspection (segment içi / segmentler arası). Modlar: **single hop** (tek ara attachment) ve **dual hop** (hem kaynak hem hedef core network edge'de inspection; iki Region'da da inspection attachment gerekir).
- **E doğru:** **Inspection VPC'de appliance mode açık olmalı** — akış simetrisi için, TGW'deki mantığın aynısı.

- **D yanlış:** **`send-to`** = **north-south** (internet veya on-prem çıkışı). Segment içi VPC↔VPC trafiği east-west'tir → `send-via`.

**Ek kısıtlar:** bir attachment ya segment'te ya NFG'de olur, **ikisinde birden olmaz**. Dual-hop kullanılan bir NFG **single-hop veya send-to için yeniden kullanılamaz**. Core network policy'deki statik rotalar NFG route table'larına **otomatik yayılmaz**.
</details>

---

## Soru 14 — Network Firewall kural motoru

Bir ekip Network Firewall policy'sinde şunları istiyor:
- Belirli ülkelerden gelen trafiği bloklama
- HTTPS trafiğini port numarasından bağımsız tanıma
- Belirli domain'lere giden istekleri **reject** (drop değil) etme
- Bloklanan trafiğin loglanması

Hangi ifadeler doğrudur? (Üç cevap seçin)

- **A.** Geo-IP filtreleme Suricata `geoip` keyword'ü ile yapılır, ISO 3166 alpha-2 kodları kullanılır.
- **B.** Port'tan bağımsız HTTPS tanıma, stateful motorun stateful protokol tespiti ile yapılır.
- **C.** Domain list rule group'ları REJECT aksiyonunu destekler.
- **D.** Logging yalnızca stateful engine'e forward edilen trafik için mevcuttur.
- **E.** Geo-IP filtreleme stateless motorda yapılır.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevaplar: A, B, C ve D** — soru "üç seç" diyorsa **A, C, D** en spesifik olanlardır; E tek yanlış seçenektir.

- **A doğru:** `geoip:dst,CN,US;` biçiminde. Yön seçiciler **`src`, `dst`, `both`** (AND), **`any`** (OR). **ISO 3166 alpha-2** kodları; geçersiz kodlar kaydetme anında reddedilir. **MaxMind GeoIP** DB, **IPv4 ve IPv6** (AWS'in stock Suricata'ya eklentisi). AWS uyarısı: IP→ülke eşlemeleri değişir, geo allow kuralı kullanırken **default action'ı Drop** yapın ve daha yüksek priority'li açık deny kuralları ekleyin.
- **B doğru:** Stateful motor *"stateful protokol tespiti"* yapar — doküman örneği tam olarak **"portundan bağımsız olarak HTTPS trafiğini filtreleme"**dir.
- **C doğru:** **26 Eyl 2025**'ten beri domain list rule group'ları **`REJECT` ve `ALERT`** de destekliyor (eskiden yalnızca allow/deny). REJECT, TCP RST göndererek client'ı hemen bilgilendirir; drop sessizce düşürür.
- **D doğru ve kritik:** *"Firewall logging yalnızca **stateful rules engine'e forward ettiğiniz** trafik için mevcuttur."* → stateless-only düşüşler **loglanmaz**. Log tipleri: **FLOW, ALERT** (DROP/ALERT/REJECT aksiyonlu stateful kurallar), **TLS**.
- **E yanlış:** `geoip` bir **Suricata** keyword'üdür → **stateful** motorda çalışır. Stateless motor yalnızca 5-tuple match ayarlarını bilir, yön farkındalığı bile yoktur.
</details>

---

## Soru 15 — TGW kotası ve ölçek

Bir şirket tek bir TGW'de 800 VPC attachment'ı planlıyor. Her VPC'nin ortalama 3 CIDR bloğu var ve on-prem'den 400 prefix duyuruluyor. Ekip 8 route table kullanacak. Hangi kota problemi çıkar?

- **A.** TGW başına attachment kotası (5.000) aşılır.
- **B.** TGW başına toplam route kotası (10.000) risk altındadır: 800 × 3 = 2.400 VPC rotası, 8 tabloya propagate edilirse 19.200 route olur.
- **C.** TGW başına route table kotası (20) aşılır.
- **D.** VPC başına TGW kotası (5) aşılır.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: B**

Bu, en çok yanlış anlaşılan TGW kotasıdır. **Kota, route table başına değil, TGW BAŞINA TOPLAMDIR: 10.000 route** (tüm route table'lardaki rotaların toplamı).

Hesap: 800 VPC × 3 CIDR = **2.400 VPC prefix'i**. Bunlar 8 route table'a propagate edilirse **2.400 × 8 = 19.200 route** → kota **aşılır**. Buna on-prem'in 400 prefix'i × ilgili tablolar da eklenir.

**Çözümler:**
- Propagation'ı **seçici** yapmak (her attachment'ı her tabloya propagate etmemek) — segmentasyon zaten bunu gerektirir
- Route table sayısını azaltmak
- **Prefix summarization** (VPC CIDR'larını planlarken bitişik bloklar kullanmak)
- Kota artırımı talep etmek (SA/TAM üzerinden artırılabilir)
- Çok büyük ölçekte **Cloud WAN**'a geçmek

- **A yanlış:** 800 < 5.000. Attachment kotası rahat.
- **C yanlış:** 8 < 20. Route table kotası rahat.
- **D yanlış:** **VPC başına TGW = 5** kotası, bir VPC'nin kaç TGW'ye bağlanabileceğini sınırlar — 800 VPC'nin **tek** TGW'ye bağlanmasıyla ilgisi yok. Bu, kotayı ters okuma tuzağıdır.
</details>

---

## Soru 16 — Prefix limiti geçiş dönemi

Bir mimar 2026 ortasında bir hibrit tasarım yapıyor. On-prem'den AWS'e **480 prefix** duyurulacak, transit VIF kullanılıyor. Aşağıdakilerden hangisi **en doğru** yaklaşımdır?

- **A.** Prefix'leri summarize ederek 100'ün altına indir.
- **B.** VIF üzerinde inbound prefix control ile tahsisi artır; 1.000'e kadar mümkün.
- **C.** İki transit VIF kullan, prefix'leri böl.
- **D.** Transit VIF'i private IP VPN ile değiştir; route ölçeği 5.000'e çıkar.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: B**

**20 Ağustos 2026**'da AWS **inbound prefix controls**'ü duyurdu: private ve transit VIF'te müşteriden duyurulan prefix limiti varsayılan **100**'den, **IPv4 ve IPv6 için ayrı ayrı 1.000'e kadar** tahsis edilebilir hale geldi. Dedicated ve hosted bağlantılarda geçerli, **ek ücretsiz**.

- **A — 2026 öncesinin doğru cevabı, artık en iyisi değil:** Summarization hâlâ iyi bir pratiktir (route tablolarını küçültür, TGW'nin 10.000 route kotasını korur) ama artık **zorunlu ilk aksiyon değil**. Piyasadaki kursların hâlâ verdiği cevap budur.
- **C yanlış:** Aynı DXGW'ye iki transit VIF'in farklı prefix setleri duyurması yönetimsel karmaşa yaratır; ayrıca **dedicated bağlantı başına 4 transit VIF** limiti var ve prefix control varken gereksiz.
- **D yanlış — doğru bilgi, yanlış kullanım:** Private IP VPN'in route ölçeği gerçekten **5.000 çıkış / 1.000 giriş**'tir (yalnız DX'te 200/100) ama bu **VPN'in kendi route kotası**dır, VIF'in prefix limiti değil. Ayrıca sadece prefix sayısı için IPsec katmanı eklemek gereksiz karmaşıklık ve throughput kaybıdır.

**Sınav stratejisi:** Prefix limiti soruları geçiş döneminde. Senaryoda **tarih verilmişse** ona göre karar verin. Verilmemişse ve seçenekler arasında hem "summarize et" hem "prefix control ile artır" varsa, **daha az invaziv olan prefix control'ü** seçin.
</details>

---

## Soru 17 — Firewall Manager kapsamı

Bir organizasyonda 5 OU ve 400 hesap var. Güvenlik ekibi şunları istiyor:
- Tüm `prod` etiketli VPC'lerde baseline NACL
- Tüm hesaplarda kullanılmayan security group'ların raporlanması (düzeltme **yapılmadan**)
- Yeni hesapların otomatik kapsama girmesi
- İki etiketin (`env=prod` **VE** `tier=web`) birlikte eşleştiği kaynaklar için özel bir WAF politikası

Hangi ifadeler doğrudur? (Üç cevap seçin)

- **A.** `NETWORK_ACL_COMMON` politika tipi baseline NACL için kullanılır.
- **B.** `SECURITY_GROUPS_USAGE_AUDIT` politikası, remediation kapalı modda çalıştırılabilir.
- **C.** Kaynak etiketi kapsamı birden fazla etiketi AND/OR ile birleştirebilir.
- **D.** Yeni hesapların kapsama girmesi için her seferinde politikayı yeniden uygulamak gerekir.
- **E.** Baseline NACL için AWS Config gerekmez.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevaplar: A, B ve C**

- **A doğru:** **`NETWORK_ACL_COMMON`**, FMS'in 11 politika tipinden biridir ve org çapında baseline NACL'leri zorlar. **Bu politika tipi çoğu eski çalışma kaynağında YOKTUR** — bilmek fark yaratır.
- **B doğru:** Her FMS politikası için iki mod vardır: **uyumsuz hesapları/kaynakları yalnızca görüntüleme** veya **otomatik düzeltme**. `SECURITY_GROUPS_USAGE_AUDIT` kullanılmayan/gereksiz SG'leri bulur ve raporlama modunda çalıştırılabilir.
- **C doğru:** **9 Oca 2025**'ten beri kaynak-etiketi kapsamı **birden fazla etiketi AND veya OR** ile birleştirebiliyor, ayrıca **wildcard** operatörü (herhangi bir key veya value) var.

- **D yanlış:** FMS'in doğal davranışı tam tersi: *"Organizasyona yeni eklenen hesaplar **otomatik olarak** kapsama alınır ve politika kapsamdaki kaynaklarına uygulanır."*
- **E yanlış:** **AWS Config, FMS'in dokümante edilmiş bir ön koşuludur** — FMS kaynak uyumluluğunu Config ile değerlendirir. Diğer ön koşullar: **AWS Organizations + FMS administrator hesabı + AWS RAM resource sharing** (Network Firewall ve DNS Firewall politikaları için zorunlu) + Marketplace aboneliği (3. parti firewall için).
</details>

---

## Soru 18 — MTU zinciri

Bir uygulama on-prem'den TGW inter-Region peering üzerinden başka bir Region'daki VPC'ye 8.000 baytlık paketler gönderiyor. Küçük istekler çalışıyor, büyük transferler takılıyor. Yol: on-prem → transit VIF (MTU 8500) → DXGW → TGW-A → **peering** → TGW-B → VPC. Sorun ne ve nasıl teşhis edilir?

- **A.** Transit VIF MTU'su 1500'e düşmüş; jumbo frame etkinleştirilmeli.
- **B.** TGW peering attachment'ında PMTUD desteklenmediği için 8500 üstü paketler sessizce düşüyor; TGW Flow Logs'ta `packets-lost-mtu-exceeded` sayacı bunu gösterir.
- **C.** VPC route table'da eksik rota var.
- **D.** Security group 8000 baytlık paketleri blokluyor.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: B**

Yol boyunca MTU zinciri: transit VIF **8500**, TGW VPC attachment **8500**, TGW peering **8500**. 8.000 baytlık payload teorik olarak geçer — **ama** kapsülleme ek yükü eklendiğinde 8500'ü aşabilir ve **TGW peering'de PMTUD desteklenmez**. PMTUD olmadığı için gönderen taraf "paketi küçült" mesajı **almaz**; paketler sessizce düşer. Küçük istekler çalıştığı için bağlantı "var" görünür — klasik MTU black hole.

**Teşhis:** **TGW Flow Logs**'un dört kayıp sayacından biri tam bunu söyler: **`packets-lost-mtu-exceeded`**. Diğerleri: `packets-lost-no-route`, `packets-lost-blackhole`, `packets-lost-ttl-expired`. Bu alanlar **VPC Flow Logs'ta yoktur** — TGW Flow Logs'un ayırt edici değeri budur. Kaynak tipi `TransitGateway` veya `TransitGatewayAttachment`, toplama aralığı **60 sn'ye zorlanır**.

**PMTUD durumu hatırlatması:** VPC attachment ✅, Connect attachment ✅, **DX attachment ❌, VPN ❌, peering ❌**.

- **A yanlış:** Transit VIF 8500 destekler ve senaryo bunu veriyor. (Ama gerçek bir tuzak var: **iki private VIF aynı rotayı farklı MTU ile duyurursa veya aynı rotayı bir S2S VPN de duyurursa MTU 1500'e düşer.** Senaryoda böyle bir ipucu yok.)
- **C yanlış:** Rota eksikse **hiçbir** trafik geçmez, küçük istekler de çalışmazdı. "Küçük çalışıyor, büyük takılıyor" ifadesi routing'i eler ve doğrudan MTU'ya işaret eder.
- **D yanlış:** Security group'lar paket boyutuna göre filtre yapmaz — port, protokol ve IP'ye göre çalışır.
</details>

---

## Soru 19 — Route 53 Profiles vs RAM

Bir organizasyonda 150 VPC var. Her VPC'nin şunlara erişmesi gerekiyor: 12 PHZ, 20 resolver rule, 3 DNS Firewall rule group, 8 merkezî interface VPC endpoint ve query logging. Ekip önce RAM + tek tek ilişkilendirme yaklaşımını denedi ve bir kotaya takıldı. Hangi kota ve doğru çözüm nedir?

- **A.** Region başına resolver rule kotası (1.000) aşıldı → summarization yap.
- **B.** Rule↔VPC association kotası (Region başına 2.000) aşıldı (150 × 20 = 3.000) → Route 53 Profile kullan, VPC başına tek ilişkilendirme.
- **C.** VPC başına PHZ kotası aşıldı → PHZ'leri birleştir.
- **D.** DNS Firewall rule group kotası (VPC başına 5) aşıldı → rule group'ları birleştir.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevap: B**

Hesap: 150 VPC × 20 resolver rule = **3.000 rule↔VPC association**. Kota **Region başına 2.000**'dir → aşılır.

**Route 53 Profile** bu fan-out problemini çözer: **VPC başına YALNIZCA 1 Profile** ilişkilendirmesi. Profile'ın içeriği: **PHZ'ler, resolver rule'lar (forwarding + system), DNS Firewall rule group'ları, interface VPC endpoint'ler, query logging yapılandırmaları**, artı Profile seviyesinde reverse-DNS, DNS Firewall failure mode ve DNSSEC validation.

Bu senaryo Profile kotalarına rahat sığar: Profile başına **1.000 VPC**, **5.000 PHZ**, **1.000 resolver rule**, **5 DNS Firewall rule group**, **2 query logging config**. 150 VPC → 150 ilişkilendirme (2.000'lik kotanın çok altında ve o kota artık geçerli değil, Profile ayrı sayılır).

**Ücret:** **100 ilişkilendirmeye kadar hesap başına $0,75/saat**, sonrası ilişkilendirme başına $0,0014/saat. 150 için ≈ $0,75 + 50 × $0,0014 ≈ **$0,82/saat**.

Bonus: 8 merkezî interface VPC endpoint de Profile'a girer (**Nis 2025** özelliği) — eski "her spoke'ta PHZ + alias kaydı" deseni artık AWS tarafından **önerilmiyor**.

- **A yanlış:** Region başına **rule** kotası 1.000'dir ve burada 20 kural var — aşılmıyor. Aşılan **association** kotasıdır. Ayrıca kota muhasebesi nüansı: *rule sayısı* **oluşturana**, *rule↔VPC association* **tüketiciye** sayılır.
- **C yanlış:** Bir VPC'ye ilişkilendirilebilen PHZ sayısı bu ölçekte darboğaz değil (Profile başına 5.000).
- **D yanlış:** **VPC başına 5** DNS Firewall rule group kotası var ve senaryoda **3** kullanılıyor — aşılmıyor.
</details>

---

## Soru 20 — Bütünleşik final sorusu

Bir bankanın mimarisi: 3 Region, 120 VPC, 2 veri merkezi (her birinde 2 DX bağlantısı, farklı lokasyonlarda), 90 şube (VPN). Yeni gereksinimler:
1. DX SLA **%99,99** olmalı
2. VPC↔VPC TGW trafiği şifreli olmalı
3. Tüm East-West trafiği inspection'dan geçmeli, **inspection VPC işletmek istemiyorlar**
4. 90 şube tek attachment'ta toplanmalı
5. Uyumluluk için TGW'de kaybolan paketlerin sebebi raporlanabilmeli

Aşağıdaki hangi **beş** aksiyon gereklidir? (Beş cevap seçin)

- **A.** Her veri merkezinde DX bağlantı sayısını 2'den 4'e çıkar (lokasyon başına 2), Enterprise Support'a geç ve Well-Architected Review tamamla.
- **B.** TGW'de `modify-transit-gateway` ile encryption support'u etkinleştir; önce peering ve Connect attachment'larını kaldır.
- **C.** Network Firewall'ı native TGW attachment olarak dağıt.
- **D.** VPN Concentrator kur (1 concentrator, 90 site).
- **E.** TGW Flow Logs'u etkinleştir ve `packets-lost-*` sayaçlarını raporla.
- **F.** MACsec'i tüm DX bağlantılarında `must_encrypt` modunda etkinleştir.
- **G.** Inspection VPC'de GWLB + üçüncü parti appliance kur.

<details>
<summary><b>Cevap ve analiz</b></summary>

**Doğru cevaplar: A, B, C, D ve E**

- **A → gereksinim 1.** %99,99 SLA (24 Tem 2026 metni): **≥4 bağlantı, ≥2 farklı DX lokasyonu, lokasyon başına ≥2 bağlantı, her bağlantı farklı AWS cihazında, + Enterprise Support + tamamlanmış Well-Architected Review.** Mevcut durum (lokasyon başına 1 bağlantı) **%99,9**'dur.
- **B → gereksinim 2.** TGW Encryption Support (Kas 2025). **Peering, Connect, Network Firewall, VPN Concentrator, Client VPN attachment'ları ve security group referencing/multicast uyumsuzdur** ve etkinleştirmeden önce var olmamalıdır. Sadece `modify-transit-gateway` ile açılır (oluşturmada değil), **14 güne kadar** sürebilir.
  > **Dikkat — burada gerçek bir mimari çelişki var:** Network Firewall attachment'ı da encryption support ile uyumsuzdur. Yani C ve B aynı TGW'de birlikte olamaz. Gerçek sınavda bu tür çelişkiler "hangisi öncelikli" ya da "hangi Region'da hangisi" şeklinde ayrıştırılır. **Bu çelişkiyi fark etmek, sorunun asıl testi.** Pratikte çözüm: şifrelemeyi gerektiren VPC'leri ayrı bir TGW'de tutmak veya inspection'ı GWLB ile inspection VPC'de yapmak.
- **C → gereksinim 3.** "Inspection VPC işletmek istemiyorlar" ifadesinin tek cevabı **native TGW attachment** (Haz 2025): inspection VPC yok, firewall subnet'i yok, endpoint yok, VPC route table plumbing'i yok, **appliance mode her zaman açık**.
- **D → gereksinim 4.** VPN Concentrator **100'e kadar** site → 90 şube **tek concentrator**, tek attachment. BGP zorunlu; **ECMP yok, private IP VPN yok, VGW yok, mevcut VPN'ler taşınamaz**.
- **E → gereksinim 5.** TGW Flow Logs'un dört sayacı: `packets-lost-no-route`, `packets-lost-blackhole`, `packets-lost-mtu-exceeded`, `packets-lost-ttl-expired`. **VPC Flow Logs'ta bu alanlar yoktur.** Toplama TGW'de **60 sn'ye zorlanır**.

- **F yanlış:** MACsec **SLA ile ilgisizdir** ve gereksinim 2 **VPC↔VPC TGW trafiğini** istiyor — MACsec DX linkini korur, **noktadan noktaya**dır ve TGW içi trafiği hiç kapsamaz. Ayrıca `must_encrypt` modu, müzakere başarısız olursa **tüm trafiği düşürür** — bir bankada erişilebilirlik riski.
- **G yanlış:** Gereksinim 3 açıkça **"inspection VPC işletmek istemiyorlar"** diyor. GWLB mimarisi bir inspection VPC gerektirir. (Doğru cevap C ile çelişir.)
</details>

---

## Soru bankası özeti — en çok tekrarlanan tuzaklar

Bu 20 soruda test edilen tuzakların sıklık sıralaması:

| Sıra | Tuzak | Kaç soruda |
|---|---|---|
| 1 | **Doğru bilgi, yanlış yer/bağlam** (rule group admin hesapta vs üye hesapta; route ölçeği VPN'in vs VIF'in) | 5 |
| 2 | **Longest prefix match'in her şeyin önünde olması** | 3 |
| 3 | **Tek akış = tek tünel/tek yol** (ECMP tek akışı hızlandırmaz) | 2 |
| 4 | **MACsec ≠ uçtan uca, hosted'da yok** | 3 |
| 5 | **Private DNS açık = merkezî endpoint çalışmaz / gateway endpoint bypass edilir** | 2 |
| 6 | **Appliance mode'un gizli şartı: route propagation açık** | 2 |
| 7 | **Statik/kavramsal analiz vs veri düzlemi** (Reachability paket göndermez) | 2 |
| 8 | **Kotayı ters okuma** (VPC başına TGW vs TGW başına attachment; TGW başına toplam route) | 2 |
| 9 | **"Hatasız başarısızlık"** (NLB'de UDP 4789 listener'ı yok) | 1 |
| 10 | **Uyumsuz özellik çiftleri** (encryption support + peering/NFW; appliance mode + 2/3-tuple) | 3 |

---

[← Bölüm 8: Karar matrisleri](08-karar-matrisleri.md) · [Sonraki bölüm: Son tekrar cheatsheet →](10-son-tekrar-cheatsheet.md)
