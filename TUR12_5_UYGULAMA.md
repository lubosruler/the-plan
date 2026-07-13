# TUR 12.5 GENEL RAPOR

**Kapsam:** `lubosruler/budlum` (budlum-core) + `lubosruler/BudZero` (BudZKVM)
**Tarih:** 2026-07-14 (Europe/Istanbul)
**Tür:** Bağımsız güvenlik incelemesi ("hacker gibi" bulgu taraması) — henüz fix/push yok
**Yöntem:** İki reponun tam kaynak arşivi indirilip (`budlum-main.zip`, `BudZero-main.zip`) satır satır incelendi; önceki turlarda "incelenmemiş" olarak işaretlenen alanlara (RPC auth, key management, bridge, settlement proof verifier, storage) öncelik verildi.
**Durma kuralı:** Bu tur yalnızca bulgu tespiti içindir; kod değişikliği/push yapılmadı.

---

## 1. Özet

Önceki turlarda (Tur 1–11.9) kapatılan A1–A13 bulguları kod üzerinde doğrulandı — gerçek, uydurma değil (bkz. §4 Doğrulama). Bu turda 4 yeni bulgu (B1–B4) tespit edildi. En kritik olanı (B1), önceki turlarda "mainnet güvenliği" olarak kapatılan bir kontrolün aslında iddia ettiği korumayı tam sağlamadığını gösteriyor.

| ID | Önem | Konu | Dosya |
|----|------|------|-------|
| B1 | **Kritik** | PKCS#11/HSM mainnet kuralı BLS + PQ anahtarlarını kapsamıyor | `src/consensus/pos.rs`, `src/crypto/primitives.rs`, `src/main.rs` |
| B2 | **Yüksek** | RPC IP allowlist, `X-Real-IP` sahteciliğiyle atlatılabiliyor | `src/rpc/server.rs` |
| B3 | Orta | API key karşılaştırması constant-time değil | `src/rpc/server.rs` |
| B4 | Düşük | Yerel validator anahtar dosyasında parola/KDF yok | `src/crypto/primitives.rs` |

---

## 2. Bulgu detayı

### B1 — PKCS#11/HSM mainnet kuralı BLS + PQ anahtarlarını kapsamıyor

**Dosyalar:** `src/cli/commands.rs` (`validate_strict_rules`), `src/main.rs:335–364`, `src/consensus/pos.rs:531` (`bls_secret_key`), `src/crypto/primitives.rs:217` (`ValidatorKeys::save/load`)

`validate_strict_rules()` mainnet'te `role == "validator"` için `signer_backend == "pkcs11"` şartını zorunlu kılıyor ve eksikse `std::process::exit(1)` ile durduruyor ("CRITICAL SECURITY FAILURE"). Ancak bu kontrol yalnızca `Pkcs11Signer`'ı (`sign_block` → Ed25519 konsensüs imzası) devreye sokuyor.

`main.rs` PoS modunda, `hsm_signer` var olsun olmasın **koşulsuz** olarak şunu çalıştırıyor:

```rust
let keys = if let Some(ref path) = config.validator_key_file {
    match budlum_core::crypto::primitives::ValidatorKeys::load(path) { ... }
}
```

`ValidatorKeys` — `sig_key`, `vrf_key`, `pq_key` (Dilithium5) ve `bls_key` alanlarının hepsini içeriyor. `consensus/pos.rs:531`:

```rust
fn bls_secret_key(&self) -> Option<bls12_381::Scalar> {
    self.validator_keys.as_ref().and_then(|k| k.bls_key.as_ref()).map(|b| b.secret_key)
}
```

Yani BLS finality imzası (`sign_prevote`/`sign_precommit`, `blockchain.rs:3059,3102`) her zaman `validator_keys`'ten, yani diskteki düz-metin anahtar dosyasından geliyor — HSM'den değil. Aynı şey Dilithium5 (PQ) anahtarı için de geçerli.

**Sonuç:** "Mainnet validators require PKCS#11" iddiası yanıltıcı bir güvenlik garantisi veriyor. Projenin kendi tanıttığı "Dilithium5 finality core'a dokunmuş" ve BLS+Dilithium5 hibrit finality mimarisinin asıl imza anahtarları, mainnet'te bile diskte düz metin dosyada duruyor.

**Öneri:** `validate_strict_rules()` mainnet'te BLS ve PQ anahtarları için de HSM/ayrı bir donanım-destekli yol zorunlu kılmalı; ya da en azından `ValidatorKeys::load` çağrısını `signer_backend=pkcs11` olduğunda BLS/PQ secret alanlarını `None` bırakacak şekilde ayırıp, bu anahtarlar için de bir HSM soyutlaması eklemeli.

---

### B2 — RPC IP allowlist, `X-Real-IP` sahteciliğiyle atlatılabiliyor

**Dosya:** `src/rpc/server.rs:403–435` (`extract_client_ip`)

