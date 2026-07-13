# BUDLUM Ağ Güvenlik Denetimi — Bulunan Hatalar / Bugfix Raporu

- **Hedef:** `github.com/lubosruler/budlum` (budlum-core — "Evrensel Mutabakat Katmanı", Rust L1)
- **Tarih:** 2026-07-13
- **Yöntem:** Deposu klonlandı (`git clone`), `src/` altındaki 96 Rust modülü + `CLAUDE.md` + commit geçmişi (TUR 5/6/7 güvenlik audit'leri) satır satır incelendi.
- **Kapsam:** Kripto, konsensüs (PoW/PoS/PoA/BFT), finality (BLS + QC blob), slashing/registry, cross-domain bridge, RPC sunucusu, transaction/account katmanı.

> **Genel değerlendirme:** Canlı (live) güvenlik yolları son TUR audit commit'leriyle ciddi şekilde sertleştirilmiş (2/3 quorum, RPC auth varsayılan açık, keyfile `0o600`, snapshot timeout, bridge sweep, VRF/liveness doğrulama). Bununla birlikte **önceki bir "güvenlik denetiminden sağ kurtulan" (latent) kritik kod, birkaç mantık/konsensüs tutarlılığı ve anahtar-bütünlüğü açığı** hâlâ mevcut. Aşağıdaki bulgular önem sırasına göre listelenmiştir.

---

## Özet Tablosu

| # | Önem | Dosya / Satır | Başlık |
|---|------|---------------|--------|
| 1 | 🔴 YÜKSEK (latent) | `src/consensus/slashing.rs` (`verify_double_sign`/`verify_double_proposal`/`verify_double_vote`) | Slashing kanıtı "doğrulayıcıları" imza doğrulaması yapmıyor (sahte-verify API) |
| 2 | 🟠 ORTA | `src/chain/blockchain.rs:1796` + `src/consensus/qc.rs:205` | QC blob imza sayısı **dedup'tan önce** kontrol ediliyor (aynı `validator_index` atlatması) |
| 3 | 🟠 ORTA-YÜKSEK | `src/consensus/pow.rs:11,122` | PoW zorluğu blokta/state'te taşınmıyor; `validate_block` yan etkiyle motor state'ini bozuyor |
| 4 | 🟠 ORTA | `src/chain/blockchain.rs:1762` + `src/consensus/qc.rs:395` | QC fault-proof tespiti pratikte **ulaşılamaz** → finality asla geri alınamaz (yanıltıcı güvence) |
| 5 | 🟠 ORTA | `src/crypto/primitives.rs:120` | `BlsKeypair::from_bytes` anahtar bütünlüğünü doğrulamıyor |
| 6 | 🟡 DÜŞÜK | `src/core/account.rs:557` | Double-sign slash'ı registry'de `LivenessFault` olarak etiketleniyor (yanlış koşul) |
| 7 | 🟡 DÜŞÜK | `src/crypto/primitives.rs:295,350` | `KeyPair::generate/save` gizli anahtar yolunu/public key'i `stdout`'a basıyor |
| 8 | 🟡 DÜŞÜK | `src/consensus/slashing.rs` (SlashingType/slash_amount) | Eski slash oranları registry ile tutarsız ve kullanılmıyor |
| 9 | 🔴 YÜKSEK | `src/domain/finality_adapter.rs` (`PoWFinalityAdapter`) + `src/domain/commitment_registry.rs:54` | PoW cross-domain finality **tamamen self-declared** → sahte commitment ile bridge **mint forgery** (enflasyon) |
| 10 | 🟠 ORTA | `src/execution/executor.rs` (`apply_transaction_checked` üst kontrol) | `Unstake`/`Vote` tx'lerinde `total_cost` stake'i liquid balance'a yanlış yüklüyor → meşru unstake **DoS** |

---

## 1. 🔴 Slashing kanıtı "doğrulayıcıları" kriptografik imza kontrolü yapmıyor (LATENT KRİTİK)

**Konum:** `src/consensus/slashing.rs` — `verify_double_sign` (~satır 232), `verify_double_proposal` (~satır 270), `verify_double_vote` (~satır 300).

**Açıklama:** Bu üç fonksiyon *yapısal* kontrollerden ibarettir. Örnek `verify_double_sign`:

```rust
pub fn verify_double_sign(&self) -> Result<(), String> {
    // ... sadece:
    if hash1 == hash2 { return Err("Block hashes are identical"); }
    if sig1 == sig2  { return Err("Signatures are identical"); }
    // pubkey uzunluğu kontrolü
    Ok(())   // <-- İMZA HİÇBİR ZAMAN DOĞRULANMIYOR
}
```

`signature_1` ve `signature_2` üzerinde **hiçbir ed25519/Dilithium doğrulaması yapılmıyor**; sadece "iki ayrı bayt dizisi mi?" deniyor. `verify_double_vote` ve `verify_double_proposal` da aynı şekilde sadece yapısal.

**Neden tehlikeli:** Bu modül şu an *ölü kod* (`consensus/mod.rs`'ta `pub mod slashing;` ile derleniyor ama hiçbir yer `consensus::slashing::SlashingEvidence`'i import etmiyor — canlı yol `consensus::pos::SlashingEvidence` kullanıyor, ve o yol imzaları gerçekten doğruluyor). Ancak modül, adı **`verify_*`** olan ve "doğrulama" yaptığını ima eden bir API sunuyor. Bu API gelecekte RPC/consensus yoluyla slash mekanizmasına bağlanırsa (veya bir geliştirici "zaten verify var" diyerek onu kullanırsa) **herhangi bir saldırgan, iki rastgele bayt dizisiyle istediği validator'ı slash edebilir** — çünkü "doğrulayıcı" imzayı kontrol etmiyor.

**Sömürü senaryosu (eğer canlıya bağlanırsa):** Saldırgan, kurban validator'ın 32-byte pubkey hex'ini bilir (herkese açık). `SlashingEvidence::double_sign(victim, 100, "h1", "h2", vec![1,2,3], vec![4,5,6], "attacker")` üretir; `verify_double_sign()` `h1 != h2` ve `sig != sig` olduğu için **geçer**. Kanıt `is_actionable()` tetikler ve kurbanın tüm stake'i slash edilir (griefing / zincir sabotajı).

**Önerilen düzeltme:**
- Bu modülü ya tamamen silin, ya da `verify_*` fonksiyonlarını **gerçek kriptografik doğrulama** ile doldurun:

```rust
pub fn verify_double_sign(&self) -> Result<(), String> {
    if self.offense_type != SlashingType::DoubleSign { return Err("wrong type".into()); }
    let (h1, h2, s1, s2) = /* ... */;
    if h1 == h2 { return Err("identical hashes".into()); }
    let pk = hex::decode(&self.validator).map_err(|_| "bad pubkey".to_string())?;
    // Gerçek imza doğrulaması:
    crate::crypto::primitives::verify_signature(h1.as_bytes(), s1, &pk)
        .map_err(|e| format!("sig1 invalid: {e}"))?;
    crate::crypto::primitives::verify_signature(h2.as_bytes(), s2, &pk)
        .map_err(|e| format!("sig2 invalid: {e}"))?;
    Ok(())
}
```

- Veya en azından fonksiyonlara `#[deprecated]` + doc-comment "KULLANMAYIN — imza doğrulamaz" ekleyin ve modülü `pub mod` yerine test-only yapın.

---

## 2. 🟠 QC blob imza sayısı dedup'tan ÖNCE kontrol ediliyor

**Konum:** `src/chain/blockchain.rs:1796` (`import_qc_blob`) ve `src/consensus/qc.rs:205` (`verify_against_snapshot`).

**Açıklama:** `import_qc_blob` quorum kontrolünü **ham imza sayısına** göre yapar:

```rust
let n_validators = snapshot.validators.len();
let min_signers = (n_validators * 2 + 3 - 1) / 3; // ceil(2/3 n)
if blob.pq_signatures.len() < min_signers { return Err(/* yetersiz imza */); }
blob.verify_against_snapshot(&snapshot, None, Some(self.state.epoch_index))?;
```

Ama `verify_against_snapshot` içinde imzalar **`validator_index` bazında dedup** edilir (`verified_indices.insert(idx)`). Yani bir saldırgan, `min_signers` adet girdi gönderir ve hepsine **aynı `validator_index = 0`** + aynı geçerli imzayı koyarsa:
- ham `pq_signatures.len() == min_signers` → sayı kontrolünden geçer,
- `verify_against_snapshot` dedup sonrası **1 benzersiz** imzacı görür, yine "geçer" der.

**Etki:** PQ (kuantum-dayanıklı) finality quorum garantisi aslında tek bir validator'a düşer. BLS `FinalityCert` hattı (`handle_finality_cert` içindeki `required_signers` kontrolü) bunu büyük ölçüde maskeleyse de, PQ katmanının 2/3 quorum sözü kırılmış olur ve kötü niyetli bir validator seti QC blob'u şişirerek denetimi atlatabilir.

**Önerilen düzeltme:** Quorum kontrolünü dedup *sonrası* benzersiz imzacı sayısına göre yapın:

```rust
let unique_signers = blob.pq_signatures
    .iter()
    .map(|e| e.validator_index as usize)
    .collect::<std::collections::HashSet<_>>()
    .len();
if unique_signers < min_signers {
    return Err(format!("QcBlob has only {unique_signers} unique signers, need {min_signers}"));
}
```

---

## 3. 🟠 PoW zorluğu konsensüs dışı (yerel mutable state + validate yan etkisi)

**Konum:** `src/consensus/pow.rs:11` (`current_difficulty: RwLock<usize>`), `validate_block` ~satır 122-160.

**Açıklama:** Zorluk (`difficulty`) blok başlığında veya state root'ta **taşınmıyor**; sadece her nodun belleğinde tutulan bir `RwLock<usize>` alanı. Daha kötüsü, `validate_block` *okuma amaçlı* bir doğrulama fonksiyonu olmasına rağmen **yan etki** üretiyor:

```rust
if block.index > 0 && block.index.is_multiple_of(self.config.adjustment_interval) {
    let new_diff = self.calculate_new_difficulty(chain);
    if let Ok(mut d) = self.current_difficulty.write() { *d = new_diff; } // <-- doğrulama sırasında state değişti!
}
if !self.meets_difficulty(&block.hash) { return Err(...); }
```

Bu, reorg sırasında (`try_reorg` → `is_valid_chain(new_chain)`) rakip zincirleri doğrularken motorun `current_difficulty`'sini rakip zincirin zamanlamasına göre değiştirir. Eğer reorg reddedilirse, motorun zorluğu artık **kanonik zincir için yanlış** kalır → bu nodun kendi `mine()`'ı ve gelecekteki doğrulamaları bozulur.

**Etki:** Node'lar arası determinizm kaybı / zincir bölünmesi riski (özellikle yeniden org ve eşler-arası farklı görüşlerde). Ayrıca `validate_block`'ın yan etkili olması, test ve eşzamanlılık açısından kırılgan.

**Önerilen düzeltme:** Zorluğu deterministik hale getirin — ya bloğa `difficulty` alanı ekleyip başlık hash'ine katın, ya da `(chain_height)` fonksiyonu olarak tanımlayın (global state'e yazmadan). `validate_block` **kesinlikle** hiçbir mutable motor state'i değiştirmemelidir.

