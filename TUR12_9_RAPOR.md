# TUR 12.5 — Uygulama (B1–B4 + README)

**Kaynak:** `TUR12_5_GENEL_RAPOR.md` (bağımsız AI) — bulgular kodda doğrulandı.  
**budlum HEAD:** `94916b4`  
**BudZero HEAD:** `f33982a` (README)

## Fixler

| ID | Değişiklik |
|----|------------|
| **B1** | Mainnet: disk `ValidatorKeys` yasak (`validate_strict_rules` + `main.rs` PoS load). PKCS#11 yalnızca Ed25519 consensus; BLS/PQ plaintext iddiası kapatıldı (fail-closed). |
| **B2** | `X-Real-IP` yalnızca `trusted_proxies` doluyken; spoof testi |
| **B3** | API key / Bearer: `subtle::ConstantTimeEq` |
| **B4** | `ValidatorKeys::save` docs + warn; mainnet zaten B1 ile kapalı |

## Test / CI
- Yerel: fmt, clippy `-D warnings`, **450** lib test  
- CI: (poll)

## README (her tur bonus)
- **budlum** + **BudZero** profesyonel yeniden yazım: vizyon, mimari, güvenlik, dürüst roadmap, quick start.

## Sonraki
**Tur 12.9** — ölü / sahte-yeşil / ağa bağlı olmayan yollar + BudZero pin rebind + Z-B 3.5.
