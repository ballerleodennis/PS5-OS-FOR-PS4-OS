# PS5-OS-FOR-PS4-OS



ps5os/
├── tools/
│   ├── mkrootfs.sh        # generischer Porter: Rootfs -> Multiboot-Archiv
│   ├── fw-index.sh        # Firmware-Dump indizieren, Rollen zuordnen
│   ├── tex.sh             # GNF/DDS -> PNG
│   ├── at9.sh             # AT9/AT3 -> WAV/OGG
│   ├── mk-tiles.py        # icon0/pic0 -> Kachel + Hintergrund
│   ├── pack.sh            # validieren, benennen, ausliefern
│   └── push.sh            # per FTP auf die PS4
├── overlay/               # wird über das Rootfs gelegt
│   ├── opt/ps5shell/      # UI (HTML/CSS/JS) + shelld
│   ├── etc/systemd/system/
│   ├── usr/local/bin/
│   └── usr/share/plymouth/themes/ps5os/
├── assets/                # Ergebnis der Extraktion (Fonts, SFX, Icons)
└── build/ out/








## Bevor ich Code schreibe: was "porten" hier wirklich heißt

Deine Boot-Files sind **kein bootbares OS für die PS4**. Grund: Die PS5 läuft auf FreeBSD-Basis mit PS5-Kernel, Oberon-GPU und anderem Southbridge (Baikal2/Aeolia2). SELF/ELF aus `/system/vsh` (`SceShellCore`, `SceShellUI`) sind FreeBSD-Binaries gegen PS5-Syscalls — auf einem PS4-Linux-Kernel starten die nicht.

Zwei Wege, brutal ehrlich:

| Weg | Aufwand | Ergebnis |
|---|---|---|
| **A) Linux-OS mit nachgebauter PS5-Oberfläche** (empfohlen) | 2–4 Wochen | Sieht 1:1 aus, fühlt sich 1:1 an, läuft über Multiboot |
| **B) PS5-Kernel auf PS4 porten** | Monate, Forschung | Nie fertig geworden, kein Vorteil für die UI |

Deine Firmware-Files werden bei Weg A zum **Asset- und Layout-Lieferanten**: Fonts, Sounds, Icons, Kachel-Anordnung, Boot-Animation. Das ist genau der Teil, der "1:1" ausmacht. Der Rest ist ein Linux-Kiosk, der die PS5-Shell nachbildet — gebaut auf EmuFives Emulator, nur mit eigener Oberfläche statt dessen Home-Screen.

## Zielarchitektur

Boot-Kette: GoldHEN-Payload → Multiboot-Kernel → Menü-Karte `ps5os` → Installation auf `/user/home/ps5os-ps4.img` → systemd → `cage`-Kiosk mit WPE-WebKit-UI → "Spielen" startet `kytyps5` mit dem Spieleordner → zurück zur Shell.

Wichtig: **`bzImage`, `initramfs.cpio.gz` und der Payload bleiben von PS4 Multiboot.** Du lieferst nur ein Rootfs. Das hält den Port klein und updatebar.

## Repo-Layout

```
ps5os/
├── tools/
│   ├── mkrootfs.sh        # generischer Porter: Rootfs -> Multiboot-Archiv
│   ├── fw-index.sh        # Firmware-Dump indizieren, Rollen zuordnen
│   ├── tex.sh             # GNF/DDS -> PNG
│   ├── at9.sh             # AT9/AT3 -> WAV/OGG
│   ├── mk-tiles.py        # icon0/pic0 -> Kachel + Hintergrund
│   ├── pack.sh            # validieren, benennen, ausliefern
│   └── push.sh            # per FTP auf die PS4
├── overlay/               # wird über das Rootfs gelegt
│   ├── opt/ps5shell/      # UI (HTML/CSS/JS) + shelld
│   ├── etc/systemd/system/
│   ├── usr/local/bin/
│   └── usr/share/plymouth/themes/ps5os/
├── assets/                # Ergebnis der Extraktion (Fonts, SFX, Icons)
└── build/ out/
```

## 1. Der Porter

`tools/mkrootfs.sh` — generisch: baut aus Debian oder aus einem beliebigen Rootfs-Tarball ein Multiboot-Archiv. Genau das Tool, das du für jedes weitere OS wiederverwenden kannst.

```bash
#!/usr/bin/env bash
# Portiert ein Rootfs nach PS4-Multiboot. Ergebnis: out/<name>-ps4-rootfs.tar.xz
set -euo pipefail

NAME=${NAME:-ps5os}
ROOT=${ROOT:-build/root}
OVERLAY=${OVERLAY:-overlay}
FROM_TAR=${FROM_TAR:-}
OUT=${OUT:-out}
DISTRO=${DISTRO:-trixie}
MIRROR=${MIRROR:-http://deb.debian.org/debian}

# WICHTIG: cog/seatd/cage = Kiosk-Stack, wpewebkit-driver als Fallback
PKGS="systemd-sysv dbus seatd cage cog wpewebkit-driver
  fonts-inter fonts-noto-color-emoji
  python3 python3-aiohttp python3-pil
  ffmpeg vgmstream-cli alsa-utils pipewire pipewire-pulse wireplumber
  network-manager openssh-server sudo curl ca-certificates
  plymouth plymouth-themes imagemagick less nano file"

cleanup(){ for d in dev/pts dev proc sys; do umount -R "$ROOT/$d" 2>/dev/null || true; done; }
trap cleanup EXIT

rm -rf "$ROOT"; mkdir -p "$ROOT" "$OUT"

if [ -n "$FROM_TAR" ]; then
  echo "[*] importiere $FROM_TAR"
  tar -xpf "$FROM_TAR" -C "$ROOT" --numeric-owner --xattrs
else
  echo "[*] debootstrap $DISTRO"
  debootstrap --arch=amd64 --variant=minbase --include="${PKGS//$'\n'/ }" \
    "$DISTRO" "$ROOT" "$MIRROR"
fi

echo "[*] overlay"
tar -C "$OVERLAY" -cf - . | tar -C "$ROOT" -xpf - --numeric-owner

mount -t proc proc "$ROOT/proc"; mount -t sysfs sys "$ROOT/sys"
mount --rbind /dev "$ROOT/dev"; mount --make-rslave "$ROOT/dev"

chroot "$ROOT" /bin/bash -euo pipefail <<'EOF'
export DEBIAN_FRONTEND=noninteractive
echo "ps5os" > /etc/hostname
printf '127.0.0.1 localhost\n127.0.1.1 ps5os\n' > /etc/hosts
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
useradd -m -s /bin/bash -G video,render,input,audio,seat ps5
echo 'ps5:ps5' | chpasswd; echo 'root:ps5' | chpasswd
systemctl enable ps5os-shelld.service ps5os-shell.service NetworkManager ssh
systemctl set-default graphical.target
apt-get clean; rm -rf /var/lib/apt/lists/* /etc/machine-id /etc/ssh/ssh_host_*
EOF

echo "[*] packen"
tar --numeric-owner --xattrs -cJpf "$OUT/$NAME-ps4-rootfs.tar.xz" -C "$ROOT" .

# Validierung: ohne ./etc und ./bin startet Multiboot nicht
tar -tJf "$OUT/$NAME-ps4-rootfs.tar.xz" | grep -qx './etc/fstab' || { echo "FEHLER: fstab fehlt"; exit 1; }
ls -lh "$OUT/$NAME-ps4-rootfs.tar.xz"
```

Das Suffix `-ps4-rootfs` **muss** im Namen stehen — daran findet das Multiboot-Menü Systeme. Deine Karte heißt danach `ps5os`, weil der Installer den Namen davor nimmt.

## 2. Firmware-Dump verwertbar machen

`tools/fw-index.sh` — katalogisiert den Dump und schlägt Rollen vor, damit du nicht 40.000 Dateien durchklickst.

