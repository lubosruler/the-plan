# TUR 11 GENEL RAPOR

**Ajan:** Lubo  
**Tarih:** 2026-07-13 / 2026-07-14 (Europe/Istanbul)  
**Durum:** Tamamlandı — tüm push’lar GitHub’da, CI yeşil  
**Durma kuralı:** Push + CI success → dur; fail → aynı turda çöz (uygulandı)

---

## 1. Özet

Tur 11 serisi, `BUDLUM_ICIN_BULGULAR.md` A bölümündeki açık güvenlik maddelerini kapatmak için planlandı ve yürütüldü. İş üç alt tura bölündü:

| Alt tur | Kapsam | Repo | Sonuç |
|--------|--------|------|--------|
| **11** | A1, A2, A3, A11 + stake recycling + `RegistryParams::validate` | `lubosruler/budlum` | CI yeşil |
| **11.5** | A9, A10 (BudZero state) | `lubosruler/BudZero` | CI yeşil |
| **11.7** | A4, A5, A7, A8 | `lubosruler/budlum` | CI yeşil |
| **11.9** | A6, A12, A13 | budlum + BudZero | CI yeşil |

**Kapanan bulgular:** A1–A13 (A13: prod gate + ignore; tam Z-B math kapanışı Commit 3.5’e bırakıldı).

---

## 2. Commit haritası

### lubosruler/budlum (`main`)

| SHA | Commit |
|-----|--------|
| `83eff2c` | **tur11:** A1/A2/A3/A11 cheap-check order, gov gates, heap mem, param bounds |
| `1fcdc8b` | **tur11.fix1:** clippy-clean RegistryParams validate tests |
| `59309fc` | **tur11.7:** A4 ban expiry, A5 cross-role slash, A7 PoP chain_id, A8 gov bounds |
| `8210ad0` | **tur11.7.fix1:** simplify ban-active predicate (`nonminimal_bool`) |
| `8ac9e01` | **tur11.9:** A6 hash-mix PoA leader selection |
| `6d3ae58` | **tur11.9.fix1:** stabilize PoA fee test under hash-mix |

**HEAD:** `6d3ae58`  
**CI:** success — https://github.com/lubosruler/budlum/actions/runs/29286186878  
**Lib test sayısı (son):** 446 passed

### lubosruler/BudZero (`main`)

| SHA | Commit |
|-----|--------|
| `0a31701` | **tur11.5:** A9 nested backup stack + A10 `save() -> Result` |
| `912213a` | **tur11.5.fix1:** rustfmt `plonky3_air` (CI fmt gate) |
| `a538835` | **tur11.9:** A12 storage gas + A13 Production gate for VerifyMerkle |

**HEAD:** `a538835`  
**CI:** success — https://github.com/lubosruler/BudZero/actions/runs/29285852494

### BudZero pin (budlum CI)

budlum CI hâlâ BudZero’yu **`06246f0`** (tur10.zk) ile pin’liyor.  
Sebep: tur10.5.z_a_phase1+ sonrası basit program STARK verify (`InvalidProof`) — budlum public-input yolu ile BudZero main uyumsuz.  
A12/A13 BudZero `main`’de; L1 prove/verify path pin’de. **Tur 12 önerisi:** pin’i yeşil bir BudZero main’e taşımak (Z-A public input hizası).

---

## 3. Bulgu kapanış detayı

### A1 — İmza / nonce sırası (DoS) ✅
- **Dosya:** `src/core/account.rs` — `validate_transaction_with_context`
- **Önce:** `tx.verify()` → nonce → fee → balance  
- **Sonra:** nonce → fee → balance → `tx.verify()`  
- Ucuz kontroller pahalı imzadan önce.

### A2 — Governance teklif spam ✅
- **Dosya:** `src/execution/executor.rs`
- Teklif oluşturma: `proposer_stake == 0` → reddet (oy ile aynı kapı).
- Bonus: unstake sırasında aktif proposal oy ağırlığı düşer (stake recycling).

### A3 — `SystemTime` panic ✅
- **Dosya:** `src/core/transaction.rs`
- `.unwrap()` → `.map(|d| d.as_millis()).unwrap_or(0)`

### A4 — Ban süresi restart’ta sıfırlanma ✅
- **Dosyalar:** `peer_manager.rs`, `node.rs`
- `PersistedBan { peer_id, expires_unix }`
- Reload: kalan süre = `expires_unix - now` (full window değil)
- Legacy `Vec<String>` hâlâ okunur (`reload_banned_peers_legacy`)