---

## 4. 🟠 QC fault-proof tespiti pratikte ulaşılamaz (finality geri alınamaz)

**Konum:** `src/chain/blockchain.rs:1762` (`maybe_apply_detected_qc_faults`) → `src/consensus/qc.rs:395` (`detect_fault_proofs`).

**Açıklama:** Finality'nin "kötü PQ attestation bulunursa geri al" mekanizması şöyle işliyor:
- `verified_qc_blobs`'a sadece `verify_against_snapshot` **tüm imzalar geçerli** dediği için** eklenir.
- `detect_fault_proofs` *sadece* doğrulaması **başarısız** olan imzaları raporlar.

Yani doğrulanmış bir blob'da zaten tüm imzalar geçerli olduğundan `detect_fault_proofs` **her zaman boş** döner → `maybe_apply_detected_qc_faults` hiçbir şey yapmaz → `QcProofAction::InvalidateFinality` / `SlashValidator` dalları **ölü koddur**. Dokümantasyonda vaat edilen "finality'yi geçersiz kılma" özelliği hiçbir zaman tetiklenemez.

**Etki:** Yanıltıcı güvenlik garantisi (false sense of security). Bir saldırgan, kanıtlanabilir bir PQ hatası olsa bile finality'yi geri aldıramaz; sistem "geri alınamaz" tasarlanmış gibi davranır ama bu tasarım bilinçli değil, çelişkiden doğmuş.

