# pelican-egg-rust-toros

Pelican Panel için Rust dedicated server egg'i — Facepunch resmi `pelican-eggs/games-steamcmd/rust/vanilla` üzerine **Toros Clan**'ın özel düzenlemeleri.

Plugin'lerden bağımsız (Rust C# pluginleri / Oxide modları ayrı repo'da). Bu repo sadece **Pelican egg JSON'unu** ve egg'in çalışması için gereken konvansiyonları içerir.

## Upstream'den farklar

Toplamda 3 değişiklik. Hepsi Pelican Panel'in standart Filament admin > Eggs > Import sekmesinden uygulanabilir.

### 1. Shell-quoting bug bypass (`config.files`)

Facepunch'ın resmi egg'i hostname ve description gibi kullanıcı-input alanlarını CLI argümanı olarak geçiriyor (`+server.hostname "{{SERVER_HOSTNAME}}"`). Pelican Wings'in `entrypoint.sh`'i `eval echo -e ${STARTUP}` pattern'i kullandığı için kullanıcı hostname'inde boşluk, `|`, `(`, `'`, `\n` varsa shell field-split olup boot kırılıyor.

Bu egg yerine **Pelican `config.files` parser**'ı ile `server/rust/cfg/server.cfg` dosyasına yazıyor:

- Hostname, description, URL, header image, logo image, level, worldsize, seed, tags, **levelurl** (custom map URL) artık CLI'dan değil cfg'den okunuyor.
- Cfg dosyası shell parser'ından geçmiyor → kullanıcı her özel karakteri yazsa sorunsuz boot.
- Rust dedicated server `server.cfg`'yi CLI'a tercih eder (Facepunch wiki).
- Upstream'in `MAP_URL` koşullu CLI argümanı (`$( [ -z ${MAP_URL} ] && ... )`) kaldırıldı; aynı işlevsellik `server.levelurl` cfg satırından sağlanıyor (boş URL = procedural map).

İlgili upstream issue'lar:
- [pelican-dev/wings#74](https://github.com/pelican-dev/wings/issues/74) — `|` karakteri startup'ı kırıyor
- [pterodactyl/panel#721](https://github.com/pterodactyl/panel/issues/721) — quoting genel
- [pelican-eggs/games-steamcmd#146](https://github.com/pelican-eggs/games-steamcmd/issues/146) — Rust-spesifik

### 2. `install_script` server.cfg seed ekledi

Pelican `file` parser bulamadığı satırı **skip** ediyor (append yapmıyor — kaynak: [pelican-dev/wings parser/parser.go#L575-609](https://github.com/pelican-dev/wings/blob/main/parser/parser.go#L575-L609)). Yeni server install'da cfg dosyası yoksa parser hiçbir satır eşleştiremez → hostname/description Rust'a hiç ulaşmaz.

Çözüm: install_script sonuna seed bloğu eklendi. Server ilk install'da `server/rust/cfg/server.cfg` placeholder satırlarla yaratılıyor, sonraki her boot'ta parser bu satırları override ediyor.

### 3. `SERVER_TAGS` egg variable

Steam server browser tag'leri için ayrı variable. Default `weekly,vanilla`. Cfg'de `server.tags "..."` satırına yazılıyor.

## Port konvansiyonu (Toros standart)

Tüm Toros Rust server'ları aynı pattern kullanır (Facepunch default ile uyumlu):

| Convar | Port | Notes |
|---|---|---|
| `+server.port` | **X15** | Oyuncu Steam connect, primary allocation |
| `+rcon.port` | **X16** | RCON admin |
| `+server.queryport` | **X17** | Steam A2S query |
| `+app.port` | X82 (örn 28082) | Rust+ companion |

Yeni server yaratırken admin Pelican allocation pool'undan **X15 ile biten** port'u primary olarak seçer; `RCON_PORT` ve `QUERY_PORT` egg variable'ları manuel olarak X16/X17'ye set edilmeli (egg default'u gelecekte port detection ile dinamikleştirilebilir).

## Kurulum

1. Pelican Panel admin paneline gir
2. Eggs > Import Egg sekmesine git
3. `egg-rust.json`'u yükle
4. (Yeni nest yaratmadıysan) Existing nest seç, ya da yeni "Toros Games" nest aç
5. Import et

Mevcut Rust server'larını yeni egg'e taşımak istersen Pelican admin > Server > "Change Egg" seçeneği kullanılabilir; environment variable'lar otomatik korunur (yeni egg variable'lar boş gelir, manuel doldurulur).

## Bakım

Facepunch upstream'i değişirse:

```sh
# Upstream'den fresh egg al
curl -L -o /tmp/egg-rust-upstream.yaml \
  https://raw.githubusercontent.com/pelican-eggs/games-steamcmd/refs/heads/main/rust/vanilla/egg-rust.yaml

# Manuel diff + bizim 3 değişikliği patch et:
#   - config.files (server.cfg parser)
#   - script_install (cfg seed bloğu)
#   - SERVER_TAGS variable
# Sonra panel'den re-import.
```

## Lisans

Pelican egg formatı genel olarak public domain / no-license. Bu repo'daki düzenlemeler de aynı.
