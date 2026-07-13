# TUR 13 RAPOR — Personas + Z-B ilerleme + org roadmap hizası

**Tarih:** 2026-07-14  
**budlum HEAD:** `03c3bf5`  
**BudZero HEAD:** `c9c5f47`

## Yapılanlar

### 1. Persona paketleri (user / developer / enterprise PoA)
- `config/personas/user-devnet.toml` — `role=rpc`, localhost, auth off  
- `config/personas/developer.toml` — validator devnet, local keys, operator RPC, features on  
- `config/personas/enterprise-poa.toml` — mainnet-shaped, PKCS#11, no disk keys, no mDNS  
- `docs/PERSONAS.md` uyumluluk matrisi  
- `tur13_persona_configs_parse_as_strict_v2` testi  

### 2. Org roadmap denetimi
- `docs/ORG_ROADMAP_AUDIT.md` — budlum-xyz Budlum/BudZero/B.U.D. maddeleri  
- B.U.D. **Tur 14**’e ayrıldı  
- README status tablosu org Research Roadmap + ch12 ile hizalı  

### 3. Z-B Commit 3.5 *ilerleme* (BudZero)
- `merkle_poseidon_round` + pre-round expansion currents  
- original-only root check, expand gas, key continuity  
- **Production gate korundu** (64-depth hâlâ ignore / InvalidProof)  
- Dürüst README: org “31 opcode production” iddiası bu fork’ta *VerifyMerkle için* reddedildi  

### 4. Kapılar
- budlum: 452 lib tests, clippy -D warnings  
- BudZero: clippy clean, proof suite 47 pass / 1 ignore  

## Tur 13.5 / 13.9 / 14 (plan sabit)
- **13.5:** PoW light-client/mint, archive/runbook, benches, RPC quota  
- **13.9:** BLS/PQ key policy, audit checklist, DEVİR notu, roadmap kapanış (B.U.D. hariç)  
- **14:** B.U.D. only  

## Durma
Push + CI success → DUR.
