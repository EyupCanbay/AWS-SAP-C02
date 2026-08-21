# Bölüm 10 — Son tekrar: tek sayfa cheatsheet

> **Kullanım:** Sınavdan 1–2 saat önce yalnızca bu bölümü okuyun. Yeni bir şey öğrenmeye çalışmayın; sayıları tazeleyin.

---

## ⚡ Transit Gateway

**Association** = giriş, **1 route table**. **Propagation** = duyuru, **N route table**. Tek yön = tek yönlü blackhole.

**Route sırası:** longest prefix → statik > propagated → sabit tip sırası → AS_PATH → MED
**Tip sırası (1→10):** VPN statik · prefix-list · **VPC** · **DXGW** · Connect · private-DX-VPN · VPN · VPN Concentrator · Client VPN · peering/Cloud WAN
Varsayılan MED: **DX = 0**, VPN ve Connect = **100**

**Appliance mode:** yalnızca **VPC attachment**; 5-tuple hash; cross-Region çalışır; **gizli şart: route propagation açık**; GWLB'de **5-tuple stickiness zorunlu**; native NFW attachment'ta **her zaman açık**.

**Peering:** **statik route only**, BGP yok, ECMP yok, multicast yok, **transitive değil**, AES-256 çift şifreli, MTU 8500, PMTUD yok, TGW başına 50, aynı çift arasında 1, ayrı ASN önerilir.

**Connect:** GRE+BGP, transport = **VPC veya DX** (VPN değil), peer başına **2 BGP oturumu**, `ebgp-multihop TTL=2`, inside CIDR `169.254.0.0/16`'dan **/29**, attachment başına **4 peer**, tünel **5 Gbps** → attachment **20 Gbps**, **BFD/graceful-restart/statik route/multicast YOK**.

**Multicast:** **IGMPv2 only** (v3/SSM yok), TGW oluşturmada açılmalı, subnet 1 domain, **yalnızca VPC attachment**, non-Nitro gönderici olamaz, fragmentasyon yok, akış 1 Gbps / AZ 20 Gbps, 3 kaçırılan QUERY = düşüş, **IPv6 yalnızca statik**.

**Kotalar:** 5 TGW/hesap · **5.000 attachment/TGW** · **5 TGW/VPC** · 20 route table · 5 CIDR · **TGW BAŞINA TOPLAM 10.000 ROUTE** · 50 peering · 4 Connect peer · 5 concentrator (100 site)

**Bant genişliği (AZ/yön):** VPC / DX / peering = **100 Gbps, 7,5M PPS** · Connect tünel 5 Gbps · VPN standard 1,25 / **LBT 5** · concentrator 100 Mbps

**2025-26:** native NFW attachment · Encryption Support (peering/Connect/NFW/concentrator/Client VPN/SG-referencing/multicast **uyumsuz**, `modify` ile, **14 gün**, use1-az3 yok) · Flexible Cost Allocation (**50 kural**, TGW başına 1 policy) · Client VPN native (**SNAT yok**, /12–/22) · SG referencing (**peering üzerinden yok**, inbound only, GWLB/NFW yolunda çalışmaz)

---

## ⚡ Direct Connect

**Dedicated:** 1, 10, 100, **400 Gbps** (400G seçili lokasyon). **Port hızı değişmez.**
**Hosted:** 50–500 Mbps, 1, 2, 5, 10, **25 Gbps**. **400G hosted YOK.** **Tip başına 1 VIF.**

| | Private VIF | Public VIF | Transit VIF |
|---|---|---|---|
| Bağlanır | VGW / DXGW | gateway YOK | **yalnızca DXGW** |
| MTU | 1500 / **9001** | **1500** | 1500 / **8500** |
| SiteLink | ✅ (DXGW ile) | ❌ | ✅ |
| Accelerated VPN | — | ❌ | — |

**Prefix:** private/transit **1.000'e kadar** (eskiden 100) · public **1.000 sabit** · Cloud WAN DXGW→on-prem **5.000**
**VIF sayısı:** dedicated **50 + 4 transit = 51** (hızdan bağımsız) · hosted **1**