**Önerilen düzeltme:** Ya fault tespiti, blob *insert edildikten sonra* ikinci bir bağımsız kaynaktan (ör. ayrı bir "challenge" akışı) karşı doğrulansın; ya da bu yol tamamen kaldırılıp "finality geri alınamaz" tasarımı belgelensin. Mevcut haliyle kod ile doküman çelişiyor.

---

## 5. 🟠 `BlsKeypair::from_bytes` anahtar bütünlüğünü doğrulamıyor

**Konum:** `src/crypto/primitives.rs:120` (`BlsKeypair::from_bytes`), `to_bytes` ~satır 110.

**Açıklama:**
```rust
pub fn from_bytes(bytes: &[u8]) -> Result<Self, CryptoError> {
    // sk Scalar olarak parse ediliyor (tamam)
    let pk = bytes[32..128].to_vec(); // <-- ham Vec, G2 parse/validate YOK
    Ok(BlsKeypair { secret_key: sk, public_key: pk })
}
```
- `public_key`, `bytes[32..128]`'den olduğu gibi alınıyor; **G2 noktası olup olmadığı (`G2Affine::from_compressed`) hiç kontrol edilmiyor** (geçersiz encoding sessizce kabul edilir).
- `pk`'nın `sk`'den türetilip türetilmediği **hiçbir zaman yeniden hesaplanmıyor**. Bozuk/kötü niyetli bir keyfile (veya serileştirme yolu) `sk`'ye uymayan bir `pk` içeriyorsa node, farklı/çöp bir BLS anahtarına sahip olduğunu sanır.

