# TUR 14.5 PLANI — Depolama Pazarı: Parçalanmış İçerik + NFT Bağlama (Filecoin-benzeri)

**Tarih:** 2026-07-14
**Önceki tur:** Tur 14 (B.U.D. Faz 1-2 iskeleti — Storage ConsensusDomain +
içerik-adresleme). Bu belge onun üzerine inşa eder, onu tekrar etmez.
**Sıradaki tur:** Tur 14.9 — kontrol turu (bkz. §7).

---

## 0. Kullanıcı senaryosu (kendi tarifin)

> Validatörler, zaten doğrulayıcı oldukları gibi, B.U.D.'u kullanarak Budlum
> ağında Filecoin mantığına benzer bir hizmet verebiliyorlar. Budlum'da NFT
> olarak temsil edilen ama aslında bir video olan ya da bir dapp'in oyunu olan
> içerik, bu depolarda birden fazla parçaya bölünerek saklanabiliyor.

Bu, B.U.D. roadmap'indeki **Faz 5 (operatör bond/slash ekonomisi)**'in somut
bir uygulaması ve **Faz 2 (içerik-adresleme)**'nin bir genişlemesi —
"tek parça CID" yerine "çok parçalı içerik + dağıtık depolama sözleşmesi."
Faz 3 (Proof-of-Storage / `VerifyMerkle`) hâlâ blokta olduğu için bu tur, gerçek
kriptografik depolama-kanıtı OLMADAN çalışacak şekilde tasarlandı — bkz. §4.

---

## 0.5. Revizyon gerekçesi — Yeni ilke: Veri Egemenliği / Ekip-Bağımsızlığı

Paylaştığın `BUDLUM_PARADIGMA_ANALIZI` raporu ve az önceki notun ("ekip
çalışmasa bile tüm appler/sistem çalışabilir olmalı") aynı yöne işaret ediyor
— raporun kendi paradigma #6 (dijital devlet egemenliği: "her ülke domain'ini
koruyarak bağımsız kalır") ve #7'sinin (trustless bridge: "matematiksel kanıt
var, insan onayı yok") B.U.D.'a uygulanmış hali. Yani bu bir yeni fikir değil,
raporun zaten savunduğu ilkenin depolama katmanına genişletilmesi.

**Somut kural:** B.U.D.'un hiçbir kritik fonksiyonu (deal açma, ücret ödeme,
operatör keşfi, erişilebilirlik denetimi, slashing) **"Budlum ekibinin
çalıştırdığı bir sunucu/servis"e bağımlı olamaz.** Ekip yarın ortadan kalksa,
zincir üzerindeki her şey (deal'lar, kayıtlı operatörler, manifest'ler)
**herhangi bir bağımsız node tarafından** okunabilir/işletilebilir kalmalı.

Bu, aşağıdaki 3 maddede §2.5, yeni §2.6 ve §3'te somut mimari değişikliğe
dönüşüyor — orijinal tasarımda "kim challenge açar" ve "kim shard'ı bulur"
soruları örtük bırakılmıştı; şimdi açıkça permissionless hale getiriliyor.

---

## 1. Kapsam

| Roadmap fazı | Bu turda ne kadarı |
|---|---|
| Faz 1 (Storage ConsensusDomain) | Devralınır, değişmez |
| Faz 2 (içerik-adresleme) | **Genişletilir:** tekil CID → çok-parçalı `ContentManifest` |
| **Faz 5 (operatör ekonomisi)** | **Bu turun ana hedefi:** depolama sözleşmesi (deal), bond, ücret |
| Faz 3 (Proof-of-Storage) | **Hâlâ YOK** — Z-B kapanmadan gerçek kanıt yazılmaz |
| Faz 6 (BNS/.bud) | Bu turun kapsamı dışı |
| NFT minting / DeArt | Bu turun kapsamı dışı — bkz. §3.4 sınırı |

---

## 2. Mimari

### 2.1. Çok-parçalı içerik: `ContentManifest`

Faz 2'deki tekil `ContentId` (Poseidon4 tabanlı) üzerine:

```
ContentManifest {
    manifest_id: ContentId,        // manifest'in kendi hash'i
    total_size: u64,
    shard_count: u32,
    shards: Vec<ShardRef>,
}

ShardRef {
    index: u32,
    shard_id: ContentId,           // bu parçanın içerik-adresi
    size: u64,
}
```

