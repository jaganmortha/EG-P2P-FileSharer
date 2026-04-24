# DROP — peer-to-peer file transfer

Direct, encrypted file transfer between two browsers. No file uploads, no accounts, no size limit. Single static HTML file — host it anywhere.

- **No relay for your data.** Files move over a WebRTC data channel directly between browsers. DTLS-SRTP (AES-GCM) encrypts every byte on the wire.
- **Optional second encryption layer.** Type a passphrase and your chunks get a second wrap of AES-GCM-256 keyed via PBKDF2-SHA256 (210 k iterations) before they ever leave your machine.
- **Parallel channels + backpressure-aware pipelining.** Four simultaneous data channels, 64 KB chunks, tuned buffer thresholds. Realistic throughput: 500 Mbps – 1.5 Gbps between fast endpoints.
- **Stream-to-disk for huge files.** Uses the File System Access API on supported browsers (Chrome/Edge/Opera desktop) — no RAM cap, multi-GB files work fine. Falls back to in-memory blob download elsewhere.
- **Mobile-friendly handshake.** Copy-paste a code on desktop, or scan the QR on mobile.
- **Zero build step.** One HTML file. Open it locally, or drop it on any static host.

---

## Quick start — use it

1. Open `index.html` in a modern browser (Chrome/Edge/Firefox/Safari, desktop or mobile).
2. One person picks **SEND**, drops in a file, optionally sets a passphrase, hits **generate invite**. They copy the invite code (or show the QR) and send it to their partner via any channel.
3. The other person picks **RECEIVE**, pastes the invite (or scans the QR), enters the same passphrase if one was set, hits **generate response**, and sends the response code back.
4. The sender pastes the response and hits **connect**. Connection goes direct; transfer begins.

---

## Deploy on GitHub Pages

1. Create a new repo, commit `index.html` to the root.
2. In **Settings → Pages**, set source to `main` branch, root folder.
3. Visit `https://<your-username>.github.io/<repo-name>/`.

That's it. Works the same on Netlify, Cloudflare Pages, Vercel, or `python3 -m http.server`.

### One-liner Netlify/CF Pages

Drag-drop the single `index.html` into Netlify Drop or Cloudflare Pages — it's live in seconds.

---

## How the "no server" claim actually works

Two things a browser literally cannot do without *some* help from the network:

1. **Signal.** Two browsers can't establish a P2P link without first exchanging a ~2 KB handshake (SDP offer/answer + ICE candidates). DROP does this by having **you** transport the handshake — pasting a code, or scanning a QR. No third party is involved in signaling.
2. **Discover a public address.** Browsers behind NAT don't know what IP they look like from the outside. DROP uses Google's and Cloudflare's **public STUN** servers to answer that question. STUN servers never see your files — they only answer "what address do you look like from my side?".

When a direct P2P link is impossible (some corporate networks, strict carrier-grade NAT), DROP falls back to a **TURN relay** (Metered's free `openrelay.metered.ca`). TURN *does* see encrypted packets — but because the WebRTC DTLS session is end-to-end between the two browsers, TURN cannot decrypt them. If even that is too much trust for you, you can remove the TURN config and accept that ~10–15 % of connections will fail with "failed to establish".

### Making it fully self-hosted

If you want zero third-party dependencies:

- Swap the CDN `<script>` tags (`lz-string`, `qrcode`, `html5-qrcode`) for vendored local copies.
- Replace the Google/Cloudflare STUN URLs with your own [coturn](https://github.com/coturn/coturn) server.
- Either remove the TURN block, or point it at your own coturn.

The app is single-file and has no build step, so editing `index.html` directly is the entire process.

---

## Security notes

- **Transport encryption is always on** (WebRTC mandate: DTLS-SRTP, AES-GCM-128 or AES-GCM-256 negotiated per connection).
- **App-layer encryption is optional.** If you add a passphrase, every chunk is encrypted independently with a fresh 96-bit IV before it hits the wire. Even if the DTLS session were compromised, contents stay sealed unless the passphrase is known to the attacker.
- **The passphrase is never transmitted.** Only a random salt is; key derivation happens on both sides from `PBKDF2(passphrase, salt, 210000, SHA-256)`.
- **The invite/response codes do not contain file data.** They contain the SDP offer/answer (which includes your public IP as discovered by STUN — if that's privacy-sensitive, use a VPN or Tor).
- **No analytics, no telemetry.** The only outbound requests are: to STUN/TURN for connection setup, and to CDNs for library/font loads (which you can vendor if you prefer).

### Threats this does *not* protect against

- A compromised browser or machine.
- A clipboard hijacker swapping your code during the copy/paste step (visual QR scanning is slightly safer).
- A recipient you shouldn't have sent the file to in the first place.

---

## Browser support

| Feature | Chrome | Firefox | Safari | Edge | Mobile |
| --- | --- | --- | --- | --- | --- |
| WebRTC data channels | ✅ | ✅ | ✅ | ✅ | ✅ |
| File System Access API (stream-to-disk) | ✅ | ❌ | ❌ | ✅ | Chrome Android: ✅ |
| Camera + QR scan | ✅ | ✅ | ✅ | ✅ | ✅ |
| AES-GCM + PBKDF2 (WebCrypto) | ✅ | ✅ | ✅ | ✅ | ✅ |

On Firefox/Safari, multi-GB files still work but accumulate in memory; realistic cap ~2 GB per file. Switch receiver to Chromium for no-cap streaming.

---

## Tuning

All knobs are constants at the top of the `<script>` block:

```js
const NUM_DATA_CHANNELS = 4;                       // parallel channels
const CHUNK_SIZE = 64 * 1024 - 256;                // per-chunk payload
const HIGH_WATER_MARK = 8 * 1024 * 1024;           // per-channel send buffer cap
const LOW_WATER_MARK  = 1 * 1024 * 1024;           // resume threshold
```

Raising `NUM_DATA_CHANNELS` to 8 can help on very high-bandwidth links at the cost of CPU. `CHUNK_SIZE` is capped by SCTP's per-message limit (~64 KB for Safari compat; Chromium will happily go higher if you know both sides are Chromium).

---

## License

MIT — do what you want with it.
