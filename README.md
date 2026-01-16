# evse-ota
ORGE EVSE OTA mini-DM: manifests + rollout channels

**Created:** 2026-01-13 16:17  
**Last Modified:** 2026-01-16 08:03  
**Description:** GitHub üzerinden servis edilen OTA manifest’leri (stable/staging/canary) ve bunların işaret ettiği source-tar artefact’leri (GitHub Release asset). Secret-safe.

---

## Repo layout

- `stable/manifest.json`
- `staging/manifest.json`
- `canary/manifest.json`

Manifest şeması kaynak repo’daki station-side parser ile uyumlu tutulur: `CaiserSz/evse` → `src/orge_evse/ota/manifest.py`.

---

## Manifest schema (Phase-1 / Phase-2)

Top-level alanlar:

- `channel`: `stable|staging|canary`
- `build_id`: `YYYYMMDD-HHMM-<gitsha>` (ör: `20260115-1614-9121078`)
- `created_utc`: ISO8601 UTC (ör: `2026-01-15T13:55:03Z`)
- `min_build_id`: (opsiyonel) downgrade engeli için alt limit
- `notes`: kısa açıklama
- `signature`: (opsiyonel, Phase-2) Ed25519 base64 signature (canonical JSON; `signature` alanı imzaya dahil değil)

Artifact:

- `artifact.kind`: Phase-1: `source-tar`
- `artifact.url`: GitHub Release asset URL
- `artifact.sha256`: hex sha256

---

## Station config (özet)

Station tarafı env (örnek):

- `ORGE_EVSE_OTA_MANIFEST_URL=https://raw.githubusercontent.com/CaiserSz/evse-ota/main/stable/manifest.json`
- `ORGE_EVSE_OTA_AUTO_APPLY=1` (timer auto-mode)

Phase-2 (opsiyonel, önerilen):

- `ORGE_EVSE_OTA_REQUIRE_SIGNATURE=1`
- `ORGE_EVSE_OTA_PUBKEY_PATH=/etc/orge/evse/ota_pubkey.pem`
- `ORGE_EVSE_OTA_STRICT_MANIFEST_KEYS=1` (önerilen)

Restart safety guard (systemd unit `--restart` kullanıyorsa):

- `ORGE_EVSE_OTA_STRICT_RESTART=1`
- `ORGE_EVSE_OTA_RESTART_ACK=1` (bilinçli operatör onayı)

Not: Station-side akış `CaiserSz/evse` içinde (`orge-evse ota check/apply`) ve deploy layout `/opt/orge/evse/releases/<build_id>` + `/opt/orge/evse/current` symlink (ADR_0016).

---

## Yeni build publish etme (ops akışı)

### 1) Source tar üret (secret-safe)

`CaiserSz/evse` repo’sunda:

```bash
BUILD_ID="YYYYMMDD-HHMM-<gitsha>"
git archive --format=tar.gz --prefix="evse-${BUILD_ID}/" <gitref> -o "/tmp/evse-src-${BUILD_ID}.tar.gz"
sha256sum "/tmp/evse-src-${BUILD_ID}.tar.gz"
```

### 2) GitHub Release oluştur + asset upload

Bu repo’da (`CaiserSz/evse-ota`):

```bash
gh release create "${BUILD_ID}" "/tmp/evse-src-${BUILD_ID}.tar.gz" --title "EVSE OTA ${BUILD_ID}"
```

### 3) Channel manifest’i güncelle

İlgili kanal dosyasını güncelle:

- `stable/manifest.json` (prod)
- `staging/manifest.json` (smoke)
- `canary/manifest.json` (erken geri bildirim)

`artifact.url` şu formatta olmalı:

`https://github.com/CaiserSz/evse-ota/releases/download/<build_id>/evse-src-<build_id>.tar.gz`

### 3.1) Doğrulama checklist (önerilen)

- `build_id` ↔ GitHub Release tag eşleşiyor mu?
- `artifact.url` doğru release asset’e gidiyor mu?
- `sha256` doğrulandı mı?
  - `curl -L -o /tmp/evse-src-<build_id>.tar.gz <artifact.url>`
  - `sha256sum /tmp/evse-src-<build_id>.tar.gz` → manifest ile eşleştir
- Phase‑2 imza politikası:
  - `ORGE_EVSE_OTA_REQUIRE_SIGNATURE=1` ise `signature` boş olamaz.
  - İmza doğrulaması istasyon tarafında `orge-evse ota check` ile test edilir.

### 4) (Opsiyonel) Manifest sign (Phase-2)

Offline private key ile imzala (private key **repoya girmez**). İmzalama helper’ı `CaiserSz/evse` repo’sunda:

```bash
python3 scripts/ota_sign_manifest.py \
  --in stable/manifest.json \
  --out stable/manifest.json \
  --privkey-pem /path/to/offline_ed25519_privkey.pem \
  --strict-keys
```

### 5) PR + merge

PR aç (küçük/atomik) ve merge et. Bu repo’daki `stable/manifest.json` değişikliği, `stable` kanalına bağlı cihazlarda OTA check/apply akışını etkiler.