Karşılaştırma: aynı dosyadaki `PqKeyPair::from_bytes` uzunlukları ve `ed25519` yolu katı; BLS yolu bu standardın gerisinde.

**Önerilen düzeltme:**
```rust
pub fn from_bytes(bytes: &[u8]) -> Result<Self, CryptoError> {
    if bytes.len() < 128 { return Err(CryptoError::InvalidKey("short".into())); }
    let mut sk_bytes = [0u8;32]; sk_bytes.copy_from_slice(&bytes[0..32]);
    let sk_opt = Scalar::from_bytes(&sk_bytes);
    if sk_opt.is_none().into() { return Err(CryptoError::InvalidKey("bad sk".into())); }
    let sk = sk_opt.unwrap();
    let pk_bytes: [u8;96] = bytes[32..128].try_into().unwrap();
    let pk_affine = G2Affine::from_compressed(&pk_bytes);
    if pk_affine.is_none().into() { return Err(CryptoError::InvalidKey("bad pk".into())); }
    // Bütünlük: pk sk'den türetilmeli
    let expected = G2Affine::from(G2Projective::generator() * sk);
    if expected.to_compressed().as_slice() != &bytes[32..128] {
        return Err(CryptoError::InvalidKey("pk does not match sk".into()));
    }
    Ok(BlsKeypair { secret_key: sk, public_key: bytes[32..128].to_vec() })
}
```

---

## 6. 🟡 Double-sign slash'ı registry'de yanlış koşulla etiketleniyor

**Konum:** `src/core/account.rs:557` (`slash_validator`) → `apply_slashing` (satır 545, double-sign kanıtından çağrılır).

**Açıklama:** `apply_slashing` (blok içindeki double-sign kanıtından tetiklenir) `slash_validator`'ı çağırır; bu fonksiyon permissionless registry'ye yapılan yansımada **her zaman** `SlashingCondition::LivenessFault` kullanır:

```rust
let _ = self.registry.slash(*address, roles::VALIDATOR,
    SlashingCondition::LivenessFault, slash_ratio_fixed);
```

Sonuç: bir equivocation (double-sign) saldırganı registry audit log'unda (`slashing_history_for`) **liveness** olarak görünür; `slash_amount` ve koşul tutarsız. Ekonomik ceza oranı doğru olsa da denetim/raporlama yanıltıcı.

**Önerilen düzeltme:** `apply_slashing`'a gerçek `SlashingCondition` parametresini geçirin ve registry mirror'ında onu kullanın.

---

## 7. 🟡 `KeyPair::generate` / `save` gizli anahtar bilgisini `stdout`'a basıyor

**Konum:** `src/crypto/primitives.rs:295` (`println!("Public key: {}", ...)`) ve `:350` (`println!("Keypair saved to {:?}", path)`).

**Açıklama:** Kütüphane kodu (`generate`/`save`) `println!` ile üretim (production) ortamında log'lara/public key ve dosya yoluna yazıyor. Konsolu/ortak log'ları kullanan bir node'da bu, gürültü ve hafif bir bilgi sızıntısı/side-channel riski (ayrıca hot-path'te I/O).