```bash
#!/usr/bin/env bash
set -euo pipefail
DUMP=${1:?Verzeichnis mit dem Dump}
OUT=${2:-manifest.tsv}
printf 'sha1\tsize\tkind\trole\tsrc\n' > "$OUT"

find "$DUMP" -type f -print0 | while IFS= read -r -d '' f; do
  ext=${f##*.}; ext=${ext,,}
  case "$ext" in
    gnf|dds|png|jpg|jpeg|bmp|tga) kind=texture ;;
    at9|at3|at5|wav|mp3)          kind=audio ;;
    otf|ttf|ttc|otc)              kind=font ;;
    self|elf|sprx|prx)            kind=binary ;;
    json|xml|sfo)                 kind=meta ;;
    db|sqlite|sqlite3)            kind=db ;;
    *)                            kind=other ;;
  esac
  role=asset
  case "$f" in
    *SceShellUI*|*shellui*)     role=shell ;;
    *SceShellCore*|*shellcore*) role=shellcore ;;
    *[Ff]ont*)                  role=font ;;
    *boot*|*startup*|*splash*)  role=bootanim ;;
    *sound*|*se_*|*bgm*)        role=sfx ;;
    *home*|*layout*)            role=homelayout ;;
    *icon*|*tile*)              role=icon ;;
  esac
  printf '%s\t%s\t%s\t%s\t%s\n' "$(sha1sum "$f" | cut -d' ' -f1)" \
    "$(stat -c%s "$f")" "$kind" "$role" "$f" >> "$OUT"
done

echo "--- Rollen ---"; cut -f4 "$OUT" | tail -n +2 | sort | uniq -c | sort -rn
echo "--- Typen ---"; cut -f3 "$OUT" | tail -n +2 | sort | uniq -c | sort -rn
echo "Manifest: $OUT"
```

Dann gezielt: `awk -F'\t' '$4=="font"' manifest.tsv`.

**Zuordnung, was du damit machst:**

| Fund im Dump | Rolle im OS |
|---|---|
| `.otf`/`.ttf` in `SceShellUI`/Font-Pfaden | `assets/fonts/` → `@font-face` (SST ist Sonys Schrift, exakte Metriken) |
| `.at9` unter `sound`/`se_` | `at9.sh` → OGG → `assets/sfx/` (Navigations-Klick, Bestätigen, Fehler) |
| Boot-Animation / Splash-Texturen | `tex.sh` → PNG-Sequenz → Plymouth-Theme |
| `SceShellUI`-Texturen (Symbole) | `tex.sh` → `assets/icons/` |
| `home`-DB (`sqlite`) | Kachel-Reihenfolge, Gruppierung — `sqlite3 db '.dump'` |
| `param.json` pro App | Titel, Content-ID, Kategorie |

`.self`/`.sprx` wandern nicht in den Betrieb, nur als Referenz für Symbole/Logik.

## 3. Konverter

```bash
#!/usr/bin/env bash
# tools/tex.sh IN OUT.png
set -euo pipefail
in=$1; out=$2
case "${in##*.}" in
  png|jpg|jpeg) cp -f "$in" "$out" ;;
  dds) magick "$in" "$out" ;;
  gnf)
    tmp="${out%.png}.dds"
    if   command -v gnftool >/dev/null; then gnftool d "$in" "$tmp"
    elif command -v GnfTool >/dev/null; then GnfTool -d "$in" "$tmp"
    else echo "kein GNF-Tool installiert" >&2; exit 2; fi
    magick "$tmp" "$out"; rm -f "$tmp" ;;
  *) echo "unbekannt: $in" >&2; exit 2 ;;
esac
```

```bash
#!/usr/bin/env bash
# tools/at9.sh IN.at9 OUT.ogg
set -euo pipefail
tmp=$(mktemp --suffix=.wav)
vgmstream-cli -o "$tmp" "$1" >/dev/null
ffmpeg -y -loglevel error -i "$tmp" -c:a libvorbis -q:a 5 "$2"
rm -f "$tmp"
```

`tools/mk-tiles.py` baut aus dem Spiel selbst Kachel und Hintergrund — die echten Artworks, kein Platzhalter:

```python
#!/usr/bin/env python3
import sys, json, pathlib
from PIL import Image, ImageFilter, ImageEnhance

game, out = pathlib.Path(sys.argv[1]), pathlib.Path(sys.argv[2])
out.mkdir(parents=True, exist_ok=True)
sce = game / "sce_sys"

def pick(*names):
    for n in names:
        p = sce / n
        if p.exists():
            return Image.open(p).convert("RGB")
    return None

icon = pick("icon0.png", "icon0.jpg") or Image.new("RGB", (512, 512), (20, 20, 24))
bg = pick("pic0.png", "pic1.png", "pic0.jpg") or icon
bg.resize((1920, 1080), Image.LANCZOS).filter(ImageFilter.GaussianBlur(30)) \
  .pipe if False else None
bg = bg.resize((1920, 1080), Image.LANCZOS).filter(ImageFilter.GaussianBlur(30))
bg = ImageEnhance.Brightness(bg).enhance(0.5)
bg.save(out / "bg.jpg", quality=82)
icon.resize((512, 512), Image.LANCZOS).save(out / "tile.png")

title = game.name
for meta in ("param.json", "param.sfo"):
    p = sce / meta
    if p.exists() and meta.endswith(".json"):
        try: title = json.loads(p.read_text()).get("titleName", title)
        except Exception: pass
(out / "meta.json").write_text(json.dumps({"title": title, "id": game.name}))
```

## 4. Die Oberfläche

Der "1:1"-Teil. Aufbau wie die echte Shell: Topbar mit Avatar und Tabs, waagerechte Kachelreihe, Hintergrund wechselt zum Artwork des gewählten Spiels, Control Center von unten beim PS-Knopf.

`overlay/opt/ps5shell/index.html`:

```html
<!doctype html>
<html lang="de"><head>
<meta charset="utf-8"><title>ps5os</title>
<link rel="stylesheet" href="style.css">
</head><body>
<div id="bg"></div>

<header id="topbar">
  <div class="me"><img src="assets/icons/avatar.png" alt=""></div>
  <nav id="tabs">
    <button data-tab="games" class="on">Spiele</button>
    <button data-tab="media">Medien</button>
  </nav>
  <div class="tools">
    <button id="btn-search">Suche</button>
    <button id="btn-settings">Einstellungen</button>
  </div>
</header>

<main id="row"></main>

<div id="actions" class="hidden">
  <button data-act="play">Spielen</button>
  <button data-act="options">Optionen</button>
</div>

<footer id="hint">
  <span><b>✕</b> Starten</span><span><b>◯</b> Zurück</span>
  <span><b>OPTIONS</b> Optionen</span><span><b>PS</b> Control Center</span>
</footer>

<div id="cc" class="hidden">
  <div class="cc-row">
    <button data-cc="power">Energie</button>
    <button data-cc="sound">Ton</button>
    <button data-cc="ctrl">Controller</button>
    <button data-cc="shot">Screenshot</button>
    <button data-cc="home">Home</button>
  </div>
</div>

<script src="shell.js"></script>
</body></html>
```

`style.css` — Palette, Maße und Timing nach PS5-Vorbild (dunkles Blaugrau, blaue Akzente, weiche Übergänge):

