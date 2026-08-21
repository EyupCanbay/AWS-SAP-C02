# SAP-C02 — Ağ (Networking) Masterclass

**AWS Certified Solutions Architect – Professional** sınavının en çok soru çıkan alanı olan ağ konularının, en ince ayrıntısına kadar Türkçe anlatımı.

> **Hazırlanma tarihi:** 21 Ağustos 2026
> **Doğrulama:** Her sayı, kota ve davranış `docs.aws.amazon.com`, AWS What's New ve AWS blog kaynaklarından **Ağustos 2026 itibarıyla** doğrulandı. 2025–2026'da değişen ve piyasadaki kursları eskiten noktalar belgede **açıkça işaretlendi**.

---

## ⚠️ Bu belge neden farklı: 2025–2026'da değişen 12 şey

Piyasadaki hemen her SAP-C02 kaynağı aşağıdaki noktalarda **yanlış** bilgi veriyor. Bunları bilmek, hem sınavda hem gerçek işte fark yaratır.

| # | Piyasadaki kaynaklarda | Gerçek (Ağustos 2026) |
|---|---|---|
| 1 | DX private/transit VIF'te max **100 prefix** | **1.000'e kadar** (20 Ağu 2026, inbound prefix controls) |
| 2 | TGW VPC attachment **50 Gbps burst** | **100 Gbps / AZ / yön**, 7,5M PPS |
| 3 | VPN tüneli sabit **1,25 Gbps** | **Large Bandwidth Tunnel = 5 Gbps**; ayrıca **kesintisiz** upgrade (May 2026) |
| 4 | Inspection için ayrı VPC + GWLB/firewall endpoint şart | **TGW → Network Firewall native attachment** (Haz 2025) — inspection VPC gerekmiyor |
| 5 | TGW route table başına **10.000 route** | Aslında **TGW başına toplam 10.000** — tablo başına limit yok |
| 6 | DX SLA %99,99 = **2 lokasyon, 2 bağlantı** | **≥4 bağlantı + ≥2 lokasyon + lokasyon başına ≥2 + Enterprise Support + Well-Architected Review** (24 Tem 2026 SLA) |
| 7 | Interface endpoint **10 Gbps, 40 Gbps'e burst** | **10 Gbps/AZ varsayılan, otomatik 100 Gbps'e** ölçeklenir |
| 8 | NAT gateway **45 Gbps / 4M PPS** | **5 → 100 Gbps / 1M → 10M PPS** |
| 9 | Hibrit DNS için **EC2 conditional forwarder** | Artık gereksiz. Resolver endpoint + **`DELEGATE` rule** (Haz 2025) |
| 10 | 1.024 PPS limiti **DNS'e özel** | **DNS + IMDS + NTP + Windows lisanslama TOPLAMI** |
| 11 | Çakışan CIDR'da tek çözüm PrivateLink | Ayrıca **VPC Lattice resource endpoint** (Ara 2024) — load balancer gerektirmez, RDS'i ARN ile paylaşır |
| 12 | Cross-Region PrivateLink yok | Müşteri servisleri **Kas 2024**, **AWS servisleri Kas 2025** (S3, IAM, ECR, KMS, ECS, Lambda…) |

**Bunların dışında tamamen yeni ve eski kaynaklarda hiç olmayan konular:**
VPN Concentrator (100 site / 1 attachment) · TGW Encryption Support · TGW Flexible Cost Allocation · Client VPN native TGW attachment · VPC Route Server · Cloud WAN Routing Policy · DX VIF Rate Limiters · DX BGP route visibility · Route 53 Profiles · Route 53 Global Resolver · DNS Firewall Advanced · Network Firewall active threat defense · CloudWatch Network Flow Monitor.

---

## 📚 Bölümler

