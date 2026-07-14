# TUR 14.9 TALİMATI — Kontrol Turu Kapanışı

> Kaynak: `docs/ORG_ROADMAP_AUDIT.md` §4a (Tur 14.9 audit, 2026-07-14) +
> `docs/DEVIR_RAPORU.md` §7-8 + `the-plan/TUR14_5_PLAN.md` §6 (repo dışı,
> önce oku). **Bu bir DOĞRULAYAN turdur — yeni B.U.D. özelliği YAZILMAZ.**

## 0. Önce oku
- `CLAUDE.md` (root) — genel proje bağlamı, Tur 1-15 özet log
- `docs/ORG_ROADMAP_AUDIT.md` §4a — bu turun tam denetim tablosu
- `docs/DEVIR_RAPORU.md` — önceki devir, doğrulama komut seti
- (repo dışı) `the-plan/TUR14_5_PLAN.md` §6

## 1. Kapsam — bu turda YAPILACAKLAR

### 1.1 PR #6 (Tur 14.5) CI'ını yeşile çek — tek somut kod dokunuşu
Durum: `cargo fmt --check` adımı 7+ push'tur (`1a5992f` → `a2cd5b1`) kırık;
Azure blob log erişimi çalışmadığından dışarıdan teşhis edilemiyor.

1. `cargo fmt --all -- --check` çalıştır, diff'i incele.
2. `cargo fmt --all` ile düzelt — **sadece format, mantık değiştirme.**
3. Aynısını budzero workspace için yap: `cargo fmt --manifest-path budzero/Cargo.toml --all -- --check`.
4. `cargo clippy --lib --tests -- -D warnings` (+ budzero eşleniği) kırıksa:
   mantık değişikliği gerekiyorsa YAPMA — `// TODO(karar-gerekli): ...` bırak,
   yeni bulgu olarak işaretle.
5. Test seti tam koş (bkz. §3).
6. Push sonrası `gh pr checks 6` ile gerçekten yeşil olduğunu doğrula. Log
   hâlâ erişilemezse bunu bulgu olarak yaz; kullanıcıdan PR sayfasından log
   istenmesi gerekir.

**Kabul kriteri:** PR #6 tüm check'leri yeşil, VEYA yeşil olmayan her check
için isimlendirilmiş somut bir engelleyici bulgu.

### 1.2 Kapalı bulguları re-confirm et (yeniden yazma, sadece doğrula)
- `src/tests/bud_e2e.rs` — 3 aktörlü (operatör/kullanıcı/geliştirici) E2E hâlâ geçiyor mu?
- `storage_registry_has_no_admin_or_pause_or_freeze_hook` hâlâ geçiyor mu?
- `src/tests/permissionless.rs:396-403` PoA izolasyon testleri hâlâ geçiyor mu?

### 1.3 Denetim dosyasını güncelle
`docs/ORG_ROADMAP_AUDIT.md` §4a tablosunu ve "Açık bulgular" / "Kapalı
bulgular" listelerini bu turun sonucuna göre güncelle (kapanan madde varsa
taşı, yeni bulgu varsa ekle).

## 2. Bilinçli YAPILMAYACAKLAR (kapsam dışı)
- **Faz 3** (Proof-of-Storage / `VerifyMerkle` production gate) — Z-B gate
  hâlâ kapalı; `RetrievalChallenge`'ı gerçek kanıta dönüştürme YOK.
- Yeni B.U.D. özelliği, yeni RPC uç noktası, yeni domain tipi.
- **Tur 13.9** maddeleri (BLS/PQ HSM, finality live-path taraması,
  ConsensusStateV2 migration notları, harici audit checklist, README
  kapanış tablosu) — bu tur bunları gölgelemez ama kapatmaz da; açık kalsın.
- `budlum-xyz/B.U.D.` upstream'deki eksik vizyon dokümanını tahmin ederek
  "tamamlama" — bulgu olarak kalsın, kod kararına dönüştürme.

## 3. Doğrulama komutları
```bash
cargo fmt --all -- --check
cargo clippy --lib --tests -- -D warnings
cargo test --lib
cargo fmt --manifest-path budzero/Cargo.toml --all -- --check
cargo clippy --manifest-path budzero/Cargo.toml --workspace --all-targets -- -D warnings
cargo test --manifest-path budzero/Cargo.toml --workspace
cargo test --lib bud_e2e
cargo test --lib permissionless
gh pr checks 6
```

## 4. Beklenen çıktı
- PR #6 CI durumu (yeşil/kırmızı + net sebep)
- Güncellenmiş `docs/ORG_ROADMAP_AUDIT.md` §4a
- Varsa yeni bulgu satırı(ları)
- Tur 15 (Faz 3) ayrı bütçe/gate bekliyor notu — bu turda BAŞLATILMADI

## 5. Son hatırlatma
Başarı ölçütü yeni özellik değil, **söz ile kod arasındaki farkın sıfıra
yaklaşmasıdır.** CI gevşetilmeyecek, roadmap maddesi kaybolmayacak, B.U.D.
Tur 14 sınırı korunacak.
