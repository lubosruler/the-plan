# TUR 14 TANIMI — B.U.D. (Broad Universal Database) Başlangıcı

**Hazırlayan:** Claude · **Kaynak:** `the-plan` reposundaki tüm TUR11–13 belgeleri
(içerik bazında okundu, dosya adları karışık/yanlış etiketlenmiş olsa da),
artı `budlum-main__10_.zip` içindeki gerçek `docs/DEVIR_RAPORU.md`,
`docs/ORG_ROADMAP_AUDIT.md`, `docs/TUR13_5_RAPOR.md`, `docs/PERSONAS.md`.
**Doğrulama yöntemi:** Rapor metnine değil, zip içindeki gerçek dosyalara bakıldı.

---

## 0. Önce bir sıralama uyarısı — atlamadan önce oku

Planın kendisi (`TUR13_RAPOR.md` / org roadmap denetimi) zaten çok net:
**Tur 14 = yalnızca B.U.D.**, 13 serisiyle karıştırılmayacak. Ama gerçek repodaki
`docs/DEVIR_RAPORU.md` şunu da açıkça söylüyor: **Tur 13.9 henüz yapılmadı.**
Kalan 13.9 maddeleri:

1. **BLS/PQ anahtar koruması (B1)** — bu, `TUR12_5` bağımsız incelemesinde
   **Kritik** olarak işaretlenmişti: mainnet'te "PKCS#11 zorunlu" iddiası sadece
   Ed25519 konsensüs imzasını kapsıyor, BLS finality ve Dilithium5 (PQ) anahtarları
   hâlâ diskte düz metin. Bu hâlâ kapanmamış.
2. Finality live-path son taraması
3. ConsensusStateV2 migration notu
4. Harici audit teslim checklist'i
5. README roadmap kapanış tablosu + devir notu güncellemesi

**Önerim:** Tur 14'ü B.U.D. ile başlatmak mimari olarak sorun değil — B.U.D.
consensus/anahtar katmanına dokunmuyor, ayrı bir domain. Ama **B1'i (BLS/PQ
HSM) Tur 14 bahanesiyle süresiz ertelemeyin.** İki seçenek:

- **(a)** Tur 13.9'u önce, kısa bir tur olarak kapatın, sonra Tur 14'e geçin, veya
- **(b)** Tur 14'ü şimdi başlatın ama Tur 13.9 maddelerini "13.9 — Tur 14 ile
  paralel, ihmal edilmemiş borç" olarak DEVİR_RAPORU'nda açıkça işaretli tutun.

Aşağıdaki Tur 14 tanımı, (b)'yi varsayarak yazıldı — ama nihai karar sende.

---

## 1. Kapsam — Tur 14 SADECE B.U.D.

Org roadmap denetimindeki B.U.D. fazları (`budlum-xyz/B.U.D.` reposundan,
`BUD_Merkeziyetsiz_Depolama_Vizyonu.md` kaynaklı):

| Faz | Konu | Tur 14'te mi? |
|---|---|---|
| 0 | Kavramsal harita | Sadece referans (zaten var) |
| **1** | **Storage ConsensusDomain** | **Bu turun hedefi** |
| **2** | **İçerik-adresleme (CID/Poseidon)** | **Bu turun hedefi (başlangıç)** |
| 3 | Proof-of-Storage (`VerifyMerkle` bağlama) | **HAYIR — Z-B önkoşulu kapanmadı** (bkz. §2) |
| 4 | GlobalBlockHeader StorageRoot | Faz 1-2 oturduktan sonra, aynı turda denenebilir |
| 5 | Operator bond / slash ekonomisi | Faz 1 domain'i varsa doğal uzantı |
| 6 | BNS/.bud + devnet pilot | Sonraki tur |

**Not:** Elimde `BUD_Merkeziyetsiz_Depolama_Vizyonu.md` dosyasının kendisi yok —
sadece org roadmap denetimindeki özet tablo var. Eğer o vizyon dosyası
paylaşılırsa Faz 1-2 tanımını daha isabetli detaylandırabilirim. Şimdilik
aşağıdaki tanım, Budlum'ın zaten var olan `ConsensusDomain` / permissionless
registry mimarisinden **çıkarım** yapıyor.

