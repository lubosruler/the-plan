# TUR 12.9 RAPOR — Ölü / sahte-yeşil yollar + BudZero pin rebind

**Tarih:** 2026-07-14  
**budlum HEAD:** `cdc1824`  
**BudZero HEAD:** `78e0ef7`

---

## 1. Ne aradık

Tur 12.9 hedefi: testlerde yeşil görünen ama ana ağa / gerçek settlement yoluna bağlı olmayan veya kırık parçalar.

| Bulgu | Tür | Sonuç |
|-------|-----|--------|
| BudZero CI pin `06246f0` | Sahte-yeşil uyumluluk | **Kaldırıldı** → pin `78e0ef7` (main) |
| Log program prove/verify `InvalidProof` | Z-A event_digest bug | **Düzeltildi** (AIR + padding) |
| `event_digest = keccak(events)` | Public input uyumsuzluğu | **AIR limb packing** |
| `Vm::run` vs `run_receipt` | Trace uyumsuzluğu | **`run_receipt`** |
| `ConsensusKind::Zk` → her zaman Rejected | Ölü settlement path | **`verify_finality_with_claim` + registry** |
| QC post-import fault scan | Boş by construction | **Belgelendi** (external challenge) |
| Z-B 64-depth | Hâlâ ignore | Bilinçli borç (Commit 3.5) |

---

## 2. Kök neden: event_digest

Z-A phase2 `event_digest` transition şu formdaydı:

`digest[i+1] = digest[i] + is_log[i] * rs1[i]`

Prover ise Log **satırında** accumulator’ı güncelliyor. Bu yüzden **her Log programı** STARK verify’de düşüyordu. Budlum testleri Log kullandığı için BudZero main’e pin atılamıyordu.

**Fix (BudZero `78e0ef7`):**
1. Transition: `+ is_log[i+1] * rs1[i+1]`
2. Padding satırları son gerçek satırın digest/root/exit/trace_len değerlerini taşır
3. `proves_log_event_digest` pozitif testi

**Fix (budlum `cdc1824`):**
1. `event_digest_air_limbs` (limb0 = sum of log values)
2. `run_receipt` prove path
3. CI pin → `78e0ef7`

---

## 3. Zk finality — sahte bağlantı

`verify_domain_commitment_finality` Zk dalında trait `verify_finality` çağrılıyordu; bu **daima Rejected**.  
`verify_finality_with_claim` + `ProofClaimRegistry` hiç devreye girmiyordu.

Artık Zk commitment finality, önce `submit_zk_proof` ile kabul edilmiş claim root’una bağlanıyor.

---

## 4. Kapılar

| Repo | fmt | clippy -D | tests | CI |
|------|-----|-----------|-------|-----|
| BudZero | ✓ | ✓ | workspace ✓ | (poll) |
| budlum | ✓ | ✓ | **451** lib ✓ | (poll) |

Lint allow yok.

---

## 5. Kalan borç (sonraki tur)

1. Z-B Commit 3.5 — `proves_verify_merkle_valid_64_depth` ignore kaldır  
2. PoW light-client — mint ban kaldırılabilir hale gelsin  
3. BLS/PQ HSM — B1’in tam kapanışı  
4. QC external challenge RPC (post-import scan bilinçli boş)  
5. event_digest 32-bit range-proof (AIR hâlâ yaklaşık)

---

## 6. Durma kuralı

Push atıldı; CI success → **DUR**.
