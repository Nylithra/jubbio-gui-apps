<p align="center"><img src="assets/logo-192.png" width="120" alt="Jubbio Devs"></p>

# Jubbio GUI Uygulamaları — Jubbio Devs Docs (Topluluk)

> ⚠️ **Resmi dokümantasyon değildir.** Bu rehber **Jubbio Devs topluluğu** tarafından, jubbio.com web istemcisinin davranışı incelenerek ve gerçek bir GUI uygulaması üzerinde denenerek hazırlandı (Ekim 2026). Jubbio ile resmi bir bağı yoktur. Jubbio resmi GUI dokümantasyonunu yayınladığında esas kaynak odur; buradaki bilgiler değişebilir.
> Resmi dokümantasyon: https://jubbio.com/dev/docs

**Etiketler:** ✅ **Doğrulandı** = gerçek Jubbio ortamında denendi · 🔎 **İstemci** = web istemcisinin koduna göre, sunucu tarafı ayrıca denenmedi · ❓ **Bilinmiyor** = henüz çözülemedi

## İçindekiler

1. [GUI uygulaması nedir?](#1-gui-uygulaması-nedir)
2. [Hızlı başlangıç](#2-hızlı-başlangıç)
3. [Yayınlama ve kurulum](#3-yayınlama-ve-kurulum)
4. [Manifest](#4-manifest)
5. [Çalışma ortamı (iframe ve CSP)](#5-çalışma-ortamı-iframe-ve-csp)
6. [Host protokolü](#6-host-protokolü)
7. [Metotlar](#7-metotlar)
8. [Documents (ortak durum)](#8-documents-ortak-durum)
9. [Olaylar ve sunucu saati](#9-olaylar-ve-sunucu-saati)
10. [Hata kodları](#10-hata-kodları)
11. [Mini SDK](#11-mini-sdk)
12. [Örnek: Ortak İzle](#12-örnek-ortak-i̇zle)
13. [Yerel geliştirme](#13-yerel-geliştirme)
14. [Bilinen sorunlar ve SSS](#14-bilinen-sorunlar-ve-sss)

---

## 1. GUI uygulaması nedir?

GUI uygulaması, Jubbio Geliştirici Portalı'na **statik bir build klasörü** (`index.html` + JS/CSS) olarak yüklenen web sayfasıdır. Sunucularda **"Uygulama" tipindeki kanallarda** açılır. Sayfa Jubbio istemcisinin içinde bir `<iframe>` olarak çalışır ve Jubbio ile yalnızca bir **MessagePort** üzerinden konuşur.

Bir backend yazmadan şunları yapabilir:
- Ortak veri tutabilir (`documents`).
- Kullanıcıyı ve kanalı tanıyabilir (`context`).
- Sunucu saatine göre senkron çalışabilir (`clock`).

### Mimari

```
[Senin uygulaman]  ⇄ postMessage ⇄  [Jubbio istemcisi (host)]  ⇄ HTTPS + Bearer ticket ⇄  [gateway.jubbio.com]
 iframe, <appId>.jubbio.net          jubbio.com, kanal görünümü                              /api/v1/apps/…
```

1. Kullanıcı uygulama kanalını açar. Host `POST /api/v1/apps/channels/{guildId}/{channelId}/launch` ile uygulamayı başlatır ve bir **context**, kısa ömürlü bir **ticket** ve `entryUrl` alır.
2. Host `entryUrl`'i sandbox'lı bir iframe'de açar.
3. Uygulama `jubbio.app.hello` gönderir, host bir `MessagePort` verir.
4. Uygulamanın her isteğini (`documents.get` vb.) host ticket ile gateway'e iletir; cevap porttan geri döner.
5. Sunucu olaylarını (`document_changed`, saat örnekleri…) host uygulamaya `event` mesajı olarak aktarır.

### Bu bilgiler nasıl toplandı?

Ekim 2026 itibarıyla resmi dokümantasyonda GUI uygulamalarına dair bir bölüm yoktu. Bilgiler üç yoldan toplandı:
- **Web istemcisinin davranışı:** portalın manifest doğrulaması, host'un handshake kodu ve mesaj sınırları incelendi.
- **Canlı deneme:** gerçek bir sunucuda bir "keşif" uygulaması yayınlandı ve parametre şekilleri tek tek denendi.
- **Ölçüm:** uygulama sayfasının güvenlik politikası (CSP) gerçek ortamda ölçüldü.

---

## 2. Hızlı başlangıç

Örnek: kanaldaki herkesin aynı sayacı gördüğü ortak bir "tık sayacı". Derleme aracı gerekmez.

**Gerekenler:**
- Jubbio'da bir uygulama (https://jubbio.com/dev/applications → yeni uygulama).
- Yönetme yetkin olan bir sunucu.

### 1) Dosyalar

```
dist/
├── index.html
├── jubbio.js      ← Mini SDK (bölüm 11)
└── app.js
```

**dist/index.html**

```html
<!doctype html>
<html lang="tr">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Tık Sayacı</title>
  <style>
    body { font: 16px system-ui; background: #0f172a; color: #f8fafc; display: grid; place-items: center; height: 100vh; margin: 0; }
    button { font-size: 28px; padding: 16px 32px; border-radius: 14px; border: 0; background: #10b981; color: #fff; cursor: pointer; }
  </style>
</head>
<body>
  <div>
    <p id="status">Bağlanıyor…</p>
    <button id="btn" disabled>0</button>
  </div>
  <!-- Klasik script kullan: sandbox'ta module script'ler CORS ister. -->
  <script src="jubbio.js"></script>
  <script src="app.js"></script>
</body>
</html>
```

**dist/app.js**

```js
const jb = new Jubbio();
const KEY = "counter";
let version = 0;

const $ = (id) => document.getElementById(id);

async function load() {
  const res = await jb.call("documents.get", { scope: "channel", key: KEY });
  version = res.version;
  $("btn").textContent = res.value?.count ?? 0;
}

async function increment() {
  for (let i = 0; i < 3; i++) {
    try {
      const cur = Number($("btn").textContent) || 0;
      const res = await jb.call("documents.update", {
        scope: "channel",
        key: KEY,
        value: { count: cur + 1 },
        expectedVersion: version,          // zorunlu: iyimser kilit
      });
      version = res.version;
      $("btn").textContent = res.value.count;
      return;
    } catch (err) {
      if (err.code !== "VERSION_CONFLICT") throw err;
      await load();                        // başkası yazmış → tazele, tekrar dene
    }
  }
}

jb.connect().then(async () => {
  const ctx = await jb.call("context.get");
  $("status").textContent = `Merhaba ${ctx.user?.displayName ?? "misafir"}!`;
  await load();
  $("btn").disabled = false;
  $("btn").onclick = increment;
  jb.on("document.changed", (d) => d.key === KEY && load());
  setInterval(load, 5000);                 // olay kaçarsa diye yedek
});
```

### 2) Manifest

```json
{
  "schemaVersion": 1,
  "name": "Tık Sayacı",
  "version": "1.0.0",
  "entryPoint": "index.html",
  "capabilities": ["documents"],
  "room": { "kind": "workspace", "maxParticipants": 32, "spectators": true },
  "backend": { "mode": "managed" },
  "networkOrigins": []
}
```

### 3) Yayınla

1. Portal → uygulaman → **GUI Apps**.
2. Manifest JSON'unu **Manifest** kutusuna yapıştır.
3. Build klasörü olarak `dist/`'i seç.
4. **Doğrula ve yayınla** → "1.0.0 yayınlandı." mesajı gelmeli.
5. **Sunucuya kur**: sunucu ID'si + sürüm.
6. Sunucuda **+** → Oda Tipi **Uygulama** → uygulamayı ve sürümü seç → oluştur.

### 4) Dene

Kanalı iki farklı hesapla aç. Birinin tıklaması ~1 saniye içinde diğerinde görünmelidir.

> ⚠️ Yayınlanan sürüm değiştirilemez; her güncellemede `version`'ı artır (`1.0.0` → `1.0.1`). Mevcut kanalda sürüm değiştirme şu an hata verebiliyor; o durumda yeni sürümle yeni bir kanal oluştur (bölüm 14).

---

## 3. Yayınlama ve kurulum

Portal adresi: `https://jubbio.com/dev/applications/<APP_ID>/gui`

### Build klasörü kuralları 🔎

Seçilen klasörün **kendisi kök kabul edilir**: `webkitRelativePath`'in ilk parçası atılır. Yani `dist/js/app.js` → `js/app.js` olur.

| Kural | Değer |
|---|---|
| Dosya sayısı | 1 – 512 |
| Toplam boyut | ≤ 100 MB |
| Tek dosya | ≤ 16 MB |
| Dosya yolu | `^[-a-zA-Z0-9_./]+$`, `..` yasak. Boşluk, Türkçe karakter, `@`, `~` kullanılamaz |
| Giriş dosyası | `manifest.entryPoint` klasörde bulunmalı |

Vite kullanıyorsan `base: "./"` ayarla. Hash'li dosya adları kurallara uyar.

### Sürümler

- Sürüm numarası `version` alanıdır: `^\d+\.\d+\.\d+(-[a-zA-Z0-9.-]+)?$`.
- **Yayınlanan sürüm değiştirilemez.** Yayınlanmış sürümler **Geri çek** ile geri çekilebilir.
- Her sürüm kendi adresinden sunulur ✅: `https://<APP_ID>.jubbio.net/assets/<APP_ID>/<version>/index.html`

### Sunucuya kurulum

Sunucu kimliği (`^[1-9]\d{0,19}$`) ve yayınlanmış bir sürüm seçilir. Sunucuyu yönetme yetkisi gerekir.

**Kurulum, o sürümün manifestindeki yeteneklere izin verir.** Manifestte olmayan bir yeteneğin metotları `PERMISSION_DENIED` döner. Her sürüm ayrı bir kurulum olarak listelenir ("Test v1.0.2", "Test v1.0.1"…).

### Uygulama kanalı

Kanal oluştururken Oda Tipi olarak **Uygulama**'yı seç. Listede Jubbio'nun hazır oyunları (Satranç, Okey…) ve senin kurulumların görünür. Kanal seçilen sürüme bağlanır.

### Güncelleme

1. `version`'ı artır ve yayınla.
2. Yeni sürümü sunucuya kur.
3. Kanal menüsü → **Uygulama sürümü** → yeni sürüm → **Bu sürümü kullan**.

> ⚠️ **Bilinen sorun:** 3. adım denemelerimizde her seferinde *"Sürüm değiştirilemedi. Kurulumun aktif ve sürümün yayınlanmış olması gerekir."* hatası verdi. Çalışan yol: yeni sürümle **yeni bir uygulama kanalı** oluşturmak. Dokümanlar kanal bazlı olduğu için yeni kanal boş başlar.

### Portalın iç API'si 🔎 (bilgi amaçlı)

| Metot | Yol | Gövde |
|---|---|---|
| POST | `/api/v1/applications/{appId}/gui/versions` | `{ manifest, bundleHash }` |
| PUT | `/api/v1/apps/assets/{appId}/{version}?path=…` | dosya (octet-stream) |
| POST | `/api/v1/apps/assets/{appId}/{version}/receipt` | `{ files: [{ path, sha256, bytes }] }` |
| POST | `/api/v1/applications/{appId}/gui/versions/{version}/publish` | `{ receipt }` |
| POST | `/api/v1/apps/guilds/{guildId}/installations` | `{ applicationId, version, approvedCapabilities }` |

"Geliştirici backend'i / Backend anahtarı" bölümü yalnızca `backend.mode: "external"` içindir. Anahtar yalnızca kendi sunucunda kullanılır; bundle'a ya da manifeste konmaz. Anahtar 30 gün geçerlidir.

---

## 4. Manifest

Manifest bir dosya olarak yüklenmez; portaldaki Manifest kutusuna yapıştırılır.

```json
{
  "schemaVersion": 1,
  "name": "Ortak İzle",
  "version": "1.1.0",
  "entryPoint": "index.html",
  "capabilities": ["documents", "rooms"],
  "room": { "kind": "workspace", "maxParticipants": 64, "spectators": true },
  "backend": { "mode": "managed" },
  "networkOrigins": ["https://res.cloudinary.com"]
}
```

### Alanlar 🔎

| Alan | Tip | Kural |
|---|---|---|
| `schemaVersion` * | number | Tam olarak `1` |
| `name` * | string | Boş olmayan, ≤ 80 karakter |
| `version` * | string | `^\d+\.\d+\.\d+(-[a-zA-Z0-9.-]+)?$`, ≤ 80 |
| `entryPoint` * | string | `^[a-zA-Z0-9_./-]+\.html$`, `/` ile başlamaz, `.`/`..`/boş parça yok, ≤ 240 |
| `capabilities` * | string[] | Yalnızca `rooms`, `documents`, `notifications`, `jobs`, `voice`; tekrar yok |
| `room` * | object | Her zaman verilmeli |
| `room.kind` | string | `workspace` \| `turn_based` \| `realtime` |
| `room.maxParticipants` | integer | 1 – 128 |
| `room.spectators` | boolean | İzleyici rolüne izin |
| `backend.mode` * | string | `managed` \| `external` |
| `backend.endpoint` | string | Yalnızca `external`'da zorunlu: `wss://…`; origin'i `networkOrigins`'te olmalı |
| `networkOrigins` | string[] | ≤ 10; **yalnızca origin** (`https://` / `wss://`, path/query/hash yok); tekrar yok |

`*` = zorunlu

### Yetenekler

| Yetenek | Açtığı metotlar |
|---|---|
| `documents` | `documents.get`, `documents.update` |
| `rooms` | `rooms.list/create/join/leave/close/delete/command` |
| `jobs` | `jobs.list/create/delete` |
| `notifications` | `notifications.send` |
| `voice` | `voice.join`, `voice.leave` |

`context.get` her zaman kullanılabilir.

### networkOrigins ne işe yarar? ✅

Listelenen origin'ler CSP'de **yalnızca `connect-src`**'ye eklenir. Yani `fetch`, `XMLHttpRequest`, `WebSocket` ve service worker'daki `fetch` bu adreslere gidebilir.

> 🚫 Origin eklemek `frame-src`, `script-src`, `img-src` veya `media-src`'yi genişletmez. YouTube embed'i bu yüzden çalışmaz.

### Sık yapılan hatalar

- `"https://ornek.com/"`: sondaki `/` yasaktır (path sayılır).
- `"/index.html"`: entryPoint `/` ile başlayamaz.
- `version`'ı artırmamak: yayınlanmış bir sürüm değiştirilemez.
- `"document"` yazmak: doğrusu çoğul, `documents`.

---

## 5. Çalışma ortamı (iframe ve CSP)

### iframe ✅

```html
<iframe src="https://<APP_ID>.jubbio.net/assets/<APP_ID>/<version>/index.html"
        sandbox="allow-scripts allow-forms allow-same-origin"
        referrerpolicy="no-referrer"></iframe>
```

- **Origin:** `https://<APP_ID>.jubbio.net`, yani her uygulamanın kendi alt alan adı var.
- **`allow-same-origin`:** Host bunu yalnızca entryUrl https ve Jubbio'dan farklı bir origin ise ekler (üretimde öyle). Bu sayede `localStorage`, `crypto.randomUUID` ve service worker çalışır.
- **`allow` özniteliği yok:** autoplay, fullscreen ve picture-in-picture izinleri devredilmez.
- **`allow-popups` yok:** `window.open` ve `target="_blank"` çalışmaz. `allow-top-navigation` da yok.

### CSP ✅ (Ekim 2026'da ölçüldü)

```
default-src 'none';
script-src 'self' 'wasm-unsafe-eval';
style-src 'self' 'unsafe-inline';
img-src 'self' data: blob: https://cdn.jubbio.com;
font-src 'self' data:;
connect-src wss://gateway.jubbio.com <manifest.networkOrigins>;
worker-src 'self' blob:;
media-src 'self' blob:;
object-src 'none';
base-uri 'none';
form-action 'none'
```

| İstediğin | Durum | Not |
|---|---|---|
| Kendi JS/CSS dosyaların | ✅ | `'self'` |
| Satır içi `<style>` / `style=""` | ✅ | `'unsafe-inline'` yalnızca stil için |
| Satır içi `<script>`, `onclick=""`, `eval` | ❌ | Tüm JS ayrı dosyada olmalı |
| CDN'den script | ❌ | Kütüphaneleri bundle'a koy |
| WebAssembly | ✅ | `.wasm` dosyası bundle'da olmalı |
| Dış font (Google Fonts) | ❌ | Fontları bundle'a koy |
| Dış resim | ❌ | Yalnızca bundle, `data:`, `blob:`, `cdn.jubbio.com` (avatarlar) |
| Dış resim (çözüm) | ✅ | networkOrigins'ten `fetch` → `blob:` URL |
| `<iframe>` (YouTube, Twitch…) | ❌ | `frame-src` yok |
| Dış video doğrudan `<video src>` | ❌ | `media-src 'self' blob:` |
| Dış video (çözüm) | ✅ | Service worker proxy veya `blob:` |
| Kendi API'ne fetch / WebSocket | ✅ | Origin'i networkOrigins'e ekle + CORS |
| Web Worker / Service Worker | ✅ | `worker-src 'self' blob:` |
| `<form action>` | ❌ | `form-action 'none'`; `fetch` kullan |

### Dış medya: service worker proxy

`media-src` dış videoyu engeller, ama `connect-src` networkOrigins'e açık ve `worker-src 'self'` izinli. Bu ikisini birleştiren bir service worker, videoyu **aynı origin'den geliyormuş gibi** sunar. Range istekleri iletildiği için akış ve sarma çalışır.

**sw.js** (bundle kökünde)

```js
self.addEventListener("install", () => self.skipWaiting());
self.addEventListener("activate", (e) => e.waitUntil(self.clients.claim()));

self.addEventListener("fetch", (e) => {
  const u = new URL(e.request.url);
  if (u.origin !== self.location.origin || !u.pathname.endsWith("/__media")) return;
  const target = u.searchParams.get("u");
  if (target) e.respondWith(proxy(e.request, target));
});

async function proxy(req, target) {
  const headers = {};
  const range = req.headers.get("range");
  if (range) headers.Range = range;                     // sarma için şart
  const r = await fetch(target, { headers, mode: "cors", credentials: "omit" });
  const h = new Headers();
  for (const k of ["content-type", "content-length", "content-range", "accept-ranges"]) {
    const v = r.headers.get(k);
    if (v) h.set(k, v);
  }
  return new Response(r.body, { status: r.status, statusText: r.statusText, headers: h });
}
```

**Sayfada kullanım**

```js
await navigator.serviceWorker.register("sw.js", { scope: "./" });
if (!navigator.serviceWorker.controller) {
  await new Promise((r) => navigator.serviceWorker.addEventListener("controllerchange", r, { once: true }));
}
video.src = "__media?u=" + encodeURIComponent("https://res.cloudinary.com/demo/video/upload/dog.mp4");
```

**Video sunucusunun göndermesi gereken başlıklar**

```
Access-Control-Allow-Origin: *                (veya https://<APP_ID>.jubbio.net)
Access-Control-Allow-Headers: Range
Access-Control-Expose-Headers: Content-Range, Accept-Ranges, Content-Length
Accept-Ranges: bytes
```

- **Cloudinary** (`https://res.cloudinary.com`) bu başlıkları varsayılan olarak gönderiyor (başlıklar kontrol edildi).
- **Cloudflare R2**'de aynısı bucket'ın CORS politikasıyla ayarlanabilir.
- Google Drive ve Dropbox linkleri uygun değil.
- Service worker kurulamazsa dosya `fetch` ile tamamen indirilip `URL.createObjectURL(blob)` ile oynatılabilir; büyük dosyalarda belleği doldurur.

### Otomatik oynatma, tam ekran, pencereler

- **Sesli otomatik oynatma** genelde engellenir. Kullanıcıya bir kez "▶ Katıl" butonu tıklat. Engellenirse `muted` başlat ve "Sesi aç" butonu göster.
- **Tam ekran** çalışmaz. `document.fullscreenEnabled` `false` ise butonu gizle.
- **Yeni pencere** açılamaz. Kullanıcının kopyalayabileceği bir link göster.
- **Pano** (`navigator.clipboard`) engellenebilir. `execCommand("copy")` yedeği koy.

### Depolama

- `localStorage` ve IndexedDB çalışır, ama tarayıcı bunları bölümlenmiş (partitioned) tutar ve her an temizlenebilir. Yalnızca kişisel tercihler için kullan.
- Kullanıcılar arasında paylaşılan veriler için `documents` kullan.

### Module script uyarısı

`<script type="module">` CORS modunda yüklenir. Origin `"null"` olursa engellenebilir. En sorunsuz yol klasik `<script src>` ya da tek dosyalık bir bundle.

---

## 6. Host protokolü

### El sıkışma ✅

1. Uygulama üst pencereye `{ type: "jubbio.app.hello", v: 1, nonce }` yazar. `nonce` küçük harfli bir UUID olmalı (`^[a-f0-9-]{36}$`). Mesaja port eklenmez.
2. Host bir `MessageChannel` açar ve `{ type: "jubbio.app.port", v: 1, nonce }` ile cevap verir. `event.ports[0]` artık senin portundur.
3. Sonraki tüm iletişim bu port üzerinden yapılır.

```js
const nonce = crypto.randomUUID();
const hello = () => parent.postMessage({ type: "jubbio.app.hello", v: 1, nonce }, "*");

addEventListener("message", (e) => {
  const d = e.data;
  if (e.source !== parent || d?.type !== "jubbio.app.port" || d.v !== 1 || d.nonce !== nonce) return;
  clearInterval(timer);
  const port = e.ports[0];
  port.onmessage = (ev) => handle(ev.data);
  port.start();
});

hello();
const timer = setInterval(hello, 1200);   // host hazır olana kadar tekrar et
```

**Dikkat edilecekler:**
- **Hello'yu tekrar et.** Host, launch bitmeden gelen hello'ları yok sayar; ~1–1,5 sn aralıkla tekrar gönder.
- **Hız sınırı:** host saniyede en fazla 5 hello kabul eder.
- **Doğrulama:** `e.source === window.parent` ve `nonce` eşleşmesini kontrol et.
- **Yeniden bağlanma:** Yeni bir hello'ya yeni port verilir ve host eskisini kapatır. Eski porttaki bekleyen istekleri `DISPOSED` say.

### Mesaj biçimleri

```json
// istek (uygulama → host)
{ "v": 1, "type": "request", "id": "r1", "method": "documents.get", "params": { "scope": "channel", "key": "state" } }

// başarılı cevap (result undefined ise alan hiç gelmez)
{ "v": 1, "type": "response", "id": "r1", "result": { "value": null, "version": 0 } }

// hata
{ "v": 1, "type": "response", "id": "r1", "error": { "code": "BAD_REQUEST", "message": "BAD_REQUEST" } }

// olay (host → uygulama)
{ "v": 1, "type": "event", "name": "document.changed", "data": { "scope": "channel", "key": "state", "version": 3 } }
```

### İstek sınırları 🔎

| Kural | Değer |
|---|---|
| `id` | String, 1–128 karakter; aynı id cevap gelmeden tekrar kullanılamaz |
| `params` | **Düz obje zorunlu** (dizi ya da null olamaz); parametre yoksa `{}` |
| JSON boyutu | ≤ 64 KB |
| Derinlik / düğüm | Derinlik ≤ 16, toplam düğüm ≤ 8192 |
| Yasak anahtarlar | `__proto__`, `prototype`, `constructor` |
| Hız | Saniyede ≤ 100 istek, aynı anda ≤ 32 açık istek → `RATE_LIMITED` |

Host isteği doğrular, sonra kısa ömürlü bir ticket ile `https://gateway.jubbio.com/api/v1/apps/<yol>` adresine iletir. **Parametreler sunucuya olduğu gibi gider**; şekil kontrolü sunucuda yapılır.

> ⚠️ **Hata ayrıntısı gelmez.** Host sunucunun hata metnini iletmez; yalnızca kodu iletir. Yanlış bir parametre şekli sadece `BAD_REQUEST` olarak görünür.

---

## 7. Metotlar

| Metot | Yetenek | Gateway yolu | Durum |
|---|---|---|---|
| `context.get` | — | (yerel) | ✅ |
| `documents.get` | documents | `POST documents/get` | ✅ |
| `documents.update` | documents | `POST documents/update` | ✅ |
| `rooms.list` | rooms | `GET rooms` | ✅ |
| `rooms.create` | rooms | `POST rooms` | ❓ |
| `rooms.join` | rooms | `POST rooms/join` | 🔎 |
| `rooms.leave` | rooms | `POST rooms/leave` | 🔎 |
| `rooms.close` / `rooms.delete` | rooms | `POST rooms/close` / `rooms/delete` | ❓ |
| `rooms.command` | rooms | `POST rooms/command` | ❓ |
| `jobs.list` / `jobs.create` | jobs | — | ❓ |
| `jobs.delete` | jobs | `DELETE jobs/{jobId}` | 🔎 |
| `notifications.send` | notifications | — | ❓ |
| `voice.join` / `voice.leave` | voice | — | ❓ |

### context.get

Yetenek gerektirmez. Örnek cevap:

```json
{
  "applicationId": "600000000000000001",
  "applicationVersion": "1.0.2",
  "installationId": "3f1c2a9e-0000-4000-8000-000000000000",
  "guildId": "600000000000000002",
  "channelId": "600000000000000003",
  "user": {
    "id": "42",
    "displayName": "Ayşe",
    "username": "ayse",
    "avatarUrl": "https://cdn.jubbio.com/users/42/avatars/….webp"
  },
  "capabilities": ["documents", "rooms"],
  "permissions": { "canUse": true, "canManage": true },
  "locale": "tr",
  "theme": "dark"
}
```

| Alan | Açıklama |
|---|---|
| `applicationId` / `applicationVersion` | Uygulama ve kanalın bağlı olduğu sürüm |
| `installationId` | Bu sürümün sunucudaki kurulumu (uuid) |
| `guildId`, `channelId` | Sunucu ve kanal |
| `user` | `id`, `displayName`, `username`, `avatarUrl` |
| `capabilities` | Kurulumun izin verdiği yetenekler |
| `permissions` | `canUse`, `canManage`: kullanıcının bu uygulamadaki yetkisi |
| `locale`, `theme` | Ör. `tr`, `dark` / `light` |
| `initialRoom` | Kanal bir odayla açıldıysa eklenir |

Context değişirse `context.changed` olayı gelir. `permissions.canManage` ile "yalnızca yöneticiler değiştirebilsin" gibi kurallar koyabilirsin; ama bu kontrol yalnızca istemci tarafındadır.

### rooms.*

- **`rooms.list`:** `{}` gönderilir, sonuç bir dizidir. Yeni kanalda `[]` döndü ✅.
- **`rooms.join`:** `{ roomId: string, role: "player" | "spectator" }`. Host bu ikisini kendisi doğrular, uymazsa `BAD_REQUEST`. Odanın sürümü farklıysa host kanalı o sürümle yeniden açar.
- **`rooms.leave`:** `{ roomId }`.
- **`rooms.command`:** Önce `rooms.join` gerekir; yoksa `PERMISSION_DENIED` ("Join room first").
- `rooms.create`, `rooms.close`, `rooms.delete` ve `rooms.command` parametreleri henüz çözülmedi.

> ⚠️ **Kanal modunda oda durumu gelmez.** GUI uygulama kanalında host, odanın `publicState`/`privateState` snapshot ve delta mesajlarını uygulamaya **iletmiyor**. Ortak durum için `documents` kullan.

### jobs.*, notifications, voice

- `jobs.delete`: `{ jobId }`. `jobId` bir UUID olmalı; host bunu doğrular.
- `voice.join` kullanıcıya bir onay penceresi açar.
- Diğer metotların parametreleri henüz denenmedi.

---

## 8. Documents (ortak durum)

Kanaldaki herkesin okuyup yazabildiği, sürüm numarasıyla korunan anahtar-değer dokümanlarıdır.

> ✅ **Gerçek ortamda doğrulandı (Ekim 2026).** Otomatik bir "probe" ile 30'dan fazla aday parametre denendi. Kabul edilen **tek** şekil aşağıdakidir.

### documents.get

| Param | Tip | Açıklama |
|---|---|---|
| `scope` * | `"channel"` | Çalıştığı doğrulanan tek değer; veri kanala özeldir |
| `key` * | string | Doküman adı, ör. `"state"`; bir kanalda birden çok anahtar olabilir |

```js
const { value, version } = await jb.call("documents.get", { scope: "channel", key: "state" });
```

```json
{ "value": { "videoId": "abc", "playing": true }, "version": 7 }
```

Henüz yazılmamış bir anahtar için cevap `{ "value": null, "version": 0 }` olur.

### documents.update

Değeri **tamamen** değiştirir (patch/merge değildir). Yalnızca sürüm eşleşirse yazar.

| Param | Tip | Açıklama |
|---|---|---|
| `scope` * | `"channel"` | get ile aynı |
| `key` * | string | get ile aynı |
| `value` * | any (JSON) | Yeni değer |
| `expectedVersion` * | integer | Son okunan `version`; verilmezse `BAD_REQUEST` |

```js
const res = await jb.call("documents.update", {
  scope: "channel",
  key: "state",
  value: { videoId: "abc", playing: false },
  expectedVersion: 7,
});
// → { "value": { "videoId": "abc", "playing": false }, "version": 8 }
```

**Çalışmayan şekiller** (hepsi `BAD_REQUEST`):
- `{}`, `{ key }`, `{ documentId }`, `{ id }`, `{ name }`
- `{ data }`, `{ patch }`, `{ document }`, `{ content }`
- `{ scope, key, value }` (expectedVersion olmadan)
- `{ scope, key, data, expectedVersion }`

### Değişiklikleri dinlemek

Biri yazınca kanaldaki herkese `document.changed` olayı gelir. Olay **değeri içermez**:

```json
{ "key": "state", "scope": "channel", "version": 8 }
```

```js
jb.on("document.changed", async (d) => {
  if (d.scope !== "channel" || d.key !== "state" || d.version <= localVersion) return;
  const res = await jb.call("documents.get", { scope: "channel", key: "state" });
  apply(res.value, res.version);
});
```

Olay kanalı bazen geç bağlanıyor. 2–5 sn'de bir `documents.get` ile polling yapmak güvenli bir yedektir.

### Eşzamanlı yazma (iyimser kilit)

Doğru desen şudur: oku → değiştir → yaz → çakışırsa tekrar oku ve yeniden dene.

```js
let state = null;
let version = 0;

async function refresh() {
  const res = await jb.call("documents.get", { scope: "channel", key: "state" });
  state = res.value;
  version = res.version;
}

// fn(eskiDurum) → yeniDurum. Çakışmada en fazla 4 kez dener.
async function mutate(fn) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const next = fn(structuredClone(state));
    try {
      const res = await jb.call("documents.update", {
        scope: "channel", key: "state", value: next, expectedVersion: version,
      });
      state = res.value;
      version = res.version;
      return state;
    } catch (err) {
      if (err.code !== "VERSION_CONFLICT") throw err;
      await refresh();
    }
  }
  throw new Error("Çok fazla çakışma");
}

await mutate((s) => ({ ...s, playing: true, updatedAt: jb.serverNow() }));
```

Sürüm uyuşmazlığında dönen kesin kod henüz bilinmiyor (❓); en olası olanı `VERSION_CONFLICT`. Hangi kod gelirse gelsin, refresh sonrası tekrar denemek güvenlidir.

### Tasarım önerileri

- **Tek doküman, küçük değer.** Boyut sınırı henüz bilinmiyor (`STORAGE_LIMIT_EXCEEDED` hata kodu bir sınır olduğunu gösteriyor); istek gövdesi zaten en fazla 64 KB olabilir.
- **Kendi işaretini koy.** Değere `app: "uygulama-adi"` ve bir `rev` sayacı eklemek eski ya da yabancı veriyi ayırt etmeyi kolaylaştırır.
- **Zamanı sunucu saatiyle yaz:** `updatedAt: jb.serverNow()`.
- **Veri kanala bağlıdır;** yeni kanal boş başlar.
- **Gelen veriye güvenme;** her okumada doğrula.

---

## 9. Olaylar ve sunucu saati

GUI uygulama kanalında host **yalnızca 5 olay** iletir:

| Olay | data | Sunucu kaynağı |
|---|---|---|
| `context.changed` | Yeni context | host (yeniden launch) |
| `document.changed` | `{ scope, key, version }` ✅ | `document_changed` |
| `rooms.changed` | ❓ | `rooms_changed` |
| `lifecycle.changed` | `{ connection: "connecting" \| "connected" \| "disconnected" }` | `status` |
| `clock.updated` | `{ offsetMs, roundTripMs }` ✅ | `clock` |

Host bu olayları sunucudan ayrı bir olay kanalı üzerinden alır (launch cevabındaki `eventEndpoint`).

### lifecycle.changed

> ⚠️ **"connecting"e güvenme.** Durum uzun süre `"connecting"`'te kaldı, ama bu sırada istekler ve `document.changed` olayları sorunsuz çalıştı. Bağlantı göstergesini son isteğin başarısına göre belirle; `"disconnected"`'ı ise ciddiye al.

### clock.updated

Host sunucudan düzenli saat örnekleri alır, son 10 örnekten en düşük gecikmeliyi seçer ve saat farkını bildirir.

```json
{ "offsetMs": 2172, "roundTripMs": 56 }
```

**Sunucu zamanı ≈ `Date.now() + offsetMs`.** Kullanıcı saatleri saniyelerce farklı olabilir; denememizde fark 2,1 sn çıktı.

```js
let offsetMs = 0;
jb.on("clock.updated", (d) => { if (Number.isFinite(d.offsetMs)) offsetMs = d.offsetMs; });
const serverNow = () => Date.now() + offsetMs;

// Ortak durum: { playing, position (sn), updatedAt (sunucu ms) }
function expectedPosition(s) {
  if (!s.playing) return s.position;
  return s.position + (serverNow() - s.updatedAt) / 1000;
}

setInterval(() => {
  const exp = expectedPosition(state);
  if (Math.abs(video.currentTime - exp) > 1) video.currentTime = exp;
}, 1000);
```

---

## 10. Hata kodları

| Kod | Ne zaman? | Ne yapmalı? |
|---|---|---|
| `BAD_REQUEST` | Parametre şekli yanlış, zorunlu alan eksik, params obje değil | Parametreleri düzelt |
| `UNAUTHENTICATED` | Oturum/ticket geçersiz | Host bir kez yeniler; sürerse kanalı yeniden açtır |
| `PERMISSION_DENIED` | Yetenek yok, bilinmeyen metot, odaya katılmadan `rooms.command` | Manifeste ekle, yeni sürüm yayınla ve kur |
| `NOT_FOUND` | Kaynak yok, uygulama kaldırıldı | Durumu tazele |
| `VERSION_CONFLICT` | Kilit çakışması ya da kanalın sürümü değişti | Tekrar oku ve dene |
| `RATE_LIMITED` | > 100 istek/sn ya da > 32 açık istek | Yavaşla |
| `ROOM_EXPIRED` | Oda süresi doldu | Yeni oda |
| `ROOM_FULL` | `maxParticipants` aşıldı | İzleyici olarak katıl |
| `ROOM_BUSY` | Oda meşgul | Kısa süre sonra tekrar dene |
| `STORAGE_LIMIT_EXCEEDED` | Depolama sınırı aşıldı | Değeri küçült |
| `TIMEOUT` | Cevap gelmedi | Tekrar dene; art arda olursa yeniden el sıkış |
| `DISPOSED` | Port kapatıldı ya da kanal başka sürümle açılıyor | Yeni portu bekle |
| `SERVICE_UNAVAILABLE` | Diğer her şey | Geri çekilerek tekrar dene |

Host, sunucunun kodunu bilinen kodlar listesiyle karşılaştırır. Kod listedeyse hem `code` hem `message` o kod olur; değilse `code: "SERVICE_UNAVAILABLE"`, `message: "Application request failed"` olur. Sunucunun açıklayıcı hata metni uygulamaya **hiç ulaşmaz**.

---

## 11. Mini SDK

```js
const jb = new Jubbio();
await jb.connect();
const ctx = await jb.call("context.get");
jb.on("document.changed", (d) => console.log("değişti", d));
jb.on("*", (name, data) => console.log("olay", name, data));
console.log("sunucu saati", new Date(jb.serverNow()));
```

| Üye | Açıklama |
|---|---|
| `connect(timeoutMs = 20000)` | El sıkışır; port gelince resolve olur, olmazsa `NO_HOST` ile reject |
| `call(method, params = {}, timeoutMs = 15000)` | İstek gönderir; hata olursa `err.code` ile reject |
| `on(name, fn)` | Olay dinler; aboneliği kaldıran bir fonksiyon döndürür. `"*"` tüm olayları dinler |
| `serverNow()` | `Date.now() + offsetMs` |
| `reconnect()` | Yeniden el sıkışır; art arda 3 zaman aşımında otomatik çağrılır |

**jubbio.js**

```js
// Jubbio GUI host köprüsü — topluluk sürümü. Resmi SDK değildir.
(function () {
  function uuid() {
    try { if (crypto.randomUUID) return crypto.randomUUID(); } catch (_) {}
    let b;
    try { b = crypto.getRandomValues(new Uint8Array(16)); }
    catch (_) { b = Uint8Array.from({ length: 16 }, () => (Math.random() * 256) | 0); }
    b[6] = (b[6] & 0x0f) | 0x40;
    b[8] = (b[8] & 0x3f) | 0x80;
    const h = Array.from(b, (x) => x.toString(16).padStart(2, "0")).join("");
    return `${h.slice(0, 8)}-${h.slice(8, 12)}-${h.slice(12, 16)}-${h.slice(16, 20)}-${h.slice(20)}`;
  }
  const rpcError = (code, message) => Object.assign(new Error(message || code), { code });

  class Jubbio {
    constructor() {
      this.port = null;
      this.nonce = null;
      this.pending = new Map();
      this.handlers = new Map();
      this.seq = 0;
      this.offsetMs = 0;
      this.context = null;
      this.connection = "connecting";
      this.timeouts = 0;
      this._helloTimer = null;
      this._waiters = [];
      addEventListener("message", (e) => this._onWindowMessage(e));
    }

    connect(timeoutMs = 20000) {
      if (parent === window) return Promise.reject(rpcError("NO_HOST", "Uygulama Jubbio içinde açılmalı"));
      this._startHello();
      return new Promise((resolve, reject) => {
        const t = setTimeout(() => reject(rpcError("NO_HOST", "Jubbio host bulunamadı")), timeoutMs);
        this._waiters.push(() => { clearTimeout(t); resolve(this); });
      });
    }

    reconnect() { this._startHello(); }

    _startHello() {
      this.nonce = uuid();
      clearInterval(this._helloTimer);
      const hello = () => parent.postMessage({ type: "jubbio.app.hello", v: 1, nonce: this.nonce }, "*");
      hello();
      this._helloTimer = setInterval(hello, 1200);
    }

    _onWindowMessage(e) {
      const d = e.data;
      if (e.source !== parent || !d || d.type !== "jubbio.app.port" || d.v !== 1) return;
      if (d.nonce !== this.nonce || !e.ports || !e.ports[0]) return;
      clearInterval(this._helloTimer);
      if (this.port) {
        try { this.port.close(); } catch (_) {}
        for (const [, p] of this.pending) { clearTimeout(p.t); p.reject(rpcError("DISPOSED", "Bağlantı yenilendi")); }
        this.pending.clear();
      }
      this.port = e.ports[0];
      this.port.onmessage = (ev) => this._onPortMessage(ev.data);
      this.port.start();
      this.timeouts = 0;
      this._emit("bridge.connected", {});
      this._waiters.splice(0).forEach((fn) => fn());
    }

    call(method, params = {}, timeoutMs = 15000) {
      if (!this.port) return Promise.reject(rpcError("SERVICE_UNAVAILABLE", "Host bağlantısı yok"));
      if (!params || typeof params !== "object" || Array.isArray(params)) params = {};
      const id = `r${++this.seq}`;
      return new Promise((resolve, reject) => {
        const t = setTimeout(() => {
          this.pending.delete(id);
          if (++this.timeouts >= 3) { this.timeouts = 0; this.reconnect(); }
          reject(rpcError("TIMEOUT", "İstek zaman aşımına uğradı"));
        }, timeoutMs);
        this.pending.set(id, { resolve, reject, t });
        this.port.postMessage({ v: 1, type: "request", id, method, params });
      });
    }

    on(name, fn) {
      if (!this.handlers.has(name)) this.handlers.set(name, new Set());
      this.handlers.get(name).add(fn);
      return () => this.handlers.get(name).delete(fn);
    }

    serverNow() { return Date.now() + this.offsetMs; }

    _emit(name, data) {
      const run = (set, ...args) => set && set.forEach((fn) => { try { fn(...args); } catch (err) { console.error(err); } });
      run(this.handlers.get(name), data);
      run(this.handlers.get("*"), name, data);
    }

    _onPortMessage(m) {
      if (!m || m.v !== 1) return;
      if (m.type === "response") {
        const p = this.pending.get(m.id);
        if (!p) return;
        this.pending.delete(m.id);
        clearTimeout(p.t);
        this.timeouts = 0;
        if (m.error) p.reject(rpcError(m.error.code || "SERVICE_UNAVAILABLE", m.error.message));
        else p.resolve(m.result);
      } else if (m.type === "event") {
        if (m.name === "clock.updated" && Number.isFinite(m.data?.offsetMs)) this.offsetMs = m.data.offsetMs;
        if (m.name === "context.changed") this.context = m.data;
        if (m.name === "lifecycle.changed" && m.data?.connection) this.connection = m.data.connection;
        this._emit(m.name, m.data);
      }
    }
  }

  window.Jubbio = Jubbio;
})();
```

---

## 12. Örnek: Ortak İzle

Kanaldaki herkesin aynı videoyu aynı anda izlediği bir "watch party" uygulaması.

> ⚠️ **Doğrulama durumu:** Ortak durum, senkron ve bağlantı Jubbio'da denendi (sürüm 1.0.2). MP4 oynatma (service worker proxy) ise yerelde, Jubbio CSP'sinin birebir kopyasıyla test edildi; gerçek Jubbio kanalında henüz denenmedi.

**Özellikler:**
- Link yapıştır → herkes aynı videoyu görür.
- Oynat, duraklat ve sarma ~1 sn içinde senkron olur.
- Sonradan giren kişi o anki konumdan başlar.
- Sıra (queue) var; video bitince sıradaki otomatik başlar.

**Neden YouTube değil?** YouTube'un `iframe_api` script'i `script-src-elem`, embed iframe'i ise `frame-src` ihlali yüzünden engellendi. Videolar bu yüzden kendi sunucundan (MP4) geliyor.

### Ortak durum

```json
{
  "app": "ortak-izle",
  "src": "https://res.cloudinary.com/demo/video/upload/dog.mp4",
  "playing": true,
  "position": 42.3,
  "rate": 1,
  "updatedAt": 1792427798111,
  "updatedBy": "42",
  "updatedByName": "Ayşe",
  "rev": 17,
  "queue": [{ "src": "https://…/film-2.mp4", "start": 0, "addedByName": "Ali" }]
}
```

Beklenen konum: `playing ? position + (serverNow() − updatedAt) / 1000 : position`

### Senkron algoritması

1. **Yerel eylem → doküman:** Konumu hesapla, `updatedAt = serverNow()` ile `mutate()` kullanarak yaz.
2. **Uzak değişiklik → oynatıcı:** `document.changed` ya da polling yeni bir sürüm getirince uzlaştır: kaynak farklıysa yükle, oynatma durumunu eşitle, kayma 1 sn'yi aşarsa sar.
3. **Drift döngüsü:** Saniyede bir uzlaştır. Art arda sarmayı önlemek için 2,5 sn bekleme koy.
4. **Video bitti:** Sıradakine geç ya da `playing: false` yaz. `rev` kontrolü çift yazmayı engeller.

> 💡 **Echo döngüsünü tasarımla önlemek.** Oynatıcının kendi kontrolleri kapalı ve üstünde şeffaf bir tıklama katmanı var. Tüm eylemler uygulamanın butonlarından geldiği için oynatıcı olaylarını kullanıcı eylemi diye yorumlamak gerekmiyor; böylece sonsuz döngü hiç oluşmuyor.

```js
function reconcile() {
  const s = state;
  if (!s?.src) return;
  if (player.src !== s.src) return player.load(s.src, expected(s), joined && s.playing);
  if (!joined) return;

  const exp = expected(s);
  const off = Math.abs(player.time() - exp);
  const canSeek = Date.now() - lastSeekAt > 2500;

  if (s.playing) {
    if (player.state === PLAYING) { if (off > 1 && canSeek) seek(exp); }
    else if (player.state !== BUFFERING) { if (off > 1 && canSeek) seek(exp); player.play(); }
  } else {
    if (player.state === PLAYING || player.state === BUFFERING) player.pause();
    if (off > 1 && canSeek) seek(exp);
  }
}
```

### Video barındırma

| Servis | Kurulum | Not |
|---|---|---|
| Cloudinary | Gerekmez; CORS ve Range açık | Ücretsiz planda video başına ~100 MB, trafik kotadan düşer |
| Cloudflare R2 | Bucket CORS politikası | Trafik ücretsiz, büyük dosyalar için ideal |
| Kendi sunucun | nginx/Caddy'de CORS başlıkları | Tam kontrol |

**Cloudflare R2 CORS politikası:**

```json
[{
  "AllowedOrigins": ["https://<APP_ID>.jubbio.net"],
  "AllowedMethods": ["GET", "HEAD"],
  "AllowedHeaders": ["Range"],
  "ExposeHeaders": ["Content-Range", "Accept-Ranges", "Content-Length", "ETag"],
  "MaxAgeSeconds": 86400
}]
```

**Videoyu uygun biçime çevirmek** (H.264 + AAC MP4, `faststart`):

```bash
ffmpeg -i girdi.mkv -c:v libx264 -c:a aac -movflags +faststart cikti.mp4
```

---

## 13. Yerel geliştirme

Jubbio'yu taklit eden bir "sahte host" sayfası şunları yapar:
- Uygulamayı bir iframe'de açar ve hello'ya port verir.
- `context.get` ve `documents.*` metotlarını `localStorage` üzerinde uygular.
- Değişiklikleri `BroadcastChannel` ile diğer sekmelere duyurur.
- `clock.updated` olayı gönderir.

**dev/host.html**

```html
<!doctype html>
<html lang="tr">
<body style="margin:0">
<iframe id="app" src="../dist/index.html" sandbox="allow-scripts allow-forms allow-same-origin"
        style="border:0;width:100vw;height:100vh"></iframe>
<script>
  const qs = new URLSearchParams(location.search);
  const user = { id: String(Math.random()).slice(2, 10), displayName: qs.get("user") || "Ayşe" };
  const KEY = "mock-doc";
  const bc = new BroadcastChannel(KEY);
  const frame = document.getElementById("app");
  let port = null;

  const read = () => JSON.parse(localStorage.getItem(KEY) || '{"version":0,"value":null}');
  const emit = (name, data) => port && port.postMessage({ v: 1, type: "event", name, data });
  bc.onmessage = (e) => emit("document.changed", { scope: "channel", key: "state", version: e.data });

  function handle(method, p) {
    if (method === "context.get")
      return { result: { applicationId: "dev", applicationVersion: "dev", user, capabilities: ["documents"], locale: "tr", theme: "dark" } };
    if (method === "documents.get") {
      if (p.scope !== "channel" || typeof p.key !== "string") return { error: { code: "BAD_REQUEST", message: "BAD_REQUEST" } };
      const d = read();
      return { result: { value: d.value, version: d.version } };
    }
    if (method === "documents.update") {
      if (p.scope !== "channel" || !("value" in p) || !Number.isInteger(p.expectedVersion))
        return { error: { code: "BAD_REQUEST", message: "BAD_REQUEST" } };
      const d = read();
      if (p.expectedVersion !== d.version) return { error: { code: "VERSION_CONFLICT", message: "VERSION_CONFLICT" } };
      const next = { version: d.version + 1, value: p.value };
      localStorage.setItem(KEY, JSON.stringify(next));
      bc.postMessage(next.version);
      setTimeout(() => emit("document.changed", { scope: "channel", key: "state", version: next.version }), 20);
      return { result: next };
    }
    return { error: { code: "PERMISSION_DENIED", message: "PERMISSION_DENIED" } };
  }

  addEventListener("message", (e) => {
    const d = e.data;
    if (e.source !== frame.contentWindow || d?.type !== "jubbio.app.hello") return;
    const ch = new MessageChannel();
    port = ch.port1;
    port.onmessage = (ev) => {
      const m = ev.data;
      const r = handle(m.method, m.params || {});
      setTimeout(() => port.postMessage({ v: 1, type: "response", id: m.id, ...r }), 50);
    };
    e.source.postMessage({ type: "jubbio.app.port", v: 1, nonce: d.nonce }, "*", [ch.port2]);
    emit("lifecycle.changed", { connection: "connected" });
  });
  setInterval(() => emit("clock.updated", { offsetMs: 0, roundTripMs: 40 }), 5000);
</script>
</body>
</html>
```

**Çalıştırma:**

```bash
npx serve .
# http://localhost:3000/dev/host.html?user=Ali  ve ikinci sekmede ?user=Ayşe
```

**Gerçek ortama yaklaştırmak için:**
- Yerel sunucu `dist/` dosyalarına Jubbio'nun CSP'sini başlık olarak eklesin.
- Cevaplara 50–150 ms gecikme koy.
- İlk ~1 sn'deki hello'ları yok say (hello tekrarı kodunu test etmiş olursun).
- Bir butonla `disconnected` olayı gönder.
- `dev/` klasörünü `dist/`'in dışında tut.

---

## 14. Bilinen sorunlar ve SSS

### Bilinen sorunlar

- **Kanalda sürüm değiştirme başarısız.** "Sürüm değiştirilemedi…" hatası alınıyor. Çözüm: yeni sürümle yeni bir uygulama kanalı oluştur.
- **lifecycle "connecting"te kalıyor.** İstekler yine de çalışıyor. Göstergeyi son isteğin başarısına göre belirle ve polling yedeği koy.
- **Hata mesajları ayrıntı içermiyor.** Yalnızca kod geliyor.
- **YouTube / iframe embed çalışmıyor.** CSP'de `frame-src` yok. Medya için service worker proxy kullan.

### Henüz bilinmeyenler ❓

- `documents` için `"channel"` dışındaki `scope` değerleri
- Doküman boyut sınırı
- Sürüm uyuşmazlığında dönen kesin hata kodu
- `rooms.create/close/delete/command` parametreleri ve `rooms.changed` olayının verisi
- `jobs.*`, `notifications.send`, `voice.*` parametreleri
- `backend.mode: "external"` protokolü
- Masaüstü ve mobil Jubbio uygulamalarındaki sandbox ve otomatik oynatma davranışı

### Nasıl bulduk?

1. **Keşif sürümü:** Uygulamaya bir debug paneli eklendi: context JSON'u, olay logu, `offsetMs` ve elle metot çağırmak için bir konsol.
2. **Probe:** Bir buton onlarca aday parametre şeklini sırayla denedi ve `BAD_REQUEST` dönmeyenleri listeledi. OK, `NOT_FOUND` ya da `VERSION_CONFLICT` dönmesi, şeklin doğru olduğuna işaretti.
3. **Sonuç:** `documents.get {scope:"channel", key:"state"}` tek OK'ti ve `{value:null, version:0}` döndü. Yazmada `value` + `expectedVersion` zorunlu çıktı.
4. **CSP ölçümü:** Sayfaya bir iframe ve dış script eklenip `securitypolicyviolation` olayı dinlendi:

```js
document.addEventListener("securitypolicyviolation", (e) => {
  console.log(e.violatedDirective, e.blockedURI);
  console.log(e.originalPolicy);   // sayfanın tam CSP'si
});
const f = document.createElement("iframe");
f.src = "https://www.youtube.com/embed/x";
document.body.appendChild(f);
```

### SSS

**Backend yazmam gerekir mi?**
Hayır. `managed` mod ve `documents` yeterli. Kendi API'ni çağırmak istersen origin'ini `networkOrigins`'e ekle.

**React / Vue / Svelte kullanabilir miyim?**
Evet. Build çıktısı statik olduğu sürece olur. Vite'ta `base: "./"` ayarla ve bağımlılıkları bundle'a koy.

**Kullanıcının kim olduğunu nasıl bilirim?**
`context.get` → `user.id`, `user.displayName`, `user.avatarUrl`. Avatarlar `cdn.jubbio.com`'da ve CSP bunlara izin veriyor.

**Veriler kimler tarafından görülebilir?**
`scope: "channel"` dokümanları kanala bağlıdır. Kanalda uygulamayı açabilen herkesin okuyup yazabildiğini varsay; hassas veri koyma.

**Bu dokümantasyon resmi mi?**
Hayır. Jubbio Devs topluluğu tarafından hazırlanmıştır ve Jubbio ile resmi bir bağı yoktur. Resmi dokümantasyon: https://jubbio.com/dev/docs

---

*Jubbio Devs topluluğu · Son doğrulama: Ekim 2026*