---

## 2. Neden Faz 3 (Proof-of-Storage) bu turda DEĞİL

B.U.D.'un depolama-kanıtı (`VerifyMerkle` opcode'una bağlanması gereken kısım)
BudZero tarafında hâlâ **`is_experimental()`** ile production'da kapalı —
`docs/DEVIR_RAPORU.md`'nin kendi ifadesiyle: *"pozitif 64-depth proof hâlâ
yeşil değil."* Bunun üzerine kanıt-of-storage inşa etmek, sağlam olmayan bir
temelin üstüne ekonomik/slashing mantığı koymak demek — tam olarak projenin
"sahte-yeşil yol" olarak daha önce yakaladığı hatanın aynısı. Faz 3'ü Z-B
Commit 3.5 gerçekten yeşil olana kadar ertelemek bilinçli bir karar, unutkanlık
değil.

---

## 3. Somut ilk görev (Tur 14, alt-tur 14.1)

**Hedef:** Storage ConsensusDomain'in iskeletini, mevcut domain mimarisiyle
tutarlı şekilde kurmak — henüz depolama kanıtı yok, sadece domain kaydı ve
içerik-adresleme birimi.

1. **Yeni domain türü:** `ConsensusKind::Storage` (mevcut `ConsensusKind`
   enum'una ekle — PoW/PoS/PoA/BFT/Zk yanına). `ConsensusDomain` ve
   `DomainFinalityAdapter` deseninin aynısını izle; yeni bir bridge protokolü
   İCAT ETME.
2. **İçerik adresleme:** `ContentId` tipi — Poseidon4 hash tabanlı (BudZero'da
   zaten kullanılan `poseidon4_hash` primitive'iyle aynı aile; yeni bir hash
   fonksiyonu icat etme). `src/storage/` altına yeni bir modül (`content_id.rs`
   veya benzeri) — mevcut `src/storage/db.rs` (node'un kendi sled DB'si) ile
   KARIŞTIRMA, o farklı bir şey.
3. **Operatör kaydı:** Mevcut permissionless `Verifier Registry` deseniyle
   aynı `RoleId`-bazlı yaklaşım — `roles::STORAGE_OPERATOR` gibi yeni bir rol,
   stake yatırma + slashing kancası (henüz gerçek slashing mantığı yazma,
   sadece kayıt + bond iskeleti).
4. **Test:** Yeni domain'in PoA izolasyonunu bozmadığını doğrulayan bir test
   (`src/tests/permissionless.rs`'teki mevcut izolasyon testlerinin yanına).

**Yapılmayacaklar (bilinçli sınır):**
- `VerifyMerkle`/Proof-of-Storage bağlama — Faz 3, sonraya.
- `GlobalBlockHeader.storage_root` alanı eklemek — Faz 1-2 stabilse aynı turda
  değerlendirilebilir ama zorunlu değil.
- BNS/.bud — ayrı tur.

---

## 4. Kabul kriterleri

- `cargo fmt --check`, `cargo clippy -- -D warnings` (yeni allow yok),
  `cargo test --lib` — tümü yeşil.
- Yeni domain türü PoA domain'in izolasyonunu bozmuyor (mevcut testler kırılmıyor).
- Yeni modül, mevcut `ConsensusDomain`/`CrossDomainMessage` primitive'lerini
  kopyala-yapıştır değil, referans/genişletme olarak kullanıyor.
- README/CLAUDE.md'ye "B.U.D. Faz 1-2 iskelet, Faz 3 Z-B'ye bağımlı beklemede"
  notu düşülüyor — "storage production-ready" gibi yanlış bir iddia YOK.

---

## 5. Durma kuralı (hatırlatma, değişmedi)

Push atıldı + CI yeşil → **DUR**, sıradaki alt-tur (14.2 vb.) için talimat
bekle. Push reddedildi → durma, aynı turda çöz.

---

## 6. Eksik girdi

`BUD_Merkeziyetsiz_Depolama_Vizyonu.md` elimde yok. Varsa paylaşırsan Faz 1-2
tanımını (özellikle içerik-adresleme şeması ve operatör ekonomisi parametreleri)
tahminden çıkarıp doğrudan o belgeye göre netleştiririm.
