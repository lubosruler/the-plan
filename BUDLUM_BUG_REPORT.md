# BUDLUM + BudZero — Hacker-Mode Derin Güvenlik Denetimi (Konsolide Rapor)

- **Hedefler:**
  - `github.com/lubosruler/budlum` — budlum-core (Rust L1, "Evrensel Mutabakat Katmanı")
  - `github.com/lubosruler/BudZero` — BudZKVM + STARK prover/verifier (`bud-proof`, `bud-vm`, `bud-isa`)
- **Tarih:** 2026-07-13 (taze kod — `git` HEAD: budlum `3ef315d`, BudZero `6ff8067`)
- **Mod:** Adversarial / "sen bir hackersın" — statik kaynak kodu incelemesi + exploit zinciri türetimi.
- **Yöntem:** Her iki repo klonlandı, `bud-proof/src/plonky3_air.rs`, `plonky3_prover.rs`, `bud-vm/src/lib.rs`, `bud-proof/src/adapter.rs`, budlum `submit_zk_proof` / `finality_adapter` / `commitment_registry` / `prover/mod.rs` satır satır okundu.

> **Önceki rapor (`BUDLUM_BUG_REPORT.md`) ile ilişki:** O rapor budlum'a ait 10 bulguyu içeriyor ve ZK finality'yi **"✅ state-root bağlama doğru"** diyerek *güvenli* işaretlemişti. **Bu taze denetim o yeşil tiki GERÇEK DIŞI BULDU:** BudZero STARK AIR'i `final_state_root` dahil 48 public input'dan 40'ını hiçbir kısıta bağlamıyor. Yani BudZero ZK prover/verifier'ı **soundness break'li (forgeable proof)**. Önceki raporun ZK "✅"u bu raporda **İPTAL (REVOKED)** edilmiştir. Aşağıdaki Bölüm 1 (BudZero) en üst önceliktir.

---

## 🚨 Güncellenmiş Özet Tablosu — Tüm Bulgular (14)

| # | Önem | Dosya / Satır | Başlık |
|---|------|---------------|--------|
| **Z-A** | 🔴 **KRİTİK** | `BudZero/bud-proof/src/plonky3_air.rs:121,443-452` + `plonky3_prover.rs:815-857` | STARK AIR `final_state_root` (+40 public input) kısıtını **hiç bağlamıyor** → forgeable ZK proof → budlum'da keyfi state/mint |
| **Z-B** | 🔴 **KRİTİK** | `BudZero/bud-proof/src/plonky3_air.rs:359-360` + `bud-vm/src/lib.rs:412` | `VerifyMerkle` opcode yalnızca `assert_bool` → sahte Merkle/SMT üyelik kanıtı |
| **Z-C** | 🟠 ORTA | `BudZero/bud-proof/src/plonky3_air.rs:260-271` | Halt/termination kısıtı yok → trace yarıda bitebiliyor (program tamamlandı mı? belirsiz) |
| **Z-D** | 🟠 ORTA | `BudZero/bud-vm/src/lib.rs:145,452` | `Vm::step` hatalı adımı trace'e **atmıyor** (Z-C ile birleşir) |
| 1 | 🔴 YÜKSEK (latent) | `budlum/src/consensus/slashing.rs` | Slashing `verify_*` API'sı imza doğrulamıyor (sahte-verify) |
| 2 | 🟠 ORTA | `budlum/src/chain/blockchain.rs:1796` + `qc.rs:205` | QC blob imza sayısı dedup'tan önce kontrol ediliyor |
| 3 | 🟠 ORTA-YÜKSEK | `budlum/src/consensus/pow.rs:11,122` | PoW zorluğu blok/state'te taşınmıyor; `validate_block` yan etkili |
| 4 | 🟠 ORTA | `budlum/src/chain/blockchain.rs:1762` + `qc.rs:395` | QC fault-proof tespiti ulaşılamaz → finality geri alınamaz |
| 5 | 🟠 ORTA | `budlum/src/crypto/primitives.rs:120` | `BlsKeypair::from_bytes` anahtar bütünlüğünü doğrulamıyor |
| 6 | 🟡 DÜŞÜK | `budlum/src/core/account.rs:557` | Double-sign slash'ı registry'de `LivenessFault` olarak etiketleniyor |
| 7 | 🟡 DÜŞÜK | `budlum/src/crypto/primitives.rs:295,350` | `KeyPair::generate/save` secret/yol bilgisini `stdout`'a basıyor |
| 8 | 🟡 DÜŞÜK | `budlum/src/consensus/slashing.rs` | Eski slash oranları tutarsız/kullanılmıyor |
| 9 | 🔴 YÜKSEK | `budlum/src/domain/finality_adapter.rs` (`PoWFinalityAdapter`) + `commitment_registry.rs:54` | PoW cross-domain finality self-declared → sahte commitment ile bridge mint forgery |
| 10 | 🟠 ORTA | `budlum/src/execution/executor.rs` | `Unstake`/`Vote` `total_cost` stake'i liquid balance'a yanlış yüklüyor |