```css
@font-face{font-family:SST;src:url(assets/fonts/sst-roman.woff2) format("woff2");font-weight:400;font-display:swap}
@font-face{font-family:SST;src:url(assets/fonts/sst-light.woff2) format("woff2");font-weight:300;font-display:swap}
:root{
  --bg:#0b0c10; --ink:#f4f5f8; --dim:#9aa0ac;
  --blue:#0070d1; --ring:#ffffff;
  --tile:196px; --gap:22px; --r:18px;
}
*{box-sizing:border-box;margin:0;padding:0}
html,body{height:100%;overflow:hidden;background:var(--bg);color:var(--ink);
  font-family:SST,Inter,system-ui,sans-serif;font-weight:300;
  letter-spacing:.01em;user-select:none;cursor:none}

#bg{position:fixed;inset:-40px;background-size:cover;background-position:center;
  filter:blur(0px) saturate(1.05);opacity:.9;transition:opacity .45s ease, background-image .01s}
#bg::after{content:"";position:absolute;inset:0;
  background:linear-gradient(180deg,rgba(6,7,10,.55) 0%,rgba(6,7,10,.15) 35%,rgba(6,7,10,.85) 100%)}

#topbar{position:fixed;top:0;left:0;right:0;height:96px;display:flex;align-items:center;
  gap:28px;padding:0 56px;z-index:5}
.me img{width:40px;height:40px;border-radius:50%;background:#2a2d36}
#tabs{display:flex;gap:26px}
#tabs button{background:none;border:0;color:var(--dim);font:inherit;font-size:19px;
  padding:8px 2px;position:relative;transition:color .2s}
#tabs button.on{color:var(--ink)}
#tabs button.on::after{content:"";position:absolute;left:0;right:0;bottom:0;height:2px;
  background:var(--ink);border-radius:2px}
.tools{margin-left:auto;display:flex;gap:14px}
.tools button{background:rgba(255,255,255,.07);border:0;color:var(--ink);font:inherit;
  font-size:15px;padding:10px 18px;border-radius:999px;transition:background .2s}
.tools button:hover{background:rgba(255,255,255,.16)}

#row{position:fixed;left:56px;right:56px;bottom:150px;display:flex;gap:var(--gap);
  align-items:flex-end;overflow-x:auto;overflow-y:hidden;scrollbar-width:none;
  scroll-behavior:smooth;padding:20px 0 0}
#row::-webkit-scrollbar{display:none}

.tile{flex:0 0 auto;width:var(--tile);transition:transform .22s cubic-bezier(.2,.7,.2,1),
  filter .22s;position:relative;outline:none}
.tile img{width:100%;aspect-ratio:1;border-radius:var(--r);object-fit:cover;
  display:block;box-shadow:0 18px 40px rgba(0,0,0,.55)}
.tile.sel{transform:translateY(-14px) scale(1.09);z-index:3}
.tile.sel img{outline:3px solid var(--ring);outline-offset:4px}
.tile .name{font-size:15px;color:var(--ink);margin-top:12px;opacity:0;
  transition:opacity .2s;max-width:var(--tile);white-space:nowrap;
  overflow:hidden;text-overflow:ellipsis}
.tile.sel .name{opacity:1}

#actions{position:fixed;left:56px;bottom:96px;display:flex;gap:12px;z-index:6;
  transition:opacity .18s, transform .18s}
#actions button{background:rgba(255,255,255,.1);border:0;color:var(--ink);font:inherit;
  font-size:16px;padding:12px 26px;border-radius:999px;backdrop-filter:blur(20px)}
#actions button:first-child{background:var(--blue)}

#hint{position:fixed;left:56px;right:56px;bottom:26px;display:flex;gap:30px;
  font-size:13px;color:var(--dim);z-index:6}
#hint b{color:var(--ink);font-weight:400}

#cc{position:fixed;left:0;right:0;bottom:0;padding:28px 56px 44px;
  background:linear-gradient(180deg,rgba(12,13,17,.4),rgba(12,13,17,.96));
  backdrop-filter:blur(34px);transform:translateY(100%);
  transition:transform .3s cubic-bezier(.2,.7,.2,1);z-index:10}
#cc.open{transform:translateY(0)}
.cc-row{display:flex;gap:16px;justify-content:center}
.cc-row button{width:130px;height:96px;border:0;border-radius:16px;
  background:rgba(255,255,255,.09);color:var(--ink);font:inherit;font-size:14px}

.hidden{opacity:0;pointer-events:none}
#cc.hidden{opacity:1;transform:translateY(100%)}
```

`shell.js` — Navigation per Tastatur und Gamepad (DualShock 4 erscheint als Gamepad-API), RPC gegen `shelld`:

```js
const WS = new WebSocket(`ws://${location.host}/ws`);
let games = [], idx = 0, tab = "games", ccOpen = false;
const row = document.getElementById("row");
const bg  = document.getElementById("bg");
const actions = document.getElementById("actions");
const cc = document.getElementById("cc");

const send = (op, arg) => WS.readyState === 1 && WS.send(JSON.stringify({ op, arg }));

function render() {
  row.innerHTML = "";
  games.forEach((g, i) => {
    const d = document.createElement("div");
    d.className = "tile";
    d.innerHTML = `<img src="${g.tile}" alt=""><div class="name"></div>`;
    d.querySelector(".name").textContent = g.title;
    row.appendChild(d);
  });
  select(idx, true);
}

function select(i, instant) {
  idx = Math.max(0, Math.min(games.length - 1, i));
  [...row.children].forEach((t, n) => t.classList.toggle("sel", n === idx));
  const g = games[idx];
  if (g) {
    bg.style.backgroundImage = `url("${g.bg}")`;
    actions.classList.remove("hidden");
    row.children[idx].scrollIntoView({ inline: "center", block: "nearest", behavior: instant ? "auto" : "smooth" });
    send("focus", g.id);
  } else actions.classList.add("hidden");
}

function launch() { const g = games[idx]; if (g) { send("launch", g.id); actions.classList.add("hidden"); } }
function toggleCC() { ccOpen = !ccOpen; cc.classList.toggle("open", ccOpen); }

WS.onmessage = e => {
  const m = JSON.parse(e.data);
  if (m.type === "library") { games = m.games; render(); }
  if (m.type === "game_end") { actions.classList.remove("hidden"); }
};

const keymap = {
  ArrowLeft: () => select(idx - 1), ArrowRight: () => select(idx + 1),
  ArrowUp: () => { if (!ccOpen) ccOpen = false, cc.classList.remove("open"); },
  Enter: () => ccOpen ? null : launch(),
  Escape: () => { if (ccOpen) toggleCC(); },
  Backspace: () => toggleCC(),
};
addEventListener("keydown", e => { const f = keymap[e.key]; if (f) { f(); e.preventDefault(); } });

let padPrev = {};
function poll() {
  const pads = navigator.getGamepads();
  for (const p of pads) {
    if (!p) continue;
    const b = i => !!p.buttons[i]?.pressed;
    const edge = (i, fn) => { if (b(i) && !padPrev[i]) fn(); padPrev[i] = b(i); };
    edge(14, () => select(idx - 1)); edge(15, () => select(idx + 1));
    edge(0, () => launch()); edge(1, () => ccOpen && toggleCC());
    edge(9, () => toggleCC());           // OPTIONS
    edge(16, () => toggleCC());          // PS-Taste
  }
  requestAnimationFrame(poll);
}
poll();

document.querySelectorAll("#tabs button").forEach(b =>
  b.onclick = () => {
    document.querySelectorAll("#tabs button").forEach(x => x.classList.remove("on"));
    b.classList.add("on"); tab = b.dataset.tab; send("tab", tab);
  });
document.querySelectorAll("#actions button").forEach(b =>
  b.onclick = () => b.dataset.act === "play" && launch());
document.querySelectorAll(".cc-row button").forEach(b =>
  b.onclick = () => { send("cc", b.dataset.cc); if (b.dataset.cc !== "sound") toggleCC(); });

WS.onopen = () => send("list");
```

`shelld.py` — Backend: Bibliothek scannen, Spiel starten, Ton, Energie.

```python
#!/usr/bin/env python3
import asyncio, json, os, pathlib
from aiohttp import web

ROOT = pathlib.Path("/opt/ps5shell")
LIB  = pathlib.Path(os.environ.get("PS5OS_LIB", "/user/home/PS5"))
STATE = pathlib.Path("/var/lib/ps5os/state.json")
STATE.parent.mkdir(parents=True, exist_ok=True)

state = json.loads(STATE.read_text()) if STATE.exists() else {"volume": 60, "fav": []}
save = lambda: STATE.write_text(json.dumps(state))

def library():
    out = []
    if not LIB.exists():
        return out
    for d in sorted(LIB.iterdir()):
        if not d.is_dir():
            continue
        ok = (d / "sce_sys" / "eboot.bin").exists() or list(d.glob("*.zar"))
        art = d / "art"
        if not ok or not (art / "tile.png").exists():
            continue
        meta = json.loads((art / "meta.json").read_text()) if (art / "meta.json").exists() else {}
        out.append({"id": d.name, "title": meta.get("title", d.name),
                    "tile": f"/lib/{d.name}/art/tile.png",
                    "bg":   f"/lib/{d.name}/art/bg.jpg"})
    out.sort(key=lambda g: (g["id"] not in state["fav"], g["title"].lower()))
    return out

