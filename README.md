# cfw2ray

Worker Cloudflare buat jalanin VLESS, Trojan, VMess, dan Shadowsocks langsung dari edge Cloudflare. Karena Worker sendiri nggak bisa handle UDP, trafik UDP-nya di-relay ke VPS lewat koneksi TCP atau WebSocket.

## Cara kerjanya

```mermaid
flowchart LR
    C[Client] -->|WSS / HTTPS| W[Cloudflare Worker / worker.js]
    W -->|TCP langsung| T[TCP target]
    W -->|TCP atau WebSocket/WSS| R[VPS UDP relay]
    R -->|UDP| U[UDP target]
    W -->|DNS UDP/53| D[DNS-over-HTTPS]
```

Singkatnya: client konek ke Worker via WebSocket/HTTPS, lalu Worker yang mutusin mau diteruskan ke mana — TCP langsung, DNS-over-HTTPS (untuk query DNS di port 53), atau diteruskan ke VPS relay kalau itu trafik UDP biasa.

## Setup `worker.js`

Buka `worker.js`, terus edit bagian `CONFIG` di paling atas:

```js
const CONFIG = Object.freeze({
    UUID: 'UUID-ANDA',
    SS_METHOD: 'aes-128-gcm',

    // Pilih salah satu:
    // TCP relay. Jika host dan port valid, opsi websocket diabaikan.
    // UDP_RELAY: { host: 'IP-VPS-ANDA', port: 8443 },

    // WebSocket relay:
    UDP_RELAY: {
        websocket: 'wss://relay.example.com/',
        // websocket: 'ws://IP-VPS-ANDA:8080/',
    },

    MAX_PROTOCOL_HEADER: 4096,
    MAX_TCP_PROXY_REPLAY: 4 * 1024 * 1024,
    REJECT_UDP_443: true,
    DNS_DOH_URLS: [
        'https://cloudflare-dns.com/dns-query',
        'https://dns.google/dns-query',
    ],
    DNS_DOH_TIMEOUT_MS: 5000,
});
```

Beberapa hal yang perlu diperhatikan:

- **`UUID`** — dipakai bareng buat VLESS, VMess, Trojan, sama Shadowsocks legacy. Bikin sendiri di [uuidgenerator.net](https://www.uuidgenerator.net/).
- **`UDP_RELAY.host` + `UDP_RELAY.port`** — memakai relay TCP biasa. Keduanya harus valid; jika keduanya diisi, `websocket` tidak digunakan.
- **`UDP_RELAY.websocket`** — memakai relay WebSocket dengan URL `ws://` atau `wss://`.
- **`SS_METHOD`** — samain juga sama method yang dipakai di client Shadowsocks kamu.

## Setup `wsudprelay.js`

`wsudprelay.js` adalah UDP relay melalui WebSocket dan tidak membutuhkan package npm. Jalankan di VPS dengan Node.js versi 18 ke atas:

```bash
node wsudprelay.js
```

Konfigurasi default-nya:

```js
const CONFIG = Object.freeze({
    LISTEN_HOST: '0.0.0.0',
    LISTEN_PORT: 8080,
    WS_PATH: '/',
    MAX_WS_MESSAGE_BYTES: 4 * 1024 * 1024,
    HANDSHAKE_TIMEOUT_MS: 10000,
    IDLE_TIMEOUT_MS: 300000,
    XUDP_GRACE_MS: 60000,
    MAX_CONNECTIONS: 4096,
    REJECT_UDP_443: true,
});
```

Untuk koneksi tanpa TLS, gunakan:

```js
UDP_RELAY: {
    websocket: 'ws://IP-VPS-ANDA.sslip.io:8080/',
},
```

Untuk `wss://`, terminasi TLS di reverse proxy seperti Nginx, Caddy, atau Cloudflare Tunnel, lalu teruskan WebSocket ke `wsudprelay.js`. Contoh URL Worker:

```js
UDP_RELAY: {
    websocket: 'wss://relay.example.com/',
},
```

Nilai path pada URL harus sama dengan `WS_PATH`. Jika `WS_PATH` diubah menjadi `/udprelay`, gunakan `wss://relay.example.com/udprelay`.

## Setup `udprelay.js`

Ini yang jalan di VPS. Edit juga bagian `CONFIG`-nya:

```js
const CONFIG = Object.freeze({
    LISTEN_HOST: '0.0.0.0',
    LISTEN_PORT: 8443,
    HANDSHAKE_TIMEOUT_MS: 10000,
    IDLE_TIMEOUT_MS: 300000,
    XUDP_GRACE_MS: 60000,
    MAX_CONNECTIONS: 4096,
    REJECT_UDP_443: true,
});
```

Untuk relay TCP, `LISTEN_PORT` di `udprelay.js` harus sama dengan `UDP_RELAY.port` di Worker.

## Jalanin relay TCP

`udprelay.js` perlu jalan di VPS dengan Node.js versi 18 ke atas:

```bash
node udprelay.js
```

VPS-nya cukup buka akses TCP masuk di port `LISTEN_PORT`.

## Deploy Worker-nya

Deploy-nya lewat Cloudflare Dashboard:

1. Buka **Workers & Pages**.
2. Klik **Create application → Create Worker → Deploy**.
3. Klik **Edit code**.
4. Hapus kode contoh bawaan, ganti dengan seluruh isi `worker.js`.
5. Klik **Save and Deploy**.

Setelah itu Worker kamu udah bisa diakses lewat subdomain `workers.dev` bawaan, atau kalau mau pakai domain sendiri, tinggal tambahin di **Settings → Domains & Routes → Add → Custom Domain**.

## Cara pakai di client

Setup client-nya gampang: pakai alamat Worker kamu dan pilih transport WebSocket atau XHTTP mode stream-one

| Protokol | Path | Credential |
| --- | --- | --- |
| VLESS | `/vless` | UUID = `CONFIG.UUID` |
| Trojan | `/trojan` | Password = `CONFIG.UUID` |
| VMess | `/vmess` | ID = `CONFIG.UUID` |
| Shadowsocks | `/ss` | Password = `CONFIG.UUID` (method legacy) |

## Proxy IP

Proxy IP digunakan sebagai alamat alternatif untuk mengakses host atau IP Cloudflare yang tidak dapat diakses langsung oleh Cloudflare Worker.

Tambahkan `host:port` di belakang path:

```text
/vless/host:port
/trojan/host:port
/vmess/host:port
/ss/host:port
```

Misalnya:

```text
/vless/192.0.2.10:8080
```

## Yang perlu diketahui

- Query DNS di UDP port 53 otomatis dialihkan lewat DNS-over-HTTPS, jadi nggak lewat relay VPS.
- Relay WebSocket memakai path yang diatur oleh `WS_PATH`; default-nya `/`.
- Trafik UDP selain port 53 diteruskan ke VPS relay.
- UDP di port 443 bakal ditolak kalau `REJECT_UDP_443` diaktifkan.