**DXGW:** global (Çin hariç), **veri yolunda değil**, **transitive DEĞİL**. 20 VGW · **6 TGW** · 30 VIF · TGW/DXGW için 20 · TGW association allowed prefix **200**.
**VGW/TGW allowed prefix farkı:** VGW'de **filtre** (VPC CIDR'ından geniş/eşit olmalı, duyurulan VPC CIDR'ı) · TGW'de **doğrudan duyurulan liste** (VPC CIDR'ları otomatik duyurulmaz!)
**Mod dışlaması:** VGW ve TGW aynı DXGW'de olmaz. Cloud WAN'a bağlıysa başka association yok, **tek segment**.

**LAG:** max **4** (<100G) / **2** (100G, 400G) · **sadece dedicated** · aynı hız · **aynı cihaz** (resiliency DEĞİL) · `minimumLinks` altına düşerse **tamamen düşer** · MACsec'te **tek anahtar aktif**

**SLA (24 Tem 2026):** **%99,99 = ≥4 bağlantı + ≥2 lokasyon + lokasyon başına ≥2 + farklı cihaz + Enterprise Support + WA Review** · **%99,9 = ≥2 bağlantı + ≥2 lokasyon + Enterprise Support** · tek bağlantı **%95,0** · "unavailable" = **120 ardışık saniye**

**BGP çıkış sırası:** longest prefix → **LOCAL PREF** → AS_PATH → MED → ECMP
**Local pref community:** **7224:7300 HIGH** · **7224:7200 MEDIUM (varsayılan)** · **7224:7100 LOW**
**Public VIF scope (siz):** 7224:9100 Region · 9200 kıta · **9300 global (varsayılan)**
**Public VIF (AWS→siz):** 7224:8100 aynı Region · 8200 aynı kıta · etiketsiz diğer kıta · **hepsine NO_EXPORT (65535:65281)** · AWS min **AS_PATH 3**
**4-byte ASN:** tüm VIF'lerde (Eyl 2025), 1–4294967294, **4200000000+ ileriye 7224 ile değiştirilir**

**BFD:** AWS **300 ms × 3 ≈ 900 ms** · otomatik etkin **ama router'da yapılandırmalısın** · BGP hold **90 sn** varsayılan / min 3 · **timer'lar en düşük ortak değere, BFD YAVAŞ tarafa uyar** · graceful restart ile birlikte önerilmez

**MACsec:** **10/100/400G DEDICATED only, hosted'da YOK** · CKN/CAK **hex 64 karakter**, **AES-256, static CAK only** · 3 çifte kadar rotasyon · 10G: GCM-AES-256 veya XPN-256; **100G/400G: XPN-256 zorunlu** · SCI **her zaman açık** · `no_encrypt` / **`should_encrypt`** (varsayılan, başarısızsa şifresiz devam) / **`must_encrypt`** (başarısızsa tüm trafiği düşürür) · **ek ücretsiz** · **NOKTADAN NOKTAYA, uçtan uca DEĞİL**

**SiteLink:** private + transit VIF (**public VIF YOK, VGW'ye bağlı private VIF YOK**) · **DXGW zorunlu** · dedicated + hosted · **$0,50/saat/VIF** + GB matrisi · şube başına tekil ASN veya `allowas-in` · **multicast yok, QoS yok**

**2026:** prefix controls (20 Ağu) · BGP route visibility (30 Tem, `ListVirtualInterfaceRoutes`) · **VIF Rate Limiters** (1 Haz, sadece dedicated, 10/bağlantı, min 50 Mbps, oversubscription OK) · CloudFormation (31 Mar) · BGP CloudWatch metrikleri (30 Mar)

---

## ⚡ Site-to-Site VPN

**Tünel:** standard **1,25 Gbps / 140K PPS** · **LBT 5 Gbps / 400K PPS** (TGW+Cloud WAN only, **VGW'de YOK**) · concentrator **100 Mbps / 10K PPS**
**ECMP iki şart:** **BGP (dinamik)** + **TGW'de VPN ECMP support açık**. Statik'te YOK, VGW'de YOK.
**5-tuple hash → TEK AKIŞ TEK TÜNELE SABİTLENİR.** Daha fazla tünel tek akışı hızlandırmaz.

**Toplamlar:** 1 std ECMP kapalı **1,25** · 1 std ECMP açık **2,5** · 1 LBT **10** · 2 LBT **20** · 1 LBT+1 std **12,5**

**Route:** TGW içeri **1.000** / dışarı **5.000** · VGW içeri **100** / dışarı **1.000** / statik **100**
**Kota:** 50 CGW · 5 VGW · **50 VPN connection** · VGW başına 10 · **10 accelerated** · **50 LBT** · 50 concentrator · TGW başına 5 concentrator · concentrator başına **100 site**

**Tünel ayarları:** IKEv1 **ve** IKEv2 (ikisi varsayılan) · Phase1 lifetime **28.800**, Phase2 **3.600** · rekey margin **270**, fuzz **100%** · replay window **1.024** · DPD timeout **40 sn**, action `clear`(varsayılan)/`none`/`restart` · startup action **`add`** (CGW başlatır) / `start` (AWS başlatır) · inside IPv4 **/30** (`169.254.0.0/16`), IPv6 **/126** (`fd00::/8`) · PSK **8–64 karakter, `0` ile başlamaz** · outside IP: `PublicIpv4` / `PrivateIpv4` / **`Ipv6`**

**ELC:** **tünel bazında opt-in**, API varsayılanı **kapalı**, AWS Health bildirimi, `get-vpn-tunnel-replacement-status`, **kritik güncellemeler yine anında**

**Accelerated:** **yalnızca TGW** (VGW YOK) · **mevcut bağlantıda açılamaz** · **NAT-T zorunlu** · **IKE'yi CGW başlatır** · **DX public VIF ile ÇALIŞMAZ** · accelerator'lar **görünmez/yönetilemez** · IKE fragmentation desteklemeyen CGW'de **açma** · 10/Region

**Private IP VPN:** **TGW (private CIDR) + DXGW + TRANSIT VIF** · VGW YOK · RFC1918/RFC6598 · route **5.000/1.000** · LBT destekler · şifreli+şifresiz aynı DX'te eş zamanlı

**CloudHub:** **VGW tabanlı** · **CGW başına tekil ASN** · **BGP zorunlu** · prefix'ler çakışmamalı · VPC gerekmez · VGW'den siteye röle veri **ücretli**

**CGW ASN:** public 1–4294967295, **rezerve: 7224 (tüm), 9059 (eu-west-1), 10124 (ap-northeast-1), 17943 (ap-southeast-1)** · private **64512–65534** ve **4200000000–4294967294** · varsayılan **64512** · **private certificate + public VPN = outside IP GEREKMEZ**

**VPN Concentrator (Kas 2025):** **100 site / 1 attachment**, 25+ site için önerilir, **BGP only**, **ECMP YOK, private IP VPN YOK, VGW YOK, dual-stack YOK, mevcut VPN taşınamaz**
**Post-quantum: Site-to-Site VPN'de YOK.**

---

## ⚡ PrivateLink / Endpoints

**Gateway endpoint:** **yalnızca S3 + DynamoDB** · **PrivateLink kullanmaz** · route table'a AWS prefix-list rotası · **ÜCRETSİZ** · ENI yok, SG yok · policy'de `Principal` **`*` olmak zorunda** · **on-prem/peering/TGW/cross-Region'dan ERİŞİLEMEZ**

**Interface endpoint:** ENI + private IP + **SG zorunlu (443)** · private DNS için `enableDnsSupport` **ve** `enableDnsHostnames` true · **MTU 8500, PMTUD YOK, MSS clamping** · **10 → 100 Gbps/AZ** · **ping'e yanıt vermez** · shared subnet'te oluşturulamaz · policy **20.480 karakter** · **$0,01/saat/ENI + $0,01/GB**

**S3 çift endpoint:** gateway (VPC içi, ücretsiz) + interface with **"private DNS only for inbound endpoint"** (on-prem) — bu ayar **YALNIZCA S3'te var**
Endpoint'e özel S3 adı: `bucket|accesspoint|control.vpce-xxxx-yyyy.s3.<region>.vpce.amazonaws.com`

**Endpoint service:** **NLB veya GWLB** · **ALB doğrudan OLMAZ** → ALB'yi NLB'ye ALB-type target yap (cross-zone kapalı **50** / açık **100** target sayılır; target group başına **1 ALB**) · kaynak IP = **LB düğüm IP'leri** → **Proxy Protocol v2** (TLV'de **vpce-id**) · allowed principals (root/role/user/`*`) · **izin kaldırmak mevcut bağlantıyı koparmaz** · private DNS name **wildcard + domain doğrulama** · IPv6: **NLB dualstack olmalı**, backend IPv4 kalabilir

**Cross-Region PrivateLink:** müşteri servisleri **Kas 2024** · **AWS servisleri Kas 2025** (S3, IAM, ECR api+dkr, KMS, ECS, Lambda, Firehose, MSK-Flink, Route 53) · **`vpce:AllowMultiRegion`** · sağlayıcı **≥2 AZ**, AZ hizalaması gerekmez · desteklenmeyen AZ: **use1-az3, usw1-az2, apne1-az3, apne2-az2, apne2-az4** · **regional DNS only** · **yalnızca interface** · TCP idle timeout özel NLB'de yok · UDP fragmentation ile yok · **Region'lar arası failover YÖNETİLMEZ** · sağlayıcı **+$0,05/saat/Region**

**VPC Lattice (Ara 2024):** resource gateway (**load balancing YAPMAZ**, ENI başına IPv4 **1–62, varsayılan 16, DEĞİŞTİRİLEMEZ**, IPv6 **/80**) · resource configuration (`SINGLE`/`GROUP`/`CHILD`/**`ARN`** — ARN **yalnızca RDS**) · **resource endpoint LOAD BALANCER GEREKTİRMEZ, on-prem hedefleri destekler** · service network endpoint (VPC başına **1** service network doğrudan) · **$0,02/saat/kaynak**

**NAT gateway:** **5 → 100 Gbps**, **1M → 10M PPS**, **hedef başına 55.000 bağlantı**, **8 IPv4/NAT GW**, port 1024–65535, MTU 8500 · **VGW üzerinden VPN/DX trafiği NAT GW'ye YÖNLENDİRİLEMEZ (TGW ile YÖNLENDİRİLİR)** · **peering üzerinden de gidilemez**

**Kotalar:** **interface + GWLB endpoint BİRLİKTE VPC başına 50** · gateway endpoint 20/Region · resource endpoint 200/VPC · service network endpoint 50/VPC · NLB **3.000 target** (**cross-zone açıksa 500**), AZ başına 500, 50 listener

---

## ⚡ Route 53 Resolver / DNS

**VPC Resolver:** CIDR **taban + 2** · link-local **169.254.169.253** / **fd00:ec2::253**
**`enableDnsSupport`** varsayılan **true**, **`enableDnsHostnames`** varsayılan **false** (default VPC hariç). İnterface endpoint private DNS ve autodefined reverse-DNS için **ikisi de true**.
**1.024 PPS = Resolver + IMDS + NTP + Windows lisanslama TOPLAMI. Artırılamaz.**

**INBOUND = sorgu içeri (on-prem→AWS).** On-prem'de conditional forwarder.
**OUTBOUND = sorgu dışarı (AWS→on-prem).** AWS'te FORWARD rule.
**≥2 IP, ≥2 AZ** · **IP başına 10.000 QPS** ama **SG connection tracking veya NLB → 1.500** · 40.000+ IP+port kombinasyonu ENI'yi zorlar · **ölçekleme = IP EKLEME** (endpoint başına 6) · **4 endpoint/Region** · **$0,125/saat/ENI + $0,40/M sorgu**
Protokoller: inbound Do53/DoH/DoH-FIPS, outbound Do53/DoH · SG: TCP+UDP **53** (+443 DoH) · kategori **Default / Delegation** (delegation'da **Do53 only**)

**Rule tipleri:** **FORWARD · SYSTEM · RECURSIVE · DELEGATE** (DELEGATE Haz 2025)
**Eşleşme:** tam eşleşme VEYA **alt domain**. `acme.example.com` kuralı → `zenith.acme.example.com` ✅, `example.com` ❌, `nadir.example.com` ❌
**ÖNCELİK: numerik priority YOK — EN SPESİFİK domain kazanır.**
**SYSTEM = FORWARD'dan istisna oyma.** Tek işi bu.

**RAM paylaşımı:** kendi oluşturduğun FORWARD kuralları · tüketici **değiştiremez/silemez** · kota: *rule sayısı* oluşturana, *rule↔VPC association* **tüketiciye** · **OU değişirse o hesabın tüm ilişkilendirmeleri SİLİNİR**

**Cross-account PHZ — 3 adım, SADECE CLI:**
`create-vpc-association-authorization` → `associate-vpc-with-hosted-zone` → `delete-vpc-association-authorization`
**VPC başına 1 authorization.** 3. adım MEVCUT ilişkilendirmeyi bozmaz.
**AWS tavsiyesi:** PHZ'leri tüm VPC'lere **doğrudan paylaş/ilişkilendir**; query forwarding son çare.

**Route 53 Profiles:** PHZ + resolver rule + DNS FW rule group + **interface VPC endpoint** + query logging + reverse-DNS/failure-mode/DNSSEC · RAM, **aynı Region** · **VPC başına 1 Profile** · **5 Profile/hesap-Region** · 1.000 VPC · 1.000 rule · 5.000 PHZ · 5 DNS FW group · 2 query log config · **$0,75/saat (≤100 assoc.)**

**DNS Firewall:** VPC içi + inbound endpoint'ten geçen sorgular · **yalnızca DOMAIN ADI** (IP'ye çözmez, HTTPS/SSH/TLS/FTP filtrelemez) · rule group **priority sırası**, grup içi **priority**, **ilk eşleşen kazanır** · ALLOW/BLOCK/ALERT; BLOCK yanıtı **NODATA/NXDOMAIN/OVERRIDE** · **varsayılan FAIL-CLOSED** · kural ya domain list ya **Advanced** (ikisi birden olmaz) · query type kural başına 1 · **Advanced: DGA + DNS tunneling + dictionary DGA**, eşik High/Med/Low, **$0,16/saat/VPC** · foundation **$0,60/M** · domain **$0,0005/ay**
**Kota:** 5 rule group/VPC · 1.000 rule group/hesap-Region · **100 kural/rule group** · 1.000 domain list · **toplam 100.000 domain**

**Global Resolver (GA Mar 2026):** anycast, internetten erişilir, **VPC/endpoint/DX/VPN GEREKMEZ**, public + **PHZ** çözer, **DNS Views** (RAM ile paylaşılır, Haz 2026) · **2 global resolver/hesap** · 5 view/resolver · 1.000 PHZ/view · **$5,00/saat (ilk 2 Region)**, +$1,50/Region · **VPC Resolver'ın yerini ALMAZ**

**EC2 conditional forwarder deseni artık GEREKSİZ.**

---

## ⚡ İzleme

**Flow Logs:** v2(14 alan) → v3 → v4 → v5 → **v6 (TGW)** → v7 (ECS) → **v8 `reject-reason` (BPA/EC)** → **v9 `resource-id`** → **v10 `encryption-status` (0/1/2/3)** → **v11 (Haz 2026: instance/interface/ASG etiketleri + next-hop alanları + `interface-type`)**
Toplama **60 veya 600 sn** (varsayılan **600**) · **TGW'de 60'a zorlanır** · **Nitro'da ≤60** · hedef: CloudWatch Logs / S3 / **Firehose** · cross-account **S3 + Firehose** · **250 subscription/kaynak** · **format DEĞİŞTİRİLEMEZ** · org çapında etkinleştirme (Ağu 2025)
`traffic-path`: 1 aynı VPC · **2 IGW/gateway-endpoint** · **3 VGW** · 4 intra-Region peering · 5 inter-Region peering · 6 LZ/WZ · 7 gateway endpoint · 8 IGW
`tcp-flags`: FIN 1 · SYN 2 · RST 4 · **SYN-ACK 18**
**YAKALANMAZ:** Amazon DNS · Windows aktivasyon · **169.254.169.254** · **169.254.169.123** · DHCP · **mirror KAYNAK trafiği** · router rezerve IP · **vpce ENI ↔ NLB ENI** · ARP · kısa ömürlü regional NAT GW

**TGW Flow Logs — 4 sayaç:** `packets-lost-no-route` · `packets-lost-blackhole` · `packets-lost-mtu-exceeded` · `packets-lost-ttl-expired`
**Connect attachment'ta doğrudan yok** → transport attachment altında

**Traffic Mirroring:** kaynak **`interface` ENI** (requester-managed: **yalnızca RDS + ElastiCache**) · **instance-tipi listesi** (blanket Nitro-only DEĞİL) · hedef ENI / **NLB (UDP 4789 listener ZORUNLU — yoksa hatasız başarısız)** / GWLBE · **VXLAN UDP 4789** · filtre **yön bazlı, 10 kural, ilk eşleşen** · `amazon-dns` seçeneği · session **1–32766**, **3/kaynak ENI**, 10.000/hesap · hedef başına kaynak **10 veya 100** (NLB/GWLBE limitsiz) · **BANT GENİŞLİĞİNDEN DÜŞER (1+1 → 4 Gbps)**, limit aşımında **önce mirror düşer**, akış başına **<2,5 Gbps** tut · cross-VPC: **intra-Region peering / TGW / GWLBE** (inter-Region peering YOK) · aynı VPC farklı hesap = **VPC sharing** · `PacketLength` NLB varsayılan **8500** · overhead **+54/+74** · kesilmesiz max MTU **8947** · **IPv6-only subnet'te YOK** · mirror edilemez: ARP, DHCP, IMDS, NTP, Windows aktivasyon

**Reachability Analyzer:** **PAKET GÖNDERMEZ** (statik) · **CROSS-REGION YOK** · cross-account VAR (Organizations trusted access; delegated admin **management account'u analiz edemez**; us-east-1 CloudFormation izni; **cross-account ek ücretsiz**) · **$0,10/analiz** · **IPv4 only** · **120 gün sonra silinir** · **analiz etmez:** TGW Connect attachment, TGW policy table, NFW resource-group/**domain list**/**Suricata**, **GWLBE arkası LB+target**, target health, BYOIP durumu, traffic mirroring
**Haz 2025: Amazon Q ağ troubleshooting KALDIRILDI.**

**Network Access Analyzer:** kaynak-hedef **VERMEZSİN** → **Access Scope** (MatchPaths / ExcludePaths) · 4 Amazon scope (IGW-ingress/egress, VPC-ingress/egress) · **$0,002/ENI** · statik, aynı Region

**Monitor'lar:** **Synthetic Monitor** = AWS→on-prem **aktif prob**, paket kaybı + RTT + **NHI (DX ile)**, **ajan yok**, prob başına ücret, subnet aynı hesapta, `169.254.0.0/16` ve VPC `.2` prob edilemez, **otomatik failover yok** · **Flow Monitor** = **ajan**, TCP **RTT/retransmit/timeout/bayt**, **NHI = `Degraded`**, workload insights + monitors · **Internet Monitor** = pasif, AWS footprint, **VPC/NLB/CloudFront/WorkSpaces**, birim **city-network**, EventBridge + Health Dashboard

---

## ⚡ GWLB / Network Firewall / FMS

**GWLB:** **GENEVE UDP 6081** · **kaynak/hedef IP KORUNUR** · TLV: **GWLBE/VPCE ID + 32-bit flow cookie + attachment ID** · appliance: GENEVE ile yeniden kapsülle, **yalnızca dış IP takas, PORTLARI TAKAS ETME**, **flow cookie'yi DEĞİŞTİRME**, **aynı GWLB'ye** dön
**Stickiness:** varsayılan **5-tuple** (`stickiness.enabled=false`) · 3-tuple `source_ip_dest_ip_proto` · 2-tuple `source_ip_dest_ip` · **appliance mode açıkken 2/3-tuple DESTEKLENMEZ**
`target_failover.on_deregistration` / `on_unhealthy`: **`rebalance`/`no_rebalance`**, varsayılan **`no_rebalance`**, **ikisi aynı olmalı**
**GWLBE:** **tam 1 AZ, subnet DEĞİŞTİRİLEMEZ** · AZ başına 1/endpoint service · **cross-account: PrivateLink modeli, RAM DEĞİL** · **10 → 100 Gbps/AZ** · **NLB client IP preservation ÇALIŞMAZ** · aynı subnet'te NACL değerlendirilir · **egress-only IGW ile IPv6 DÜŞER** · route varken silinemez
**Target:** `instance` (ID) · `ip` (**yalnızca target group VPC subnet'leri + RFC1918 + 100.64.0.0/10**, public IP YOK)
**Health check:** varsayılan **TCP:80**, timeout 5, interval 10, healthy 5, unhealthy 2 · **TÜMÜ UNHEALTHY = FAIL-OPEN YOK, trafik düşer**
**MTU 8500, GENEVE +68 → appliance ≥8568, fragmentation YOK, PMTUD YOK** · subnet'te **≥8 IP**, subnet **kaldırılamaz** · **cross-zone varsayılan KAPALI**

**Network Firewall:** stateless (5-tuple, **yön farkındalığı yok**) + stateful (**Suricata**, protokol tespiti, DPI)
**Sıra:** `DEFAULT_ACTION_ORDER` (**pass > drop > reject > alert**, grup içi `priority`) · **`STRICT_ORDER`** (rule group priority + **yazım sırası**, **policy default action'ları açar**: Drop/Alert all/established + application-layer varyantları). **AWS strict order önerir; API varsayılanı ACTION ORDER.** Managed rule group **varyantı eşleşmeli**.
**Kapasite:** stateless = match ayarlarının **ÇARPIMI** (30×3×5=**450**) · stateful ≈ **kural sayısı** · max **30.000** · **DEĞİŞTİRİLEMEZ** · policy başına **20 stateful + 20 stateless** rule group, 30.000+30.000 kural
**Endpoint:** AZ başına primary · **kendi subnet'i HARİÇ** her subnet'i korur · **VPC endpoint association (May 2025): AZ'de birden fazla endpoint, 50/firewall/AZ, 300/hesap-Region** · **Firewall RAM ile paylaşılabilir**
**Domain list: REJECT + ALERT** (Eyl 2025) · **`aws_url_category` / `aws_domain_category`** (Oca 2026) · **JA4** (Mar 2025) · **flow flush/capture** (Mar 2025) · traffic analysis reports (Şub 2025)
**TLS inspection: inbound + outbound + ikisi** (ingress Mar 2023, egress tüm Region Ara 2023) · ACM sertifikası · session holding (Eyl 2025) **application-established ile uyumsuz** · TLS altında **HTTP2 kuralları da gerekir**, **HTTP/2 overloading YOK**
**Geo-IP VAR:** `geoip:dst,CN,US;` · `src`/`dst`/`both`(AND)/`any`(OR) · **ISO 3166 alpha-2** · MaxMind · IPv4+IPv6 · geo-allow'da **default action Drop** yap
**Logging: FLOW / ALERT / TLS** · ALERT = DROP/ALERT/REJECT stateful kurallar · **YALNIZCA STATEFUL ENGINE'E FORWARD EDİLEN TRAFİK** → stateless-only düşüşler **loglanmaz**
**Native TGW attachment (Haz 2025):** **inspection VPC/subnet/endpoint/route-plumbing YOK** · **appliance mode HER ZAMAN AÇIK** · TGW RAM ile paylaşılır · **her iki hesap silebilir** · **`HOME_NET`/`EXTERNAL_NET` policy varsayılanından FARKLI olmalı** · **TGW route table** ile yönlendirme · flexible cost allocation (Kas 2025)
**Active Threat Defense (Haz 2025):** **MadPot** · `AttackInfrastructure` (+`StrictOrder`/`ActionOrder`) · malware staging / botnet C2 / crypto-mining · **max 15.000 kapasite** · deep threat inspection
**Forward proxy (no-source-preservation): PUBLIC PREVIEW Ağu 2026, us-east-2 only. GA DEĞİL.**
**Container associations (Haz 2026):** ECS + EKS, 100/hesap-Region, 5 monitoring config, 30 referans/rule group

**FMS — 11 politika tipi:** `WAF | WAFV2 | SHIELD_ADVANCED | SECURITY_GROUPS_COMMON | SECURITY_GROUPS_CONTENT_AUDIT | SECURITY_GROUPS_USAGE_AUDIT | NETWORK_FIREWALL | DNS_FIREWALL | THIRD_PARTY_FIREWALL | IMPORT_NETWORK_FIREWALL | **NETWORK_ACL_COMMON**`
**Ön koşullar:** **Organizations + FMS admin hesabı + AWS CONFIG + AWS RAM** (NFW ve DNS FW için zorunlu) + Marketplace (3. parti: **Palo Alto Cloud NGFW, Fortigate CNF**)
**Rule group'lar politikadan ÖNCE ADMIN hesapta var olmalı.**
**Resource set: şu an yalnızca `AWS Network Firewall - firewalls`** · `PutResourceSet` · ARN **formatını** doğrular, varlığını doğrulamaz
**NFW dağıtım (FMS):** distributed / centralized / import · **public subnet'leri korur** · FMS route table'ları yönetir/izler
**Kapsam:** hesap/OU + kaynak tipi + **etiket (AND/OR + wildcard, Oca 2025)** · **görüntüle** veya **auto-remediate** · **yeni hesaplar OTOMATİK kapsama girer**

---

## ⚡ MTU tek bakışta

| Yol | MTU | PMTUD |
|---|---|---|
| DX private VIF | 1500 / **9001** | — |
| DX transit VIF | 1500 / **8500** | — |
| DX public VIF | **1500** | — |
| TGW VPC / Connect | **8500** | ✅ |
| TGW DX / **peering** | **8500** | ❌ |
| TGW VPN | **1500** | ❌ |
| S2S VPN payload | **1446** / MSS 1406 | — |
| Interface endpoint | **8500** | ❌ |
| **GWLB** | **8500** (+68 → appliance **8568**) | ❌ |
| Local Zones | **1468** | — |
| Mirror kesilmesiz max | **8947** | — |

**Jumbo kaybı:** iki private VIF farklı MTU ile aynı rotayı duyurursa **veya** aynı rotayı S2S VPN de duyurursa → **1500**.

---

## ⚡ "Bunu yapamazsın" — 20 saniyede eleme

Peering transitive ❌ · DXGW VPC↔VPC ❌ · TGW peering BGP/ECMP/multicast/transitive ❌ · transit VIF → VGW ❌ · public VIF → gateway ❌ · gateway endpoint on-prem/peering/TGW/cross-Region ❌ · gateway endpoint S3+DDB dışı ❌ · ALB doğrudan endpoint service ❌ · MACsec hosted ❌ · MACsec uçtan uca ❌ · accelerated VPN VGW ❌ · accelerated VPN mevcut bağlantıda ❌ · accelerated VPN DX public VIF ❌ · statik VPN'de ECMP ❌ · VGW'de ECMP/LBT ❌ · LAG resiliency ❌ · AS_PATH ile VPN > DX ❌ · appliance mode VPN/peering ❌ · appliance mode + 2/3-tuple ❌ · GWLBE RAM ❌ · GWLBE subnet değişimi ❌ · GWLB fail-open ❌ · Reachability paket/cross-Region/IPv6 ❌ · Flow Logs Amazon DNS/payload ❌ · flow log formatı değişimi ❌ · mirror bant genişliği bedava ❌ · mirror inter-Region peering ❌ · resolver rule numerik priority ❌ · cross-account PHZ konsol ❌ · NFW kapasite artırımı ❌ · STRICT_ORDER varsayılan ❌ · stateless drop logu ❌ · NFW forward proxy GA ❌ · FMS Config'siz ❌ · Amazon Q ağ troubleshooting ❌

---

## ⚡ Son 5 dakika: en çok karıştırılan 10 çift

| | A | B |
|---|---|---|
| 1 | **Association** = giriş, 1 tablo | **Propagation** = duyuru, N tablo |
| 2 | **Inbound endpoint** = sorgu içeri | **Outbound endpoint** = sorgu dışarı |
| 3 | **Gateway endpoint** = ücretsiz, S3+DDB, VPC içi | **Interface endpoint** = ücretli, her yer |
| 4 | **Reachability** = tek yol, "ne engelliyor", $0,10 | **Access Analyzer** = tarama, scope, $0,002/ENI |
| 5 | **Flow Logs** = metadata | **Mirroring** = payload |
| 6 | **MACsec** = L2, noktadan noktaya, dedicated | **Private IP VPN** = L3, uçtan uca, hosted'da da |
| 7 | **LBT (5 Gbps)** = tünel kapasitesi | **Accelerated VPN** = gecikme/jitter |
| 8 | **VGW allowed prefix** = filtre | **TGW allowed prefix** = duyurulan liste |
| 9 | **7224:73/72/71xx** = local pref (private/transit) | **7224:9xxx** = scope (public VIF) |
| 10 | **`send-via`** = east-west (Cloud WAN) | **`send-to`** = north-south |

---

**Bol şans. Soruyu iki kez okuyun; AWS'in sorularının yarısı seçeneklerde değil, soru kökündeki tek bir kelimede saklıdır.**

---

[← Bölüm 9: Sınav soru bankası](09-sinav-soru-bankasi.md) · [Ana sayfa](00-BASLA-BURADAN.md)