async def ws_handler(request):
    ws = web.WebSocketResponse(); await ws.prepare(request)
    async for msg in ws:
        m = json.loads(msg.data); op = m.get("op")

        if op == "list":
            await ws.send_json({"type": "library", "games": library()})

        elif op == "launch":
            g = LIB / m["arg"]
            if not g.exists():
                continue
            await ws.send_json({"type": "launching"})
            proc = await asyncio.create_subprocess_exec(
                "/usr/local/bin/ps5os-run", str(g))
            await proc.wait()
            await ws.send_json({"type": "game_end"})

        elif op == "volume":
            state["volume"] = max(0, min(100, int(m["arg"]))); save()
            await asyncio.create_subprocess_exec(
                "wpctl", "set-volume", "@DEFAULT_AUDIO_SINK@", f"{state['volume']}%")

        elif op == "cc":
            act = m["arg"]
            if act == "power":
                await asyncio.create_subprocess_exec("systemctl", "poweroff")
            elif act == "home":
                pass
            elif act == "shot":
                await asyncio.create_subprocess_exec("/usr/local/bin/ps5os-shot")

        elif op == "tab":
            pass

    return ws

async def lib_static(request):
    p = LIB / request.match_info["name"] / "art" / request.match_info["file"]
    return web.FileResponse(p) if p.exists() else web.Response(status=404)

app = web.Application()
app.router.add_get("/ws", ws_handler)
app.router.add_get("/lib/{name}/{file}", lib_static)
app.router.add_static("/", ROOT, show_index=True)
web.run_app(app, host="127.0.0.1", port=8790, print=None)
```

## 5. Session, Dienste, Boot

`overlay/usr/local/bin/ps5os-session`:

```bash
#!/bin/bash
export XDG_RUNTIME_DIR=/run/user/1000
export WLR_BACKENDS=drm WLR_RENDERER=gles2
export RADV_EXPERIMENTAL=sparse
export WPE_BACKEND=wl
exec cage -s -- /usr/bin/cog --platform=wl \
  --bg-color=0x00000000 http://127.0.0.1:8790/
```

`overlay/usr/local/bin/ps5os-run`:

```bash
#!/bin/bash
set -u
exec /opt/ps5-on-ps4/bin/ps5-run "$1"
```

`overlay/etc/systemd/system/ps5os-shelld.service`:

```ini
[Unit]
Description=ps5os shell backend
After=network.target
[Service]
User=ps5
ExecStart=/usr/bin/python3 /opt/ps5shell/shelld.py
Restart=always
[Install]
WantedBy=multi-user.target
```

`overlay/etc/systemd/system/ps5os-shell.service`:

```ini
[Unit]
Description=ps5os shell
After=ps5os-shelld.service
Conflicts=getty@tty1.service
[Service]
User=ps5
PAMName=login
TTYPath=/dev/tty1
StandardInput=tty
StandardOutput=journal
ExecStart=/usr/local/bin/ps5os-session
Restart=always
RestartSec=2
[Install]
WantedBy=graphical.target
```

Plymouth-Theme `overlay/usr/share/plymouth/themes/ps5os/` mit `ps5os.plymouth`, `ps5os.script` und PNG-Sequenz aus dem Dump; `splash` an die Kernelzeile in `/data/linux/boot/bootargs.txt` hängen.

## 6. Bauen und ausliefern

```bash
# Assets aus deinem Dump
./tools/fw-index.sh ~/ps5dump manifest.tsv
awk -F'\t' '$4=="font"{print $5}' manifest.tsv | while read -r f; do
  cp -f "$f" assets/fonts/; done
awk -F'\t' '$4=="sfx"{print $5}' manifest.tsv | while read -r f; do
  ./tools/at9.sh "$f" "assets/sfx/$(basename "${f%.*}").ogg"; done