### A5 — Cross-role slash evasion ✅
- **Dosya:** `src/registry/permissionless.rs`
- `slash()` primary role’dan sonra `slash_cross_role()` ile aynı `Address`’in diğer rollerini jail’ler.

### A6 — PoA öngörülebilir lider ✅
- **Dosya:** `src/consensus/poa.rs`
- `block_index % n` kaldırıldı.
- `SHA-256("BUDLUM_POA_LEADER_V1" || height || set fingerprint)` → slot.
- Deterministik; pure round-robin değil. VRF sonraki iterasyon.

### A7 — BLS PoP chain_id domain separation ✅
- **Dosyalar:** `finality.rs`, `blockchain.rs`, testler
- `pop_signing_message(chain_id, address, bls_pk)`
- `verify_pop(entry, chain_id)`
- Snapshot builder zincir `chain_id` kullanır.

### A8 — Governance fee/reward/param sınırları ✅
- **Dosya:** `src/core/account.rs`
- `MAX_BASE_FEE`, `MAX_BLOCK_REWARD` clamp
- `ParameterUpdate` → `RegistryParams` alanları + `validate()`
- Bilinmeyen key / bounds ihlali → log + reject (state değişmez)

### A9 — Nested state backup ✅
- **Dosya:** BudZero `bud-state/src/lib.rs`
- `backup: Option` → `backup_stack: Vec` (LIFO)
- Nested begin/commit/rollback frame’leri bozulmaz.

### A10 — `save()` panic ✅
- **Dosya:** BudZero `bud-state` + `bud-cli`
- `save() -> Result<(), String>` (expect yok)
- CLI hatayı `?` ile yayar.

### A11 — Heap / VM bellek uyumsuzluğu ✅
- **Dosya:** `src/execution/zkvm.rs`
- `Vm::with_gas_limit(1024, …)` → **8192** (compiler heap base 4096)

### A12 — SRead/SWrite gas ✅
- **Dosyalar:** BudZero `bud-vm`, `plonky3_air`
- Load/Store = 3; **SRead = 8**; **SWrite = 12**
- AIR `gas_cost` ifadesi VM ile hizalı (aksi halde InvalidProof)

### A13 — VerifyMerkle (Z-B) — prod gate ✅ / tam math ⏳
- `Opcode::VerifyMerkle` → `is_experimental()`
- **Production** ISA profili decode’da reddeder
- `proves_verify_merkle_valid_64_depth` hâlâ `#[ignore]` (over-constrained)
- `bud-proof` crate’i `experimental` feature ile harness’ta çalıştırabilir
- **Tam kapanış:** Z-B Commit 3.5 (Tur 12 adayı)

---

## 4. Kapı disiplini (gevşetme yok)

Her turda:

1. Kod + test  
2. `cargo fmt --check`  
3. `cargo clippy … -- -D warnings` (**allow eklenmedi**)  
4. `cargo test --lib` / workspace  
5. Push  
6. CI fail → **durmadan** fix commit; success → **DUR**

Clippy fix’leri: stil/predicate sadeleştirme veya AIR/VM hizası — lint kapatma değil.

---

## 5. Bilinçli borç / Tur 12 giriş noktaları

1. **BudZero pin yeniden hizalama** — budlum CI `06246f0` → yeşil `main` (Z-A public inputs)  
2. **A13 Commit 3.5** — valid 64-depth VerifyMerkle AIR; ignore kaldır; prod gate açılabilir  
3. **A6 VRF** — hash-mix geçici; epoch-seed VRF tam çözüm  
4. **Token** — PAT mesaj/raporlarda dolaştı; **revoke + secret store** (DEVİR_RAPORU_YENI §3)  
5. Bölüm C (STARK range-proof vb.) — uzman doğrulama  

---

## 6. Test kanıtı (örnek)

| Paket | Örnek testler |
|-------|----------------|
| budlum | `tur11_*`, `tur117_*`, `tur119_leader_not_pure_round_robin` |
| BudZero state | `tur115_nested_transaction_stack`, `tur115_save_returns_*` |
| BudZero vm/isa | `tur119_storage_gas_above_memory`, `tur119_verify_merkle_disabled_in_production` |

---

## 7. Sonuç

Tur 11 serisi **push onaylı ve CI yeşil** olarak kapanmıştır.  
Açık bulgu listesindeki A1–A12 kapatıldı; A13 prod’da devre dışı + ignore ile güvenli tutuldu.

**Sonraki adım:** Tur 12 (pin rebind + Z-B 3.5 + isteğe bağlı VRF) — kullanıcı talimatı ile başlar; ajan kendiliğinden geçmez.