**Önerilen düzeltme:** `println!` yerine `tracing::debug!` kullanın; kütüphane kodunda stdout'a yazmayın.

---

## 8. 🟡 Eski slash oranları tutarsız ve kullanılmıyor

**Konum:** `src/consensus/slashing.rs` (`SlashingEvidence::slash_amount` + `SlashingType`).

**Açıklama:** Bu modüldeki `slash_amount` (DoubleSign → tam stake, vb.) registry'nin `RegistryParams::slash_ratio` değerleriyle **tutarsız** ve canlı yol tarafından **hiç kullanılmıyor**. Ölü kod + tutarsız iş mantığı, gelecekteki bir birleştirme hatasına zemin hazırlar.

**Önerilen düzeltme:** Modül silinirse (bkz. #1) bu da gider. Silinmezse, slash oranlarını tek bir `RegistryParams` kaynağından alacak şekilde birleştirin.

---

## Olumlu bulgular (sağlam olan yollar)

Raportun dengeli olması için, denetim sırasında **doğru** bulunan kritik noktalar:

- ✅ **PoS double-sign kanıtı gerçekten doğrulanıyor:** `PoSEngine::verify_evidence` (`src/consensus/pos.rs:112`) her iki başlığın imzasını `BlockHeader::verify_signature` ile kontrol ediyor; `verify_signature` üretici adresini (== ed25519 pubkey) kullanıp `calculated_hash == self.hash` kontrolü yaptığından salahiyet (spoof) mümkün değil.
- ✅ **RPC `submit_slashing_report` güvenli:** Gelen raporun `provenance`'ı zorla `Unverified` yapılıyor; registry `is_actionable()` yalnızca `ConsensusVerified` raporları slash'lıyor → harici提交的 (RPC) kanıt asla slash'a yol açmıyor.
- ✅ **BLS finality quorum'u doğru:** `FinalityCert::verify` oydaş stake ≥ `quorum_stake()` ve geçerli agregat imza kontrolü yapıyor; `FinalityAggregator` prevote quorum'u şart koşuyor.
- ✅ **Cross-domain bridge replay koruması var:** `ReplayNonceStore` + `verify_id()` + durum geçiş makinesi (`Locked → Minted → Burned → Unlocked`) ve TUR 6 sweep ile abandoned-lock DoS'su kapatılmış.
- ✅ **Keyfile izinleri:** TUR 6 ile `0o600` (create-then-chmod penceresi kapalı) ve hata artık yutulmuyor.
- ✅ **RPC auth varsayılanı `true`** (fail-safe); `allowed_ips` localhost.

---

## Risk özeti

- **Canlı (live) saldırı yüzeyi:** Son audit'lerden sonra **düşük**. RPC/quorum/slashing/bridge yolları sağlam.
- **En yüksek kalıcı risk:** #1 (latent kritik — sahte-verify API, gelecekte bağlanırsa korkunç) ve #3 (PoW konsensüs determinizmi).
- **Öncelikli aksiyon:** (a) `consensus/slashing.rs` modülünü silin veya gerçek doğrulama ekleyin; (b) PoW zorluğunu blok/state'e taşıyın; (c) QC blob quorum kontrolünü dedup-sonrası yapın; (d) `BlsKeypair::from_bytes` bütünlük kontrolü ekleyin.

---
*Rapor, depodaki mevcut kaynak kodunun (`main` dalı, 2026-07-13 itibarıyla) statik incelemesine dayanmaktadır. Dinamik/fuzzing testleri ayrıca önerilir (repo `fuzz/` ve `benches/` dizinleri içeriyor).*

---

# BÖLÜM 2 — Derinlemesine 2. Tur İnceleme (Ek Bulgular)

İlk turdan sonra şu katmanlar satır satır incelendi: `settlement/proof_verifier.rs`, `domain/finality_adapter.rs` (tüm adapter'lar), `domain/commitment_registry.rs`, `network/node.rs` (p2p mesaj dispatch), `network/protocol.rs`, `execution/executor.rs`, `core/account.rs` ekonomik fonksiyonları. Bu turda **yeni bir HIGH ve bir MEDIUM** bulgu tespit edildi.

---

## 9. 🔴 PoW cross-domain finality tamamen self-declared → sahte commitment ile bridge mint forgery

**Konum:** `src/domain/finality_adapter.rs` — `PoWFinalityAdapter::verify_finality`; tetikleyici zincir: `src/chain/blockchain.rs` `submit_verified_domain_commitment` → `accept_domain_commitment` → `domain_commitment_registry.insert(...)`; ve `mint_bridge_transfer_from_verified_event` → `verify_event_from_registry` (`src/domain/commitment_registry.rs:54`, finality gate **yok**).

**Açıklama:** `PoWFinalityAdapter::verify_finality` yalnızca şunları kontrol eder:
- `declared_head_hash == commitment.domain_block_hash` (bağlama — tamam),
- `declared_cumulative_work != 0`,
- `declared_cumulative_work >= confirmations * min_work_per_confirmation`,
- `confirmations >= min_depth`.

**Ama `commitment.domain_block_hash`'ın gerçekten bu kadar iş içerdiğini / yeterli PoW zorluğunda olduğunu hiçbir zaman doğrulamaz.** `min_work_per_confirmation` varsayılanı **`1`** olduğu için iş eşiği pratikte yok. Yani bir PoW domain'i için finality, gönderenin *kendi beyan ettiği* sayılara dayanır; blok hash'inin gerçekten `difficulty` kadar leading-zero içerdiği (gerçek light-client PoW kontrolü) **hiçbir yerde** kontrol edilmez.

**Neden kritik — sömürü senaryosu (tamamen ulaşılabilir):**
1. Saldırgan, `bud_submitVerifiedDomainCommitment` RPC'siyle (permissionless) bir PoW domain'i `register_consensus_domain` ile kaydeder (operatör bond 10k) — **veya** `validator_set_hash == 0` olan *mevcut* bir PoW domain'ine sahte commitment üretir.
2. `FinalityProof::PoW { confirmations: 64, declared_head_hash: X, declared_cumulative_work: 64 }` ve rastgele `X` bloğa sahip bir `DomainCommitment` (istediği `event_root` ile) gönderir. `PoWFinalityAdapter` bunu kabul eder (64 ≥ 64, 64 ≥ 64×1) → `accept_domain_commitment` commitment'ı registry'ye **insert** eder.
3. Saldırgan, `event_root`'a uygun bir Merkle kanıtı olan sahte bir `DomainEvent` (`BridgeLocked`, `CrossDomainMessage` kind `BridgeLock`) üretir.
4. `mint_bridge_transfer_from_verified_event(...)` → `verify_event_from_registry` commitment'ı bulur, Merkle kanıtını doğrular → `VerifiedDomainEvent` döner → hedef domain'de **teminatsız (unbacked) token mint** edilir.

`DomainCommitmentRegistry::get` finality durumunu **gate'lemediğinden** (sadece kayıtlı mı diye bakar) ve `submit_verified_domain_commitment` permissionless olduğundan, bu zincir tek bir saldırgan tarafından uçtan uca yürütülebilir. Sonuç: **cross-domain enflasyon / köprüyü boşaltma**.

**Önerilen düzeltme:**
- `PoWFinalityAdapter` içinde `commitment.domain_block_hash`'ın **gerçekten** PoW zorluğunu karşıladığını doğrulayın (hex hash'in leading-zero bit sayısı ≥ domain'in `difficulty`'si; idealde ebeveyn blok zinciri + kümülatif iş ışık-istemci doğrulaması).
- `min_work_per_confirmation` varsayılanını anlamlı bir değere yükseltin **ve** gerçek iş doğrulaması olmadan PoW finality'sini `Finalized` yapmayı yasaklayın.
- `mint_bridge_transfer_from_verified_event` yolunu, sadece **finalize edilmiş** commitment'lara (ör. `commitment.finality_proof_hash` doğrulanmış + bir finalized set'te tutulan height) izin verecek şekilde gate'leyin.
- PoW domain'leri köprü için etkinleştiriliyorsa, en azından kaynak-domain doğrulamasını gerçek bir PoW light-client'a bağlayana kadar köprü mint'ini PoW domain'leri için devre dışı bırakın.