**Öncelik sırası (yeni):** **Z-A → Z-B → (Z-C, Z-D) → 9 → 1 → 3 → 2/5/10 → 4 → 6/7/8.**

---

# BÖLÜM 1 — BudZero ZK Prover/Verifier Soundness Breaks (YENİ)

> **Temel ilke:** plonky3 PCS (FRI + `TwoAdicFriPcs` + `MerkleTreeMmcs` + opening) **sound**'dur; sorun %100 `BudAir::eval` içindeki **under-constrained AIR**'dadır. STARK güvenliği, AIR'in *iddia edilen her özelliği trace'e bağlamasına* bağlıdır. BudZero AIR'i public input'ların büyük kısmını bağlamadığı için bir saldırgan, matematiksel olarak geçerli (FRI açısından sağlam) ama *anlamsız* kanıtlar üretebilir.

---

## Z-A. 🔴 KRİTİK — STARK AIR `final_state_root` ve 40 public input'ı hiçbir kısıta bağlamıyor (Forgeable ZK Proof)

**Konum:**
- `BudZero/bud-proof/src/plonky3_air.rs:121` — `num_public_values() -> 48`
- `BudZero/bud-proof/src/plonky3_air.rs:443-452` — tek public-input referansları
- `BudZero/bud-proof/src/plonky3_prover.rs:815-857` — `to_public_values` (48 değerin paketlenme sırası)