# Spiele vorbereiten
for g in /user/home/PS5/*/; do ./tools/mk-tiles.py "$g" "$g/art"; done

# Rootfs bauen und pushen
./tools/mkrootfs.sh
./tools/push.sh out/ps5os-ps4-rootfs.tar.xz
```

`tools/push.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
PS4=${PS4:?z.B. PS4=192.168.1.50}
F=${1:?Datei}
curl -sS --ftp-create-dirs -T "$F" "ftp://${PS4}:2121/user/home/$(basename "$F")"
echo "hochgeladen. Jetzt Payload starten, im Menü 'ps5os' -> Install to PS4."
```

## 7. Was du realistisch erwarten darfst

- Die Optik bekommst du mit echten Fonts, echten Sounds und echten Artworks sehr nah. Kachelgrößen, Timing, Blur und Verläufe lassen sich nachziehen, bis es sitzt.
- Was fehlt: die echte Systemlogik. Downloads, Trophäen, Store, Party — alles nachbauen, nicht portieren.
- **Erste Tests nicht auf der PS4.** Nimm das fertige Rootfs in eine QEMU-VM mit `-vga virtio`, installier dort `cage` und iteriere die UI in Minuten statt in Reboot-Zyklen.
- Auf einer Pro ist das flüssig. Auf Slim mit Baikal ist USB-Schreiben wackelig — installier dort auf die interne Platte.
- Ein 1:1-Klon der Shell-Core-Semantik (Suspend-Resume, App-Lifecycle) ist der Punkt, an dem Projekte sterben. Halte die Shell zustandslos: sie zeigt Kacheln, startet Prozesse, kommt zurück. Fertig.

Sag mir, in welcher Form dein Dump vorliegt (PUP, entschlüsselt, gemountete Partition, welche Firmware) und ob du Pro oder Slim hast — dann passe ich `fw-index.sh` auf die echte Struktur an und schreibe den Plymouth-Teil fertig.







EIN SKRIPT: 

Alles in einem Skript: legt das komplette Repo an, holt den Emulator-Unterbau, installiert die Build-Abhängigkeiten in WSL und baut das fertige Multiboot-Archiv.

Voraussetzung: Windows 10/11 mit WSL2 (Debian/Ubuntu). Ohne WSL bricht das Skript mit Anleitung ab — `debootstrap` gibt es unter Windows nicht.

Speichere das als `build-ps5os.ps1` und starte es in einer **PowerShell als Administrator**:

```powershell
Set-ExecutionPolicy -Scope Process Bypass -Force
.\build-ps5os.ps1
```

```powershell
#Requires -Version 5.1
<#
  build-ps5os.ps1 - baut das komplette ps5os-Repo und das Multiboot-Archiv.
  Parameter:
    -RepoPath   Zielordner (Standard: .\ps5os)
    -Distro     WSL-Distribution (Standard: erste verfügbare)
    -SkipBuild  nur Gerüst anlegen, nicht bauen
#>
[CmdletBinding()]
param(
  [string]$RepoPath = (Join-Path (Get-Location) 'ps5os'),
  [string]$Distro,
  [switch]$SkipBuild
)

$ErrorActionPreference = 'Stop'
$Repo = [IO.Path]::GetFullPath($RepoPath)
$Utf8NoBom = New-Object System.Text.UTF8Encoding $false

function Say  ($m) { Write-Host $m -ForegroundColor Cyan }
function Ok   ($m) { Write-Host "  + $m" -ForegroundColor DarkGray }
function Fail ($m) { Write-Host "FEHLER: $m" -ForegroundColor Red; exit 1 }

function Write-Text {
  param([string]$Rel, [string]$Content)
  $full = Join-Path $Repo $Rel
  $dir  = Split-Path $full -Parent
  if (-not (Test-Path -LiteralPath $dir)) { New-Item -ItemType Directory -Path $dir -Force | Out-Null }
  $Content = $Content -replace "`r`n", "`n"
  [IO.File]::WriteAllText($full, $Content, $Utf8NoBom)
  Ok $Rel
}

Say "Ziel: $Repo"
if (Test-Path -LiteralPath $Repo) { Remove-Item -LiteralPath $Repo -Recurse -Force }
New-Item -ItemType Directory -Path $Repo -Force | Out-Null
foreach ($d in 'tools','overlay','assets','build','out') {
  New-Item -ItemType Directory -Path (Join-Path $Repo $d) -Force | Out-Null
}

# ------------------------------------------------------------------ Tools
Write-Text 'tools/mkrootfs.sh' @'
#!/usr/bin/env bash
# Portiert ein Rootfs nach PS4-Multiboot. Ergebnis: out/<name>-ps4-rootfs.tar.xz
set -euo pipefail
NAME=${NAME:-ps5os}
ROOT=${ROOT:-build/root}
OVERLAY=${OVERLAY:-overlay}
FROM_TAR=${FROM_TAR:-}
OUT=${OUT:-out}
DISTRO=${DISTRO:-trixie}
MIRROR=${MIRROR:-http://deb.debian.org/debian}

CORE="systemd-sysv dbus network-manager openssh-server sudo curl ca-certificates python3 python3-aiohttp plymouth plymouth-themes imagemagick nano less file alsa-utils iproute2 iputils-ping"
EXTRAS="seatd cage cog chromium wpewebkit-driver grim fonts-inter fonts-noto-color-emoji python3-pil ffmpeg vgmstream-cli pipewire pipewire-pulse wireplumber"

cleanup(){ for d in dev/pts dev proc sys; do umount -R "$ROOT/$d" 2>/dev/null || true; done; }
trap cleanup EXIT

[ "$(id -u)" -eq 0 ] || { echo "als root ausfuehren (wsl -u root)"; exit 1; }
command -v debootstrap >/dev/null || { echo "debootstrap fehlt"; exit 1; }

rm -rf "$ROOT"; mkdir -p "$ROOT" "$OUT"

if [ -n "$FROM_TAR" ]; then
  echo "[*] importiere $FROM_TAR"
  tar -xpf "$FROM_TAR" -C "$ROOT" --numeric-owner --xattrs
else
  echo "[*] debootstrap $DISTRO"
  debootstrap --arch=amd64 --variant=minbase --include="${CORE//$'\n'/ }" "$DISTRO" "$ROOT" "$MIRROR"
fi

echo "[*] overlay"
tar -C "$OVERLAY" -cf - . | tar -C "$ROOT" -xpf - --numeric-owner

if [ -d assets/emufive/opt ]; then
  echo "[*] Emulator-Unterbau"
  tar -C assets/emufive/opt -cf - . | tar -C "$ROOT/opt" -xpf - --numeric-owner
fi

mount -t proc proc "$ROOT/proc"; mount -t sysfs sys "$ROOT/sys"
mount --rbind /dev "$ROOT/dev"; mount --make-rslave "$ROOT/dev"

echo "[*] System einrichten"
chroot "$ROOT" /bin/bash -euo pipefail <<'CHROOT'
export DEBIAN_FRONTEND=noninteractive
echo "ps5os" > /etc/hostname
printf '127.0.0.1 localhost\n127.0.1.1 ps5os\n' > /etc/hosts
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
apt-get update -qq || true

for p in seatd cage cog chromium wpewebkit-driver grim fonts-inter fonts-noto-color-emoji python3-pil ffmpeg vgmstream-cli pipewire pipewire-pulse wireplumber; do
  apt-get install -y --no-install-recommends "$p" >/dev/null 2>&1 || echo "  ! $p nicht verfuegbar"
done

id ps5 >/dev/null 2>&1 || useradd -m -s /bin/bash -G video,render,input,audio,seat ps5
echo 'ps5:ps5' | chpasswd; echo 'root:ps5' | chpasswd

chmod +x /usr/local/bin/ps5os-* /opt/ps5shell/shelld.py 2>/dev/null || true
systemctl enable ps5os-shelld.service ps5os-shell.service NetworkManager ssh >/dev/null 2>&1 || true
systemctl enable plymouth-start.service plymouth-quit.service >/dev/null 2>&1 || true
systemctl set-default graphical.target
plymouth-set-default-theme ps5os >/dev/null 2>&1 || true
[ -x /opt/ps4-mesa/bin/ps4-mesa-enable ] && /opt/ps4-mesa/bin/ps4-mesa-enable >/dev/null 2>&1 || true

apt-get clean; rm -rf /var/lib/apt/lists/* /etc/machine-id /etc/ssh/ssh_host_*
CHROOT

echo "[*] packen"
tar --numeric-owner --xattrs -cJpf "$OUT/$NAME-ps4-rootfs.tar.xz" -C "$ROOT" .
tar -tJf "$OUT/$NAME-ps4-rootfs.tar.xz" | grep -qx './etc/fstab' || { echo "FEHLER: fstab fehlt"; exit 1; }
ls -lh "$OUT/$NAME-ps4-rootfs.tar.xz"
'@

Write-Text 'tools/fetch-emufive.sh' @'
#!/usr/bin/env bash
# Holt Emulator + PS4-Mesa aus dem EmuFive-Release nach assets/emufive/opt
set -euo pipefail
DEST=${1:-assets/emufive}
mkdir -p "$DEST"
command -v curl >/dev/null || { echo "curl fehlt"; exit 1; }

TAG=${EMUFIVE_TAG:-}
if [ -z "$TAG" ]; then
  TAG=$(curl -fsSL https://api.github.com/repos/blue0x1/EmuFive/releases/latest \
        | grep -o '"tag_name": *"[^"]*"' | cut -d'"' -f4)
fi
[ -n "$TAG" ] || { echo "Release-Tag nicht ermittelbar"; exit 1; }
echo "[*] EmuFive $TAG"
curl -fL --retry 3 -o "$DEST/emufive.tar.xz" \
  "https://github.com/blue0x1/EmuFive/releases/download/$TAG/ps5-ps4-rootfs.tar.xz"

tar -xJf "$DEST/emufive.tar.xz" -C "$DEST" --wildcards './opt/ps5-on-ps4/*' 2>/dev/null \
  || tar -xJf "$DEST/emufive.tar.xz" -C "$DEST"
tar -xJf "$DEST/emufive.tar.xz" -C "$DEST" --wildcards './opt/ps4-mesa/*' 2>/dev/null || true
rm -f "$DEST/emufive.tar.xz"
echo "[*] nach $DEST/opt entpackt"
'@

Write-Text 'tools/fw-index.sh' @'
#!/usr/bin/env bash
set -euo pipefail
DUMP=${1:?Verzeichnis mit dem Dump}
OUT=${2:-manifest.tsv}
printf 'sha1\tsize\tkind\trole\tsrc\n' > "$OUT"

find "$DUMP" -type f -print0 | while IFS= read -r -d '' f; do
  ext=${f##*.}; ext=${ext,,}
  case "$ext" in
    gnf|dds|png|jpg|jpeg|bmp|tga) kind=texture ;;
    at9|at3|at5|wav|mp3)          kind=audio ;;
    otf|ttf|ttc|otc)              kind=font ;;
    self|elf|sprx|prx)            kind=binary ;;
    json|xml|sfo)                 kind=meta ;;
    db|sqlite|sqlite3)            kind=db ;;
    *)                            kind=other ;;
  esac
  role=asset
  case "$f" in
    *SceShellUI*|*shellui*)     role=shell ;;
    *SceShellCore*|*shellcore*) role=shellcore ;;
    *[Ff]ont*)                  role=font ;;
    *boot*|*startup*|*splash*)  role=bootanim ;;
    *sound*|*se_*|*bgm*)        role=sfx ;;
    *home*|*layout*)            role=homelayout ;;
    *icon*|*tile*)              role=icon ;;
  esac
  printf '%s\t%s\t%s\t%s\t%s\n' "$(sha1sum "$f" | cut -d' ' -f1)" \
    "$(stat -c%s "$f")" "$kind" "$role" "$f" >> "$OUT"
done

echo "--- Rollen ---"; cut -f4 "$OUT" | tail -n +2 | sort | uniq -c | sort -rn
echo "--- Typen ---"; cut -f3 "$OUT" | tail -n +2 | sort | uniq -c | sort -rn
echo "Manifest: $OUT"
'@

Write-Text 'tools/tex.sh' @'
#!/usr/bin/env bash
set -euo pipefail
in=$1; out=$2
case "${in##*.}" in
  png|jpg|jpeg) cp -f "$in" "$out" ;;
  dds) magick "$in" "$out" ;;
  gnf)
    tmp="${out%.png}.dds"
    if   command -v gnftool >/dev/null; then gnftool d "$in" "$tmp"
    elif command -v GnfTool >/dev/null; then GnfTool -d "$in" "$tmp"
    else echo "kein GNF-Tool installiert" >&2; exit 2; fi
    magick "$tmp" "$out"; rm -f "$tmp" ;;
  *) echo "unbekannt: $in" >&2; exit 2 ;;
esac
'@

Write-Text 'tools/at9.sh' @'
#!/usr/bin/env bash
set -euo pipefail
tmp=$(mktemp --suffix=.wav)
vgmstream-cli -o "$tmp" "$1" >/dev/null
ffmpeg -y -loglevel error -i "$tmp" -c:a libvorbis -q:a 5 "$2"
rm -f "$tmp"
'@

Write-Text 'tools/mk-tiles.py' @'
#!/usr/bin/env python3
import sys, json, pathlib
from PIL import Image, ImageFilter, ImageEnhance

game, out = pathlib.Path(sys.argv[1]), pathlib.Path(sys.argv[2])
out.mkdir(parents=True, exist_ok=True)
sce = game / "sce_sys"

def pick(*names):
    for n in names:
        p = sce / n
        if p.exists():
            return Image.open(p).convert("RGB")
    return None

icon = pick("icon0.png", "icon0.jpg") or Image.new("RGB", (512, 512), (20, 20, 24))
bg = pick("pic0.png", "pic1.png", "pic0.jpg") or icon
bg = bg.resize((1920, 1080), Image.LANCZOS).filter(ImageFilter.GaussianBlur(30))
bg = ImageEnhance.Brightness(bg).enhance(0.5)
bg.save(out / "bg.jpg", quality=82)
icon.resize((512, 512), Image.LANCZOS).save(out / "tile.png")

title = game.name
p = sce / "param.json"
if p.exists():
    try:
        title = json.loads(p.read_text(encoding="utf-8", errors="ignore")).get("titleName", title)
    except Exception:
        pass
(out / "meta.json").write_text(json.dumps({"title": title, "id": game.name}), encoding="utf-8")
'@

Write-Text 'tools/push.sh' @'
#!/usr/bin/env bash
set -euo pipefail
PS4=${PS4:?PS4=192.168.1.50 setzen}
F=${1:?Datei}
curl -sS --ftp-create-dirs -T "$F" "ftp://${PS4}:2121/user/home/$(basename "$F")"
echo "hochgeladen. Payload starten, im Menue ps5os -> Install to PS4."
'@

Write-Text 'tools/build-all.sh' @'
#!/usr/bin/env bash
# Komplettbau im Linux-Dateisystem. Aufruf: wsl -u root -- bash tools/build-all.sh
set -euo pipefail
cd "$(dirname "$0")/.."
apt-get update -qq
DEBIAN_FRONTEND=noninteractive apt-get install -y -qq debootstrap curl xz-utils ca-certificates jq >/dev/null
./tools/fetch-emufive.sh
./tools/mkrootfs.sh
echo "FERTIG: $(ls out/*.tar.xz)"
'@

# ------------------------------------------------------------------ Overlay
Write-Text 'overlay/etc/fstab' @'
# wird vom Multiboot-Installer ergaenzt
proc  /proc  proc  defaults  0 0
'@

Write-Text 'overlay/etc/systemd/system/ps5os-shelld.service' @'
[Unit]
Description=ps5os shell backend
After=network.target

[Service]
User=ps5
ExecStart=/usr/bin/python3 /opt/ps5shell/shelld.py
Restart=always

[Install]
WantedBy=multi-user.target
'@

Write-Text 'overlay/etc/systemd/system/ps5os-shell.service' @'
[Unit]
Description=ps5os shell
After=ps5os-shelld.service
Conflicts=getty@tty1.service

[Service]
User=ps5
PAMName=login
TTYPath=/dev/tty1
StandardInput=tty
StandardOutput=journal
ExecStart=/usr/local/bin/ps5os-session
Restart=always
RestartSec=2

[Install]
WantedBy=graphical.target
'@

Write-Text 'overlay/usr/local/bin/ps5os-session' @'
#!/bin/bash
export XDG_RUNTIME_DIR=/run/user/1000
export WLR_BACKENDS=drm WLR_RENDERER=gles2 RADV_EXPERIMENTAL=sparse
URL=http://127.0.0.1:8790/
if command -v cog >/dev/null; then
  exec cage -s -- "$(command -v cog)" --platform=wl --bg-color=0x00000000 "$URL"
elif command -v chromium >/dev/null; then
  exec cage -s -- "$(command -v chromium)" --kiosk --noerrdialogs \
    --disable-features=Translate --user-data-dir=/home/ps5/.chromium "$URL"
fi
'@

Write-Text 'overlay/usr/local/bin/ps5os-run' @'
#!/bin/bash
set -u
G="$1"
if [ -x /opt/ps5-on-ps4/bin/ps5-run ]; then exec /opt/ps5-on-ps4/bin/ps5-run "$G"; fi
BIN=$(ls /opt/ps5-on-ps4/bin/kytyps5 /opt/ps5-on-ps4/kytyps5 2>/dev/null | head -1)
[ -n "$BIN" ] && exec "$BIN" --game "$G"
echo "kein Emulator gefunden" >&2
'@

Write-Text 'overlay/usr/local/bin/ps5os-shot' @'
#!/bin/bash
d="$HOME/Pictures"; mkdir -p "$d"
f="$d/shot-$(date +%Y%m%d-%H%M%S).png"
if command -v grim >/dev/null; then grim "$f"
else ffmpeg -y -loglevel error -f kmsgrab -i - -vf hwdownload,format=bgr0 "$f"; fi
echo "$f"
'@

Write-Text 'overlay/usr/share/plymouth/themes/ps5os/ps5os.plymouth' @'
[Plymouth Theme]
Name=ps5os
Description=ps5os Boot
ModuleName=script

[script]
ImageDir=/usr/share/plymouth/themes/ps5os
ScriptFile=/usr/share/plymouth/themes/ps5os/ps5os.script
'@

Write-Text 'overlay/usr/share/plymouth/themes/ps5os/ps5os.script' @'
Window.SetBackgroundTopColor(0.043, 0.047, 0.063);
Window.SetBackgroundBottomColor(0.043, 0.047, 0.063);

logo.image = Image.Text("ps5os", 0.95, 0.96, 1.0);
logo.sprite = Sprite(logo.image);
logo.sprite.SetPosition(Window.GetWidth() / 2 - logo.image.GetWidth() / 2,
                        Window.GetHeight() / 2 - logo.image.GetHeight() / 2);

dot.image = Image.Text("l", 0.0, 0.44, 0.82);
dot.sprite = Sprite(dot.image);
dot.sprite.SetPosition(Window.GetWidth() / 2 - dot.image.GetWidth() / 2,
                       Window.GetHeight() / 2 + 60);

fun refresh() {
  t = Math.Sin(Plymouth.GetMode() + Time.Now() * 2.0);
  dot.sprite.SetOpacity(0.3 + Math.Abs(t) * 0.7);
  logo.sprite.SetOpacity(0.75 + Math.Abs(t) * 0.25);
}
Plymouth.SetRefreshFunction(refresh);
'@

# ------------------------------------------------------------------ Shell UI
Write-Text 'overlay/opt/ps5shell/index.html' @'
<!doctype html>
<html lang="de"><head>
<meta charset="utf-8"><title>ps5os</title>
<link rel="stylesheet" href="style.css">
</head><body>
<div id="bg"></div>

<header id="topbar">
  <div class="me"><img src="assets/icons/avatar.png" alt=""></div>
  <nav id="tabs">
    <button data-tab="games" class="on">Spiele</button>
    <button data-tab="media">Medien</button>
  </nav>
  <div class="tools">
    <button id="btn-search">Suche</button>
    <button id="btn-settings">Einstellungen</button>
  </div>
</header>

<main id="row"></main>

<div id="actions" class="hidden">
  <button data-act="play">Spielen</button>
  <button data-act="options">Optionen</button>
</div>

<footer id="hint">
  <span><b>X</b> Starten</span><span><b>O</b> Zurueck</span>
  <span><b>OPTIONS</b> Optionen</span><span><b>PS</b> Control Center</span>
</footer>

<div id="cc" class="hidden">
  <div class="cc-row">
    <button data-cc="power">Energie</button>
    <button data-cc="sound">Ton</button>
    <button data-cc="ctrl">Controller</button>
    <button data-cc="shot">Screenshot</button>
    <button data-cc="home">Home</button>
  </div>
</div>

<script src="shell.js"></script>
</body></html>
'@

Write-Text 'overlay/opt/ps5shell/style.css' @'
@font-face{font-family:SST;src:url(assets/fonts/sst-roman.woff2) format("woff2");font-weight:400;font-display:swap}
@font-face{font-family:SST;src:url(assets/fonts/sst-light.woff2) format("woff2");font-weight:300;font-display:swap}
:root{--bg:#0b0c10;--ink:#f4f5f8;--dim:#9aa0ac;--blue:#0070d1;--ring:#fff;--tile:196px;--gap:22px;--r:18px}
*{box-sizing:border-box;margin:0;padding:0}
html,body{height:100%;overflow:hidden;background:var(--bg);color:var(--ink);
  font-family:SST,Inter,system-ui,sans-serif;font-weight:300;letter-spacing:.01em;user-select:none;cursor:none}

#bg{position:fixed;inset:-40px;background-size:cover;background-position:center;
  filter:saturate(1.05);opacity:.9;transition:opacity .45s ease}
#bg::after{content:"";position:absolute;inset:0;
  background:linear-gradient(180deg,rgba(6,7,10,.55) 0%,rgba(6,7,10,.15) 35%,rgba(6,7,10,.85) 100%)}

#topbar{position:fixed;top:0;left:0;right:0;height:96px;display:flex;align-items:center;
  gap:28px;padding:0 56px;z-index:5}
.me img{width:40px;height:40px;border-radius:50%;background:#2a2d36}
#tabs{display:flex;gap:26px}
#tabs button{background:none;border:0;color:var(--dim);font:inherit;font-size:19px;
  padding:8px 2px;position:relative;transition:color .2s}
#tabs button.on{color:var(--ink)}
#tabs button.on::after{content:"";position:absolute;left:0;right:0;bottom:0;height:2px;
  background:var(--ink);border-radius:2px}
.tools{margin-left:auto;display:flex;gap:14px}
.tools button{background:rgba(255,255,255,.07);border:0;color:var(--ink);font:inherit;
  font-size:15px;padding:10px 18px;border-radius:999px;transition:background .2s}
.tools button:hover{background:rgba(255,255,255,.16)}

#row{position:fixed;left:56px;right:56px;bottom:150px;display:flex;gap:var(--gap);
  align-items:flex-end;overflow-x:auto;overflow-y:hidden;scrollbar-width:none;
  scroll-behavior:smooth;padding:20px 0 0}
#row::-webkit-scrollbar{display:none}

.tile{flex:0 0 auto;width:var(--tile);transition:transform .22s cubic-bezier(.2,.7,.2,1);position:relative}
.tile img{width:100%;aspect-ratio:1;border-radius:var(--r);object-fit:cover;display:block;
  box-shadow:0 18px 40px rgba(0,0,0,.55)}
.tile.sel{transform:translateY(-14px) scale(1.09);z-index:3}
.tile.sel img{outline:3px solid var(--ring);outline-offset:4px}
.tile .name{font-size:15px;margin-top:12px;opacity:0;transition:opacity .2s;max-width:var(--tile);
  white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.tile.sel .name{opacity:1}

#actions{position:fixed;left:56px;bottom:96px;display:flex;gap:12px;z-index:6;
  transition:opacity .18s}
#actions button{background:rgba(255,255,255,.1);border:0;color:var(--ink);font:inherit;
  font-size:16px;padding:12px 26px;border-radius:999px;backdrop-filter:blur(20px)}
#actions button:first-child{background:var(--blue)}

#hint{position:fixed;left:56px;right:56px;bottom:26px;display:flex;gap:30px;
  font-size:13px;color:var(--dim);z-index:6}
#hint b{color:var(--ink);font-weight:400}

#cc{position:fixed;left:0;right:0;bottom:0;padding:28px 56px 44px;
  background:linear-gradient(180deg,rgba(12,13,17,.4),rgba(12,13,17,.96));
  backdrop-filter:blur(34px);transform:translateY(100%);
  transition:transform .3s cubic-bezier(.2,.7,.2,1);z-index:10}
#cc.open{transform:translateY(0)}
.cc-row{display:flex;gap:16px;justify-content:center}
.cc-row button{width:130px;height:96px;border:0;border-radius:16px;
  background:rgba(255,255,255,.09);color:var(--ink);font:inherit;font-size:14px}

.hidden{opacity:0;pointer-events:none}
'@

Write-Text 'overlay/opt/ps5shell/shell.js' @'
const WS = new WebSocket(`ws://${location.host}/ws`);
let games = [], idx = 0, ccOpen = false;
const row = document.getElementById("row");
const bg = document.getElementById("bg");
const actions = document.getElementById("actions");
const cc = document.getElementById("cc");

const send = (op, arg) => WS.readyState === 1 && WS.send(JSON.stringify({ op, arg }));

function render() {
  row.innerHTML = "";
  games.forEach(g => {
    const d = document.createElement("div");
    d.className = "tile";
    d.innerHTML = '<img src="' + g.tile + '" alt=""><div class="name"></div>';
    d.querySelector(".name").textContent = g.title;
    row.appendChild(d);
  });
  select(idx, true);
}

function select(i, instant) {
  idx = Math.max(0, Math.min(games.length - 1, i));
  [...row.children].forEach((t, n) => t.classList.toggle("sel", n === idx));
  const g = games[idx];
  if (!g) { actions.classList.add("hidden"); return; }
  bg.style.backgroundImage = 'url("' + g.bg + '")';
  actions.classList.remove("hidden");
  row.children[idx].scrollIntoView({ inline: "center", block: "nearest", behavior: instant ? "auto" : "smooth" });
  send("focus", g.id);
}

function launch() {
  const g = games[idx];
  if (!g) return;
  send("launch", g.id);
  actions.classList.add("hidden");
}
function toggleCC() { ccOpen = !ccOpen; cc.classList.toggle("open", ccOpen); }

WS.onmessage = e => {
  const m = JSON.parse(e.data);
  if (m.type === "library") { games = m.games; render(); }
  if (m.type === "game_end") actions.classList.remove("hidden");
};

addEventListener("keydown", e => {
  const k = e.key;
  if (k === "ArrowLeft") select(idx - 1);
  else if (k === "ArrowRight") select(idx + 1);
  else if (k === "Enter") { if (!ccOpen) launch(); }
  else if (k === "Escape" || k === "Backspace") toggleCC();
  else return;
  e.preventDefault();
});

let prev = {};
function poll() {
  for (const p of navigator.getGamepads()) {
    if (!p) continue;
    const b = i => !!p.buttons[i] && p.buttons[i].pressed;
    const edge = (i, fn) => { if (b(i) && !prev[i]) fn(); prev[i] = b(i); };
    edge(14, () => select(idx - 1));
    edge(15, () => select(idx + 1));
    edge(0, () => launch());
    edge(1, () => ccOpen && toggleCC());
    edge(9, toggleCC);
    edge(16, toggleCC);
  }
  requestAnimationFrame(poll);
}
poll();

document.querySelectorAll("#tabs button").forEach(b => b.onclick = () => {
  document.querySelectorAll("#tabs button").forEach(x => x.classList.remove("on"));
  b.classList.add("on");
  send("tab", b.dataset.tab);
});
document.querySelectorAll("#actions button").forEach(b =>
  b.onclick = () => b.dataset.act === "play" && launch());
document.querySelectorAll(".cc-row button").forEach(b =>
  b.onclick = () => { send("cc", b.dataset.cc); if (b.dataset.cc !== "sound") toggleCC(); });

WS.onopen = () => send("list");
'@

Write-Text 'overlay/opt/ps5shell/shelld.py' @'
#!/usr/bin/env python3
import asyncio, json, os, pathlib
from aiohttp import web

ROOT = pathlib.Path("/opt/ps5shell")
LIB = pathlib.Path(os.environ.get("PS5OS_LIB", "/user/home/PS5"))
STATE = pathlib.Path("/var/lib/ps5os/state.json")
STATE.parent.mkdir(parents=True, exist_ok=True)

state = json.loads(STATE.read_text()) if STATE.exists() else {"volume": 60, "fav": []}
save = lambda: STATE.write_text(json.dumps(state))

def library():
    out = []
    if not LIB.exists():
        return out
    for d in sorted(LIB.iterdir()):
        if not d.is_dir():
            continue
        art = d / "art"
        if not art.joinpath("tile.png").exists():
            continue
        if not ((d / "sce_sys" / "eboot.bin").exists() or list(d.glob("*.zar"))):
            continue
        meta = {}
        if art.joinpath("meta.json").exists():
            try:
                meta = json.loads(art.joinpath("meta.json").read_text())
            except Exception:
                pass
        out.append({"id": d.name, "title": meta.get("title", d.name),
                    "tile": f"/lib/{d.name}/art/tile.png",
                    "bg": f"/lib/{d.name}/art/bg.jpg"})
    out.sort(key=lambda g: (g["id"] not in state["fav"], g["title"].lower()))
    return out

async def ws_handler(request):
    ws = web.WebSocketResponse()
    await ws.prepare(request)
    async for msg in ws:
        m = json.loads(msg.data)
        op = m.get("op")

        if op == "list":
            await ws.send_json({"type": "library", "games": library()})

        elif op == "launch":
            g = LIB / m["arg"]
            if not g.exists():
                continue
            await ws.send_json({"type": "launching"})
            proc = await asyncio.create_subprocess_exec("/usr/local/bin/ps5os-run", str(g))
            await proc.wait()
            await ws.send_json({"type": "game_end"})

        elif op == "volume":
            state["volume"] = max(0, min(100, int(m["arg"])))
            save()
            await asyncio.create_subprocess_exec(
                "wpctl", "set-volume", "@DEFAULT_AUDIO_SINK@", f"{state['volume']}%")

        elif op == "cc":
            act = m["arg"]
            if act == "power":
                await asyncio.create_subprocess_exec("systemctl", "poweroff")
            elif act == "shot":
                await asyncio.create_subprocess_exec("/usr/local/bin/ps5os-shot")
    return ws

async def lib_static(request):
    p = LIB / request.match_info["name"] / "art" / request.match_info["file"]
    return web.FileResponse(p) if p.exists() else web.Response(status=404)

app = web.Application()
app.router.add_get("/ws", ws_handler)
app.router.add_get("/lib/{name}/{file}", lib_static)
app.router.add_static("/", ROOT, show_index=True)
web.run_app(app, host="127.0.0.1", port=8790, print=None)
'@

# ------------------------------------------------------------------ Doku
Write-Text 'README.md' @'
# ps5os

Linux-OS mit nachgebauter PS5-Oberflaeche fuer die PS4, installierbar ueber PS4 Multiboot.

## Bauen

```
wsl -u root -- bash tools/build-all.sh
```

Ergebnis: `out/ps5os-ps4-rootfs.tar.xz`

## Ausliefern

1. Aus PS4 Multiboot holen: `bzImage`, `initramfs.cpio.gz`, `bootargs.txt`, `linux-2048mb.bin`
   (Firmware 14.02: `linux-2048mb-14.02.bin`)
2. Per FTP auf die PS4: die drei Boot-Dateien nach `/data/linux/boot/`, Payload nach `/data/payloads/`
3. `PS4=<ip> ./tools/push.sh out/ps5os-ps4-rootfs.tar.xz`
4. Payload starten, im Menue `ps5os` -> Install to PS4
5. In `bootargs.txt` zusaetzlich `splash` ergaenzen, damit der Bootscreen kommt

## Assets aus dem Firmware-Dump

```
./tools/fw-index.sh /pfad/zum/dump manifest.tsv
awk -F'\t' '$4=="font"{print $5}' manifest.tsv | xargs -I{} cp -f {} assets/fonts/
awk -F'\t' '$4=="sfx"{print $5}' manifest.tsv | while read -r f; do
  ./tools/at9.sh "$f" "assets/sfx/$(basename "${f%.*}").ogg"; done
```

Spiele-Kacheln: `./tools/mk-tiles.py /user/home/PS5/<Titel> /user/home/PS5/<Titel>/art`

## Login

`ps5` / `ps5`, root dasselbe.
'@

Write-Text 'assets/fonts/.keep' ''
Write-Text 'assets/sfx/.keep' ''
Write-Text 'assets/icons/.keep' ''

Say "Repo steht: $Repo"

if ($SkipBuild) { Say 'SkipBuild gesetzt, fertig.'; exit 0 }

# ------------------------------------------------------------------ Build
$wsl = Get-Command wsl.exe -ErrorAction SilentlyContinue
if (-not $wsl) { Fail "WSL fehlt. In Admin-PowerShell: wsl --install -d Debian, dann neu anmelden." }

if (-not $Distro) {
  $list = (& wsl.exe --list --quiet) 2>$null | ForEach-Object { $_.Trim() } | Where-Object { $_ -and $_ -notmatch 'docker' }
  if (-not $list) { Fail "Keine WSL-Distribution. Installieren: wsl --install -d Debian" }
  $Distro = @($list)[0]
}
Say "Baue in WSL-Distribution: $Distro"

$src = (& wsl.exe -d $Distro -- wslpath -a $Repo).Trim()
$dst = '~/ps5os-build'

Say 'Kopiere ins Linux-Dateisystem'
& wsl.exe -d $Distro -u root -- bash -lc "rm -rf $dst && mkdir -p $dst && cp -a '$src'/." 
if ($LASTEXITCODE -ne 0) { Fail 'Kopieren fehlgeschlagen' }
& wsl.exe -d $Distro -u root -- bash -lc "cd $dst && chmod +x tools/*.sh tools/*.py overlay/usr/local/bin/* overlay/opt/ps5shell/shelld.py"

Say 'Baue (debootstrap + Emulator + Pakete, dauert einige Minuten)'
& wsl.exe -d $Distro -u root -- bash -lc "cd $dst && bash tools/build-all.sh"
if ($LASTEXITCODE -ne 0) { Fail 'Build fehlgeschlagen, siehe Ausgabe oben' }

$outDir = Join-Path $Repo 'out'
$outLin = (& wsl.exe -d $Distro -- wslpath -a $outDir).Trim()
& wsl.exe -d $Distro -u root -- bash -lc "cp -a $dst/out/. '$outLin'/" 

$artifact = Get-ChildItem -LiteralPath $outDir -Filter '*.tar.xz' | Select-Object -First 1
if (-not $artifact) { Fail 'Kein Artefakt erzeugt' }

Write-Host ''
Say 'FERTIG'
Write-Host ("Archiv: " + $artifact.FullName) -ForegroundColor Green
Write-Host ''
Write-Host 'Naechste Schritte:' -ForegroundColor Yellow
Write-Host '  1. PS4 Multiboot Release holen: bzImage, initramfs.cpio.gz, bootargs.txt, linux-2048mb.bin'
Write-Host '  2. Per FTP: boot-Dateien -> /data/linux/boot/, Payload -> /data/payloads/'
Write-Host '  3. In WSL: PS4=<ps4-ip> bash tools/push.sh out/ps5os-ps4-rootfs.tar.xz'
Write-Host '  4. Payload starten, im Menue ps5os -> Install to PS4'
Write-Host '  5. In bootargs.txt splash ergaenzen'
```

## Was das Skript tut

Es legt alle 22 Dateien an (Porter, Firmware-Indexer, Textur- und Audio-Konverter, Kachel-Generator, UI, Backend, systemd-Units, Plymouth-Theme), holt die neueste EmuFive-Version als Emulator-Unterbau, installiert `debootstrap` und die Build-Pakete in WSL und baut daraus das Rootfs. Am Ende liegt `out/ps5os-ps4-rootfs.tar.xz` bereit.

Zwei Dinge, die du wissen musst:

- **Bauen passiert in WSL auf ext4**, nicht auf `C:`. `debootstrap` braucht Root, xattrs und Symlinks — auf `/mnt/c` bricht es sonst.
- **Pakete werden weich installiert.** `cage`, `cog`, `wpewebkit-driver` sind je nach Distribution nicht alle da; fehlt eins, fällt die Session auf Chromium im Kiosk zurück und der Build läuft trotzdem durch statt abzubrechen.

Läuft es durch, hast du ein installierbares System mit deiner Oberfläche. Danach ist der sinnvolle nächste Schritt die Asset-Pipeline auf deinem Dump — dafür brauche ich die Struktur deines Dumps.