---
name: lazycat-lpk-netdisk
description: >-
  Reusable workflow for LazyCat LPK apps needing local build/deploy plus LazyCat Drive open/save.
  懒猫网盘打开/保存、lzc-file-picker inject、COEP/ffmpeg.wasm 桥接、Blob 回写网盘。
  Use for lzc-manifest.yml, package.yml permissions, browser-only apps, lanmao wangpan integration.
---

# LazyCat LPK Netdisk

Use this skill when integrating LazyCat Drive into an LPK app, especially when the app is browser-only, uses `ffmpeg.wasm`, requires `crossOriginIsolated`, or needs to save a browser `Blob` back to LazyCat Drive.

Reply in Simplified Chinese unless the user asks otherwise.

## Core Decision

Prefer this route for COEP/`ffmpeg.wasm` apps:

```text
COEP main app + non-COEP picker bridge + IndexedDB Blob handoff + bridge-page PUT
```

Do not put LazyCat picker/inject directly inside the COEP main page if real-device testing shows picker hangs, modal freezes, or `postMessage`/iframe behavior is unreliable.

Do not remove main-page COEP if `ffmpeg.wasm` needs it.

Do not introduce server-side transcoding. Only add a backend save API when front-end `PUT /_lzc/files/home...` is proven blocked by permissions/API behavior.

## Required Reading

Before editing code, inspect the project files that define packaging, static serving, and service worker behavior:

- `lzc/lzc-manifest.yml`
- `lzc/lzc-build.yml`
- `lzc/package.yml`
- `lzc/images/nginx.conf`
- app entry HTML/JS, often `docs/index.html`
- `docs/service-worker.js`
- existing netdisk docs/test reports if present
- local LazyCat manual pages for inject/file picker/package permissions when available

Also inspect any known-good inject file in sibling repos, often:

```text
lzc/content/lazycat-injects/lzc-file-chooser-inject.js
```

## Packaging Baseline

For static apps packaged into nginx:

1. Put app static files into `docs/`.
2. Package them into `lzc/pack/docs.tar.gz`.
3. Use `lzc/images/Dockerfile` to copy/extract `docs.tar.gz` into nginx html root.
4. Add inject resources with:

```yaml
# lzc/lzc-build.yml
contentdir: ./content
```

5. Keep routes service-local, not FQDN:

```yaml
application:
  routes:
    - /=http://web:8080
```

## Permissions

If the app reads/writes user documents/media through LazyCat Drive, declare permissions in `lzc/package.yml`:

```yaml
permissions:
  required:
    - net.internet
    - document.read
    - document.write
    - media.read
    - media.write
```

Use a narrower set only when the app clearly does not need all of these. Missing `document.write`/`media.write` can make saves fail even when picker and paths look correct.

## M1: Inject + Blob Download POC

M1 is the lowest-cost test:

- Restore/copy `lzc/content/lazycat-injects/lzc-file-chooser-inject.js`.
- Add `application.injects` in `lzc/lzc-manifest.yml`.
- Configure:

```yaml
application:
  injects:
    - id: open-save-chooser
      on: browser
      when:
        - /*
      do:
        - src: file:///lzcapp/pkg/content/lazycat-injects/lzc-file-chooser-inject.js
          params:
            diskRoot: /_lzc/files/home
            fallbackMime: application/octet-stream
            locale: auto
            hooks:
              fileInput: true
              fileSystemAccess: true
```

For downloads, use `URL.createObjectURL(blob)` and `<a download>`; delay `URL.revokeObjectURL(url)`.

If M1 causes COEP/picker hang on real device, stop expanding M1 and switch to M2.

## M2: Non-COEP Picker Bridge

Use M2 for COEP apps.

### Manifest

Scope inject to the bridge page only:

```yaml
application:
  injects:
    - id: open-save-chooser
      on: browser
      when:
        - /lazycat-picker.html*
      do:
        - src: file:///lzcapp/pkg/content/lazycat-injects/lzc-file-chooser-inject.js
          params:
            diskRoot: /_lzc/files/home
            fallbackMime: application/octet-stream
            locale: auto
            hooks:
              fileInput: true
              fileSystemAccess: true
```

### nginx

Keep global COOP/COEP for the main app, but serve the bridge without COEP:

```nginx
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Embedder-Policy "require-corp" always;
add_header Cross-Origin-Resource-Policy "cross-origin" always;

location = /lazycat-picker.html {
    add_header Cross-Origin-Opener-Policy "same-origin" always;
    add_header Cross-Origin-Resource-Policy "cross-origin" always;
    try_files /lazycat-picker.html =404;
}
```

Do not remove main-page `Cross-Origin-Embedder-Policy: require-corp` when `ffmpeg.wasm` needs it.

### Opening Files

Main page:

1. Open `/lazycat-picker.html?mode=open&target=main|batch`.
2. Bridge creates `<lzc-file-picker type="file">`.
3. Bridge returns picker `stat` path.
4. Main page fetches `/_lzc/files/home + path`.
5. Main page constructs a browser `File`.
6. Call existing app file handler such as `handleFile(file)` or `handleBatchFiles(files)`.

Normalize picker paths:

```js
function normalizeLazyCatPath(path) {
  let normalized = String(path || '').trim().replace(/\.$/, '');
  normalized = normalized.replace(/^\/_lzc\/files\/home(?=\/|$)/, '');
  if (normalized && !normalized.startsWith('/')) normalized = '/' + normalized;
  return normalized;
}
```

### Saving Blob Outputs

Use this robust pattern when saving browser-generated Blob output:

1. Main page stores the output `Blob` in same-origin IndexedDB with a `saveId`.
2. Main page opens `/lazycat-picker.html?mode=save&saveId=...&filename=...`.
3. Bridge creates `<lzc-file-picker type="saveAs" choiceDirOnly>`.
4. Bridge receives picker stat and `saveName`.
5. Bridge reads the Blob from IndexedDB.
6. Bridge computes final path: `targetPath + saveName`.
7. Bridge performs `PUT /_lzc/files/home${path}`.
8. Bridge performs `GET /_lzc/files/home${path}` to verify size.
9. Bridge deletes the pending IndexedDB record.
10. Bridge reports result to main page.

Do not rely only on `window.opener.postMessage` from a non-COEP bridge to a COEP main page. Use all three channels and dedupe by `messageId`:

- `BroadcastChannel`
- `localStorage` storage event
- `window.opener.postMessage`

Expected success logs:

```text
Opening LazyCat Drive save picker for output.mp4...
LazyCat Drive save picker ready for output.mp4.
LazyCat Drive writing output.mp4 to /.../output.mp4...
Saved to LazyCat Drive: /.../output.mp4 (xxx KB)
```

If only the first log appears, bridge-to-main messaging or bridge execution is not reaching main; inspect bridge page console/network.

If the bridge logs a PUT/GET error, capture the exact HTTP status/body before changing architecture.

## Service Worker Rules

Bump cache names every time app shell or save/open flow changes.

Always bypass LazyCat files and future APIs:

```js
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);
  if (request.method !== 'GET') return;
  if (url.pathname.startsWith('/_lzc/') || url.pathname.startsWith('/api/')) {
    return;
  }
});
```

Never cache netdisk file streams in the PWA cache.

## Build And Deploy

From `lzc/`:

```powershell
tar -czf pack/docs.tar.gz -C ../docs .
lzc-cli project build
lzc-cli lpk install .\cloud.lazycat.app.<app>-v<version>.lpk
lzc-cli project info
```

When checking runtime files:

```powershell
lzc-cli project exec -s web grep "expected-string" /usr/share/nginx/html/index.html
```

## Acceptance Checklist

Verify:

- `lzc-cli project build` succeeds.
- `lzc-cli lpk install` succeeds.
- `lzc-cli project info` shows `Status_Running`.
- Main page still has `crossOriginIsolated === true` when required.
- Main app can load `ffmpeg.wasm` or equivalent COEP-dependent logic.
- LazyCat Drive open can select a small file and feed the existing handler.
- Local file open/download still works.
- LazyCat Drive save produces the expected four logs and the file appears in Drive.
- Service worker version changed and `/_lzc/` bypass is present.

## Escalation To M3

Switch to a backend save API only after M2 proves one of these:

- Front-end `PUT /_lzc/files/home...` returns a permission/API error even after `package.yml` permissions are present.
- Bridge page cannot access IndexedDB or fetch `/_lzc/files/home` in the target environment.
- Large Blob persistence in IndexedDB is unacceptable for the product.

M3 still must not do server-side transcoding unless explicitly requested. It should only save already-generated browser output.