**Kanıt (AIR'deki tek public-input kullanımları):**
```rust
// plonky3_air.rs:443 — yalnızca GAS_USED bağlı (son satır)
let expected_gas = public_inputs[34].into()
    + public_inputs[35].into() * AB::Expr::from(AB::F::from_u64(1 << 32));
builder.when_last_row().assert_zero(cur_gas - expected_gas);

// plonky3_air.rs:448-452 — yalnızca SYSCALL varsa sender/nonce/block_height bağlı
let expected_sender = public_inputs[26].into() + public_inputs[27].into() * (1<<32);
let expected_bh    = public_inputs[30].into() + public_inputs[31].into() * (1<<32);
let expected_nonce = public_inputs[28].into() + public_inputs[29].into() * (1<<32);
// ... sadece `when(is_syscall)` içinde kullanılıyor
```

AIR'in referans verdiği public-input indeksleri: **yalnızca** `[26,27]` (sender), `[28,29]` (nonce), `[30,31]` (block_height — hepsi yalnızca syscall varsa), `[34,35]` (gas_used). Geri kalan **40 indeks hiçbir kısıtta geçmiyor.**

`to_public_values` (plonky3_prover.rs:815) bu 48 değeri şu sırayla paketliyor:
```
[0,1]   chain_id
[2..10] program_hash
[10..18] initial_state_root
[18..26] final_state_root      <-- KRİTİK: AIR'DE HİÇ GEÇMİYOR
[26,27] sender
[28,29] nonce
[30,31] block_height
[32,33] gas_limit
[34,35] gas_used               <-- tek bağlı olanlardan
[36,37] exit_code
[38,39] trace_len
[40..48] event_digest
```

**Soundness açığı:** `final_state_root` (ve `initial_state_root`, `chain_id`, `gas_limit`, `exit_code`, `trace_len`, `event_digest`, `program_hash`) trace'e hiç bağlı değil. Bir prover, **herhangi geçerli bir yürütme izi (trace) alıp** `public_inputs.final_state_root`'u **rastgele 32-byte değere** ayarlayarak STARK proof üretebilir; AIR bu alanı kontrol etmediği için verifier `Ok(())` döner. Yani **48 public input'dan 40'ı attacker kontrolünde serbest** — bu bir "constraint completeness" (tamlık) hatasıdır, kriptografik bir kırılma değil ama ZK soundness'ı tamamen yok eder.

**Neden özellikle `final_state_root` korkunç:** budlum `submit_zk_proof`, STARK doğrulamasından dönen `public_inputs.final_state_root`'u doğrudan kullanıyor (aşağıdaki exploit zinciri).

**Exploit zinciri (budlum'a uzanır):**
1. `budlum/src/chain/blockchain.rs:1348` — `submit_zk_proof` çağırır:
   ```rust
   bud_proof::DefaultAdapter::verify(&submission.proof, &submission.public_inputs, &submission.program)
   ```
2. `bud-proof/src/plonky3_prover.rs::verify` içindeki `public_inputs_hash` ve `program_hash` kontrolleri **attacker'ın kendi gönderdiği değerlerinin iç tutarlılığıdır** (envelope hash'i ve keccak(program) attacker'ın `expected_inputs`'iyle eşleşecek şekilde ayarlanır) — yani soundness'a katkısı yok. Asıl STARK doğrulaması `to_public_values(expected_inputs)` ile yapılır; AIR `final_state_root`'u bağlamadığından geçer.
3. `blockchain.rs:1363` — `let final_state_root = submission.public_inputs.final_state_root;` claim root olarak yazılır (`proof_claims.record(AcceptedProofClaim { final_state_root, .. })`).
4. Tasarımın amaçladığı (ve `src/tests/settlement_prod.rs` `zk_finality_real_proof` modülünün beklediği) akış: `submit_verified_domain_commitment` → `ZkFinalityAdapter::verify_finality_with_claim` (finality_adapter.rs:514) → `commitment.state_root == *final_state_root` bağlaması.

**Saldırganın kazancı:** `final_state_root = TARGET` (istediği state kökü) ile forge edilmiş "doğrulanmış" bir ZK claim üretir; bunu `TARGET` state_root'lu bir `DomainCommitment` (ve içinde sahte `BridgeLocked` event'i taşıyan `event_root`) ile eşleştirip **Zk domain finality'sini ve cross-domain mint'i** istediği gibi ilerletir.

> **ÖNEMLİ NÜANS (dürüstlük):** Mevcut `main` kodunda `verify_domain_commitment_finality` (blockchain.rs:841) Zk için `ZkFinalityAdapter::verify_finality` (generic trait) çağırıyor; bu fonksiyon **her zaman `Rejected`** dönüyor (fail-closed default, finality_adapter.rs:580-591). Yani `verify_finality_with_claim` şu an hiç çağrılmıyor ve exploit zinciri *şu an "tesadüfen" kapalı*. Ancak:
> - `submit_zk_proof` yine de forgeable proof'ları kabul edip `proof_claims` registry'sini attacker'ın seçtiği `final_state_root` ile **zehirliyor**.
> - Kod bir kablolama adımı uzakta tam istismara açık (`zk_finality_real_proof` testleri zaten bu bağlantıyı bekliyor).
> - Bu "kazara kapalılık" bir güvenlik kontrolü DEĞİL; bir afterthought'tir ve gelecekteki her Zk-finality bağlantısı anında weaponize olur.
>
> Bu nedenle **Z-A yine de KRİTİK**; "şu an exploit edilemiyor" değil, "şu an exploit edilmiyor ama verifier mathematically unsound ve registry zehirleniyor".

**Önerilen düzeltme (Rust sketch):**

Temel kural: *her* public input ya (a) trace'den türetilip AIR'de kontrol edilmeli, ya da (b) sabit bir domain常量 olmalı. `final_state_root` için, VM'in gerçekten ürettiği bir değere bağlamak gerekir (ör. final hafıza/storage üzerinden Poseidon hash, veya register dosyası + storage writes). Minimal soundness düzeltmesi:

```rust
// plonky3_air.rs — yeni sütun(lar): COL_FINAL_ROOT_0..7 (Halt satırında VM tarafından yazılır)
// ve COL_INIT_ROOT_0..7, COL_EXIT_CODE, COL_TRACE_LEN_CTR, COL_EVENT_DIGEST_0..7 ...

// 1) final_state_root: son gerçek (cpu_active=1, is_halt=1) satırda committed root'a eşit olmalı
builder
    .when(is_halt.clone())
    .when(cpu_active.clone())
    .assert_eq(public_inputs[18].into(), committed_final_root_0)
    .assert_eq(public_inputs[19].into(), committed_final_root_1)
    /* ... [18..26] tamamı */;

// 2) initial_state_root: ilk satırda VM'e verilen init root'a eşit
builder.when_first_row().assert_eq(public_inputs[10].into(), COL_INIT_ROOT_0); /* ... [10..18] */

// 3) chain_id: sabit domain常量 (preprocessed constant ya da ilk satırda assert)
builder.when_first_row().assert_eq(
    public_inputs[0].into() + public_inputs[1].into() * (1u64<<32),
    AB::Expr::from(AB::F::from_u64(CHAIN_ID)),
);

// 4) gas_limit: VM'in gas_limit'ine eşit (ilk satırda bir trace sütunuyla)
builder.when_first_row().assert_eq(
    public_inputs[32].into() + public_inputs[33].into() * (1u64<<32),
    COL_GAS_LIMIT,
);

// 5) exit_code: 0 (normal Halt) veya 1 (error) — #C/#D düzeldikten sonra anlamlı
builder.when(is_halt.clone()).assert_eq(
    public_inputs[36].into() + public_inputs[37].into() * (1u64<<32),
    COL_EXIT_CODE,
);

// 6) trace_len: cpu_active=1 satır sayısına eşit (kümülatif sayan bir sütun)
builder.when_last_row().assert_eq(
    public_inputs[38].into() + public_inputs[39].into() * (1u64<<32),
    COL_TRACE_LEN_CTR,
);

// 7) event_digest: Log opcode ile güncellenen Poseidon accumulator (son satırda assert)
builder.when(is_halt.clone()).assert_eq(public_inputs[40].into(), COL_EVENT_DIGEST_0); /* ... [40..48] */
```

`committed_final_root_*` sütunları, `trace_matrix` içinde VM'in final state'inden (ör. `state_writes_digest` + register dosyası + storage kökü üzerinden Poseidon) hesaplanıp Halt satırına yazılmalıdır. `program_hash` zaten Program CTL LogUp ile trace'e bağlı olduğundan ek kısıt gerekmez (ama `verify()`'daki keccak kontrolü yine de tutulmalı).

---

## Z-B. 🔴 KRİTİK — `VerifyMerkle` opcode yalnızca `assert_bool` (gerçek Poseidon path doğrulaması YOK)

**Konum:**
- `BudZero/bud-proof/src/plonky3_air.rs:359-360`
- `BudZero/bud-vm/src/lib.rs:412` (`Opcode::VerifyMerkle` — VM doğru hash'i hesaplıyor ama AIR bunu ZORUNLU KILMIYOR)

**Kanıt (AIR):**
```rust
// plonky3_air.rs:359
builder
    .when(is_verify_merkle.clone())
    .assert_bool(rd_val_new.clone());   // <-- SADECE boolean mı? diye bakıyor
```

**Soundness açığı:** `VerifyMerkle` opcode'unun *tek* AIR kısıtı, sonucun 0 veya 1 (boolean) olması. VM (`bud-vm/src/lib.rs:412`) gerçek Poseidon Merkle path'ini hesaplayıp `current == root` sonucunu `rd_val_new`'e yazıyor — **ama prover trace'i doğrudan kendisi inşa ettiği için**, `rd_val_new`'i istediği booleana (0 veya 1) ayarlayabilir; AIR bu değerin *gerçek* hesapla karşılandığını hiçbir zaman kontrol etmiyor.

**Etki:** ZK programları içinde Merkle/SMT path authentication tamamen kırık. Bir prover, yanlış bir `root`/`leaf`/`path` için `VerifyMerkle`'yi istediği sonuçla (ör. "üyelik var" veya "üyelik yok") tamamlayabilir → SMT içinde sahte inclusion/exclusion kanıtı, state geçmişi tahrifi, "var olmayan varlığı kanıtlama" vb.

**Neden Z-B ayrı bir KRİTİK:** Z-A'dan bağımsız. Z-A `final_state_root`'u serbest bırakır; Z-B ise *program içi* doğrulamayı sahteciler. İkisi birleşince bir saldırgan, hem "çalıştırdım ve sonuç budur" (Z-A) hem de "ara kanıtlarım doğru" (Z-B) yalanlarını aynı proof içinde söyleyebilir.

**Önerilen düzeltme (Rust sketch):**

`VerifyMerkle` satırında path verisini trace'e taşımak gerekir (şu an path yalnızca VM hafızasında; AIR'in hafıza sütunları LogUp accumulator olduğu için doğrudan okunamaz). İki adım:

1. `trace_matrix` / `Vm` — VerifyMerkle satırına `key` + 64 `sibling` hash'ini taşıyan yeni trace sütunları ekle (`COL_VM_KEY`, `COL_VM_SIB_BASE..64`).
2. `BudAir::eval` — mevcut Poseidon witness makinesiyle root'u yeniden hesapla, `claimed_root = rs1_val` ile karşılaştır, boolean sonucu `Eq` inverse-witness ile üret ve `rd_val_new`'e bağla:

```rust
// is_verify_merkle satırında:
//  path'ten computed_root = poseidon4 hash zinciri (mevcut COL_POSEIDON_* makinesi yeniden kullanılabilir)
//  eq = (computed_root == rs1_val)  -> inverse witness ile boolean
let diff = computed_root.clone() - rs1_val.clone();
let diff_inv = /* inverse witness sütunu */;
let eq_z = diff.clone() * diff_inv.clone();
builder.when(is_verify_merkle.clone()).assert_bool(eq_z.clone());
builder.when(is_verify_merkle.clone()).assert_zero(diff * (one.clone() - eq_z.clone()));
// rd_val_new ZORUNLU olarak doğru sonuç olmalı:
builder.when(is_verify_merkle.clone()).assert_eq(rd_val_new, eq_z);
```

> **Geçici (acil) önlem:** `VerifyMerkle` düzeltilene kadar `Plonky3Adapter::verify` içinde bu opcode'lu proof'lar **reddedilmeli** (deny-list), çünkü şu an "her sonuç geçerli" durumunda.

---

## Z-C. 🟠 ORTA — Halt / termination kısıtı yok → trace yarıda bitebiliyor

**Konum:** `BudZero/bud-proof/src/plonky3_air.rs:260-271`

**Kanıt (mevcut cpu_active/is_halt kısıtları):**
```rust
builder.when_first_row().assert_one(cpu_active.clone());                       // 260
builder.when_transition().assert_zero(nxt_cpu_active.clone() * (one.clone() - cpu_active.clone())); // 263: 0->1 geçiş yok
builder.when_transition().when(cpu_active.clone()).when(is_halt.clone()).assert_zero(nxt_cpu_active.clone()); // 268: halt -> inactive
builder.when(one.clone() - cpu_active.clone()).assert_one(is_halt.clone());   // 271: inactive satırlar halt
```

**Eksik olan:** Son `cpu_active=1` satırının bir **Halt** olmasını zorlayan kısıt YOK. Yukarıdaki kurallar yalnızca "aktiflik 1→0'e sadece Halt ile geçer" ve "inactive satırlar is_halt=1" der. Ancak son gerçek satır `cpu_active=1, is_halt=0` (herhangi bir opcode) olabilir; sonraki geçiş padding'e (cpu_active=0, is_halt=1) yukarıdaki kuralları ihlal etmeden yapılır.

**Etki:** "Program gerçekten Halt etti mi?" sorusu AIR'de cevaplanamaz. Z-A ile birleşir: zaten `final_state_root` serbestken, üstüne programın *tamamlanıp tamamlanmadığı* da belirsiz. Bir "programın başarıyla bitmesi gerekiyor" garantisine dayanan her mantık (örn. Zk finality'de `exit_code == 0`) bypass edilebilir.

**Önerilen düzeltme (Rust sketch):**
```rust
// Tek 1->0 geçişi yalnızca Halt satırında olmalı:
builder
    .when_transition()
    .when(cpu_active.clone())
    .when(one.clone() - nxt_cpu_active.clone())
    .assert_zero(one.clone() - is_halt.clone());
```
Bu, son gerçek (cpu_active=1) satırın mutlaka gerçek bir Halt opcode'u olmasını zorlar (normal yürütmede tüm geçişler 1→1'dir; tek 1→0 padding'e geçiştir).

---

## Z-D. 🟠 ORTA — `Vm::step` hatalı adımı trace'e atmıyor