| # | Bölüm | Konu | Durum |
|---|---|---|---|
| 1 | [Transit Gateway](01-transit-gateway.md) | Attachment/association/propagation, segmentasyon, appliance mode, route priority, peering, Connect, multicast, kotalar, Cloud WAN karşılaştırması | ✅ |
| 2 | [Direct Connect](02-direct-connect.md) | Port hızları, VIF tipleri, DXGW, LAG, resiliency & SLA, BGP community'leri, DX+VPN failover, MACsec, SiteLink | ✅ |
| 3 | [Site-to-Site VPN](03-site-to-site-vpn.md) | Bant genişliği & ECMP matematiği, tünel seçenekleri, ELC, accelerated VPN, private IP VPN over DX, CloudHub, CGW, VPN Concentrator | ✅ |
| 4 | [PrivateLink ve VPC Endpoints](04-privatelink-vpc-endpoints.md) | Gateway vs interface endpoint, S3 çift endpoint deseni, endpoint service, cross-Region PrivateLink, VPC Lattice resource endpoint, çakışan CIDR | ✅ |
| 5 | [Route 53 Resolver ve hibrit DNS](05-route53-resolver-hibrit-dns.md) | Inbound/outbound endpoint, rule tipleri & eşleşme, RAM paylaşımı, cross-account PHZ, Route 53 Profiles, DNS Firewall, Global Resolver | ✅ |
| 6 | [İzleme ve troubleshooting](06-izleme-ve-troubleshooting.md) | VPC/TGW Flow Logs (v11'e kadar), Traffic Mirroring, Reachability Analyzer, Network Access Analyzer, Synthetic/Flow/Internet Monitor | ✅ |
| 7 | [GWLB, Network Firewall, Firewall Manager](07-gwlb-network-firewall-firewall-manager.md) | GENEVE, flow stickiness, 4 dağıtım modeli, Suricata kural motoru, kapasite hesabı, TLS inspection, native TGW attachment, FMS 11 policy tipi | ✅ |
| 8 | [Karar matrisleri ve anahtar kelime haritası](08-karar-matrisleri.md) | Tüm karar ağaçları + **anahtar kelime → cevap tablosu** + "bunu yapamazsın" listesi + sayısal ayrım tablosu | ✅ |
| 9 | [Sınav soru bankası](09-sinav-soru-bankasi.md) | 20 birleşik senaryo sorusu — birden fazla servisi birlikte test eden, gerçek sınav formatında | ✅ |
| 10 | [Son tekrar cheatsheet](10-son-tekrar-cheatsheet.md) | Sınavdan 1 saat önce okunacak tek sayfa: tüm sayılar, MTU tablosu, eleme listesi, karıştırılan 10 çift | ✅ |

---

## 🎯 Nasıl çalışmalı?

**Belge büyüklüğü:** 11 dosya · ~43.000 kelime · **55 senaryo sorusu** (35 bölüm içi + 20 birleşik) · 27 diyagram.

Her bölüm aynı iskeleti izliyor:

1. **Teknik derinlik** — servisin gerçekte ne yaptığı, mekanizma seviyesinde
2. **Tablolar** — kotalar, limitler, davranış karşılaştırmaları
3. **Diyagram** — `images/` klasöründeki görsel (Markdown'da otomatik gösterilir)
4. **"Basit anlatım"** — konunun günlük hayattan bir analojiyle özeti
5. **Sınav tuzakları** — AWS'in soru yazarken sevdiği yanıltmalar
6. **5 senaryo sorusu** — cevap **ve neden diğer üç seçeneğin yanlış olduğu** ayrı ayrı analiz edilerek
7. **60 saniyelik özet** — tekrar için

### Önerilen sıra
- **İlk okuma:** 1 → 2 → 3 (bağlantı katmanı), sonra 4 → 5 (erişim ve isim çözümleme), sonra 6 → 7 (görünürlük ve güvenlik).
- **İkinci okuma:** yalnızca "Sınav tuzağı" blokları ve 60 saniyelik özetler.
- **Sınavdan önce:** Bölüm 8 (karar matrisleri) + Bölüm 10 (sayılar).

### Soruları çözerken
Cevap bloklarını **açmadan önce** kendi cevabınızı yazın. Yanlış seçeneklerin analizini okumak, doğru cevabı okumaktan daha öğreticidir — AWS'in sorularının %70'i "iki seçenek de çalışır, hangisi *daha* uygun?" mantığındadır.

---

## 🖼️ Görseller

Tüm diyagramlar `images/` klasöründedir ve Markdown içinde otomatik görünür. Görüntüleyici öneriler: **VS Code** (Markdown Preview), **Typora**, **Obsidian**, **MacDown**. macOS Quick Look (boşluk tuşu) da görselleri gösterir.

Diyagram listesi:

| Dosya | Konu |
|---|---|
| `tgw-anatomi.png` | Attachment / association / propagation |
| `tgw-segmentasyon.png` | 3 segmentasyon deseni |
| `tgw-merkezi-inspection-akis.png` | Merkezi inspection: paket adım adım, iki route table'ın döngüyü kırması |
| `tgw-appliance-mode.png` | Appliance mode ile/olmadan akış simetrisi |
| `tgw-route-priority.png` | Route evaluation order merdiveni |
| `tgw-inter-region-peering.png` | Peering: statik route ve transitive olmayış |
| `tgw-karar-matrisi.png` | TGW vs Peering vs PrivateLink vs Cloud WAN |
| `dx-vif-tipleri.png` | Private / Public / Transit VIF |
| `dxgw-topoloji.png` | DXGW ve transitive olmayış |
| `dx-resiliency-sla.png` | Resiliency modelleri ve 2026 SLA |
| `dx-vpn-failover.png` | DX+VPN yedekliliği ve asimetrik routing |
| `dx-bgp-communities.png` | Local pref vs scope community'leri |
| `dx-sitelink.png` | SiteLink ile Region bypass |
| `vpn-ecmp-bant-genisligi.png` | ECMP bant genişliği matematiği |
| `vpn-accelerated.png` | Accelerated VPN yolu |
| `vpn-private-ip-dx.png` | Private IP VPN over Direct Connect |
| `endpoint-gateway-vs-interface.png` | Gateway vs interface endpoint |
| `overlapping-cidr-cozumleri.png` | Çakışan CIDR: üç çözüm |
| `privatelink-endpoint-service.png` | Sağlayıcı/tüketici modeli |
| `r53-hibrit-dns.png` | Inbound/outbound endpoint yönleri |
| `r53-resolver-rule-oncelik.png` | Rule eşleşme ve SYSTEM istisnası |
| `r53-merkezi-dns-profiles.png` | Route 53 Profiles ile ölçekleme |
| `izleme-arac-secimi.png` | Hangi izleme aracı? (6 araç ayrımı) |
| `flowlogs-vs-mirroring.png` | Flow Logs vs Mirroring vs TGW Flow Logs |
| `gwlb-geneve.png` | GWLB GENEVE akışı |
| `nfw-dagitim-modelleri.png` | Network Firewall 4 dağıtım modeli |
| `firewall-manager.png` | FMS politika tipleri ve hiyerarşi |

---

## 📌 Kaynak notu

Belgedeki her teknik iddia AWS'in kendi dokümantasyonundan doğrulandı. Dokümantasyonun **çelişkili veya belirsiz** olduğu noktalar belgede açıkça belirtildi (örn. VPN MTU 1500 vs 1446, LAG'de 4×400G'nin mümkün olup olmadığı, appliance mode'un peering üzerindeki durumu). Bu noktalarda sınav sorusu gelirse en makul seçeneği işaretleyin — AWS'in kendisi de net konuşmuyor.