```rust
// PoWFinalityAdapter::verify_finality içinde eklenmesi gereken GERÇEK kontrol:
fn meets_pow(hash: &Hash32, difficulty_bits: u32) -> bool {
    let zeros = hash.iter().take_while(|b| **b == 0).count() * 8
        + hash.iter().skip_while(|b| **b == 0).next().map(|b| b.leading_zeros() as usize).unwrap_or(0);
    zeros as u32 >= difficulty_bits
}
// commitment.domain_block_hash'ın domain zorluğunu karşılamadığı her durumda Rejected.
```

---

## 10. 🟠 `Unstake` / `Vote` tx'lerinde `total_cost` stake'i liquid balance'a yanlış yüklüyor

**Konum:** `src/execution/executor.rs` — `apply_transaction_checked` üst kontrol (satır ~16-23).

**Açıklama:** Fonksiyonun en üstünde tüm tx tipleri için tek bir kontrol var:
```rust
let total_cost = tx.total_cost(); // = amount + fee
let sender_account = state.get_or_create(&tx.from);
if sender_account.balance < total_cost {
    return Err("insufficient_balance");  // her tip için
}
```
Ancak `Unstake`'da `tx.amount` **stake'ten** düşülüyor, liquid `balance`'dan değil; `Vote`'da da sadece `fee` düşülüyor. Oysa üst kontrol `balance < amount + fee` diyerek `amount`'ı liquid balance'a sayıyor.