**Konum:** `BudZero/bud-vm/src/lib.rs:145` (`consume_gas` `?`), `:412` (VerifyMerkle), `:452` (`self.trace.push`)

**Kanıt:**
```rust
pub fn step(&mut self, program: &[u64]) -> Result<(), VmError> {
    // ...
    self.consume_gas(Self::gas_cost(inst.opcode))?;   // 145: HATA -> hemen return, push YOK
    // ...
    let (dst_val, next_pc) = match inst.opcode {
        Opcode::Load => { /* ... */ } // bellek hatasında `return Err(...)` (push öncesi)
        // OutOfGas / StackOverflow / StackUnderflow / InvalidOpcode / InvalidPc / AssertionFailed
        // hepsi `return Err` ile, self.trace.push'TAN ÖNCE dönüyor
    };
    // ...
    self.trace.push(Step { /* ... */ });              // 452: yalnızca başarılı adımda
    Ok(())
}
```

**Soundness açığı:** Hata veren adım trace'e yazılmadığı için, hatalı (out-of-gas/error) bir programın trace'i **son (hatalı) adımı düşmüş** olarak üretilir. Z-C (halt kısıtı yok) ile birleşince trace'in son satırı sadece padding'dir (cpu_active=0) — yani "program erken durdu" ile "program normal Halt etti" ayırt edilemez. Soundness, trace'in *her adımı (termination dahil) sadık olarak* kaydetmesini gerektirir; bu boşluk Z-C'yi istismar edilebilir yapıyor.

**Önerilen düzeltme (Rust sketch):**
```rust
pub fn step(&mut self, program: &[u64]) -> Result<(), VmError> {
    // Hata durumunda bile adımı kaydet (terminal row), sonra Err dön.
    // (Z-C düzeltmesinden sonra: son gerçek satır Halt OLMALI; hatalı bitiş
    //  AIR tarafından reddedilir → yalnızca başarıyla Halt eden programlar
    //  geçerli proof üretebilir, bu doğru soundness semantiğidir.)
    let result = self.step_inner(program);   // gövde ayrı fonksiyon
    // step_inner hata döndürse de terminal Step push edilmiş olsun:
    if let Err(e) = result {
        self.halted = true;
        self.error = Some(e.clone());
        // terminal Step (is_halt=0, cpu_active=1) zaten push edildi
        return Err(e);
    }
    Ok(())
}
```
Pratikte: `step_inner` içindeki her `return Err` yerine, önce `self.trace.push(terminal_step)` yapıp sonra `Err` dön. Z-C ile birlikte bu, "hatalı biten program = geçersiz proof" garantiler.

---

# BÖLÜM 2 — budlum-core Bulguları (önceki rapordan entegre, 10 bulgu)