Parçalama (kaç parçaya bölüneceği, parça boyutu) istemci/uygulama tarafında
yapılır — zincir sadece manifest'i ve parça referanslarını tutar, dosyayı
işlemez. Bu, mevcut projede "zincir veriyi taşımaz, kanıtını tutar" ilkesiyle
tutarlı (aynı BudZKVM'nin STARK kanıtı taşıması gibi, B.U.D. da içerik
kanıtı/adresi taşır, ham veriyi değil).

### 2.2. Depolama operatörü kaydı — mevcut registry deseninin aynısı

`validatörler ... gibi` ifadeni şöyle uyguluyorum: depolama operatörlüğü,
**ayrı bir stake ile** aynı permissionless Verifier Registry mekanizmasından
geçer — var olan bir validator otomatik olarak depolama sağlayıcısı OLMAZ,
ama aynı adres isterse `roles::STORAGE_OPERATOR` için de kayıt/bond yapabilir
(Tur 3'teki relayer kapısıyla aynı desen: kayıt sadece ödül için, gönderim
kapısı değil). Whitelist/onay YOK — bu ilke B.U.D.'a da aynen taşınır.

```
StorageDeal {
    deal_id: u64,
    manifest_id: ContentId,
    shard_id: ContentId,
    operator: Address,             // roles::STORAGE_OPERATOR kayıtlı olmalı
    operator_bond: u64,
    fee_per_epoch: u64,
    replica_index: u8,             // aynı parça birden fazla operatörde durabilir
    deal_start_epoch: u64,
    deal_end_epoch: u64,
    status: DealStatus,            // Active | Expired | Slashed
}
```

Bir `manifest`'in her `shard`'ı, biri veya birden fazla `StorageDeal` ile
eşleşir (replikasyon = birden fazla operatörde aynı parça).

### 2.3. NFT bağlama — sınırlı, ödünç alınan bir referans

Sen "NFT olarak temsil edilen ama aslında video/oyun" dedin. Bu turda NFT'nin
kendisini (mint, transfer, sahiplik) **yazmıyoruz** — o DeArt/ayrı DAO
kapsamında (memory'de zaten "excluded" olarak işaretli). Bu turun sağladığı
şey sadece şu: herhangi bir NFT/token primitive'i (DeArt'tan gelsin ya da
başka), içeriğini işaret etmek için tek bir alan kullanabilir:

```
nft.content_manifest_root: ContentId   // = ContentManifest.manifest_id
```

Yani B.U.D. burada NFT standardı icat etmiyor, sadece "bu token'ın içeriği
şu manifest'tedir" diyebileceği bir çıpa sağlıyor. NFT tarafı ayrı bir tur/DAO
kararı.

### 2.4. Ücret ve bond ekonomisi (Faz 5)

- **Depolayan taraf öder:** `fee_per_epoch × deal süresi`, deal açılışında
  kilitlenir (escrow), her epoch'ta operatöre serbest bırakılır.
- **Operatör bond yatırır:** deal açmadan önce `operator_bond` stake'i
  ayrılır — mevcut registry'deki genel stake'ten ayrı, deal-özel kilitli.
- **Slashing tetiği (bu turda SINIRLI):** gerçek kriptografik
  proof-of-storage olmadığı için slashing, §2.5'teki **interim retrieval
  challenge**'a bağlanır — "kanıtlanamayan depolama iddiası" değil,
  "istenen veriyi belirli bir sürede geri veremedi" olayına.

### 2.5. Interim (geçici) erişilebilirlik kontrolü — gerçek Proof-of-Storage DEĞİL

**[REVİZE — §0.5 ilkesi gereği]** İlk taslakta bu kontrolün kimin tarafından
tetiklendiği belirsizdi; bu, örtük olarak "Budlum'un çalıştırdığı bir izleyici
servisi" varsayımına kayabilirdi — tam da kaçınmak istediğimiz merkezi
bağımlılık. Yeni tasarım:

- Rastgele bir zincir-içi tetikleyici (`GlobalBlockHeader` hash'inden türetilen
  epoch-seed), belirli bir deal için operatörden **shard'ın bir kısmının**
  (rastgele byte-range) hash'ini isteyen bir `RetrievalChallenge` üretir.
- **Challenge'ı KİMİN açtığı önemli değil — herkese açık, izinsiz bir
  işlem:** Mevcut `SlashingReport`'taki `reporter`/fee/ödül deseninin AYNISI
  (bkz. `submit_registry_slashing_report`, harici raportör fee öder, iddia
  doğrulanırsa iade + ödül alır, asılsızsa fee yanar). Yani ekip, bağımsız bir
  kullanıcı, rakip bir operatör, ya da otomatik bir bot — kim olursa olsun
  aynı ekonomik kapıdan geçerek challenge açabilir. Team'in özel bir "resmi
  izleyici" rolü YOK.
- Operatör süre içinde cevap veremezse veya hash tutmuyorsa →
  `DealStatus::Slashed`, bond'un bir kısmı yakılır; challenge'ı açana ödül.
- **Bu, sahte bir "proof-of-storage" DEĞİLDİR** — operatör verinin tamamını
  saklamadan da (örn. sadece istenen aralığı biriktirip) bu testi
  büyük olasılıkla geçebilir. README/CLAUDE.md'de bu açıkça
  "geçici, tam kanıt değil, Faz 3 = gerçek çözüm" diye yazılmalı — aksi halde
  tam olarak projenin daha önce yakaladığı "sahte-yeşil yol" hatasının
  yenisi olur.
- **Ekip-bağımsızlığı sonucu:** Ekip ortadan kalksa bile, ekonomik teşvik
  (ödül) yeterliyse üçüncü taraflar challenge açmaya devam eder — sistemin
  dürüstlüğü tek bir organizasyonun varlığına bağlı değil, izole
  katılımcıların kendi çıkarına bağlı (tıpkı Bitcoin madencilerinin merkezi
  bir otorite olmadan da bloğu doğrulaması gibi).

### 2.6. Keşif ve erişim — merkezi kapı yok

**[YENİ — §0.5 ilkesi gereği]** İlk taslak, "bir deal'ın hangi operatörde
olduğunu nasıl bulacağız" sorusunu açık bırakmıştı. Bunu kapatıyoruz:

- `StorageDeal` ve `ContentManifest` kayıtları **zaten zincir üzerinde**
  (Faz 1 domain state'inin parçası) — herhangi bir bağımsız node, kendi RPC'si
  üzerinden bunları sorgulayabilir. "Budlum Inc. indexer API'si" veya benzeri
  şirkete özel bir aracı servis **gerekmez ve tasarıma dahil edilmemeli.**
  (`bud_storageQueryManifest`, `bud_storageQueryDealsByShard` gibi RPC uçları
  — mevcut `bud_registryQuery` deseniyle aynı, herkesin çalıştırabileceği bir
  node'da var olan uçlar.)
- Gerçek byte transferi (P2P) hâlâ §3'te belirtildiği gibi bu turun kapsamı
  dışı, ama **tasarım kısıtı olarak** not düşülüyor: o katman geldiğinde
  (sonraki tur), tek bir "resmi gateway" (örn. `gateway.budlum.com`) üzerinden
  DEĞİL, `ShardRef`'in `ContentId`'siyle anahtarlanan permissionless bir
  keşif mekanizması (ör. Kademlia-tipi DHT) üzerinden tasarlanmalı — aksi
  halde depolama merkeziyetsiz olsa bile *erişim* tek bir şirkete bağımlı
  kalır ve tüm bu turun amacı boşa çıkar.

---

## 3. Yapılmayacaklar (bilinçli sınır)

1. Gerçek kriptografik Proof-of-Storage (Faz 3) — Z-B/`VerifyMerkle`
   production'a açılmadan yazılmaz.
2. NFT mint/transfer/sahiplik mantığı — DeArt/ayrı DAO kapsamı.
3. Parçalama algoritmasının kendisi (erasure coding, Reed-Solomon vb.) —
   zincir dışı, istemci sorumluluğu; bu tur sadece manifest şemasını tanımlar.
4. Gerçek dosya transferi / P2P dağıtım protokolü — bu, node'lar arası ayrı
   bir ağ katmanı ister, bu turun kapsamı dışı (belirtilmeli, sonraki tur adayı).
5. **[YENİ]** Herhangi bir "resmi"/şirkete-özel indexer, gateway, izleyici
   servis, ya da admin/pause anahtarı — B.U.D.'un hiçbir parçası bunlara
   bağımlı tasarlanmayacak (§0.5, §2.5, §2.6).

---

## 4. Kabul kriterleri

- `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test --lib` yeşil.
- `ContentManifest`/`ShardRef`/`StorageDeal` yeni tipler, mevcut
  `ConsensusDomain`/registry primitive'lerini genişletir, kopyalamaz.
- Depolama operatörü kaydı **whitelist içermez** — mevcut permissionless
  registry testleriyle aynı desende negatif test (kayıtsız operatör deal
  açamaz, ama STAKE ile herkes kayıt olabilir).
- `RetrievalChallenge` mekanizmasının "gerçek Proof-of-Storage değil, geçici"
  olduğu README/CLAUDE.md'de en az bir cümleyle **açıkça** yazılı.
- NFT bağlama sadece tek bir referans alanı (`content_manifest_root`) —
  NFT'nin kendisi bu turda yazılmaz.
- **[YENİ] `RetrievalChallenge` herhangi bir adresten açılabilir** —
  test: takma bir "resmi izleyici" rolü/anahtarı olmadan, sıradan bir
  test hesabının challenge açıp ödül aldığını doğrulayan bir test var.
- **[YENİ] Deal/manifest sorgu RPC'leri** standart node RPC'sinde —
  ayrı bir "indexer servisi" gerektirmediğini gösteren en az bir entegrasyon
  testi (node'un kendi state'inden doğrudan sorgu).
- **[YENİ]** Storage ConsensusDomain'de veya deal registry'de herhangi bir
  admin-only/pause/freeze fonksiyonu **YOK** (kod incelemesiyle doğrulanır).

---

## 5. Durma kuralı (değişmedi)

Push atıldı + CI yeşil → **DUR**, Tur 14.9 talimatını bekle.
Push reddedildi → durma, aynı turda çöz.

---

## 6. Tur 14.9 — Kontrol Turu (senin tanımınla: "gelen kontrol")

Bu, kod yazan değil **doğrulayan** bir tur olmalı — projenin kendi
disiplinine uygun olarak (rapor metnine değil, koda/CI'ya bak):

1. Tur 14 (Faz 1-2) ve Tur 14.5 (deal/sharding) commit'lerinin gerçekten
   `main`'e gidip CI'da yeşil olduğunu GitHub Actions üzerinden doğrula
   (link ile, sadece rapor cümlesiyle değil).
2. `RetrievalChallenge`'ın gerçekten "geçici/zayıf" olarak belgelendiğini,
   yanlışlıkla "storage kanıtlandı" gibi sunulmadığını kontrol et.
3. Depolama operatörü kaydının whitelist içermediğini test kanıtıyla doğrula.
4. Yeni `ConsensusKind::Storage`'ın PoA domain izolasyonunu bozmadığını
   (mevcut izolasyon testleri hâlâ geçiyor mu) doğrula.
5. Tur 13.9'un hâlâ açık olup olmadığını tekrar kontrol et (BLS/PQ HSM) —
   B.U.D. işi bunu gölgede bırakmasın.
6. **[YENİ] Veri egemenliği / ekip-bağımsızlığı denetimi (§0.5):** Kod
   tabanında `budlum.com`, sabit kodlanmış bir şirket sunucu URL'i, admin-only
   fonksiyon veya "sadece ekibin çalıştırdığı servis çalışırsa işler" türü
   gizli bir bağımlılık olmadığını grep + manuel inceleme ile doğrula.
7. Sonuç: açık/kapalı bulgu listesi + Tur 15 önerisi (muhtemelen Faz 3,
   Z-B o zamana kadar kapanmışsa).

---

## 7. Eksik girdi (tekrar not)

`BUD_Merkeziyetsiz_Depolama_Vizyonu.md` hâlâ elimde yok. Paylaşırsan, özellikle
parçalama/replikasyon parametreleri ve ekonomi modelini tahminden çıkarıp
doğrudan o belgeye göre netleştiririm.