```rust
fn extract_client_ip<B>(config: &RpcSecurityConfig, req: &HttpRequest<B>) -> Option<IpAddr> {
    if !config.trusted_proxies.is_empty() {
        if let Some(forwarded_ip) = ... x-forwarded-for ... { return Some(forwarded_ip); }
    }
    // Fall back to X-Real-IP — trusted_proxies kontrolü YOK
    if let Some(real_ip) = req.headers().get("x-real-ip")... { return Some(real_ip); }
    None
}
```

`X-Forwarded-For` doğru şekilde `trusted_proxies` doluyken kullanılıyor, ama `X-Real-IP` fallback'i **koşulsuz** çalışıyor. Varsayılan `allowed_ips = ["127.0.0.1", "::1"]` (bkz. `RpcSecurityConfig::default()` ve `operator_default()`). Uzaktaki herhangi bir istemci, kendi HTTP isteğine `X-Real-IP: 127.0.0.1` header'ı ekleyerek `is_ip_allowed()` kontrolünü geçebilir — çünkü bu header istemci tarafından serbestçe ayarlanabilir ve önünde gerçek bir reverse-proxy yoksa hiçbir doğrulamadan geçmiyor.

**Risk büyütücü:** `RpcSecurityConfig::operator_default()`'ın kendi uyarısı, `auth_required=false` iken `bud_lockBridgeTransfer`, `bud_sealGlobalHeader`, `bud_submitSlashingEvidence` gibi state-değiştiren metodların tek korumasının IP allowlist olduğunu söylüyor. Bu bypass ile, düğüm dışarıya açıksa, auth kapalıyken bu metodlara uzaktan erişim mümkün olur.

**Öneri:** `X-Real-IP`'yi de `trusted_proxies` boşken tamamen yok say; ya da IP doğrulamasını header'a değil gerçek TCP peer adresine (hyper/tower `ConnectInfo`) dayandır.

---

### B3 — API key karşılaştırması constant-time değil

**Dosya:** `src/rpc/server.rs:379` (`is_authorized`)

`x-api-key` / `Authorization: Bearer` değeri, sabit-zamanlı olmayan `HeaderValue` eşitliğiyle karşılaştırılıyor. Ağ üzerinden pratik sömürülebilirliği düşük olsa da standart sertleştirme eksikliği; `subtle::ConstantTimeEq` (zaten `secrecy`/`subtle` ekosistemi projede mevcut, `pkcs11.rs` içinde `secrecy` kullanılıyor) ile değiştirilmesi öneriliyor.

---

### B4 — Yerel validator anahtar dosyasında parola/KDF yok

**Dosya:** `src/crypto/primitives.rs:217` (`ValidatorKeys::save`)

Tur 6'da dosya izni `0o600` olarak sertleştirilmiş (iyi), ama içerik hâlâ ham/düz-metin anahtar baytları (sig_key + vrf_key + pq_key + bls_key concatenation). Parola tabanlı şifreleme (KDF + AEAD) yok. B1 nedeniyle bu artık yalnızca devnet/testnet riski değil — mainnet'te de BLS/PQ anahtarları için tek koruma bu.

---

## 3. Doğrulama (bu turun kendi bulgusu değil, TUR11 raporunun teyidi)

Yüklenen `TUR11_GENEL_RAPOR.md` canlı repo ile karşılaştırıldı:
- `lubosruler/budlum` Actions sekmesinde raporda geçen tüm commit SHA'ları (`83eff2c`, `1fcdc8b`, `59309fc`, `8210ad0`, `8ac9e01`, `6d3ae58`) gerçek ve CI yeşil.
- `83eff2c` diff'i açılıp incelendi; A1/A2/A3/A11 değişiklikleri (`account.rs`, `executor.rs`, `params.rs`, `zkvm.rs`) satır satır raporla eşleşiyor.
- Bridge mint yolu (`mint_bridge_transfer_from_verified_event`, `blockchain.rs:982`) forgery-gate'i (`expected_block_hash` zorunluluğu, Tur 9) doğru uyguluyor.
- `DomainCommitmentRegistry::insert` kendi başına doğrulama yapmıyor, ama üretim yolunda (`#[cfg(not(test))] submit_domain_commitment`) ham commitment girişi tamamen kapalı; kabul yalnızca `submit_verified_domain_commitment` → `verify_domain_commitment_finality` (consensus-türüne özel finality adapter + `FinalityStatus::Finalized` şartı) üzerinden mümkün.
- Proof envelope gerçekten `postcard` + 10 MB sınır kullanıyor (`bud-proof/src/plonky3_prover.rs`).

Bu alanlarda ek bulgu yok — önceki turların işi sağlam.

---

## 4. Sonraki adım önerisi (Tur 13 girişi)

1. **B1 öncelikli** — BLS/PQ anahtarları için HSM zorunluluğu ya da ayrı bir korumalı-depolama yolu.
2. B2 — `X-Real-IP` güvenlik açığını kapat (tek satırlık, düşük riskli bir fix).
3. B3 — constant-time karşılaştırma.
4. B4 — isteğe bağlı: yerel anahtar dosyasına parola tabanlı şifreleme.

Onay verirsen B1/B2 için kod diff'ini hazırlayıp aynı "kapı disiplini" (fmt + clippy -D warnings + test + push + CI yeşil) ile Tur 13 olarak uygulayabilirim.