> Aşağıdakiler `BUDLUM_BUG_REPORT.md` içinde detaylı (file:line + fix sketch) bulunur; burada derin denetimin bağlamı için özeti verilir. **Tümü hâlâ geçerlidir** (taze kodda yeniden teyit edildi; TUR 5/6/7 commit'leri bazı yolları sertleştirdi ama bu 10 bulgu korunuyor).

| # | Önem | Kısa özet | Anahtar düzeltme |
|---|------|-----------|------------------|
| 1 | 🔴 YÜKSEK (latent) | `consensus/slashing.rs` `verify_double_*` imza doğrulamıyor, sadece yapısal bakıyor | Modülü silin ya da gerçek ed25519/Dilithium doğrulama ekleyin |
| 2 | 🟠 ORTA | `blockchain.rs:1796` QC blob quorum'u ham imza sayısına göre yapar, dedup sonrası değil | Quorum kontrolünü **dedup sonrası benzersiz imzacı** sayısına göre yapın |
| 3 | 🟠 ORTA-YÜKSEK | `pow.rs:11,122` zorluk blok/state'te taşınmıyor; `validate_block` `current_difficulty`'yi yan etkiyle değiştiriyor | Zorluğu blok başlığına/`(height)` fonksiyonuna taşıyın; `validate_block` yan etkisiz olmalı |
| 4 | 🟠 ORTA | `blockchain.rs:1762` + `qc.rs:395` fault-proof tespiti boş döner → finality geri alınamaz | İkinci bağımsız challenge kaynağı ekleyin ya da "geri alınamaz" tasarımı belgeleyin |
| 5 | 🟠 ORTA | `crypto/primitives.rs:120` `BlsKeypair::from_bytes` G2 parse/validate + pk==sk türetme yapmıyor | `G2Affine::from_compressed` + `expected = G2(generator*sk)` kontrolü ekleyin |
| 6 | 🟡 DÜŞÜK | `core/account.rs:557` double-sign slash'ı registry'de `LivenessFault` etiketli | Gerçek `SlashingCondition` parametresini geçirin |
| 7 | 🟡 DÜŞÜK | `crypto/primitives.rs:295,350` `println!` ile secret/yol stdout'a | `tracing::debug!` kullanın |
| 8 | 🟡 DÜŞÜK | `consensus/slashing.rs` eski slash oranları tutarsız/kullanılmıyor | Tek `RegistryParams` kaynağına bağlayın |
| 9 | 🔴 YÜKSEK | `finality_adapter.rs` `PoWFinalityAdapter` self-declared work → sahte commitment ile bridge **mint** | `domain_block_hash`'ın gerçek PoW zorluğunu doğrulayın; `min_work_per_confirmation` anlamlı + mint gate |
| 10 | 🟠 ORTA | `executor.rs` `Unstake`/`Vote` üst kontrolde `total_cost` stake'i liquid balance'a yanlış yüklüyor | Stake-tabanlı tiplerde cost-floor = `fee` olsun |

**Önceki raporun SAĞLAM dediği ama BU RAPORLA İPTAL EDİLEN madde:**
- ❌ Eski rapor: *"✅ ZK finality: `verify_finality_with_claim` ile `ProofClaimRegistry`'ye bağlı, state-root bağlama doğru"*. **İPTAL** — Z-A nedeniyle STARK verifier `final_state_root`'u bağlamadığından, registry'ye yazılan root attacker kontrolünde; "bağlama doğru" varsayımı yanlış.

---

# BÖLÜM 3 — Risk Özeti & Öncelikli Aksiyon

- **Ağ çapında en yüksek risk artık BudZero ZK soundness (Z-A, Z-B).** plonky3 PCS sağlam ama AIR under-constrained → forgeable proof. Bu, budlum'un Zk domain finality ve cross-domain bridge mint yolunu doğrudan tehdit eder.
- **Z-A'nın şu an "exploit edilememesi"** `verify_finality_with_claim`'ın generic path'te çağrılmamasına dayanıyor (fail-closed default) — bu bir güvenlik kontrolü değil, kırılgan bir afterthought. `submit_zk_proof` zaten forgeable proof kabul edip registry'yi zehirliyor.
- **Öncelik sırası:**
  1. **Z-A** — AIR'e `final_state_root` + diğer 40 public input'ı bağlayan kısıtları ekle (Bölüm 1 sketch).
  2. **Z-B** — `VerifyMerkle`'yi gerçek Poseidon path kontrolüyle donat; geçici olarak opcode'u verifier'da reddet.
  3. **Z-C + Z-D** — Halt/termination kısıtı + `Vm::step` terminal adım kaydı (birlikte: yalnızca başarıyla Halt eden program geçerli proof).
  4. **#9** — PoW finality gerçek iş doğrulaması + mint gate.
  5. **#1** — Sahte-verify slashing API'sini sil/yaz.
  6. **#3** — PoW zorluğunu blok/state'e taşı.
  7. **#2, #5, #10** — QC dedup-sonrası quorum, BLS key bütünlüğü, Unstake cost-floor.
- **Test notu:** Mevcut BudZero testleri (`proves_verify_merkle_valid/invalid_root/invalid_path`) "geçiyor" çünkü hepsi `assert_bool` ile uyumlu — yani testler açığı **maskeliyor**, yakalayamıyor. Z-B düzeltildikten sonra bu testlerin sahte-path reddini gerçekten doğrulaması gerekir.

---

*Rapor, `budlum` (`3ef315d`) ve `BudZero` (`6ff8067`) depolarının 2026-07-13 itibarıyla taze kaynak kodunun statik incelemesine dayanır. Dinamik PoC/compile-test ayrıca önerilir (özellikle Z-A için: geçerli bir trace ile rastgele `final_state_root` üreten bir proof'ın `Plonky3Adapter::verify`'ı geçtiğini gösteren bir birim test).*