**Etki:** Tamamen stake etmiş ve 0 liquid balance bırakmış bir validator, `amount > 0` olduğu için bu üst kontrolden geçemez ve **hiçbir zaman unstake yapamaz** (self-DoS / doğruluk hatası). `Vote` için de benzer aşırı-kısıtlama var (balance ≥ 1 + fee gerekiyor ama sadece fee düşülüyor). Hırsızlık açısı yok (üst kontrol yalnızca *alt sınır*; daha fazla harcatmaz), ama meşru stake çekimini bozar.

**Önerilen düzeltme:** Stake tabanlı tiplerde `total_cost` yalnızca `fee` olmalı:
```rust
let cost_floor = match tx.tx_type {
    TransactionType::Unstake | TransactionType::Vote => tx.fee,
    _ => tx.total_cost(),
};
if sender_account.balance < cost_floor {
    return Err(BudlumError::validation("insufficient_balance", "..."));
}
```
Ayrıca `Unstake`'ın gerçek stake düşümü zaten `validator.stake < tx.amount` kontrolüyle korunuyor; bu kontrolü de üstte tutup liquid balance'ı sadece fee için doğrulayın.

---

## 2. Tur'da incelenen ve SAĞLAM bulunan alanlar

Denetimin derinliğini göstermek için, aşağıdaki yollar da incelendi ve **gerçek bir açık bulunamadı** (son TUR audit'lerinin etkisi):

- ✅ **P2P finality yolu:** Gelen `QcBlob` → `import_qc_blob` (quorum kontrolü var), `FinalityCert` → `handle_finality_cert` (BLS quorum + agg imza), `VerifiedDomainCommitment` → `submit_verified_domain_commitment` (adapter'a göre finality doğrulama). Hepsi gerçek doğrulamadan geçiyor; quorum bypass yok.
- ✅ **Handshake:** `node.rs` hem `Handshake` hem `HandshakeAck`'ta `chain_id` kontrolü yapıyor ve uyuşmazlıkta peer'ı banlıyor.
- ✅ **Settlement proof verifier:** `verify_event_against_commitment` domain/height/index/leaf ve Merkle kökünü ayrı ayrı kontrol ediyor; replay koruması sağlam.
- ✅ **PoA / BFT finality adapter'ları:** Gerçek ed25519 (PoA) ve BLS agregat (BFT/PoS) imza doğrulaması + count/stake quorum; self-declared `signer_count` eski açığı kapatılmış.
- ✅ **ZK finality:** `verify_finality_with_claim` ile `ProofClaimRegistry`'ye bağlı, state-root bağlama doğru; trait defaults `Rejected` döndürerek ikinci (registry'siz) doğrulama yolunu kapatıyor.
- ✅ **Executor aritmetiği:** Bakiye işlemleri `saturating_*` ile; supply cap clamp'i mevcut; fee/blok ödülü mantığı tutarlı.

> **Güncellenmiş öncelik:** (1) PoW finality gerçek doğrulama + mint gate (HIGH); (2) `consensus/slashing.rs` sahte-verify API'sini sil/yaz (HIGH latent); (3) PoW zorluğu blok/state'e taşı (ORTA-YÜKSEK); (4) QC blob quorum dedup-sonrası (ORTA); (5) `Unstake` cost-floor (ORTA); (6) `BlsKeypair::from_bytes` bütünlük (ORTA); (7) QC fault-proof ulaşılamazlık (ORTA).
