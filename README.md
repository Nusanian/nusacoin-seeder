# Nusacoin DNS Seeder

DNS seeder / network crawler untuk jaringan Nusacoin (NUX).

Node `nusacoind` yang baru pertama kali dijalankan butuh cara menemukan peer.
Seeder ini yang menjawabnya: ia rutin merangkak (crawl) jaringan P2P Nusacoin,
mencatat node mana yang aktif dan sehat, lalu menjawab query DNS dengan daftar
IP node sehat tersebut.

Software inti memakai [sipa/bitcoin-seeder](https://github.com/sipa/bitcoin-seeder)
(implementasi referensi open-source, lisensi MIT) yang dijalankan dengan
parameter jaringan Nusacoin — tanpa perubahan kode.

## Daftar seed host (per 27 Okt 2026)

| Hostname | IP | Model | Keterangan |
|---|---|---|---|
| `seed.nusachain.org` | `202.10.38.22` | Statis (A record) | Dikelola member komunitas |
| `seednx.taobot.org` | `204.13.232.148` | **Dinamis (seeder ini)** | VPS `vps3243047`, didelegasikan via NS |
| `seedns.taobot.org` | `163.245.193.102` | Statis (A record) | Dikelola tim |

Tiga hostname = redundansi: kalau satu mati, yang lain tetap menjawab.
Ketiganya didaftarkan di `vSeeds` chainparams (PR #21 ke `TaobotX11/nusacoin`).
Model statis: bila IP node berubah, pemegang DNS cukup mengubah angka IP
di record A (TTL 5 menit). Model dinamis: jawaban selalu node yang sedang
lolos uji crawler saat itu.

## Cara kerja singkat

1. Seeder konek ke node bootstrap, jabat tangan P2P (`version`/`verack`),
   lalu minta daftar peer (`getaddr`) — berulang secara paralel.
2. Node yang lolos uji (bisa dihubungi, bicara protokol Nusacoin di port 28573,
   tinggi bloknya di atas ambang `--minheight`) masuk set "good";
   yang mati di-ban sementara.
3. Server DNS built-in (port 53) menjawab setiap query hostname seed
   dengan sampel acak node good — maksimal setengah dari set good per query
   (minimal 1). Di jaringan kecil, jawaban bisa hanya 2–3 IP; itu normal,
   bukan kerusakan.

## Syarat node masuk daftar (kriteria "good")

Seeder hanya membagikan node yang lolos semua uji berikut (dari `IsGood()`
di source upstream). Satu gagal, tidak masuk daftar:

1. Bisa dihubungi dan handshake P2P berhasil — crawler konek TCP, tukar
   `version`/`verack`, lalu membaca tinggi blok dan servis node itu.
2. Port = port default P2P (28573). Node di port custom tidak masuk daftar,
   walau sehat.
3. Full node — flag servis `NODE_NETWORK` (menyediakan blok), bukan light
   client.
4. IP routable (publik) — IP privat seperti 192.168.x.x dicoret.
5. Versi protokol ≥ 70001 (node Nusacoin sekarang 70015/70016).
6. Tinggi blok ≥ `--minheight` (deployment ini: 70000).
7. Tidak sedang di-ban sementara — node yang berulang kali gagal diuji
   dicoret untuk sementara waktu.

### Bisakah node rumahan (non-VPS) ikut dibagikan?

Ya — sebagai peer, bukan sebagai seed host. Syaratnya: node full, port
28573 bisa dijangkau dari internet (port forwarding di router + firewall
dibuka), punya IP publik, dan node mengiklankan IP publiknya (bukan IP
lokal) supaya ditemukan lewat gossip antar-node (`getaddr`). Kalau koneksi
di balik CGNAT atau ISP memblokir port masuk, crawler tidak bisa konek
sehingga node tidak masuk daftar. Kalau IP publik berubah-ubah, entri lama
mati dan harus ditemukan ulang.

Menjalankan dnsseed sendiri (menjadi seed host) secara teknis bisa, tapi
seed host yang hanya online sesekali tidak berguna saat mesinnya mati —
karena itu seed host ditaruh di VPS yang hidup 24/7.

## Prasyarat (per seeder)

- VPS Ubuntu 22.04/24.04 dengan IP publik tetap
- Port 53 UDP **dan** TCP tidak diblokir firewall
- Delegasi DNS: `NS` record untuk hostname seed menunjuk ke server tersebut,
  plus glue `A` record untuk nameserver-nya
- Node bootstrap awal (port P2P 28573):
  - `163.245.193.102`
  - `204.13.232.148`
  - `202.10.38.22`

## Instalasi

### 1. Ambil binary (cara yang disarankan)

VPS kecil tidak perlu build sendiri. Binary dibangun lewat GitHub workflow
(repo ini, `.github/workflows/build.yml`, runner `ubuntu-22.04`) dan
diterbitkan sebagai release asset dengan URL stabil:

```bash
sudo mkdir -p /opt/nusacoin-seeder
curl -L -o /opt/nusacoin-seeder/dnsseed \
  https://github.com/Nusanian/nusacoin-seeder/releases/download/dnsseed-release/dnsseed
chmod +x /opt/nusacoin-seeder/dnsseed
```

Catatan build: runner butuh `libboost-dev` (upstream memakai satu header
Boost, `boost/algorithm/string/case_conv.hpp`). Tanpa itu build gagal.

### 1b. Alternatif: build dari source

```bash
sudo apt-get update
sudo apt-get install -y build-essential libboost-dev libssl-dev git

git clone https://github.com/sipa/bitcoin-seeder.git
cd bitcoin-seeder
make
```

Hasilnya satu binary: `./dnsseed`.

### 2. Pasang service (systemd)

File `nusacoin-seeder.service` di repo ini sudah berisi nilai deployment
taobot (hostname, bind, minheight). Untuk mesin lain, sesuaikan `-h`/`-n`/
`-m`/`-a`/`-s` dengan mesin tersebut, lalu:

```bash
curl -L -o /etc/systemd/system/nusacoin-seeder.service \
  https://raw.githubusercontent.com/Nusanian/nusacoin-seeder/main/nusacoin-seeder.service
systemctl daemon-reload
systemctl enable --now nusacoin-seeder
```

Isi service file (deployment taobot):

```ini
[Unit]
Description=Nusacoin DNS Seeder
After=network-online.target
Wants=network-online.target

[Service]
User=root
WorkingDirectory=/opt/nusacoin-seeder
ExecStart=/opt/nusacoin-seeder/dnsseed \
  -a 204.13.232.148 \
  -h seednx.taobot.org \
  -n ns1.taobot.org \
  -m admin.taobot.org \
  --p2port 28573 \
  --magic 4e555341 \
  --minheight 70000 \
  -s 163.245.193.102 \
  -s 204.13.232.148 \
  -s 202.10.38.22
Restart=always

[Install]
WantedBy=multi-user.target
```

Keterangan flag:

| Flag | Arti |
|---|---|
| `-a` | IP yang di-bind server DNS — **wajib eksplisit, lihat gotcha 1** |
| `-h` | Hostname seed yang dijawab |
| `-n` | Hostname nameserver (harus resolve ke IP VPS ini) |
| `-m` | Email admin (ganti `@` dengan `.`) untuk SOA record |
| `--p2port 28573` | Port P2P Nusacoin |
| `--magic 4e555341` | Magic bytes jaringan = ASCII `"NUSA"` |
| `--minheight 70000` | Tinggi blok minimum — **wajib, lihat gotcha 2** |
| `-s` | Node bootstrap (boleh IP, boleh diulang beberapa kali) |

Database (`dnsseed.dat`, `dnsseed.dump`, `dnsstats.log`) tersimpan di
working directory — jangan dihapus, isinya hasil crawl yang terakumulasi.
Flag `-s` hanya dibutuhkan saat database masih kosong (pertama kali /
setelah dihapus).

## Dua gotcha yang wajib diketahui

Keduanya ditemukan saat deploy 27 Okt 2026 dan terbukti dari source
upstream + eksperimen. Mengabaikan salah satunya = seeder tidak menjawab.

### Gotcha 1: bind wildcard bentrok dengan stub systemd-resolved

Default dnsseed bind ke `::` (dual-stack). Di Ubuntu, stub
`systemd-resolved` sudah menduduki `127.0.0.53:53`, sehingga bind `::` 
gagal (`EADDRINUSE`). Parahnya, thread DNS **mati diam-diam tanpa log**
(kode hanya return error; log "done" hanya berarti thread dibuat) —
crawler tetap jalan, service tetap `active`, tapi **tidak ada socket
dnsseed sama sekali** dan `dig` menjawab `connection refused`.

**Jangan** mematikan `systemd-resolved` untuk ini (merusak DNS mesin
sendiri). Solusinya: bind eksplisit ke IP publik VPS dengan flag `-a`:

```bash
# Cek IP publik dulu:
curl -s ifconfig.me

# Lalu pastikan service memakai -a <IP publik> dan restart.
# Verifikasi listener hidup:
ss -tulpn | grep dnsseed
```

### Gotcha 2: default minimum height adalah nilai Bitcoin

`IsGood()` mensyaratkan tinggi blok node `>= GetRequireHeight()`.
Tanpa flag, default mainnet = **350.000** (nilai Bitcoin). Tip Nusacoin
per Okt 2026 baru ~73.650 — jadi **semua** node gagal cek itu, set "good"
kosong selamanya, dan seeder menjawab otoritatif tapi `ANSWER: 0`
(NOERROR, flag `aa`, SOA benar — server hidup, datanya kosong).
Ini bukan soal "tunggu crawler lebih lama"; tanpa flag ini tidak akan
pernah menjawab.

Solusinya: `--minheight 70000` (di bawah tip, awet — node rantai aktif
selalu di atasnya). Perhatian: `--minheight 0` **tidak** berfungsi
(0 dianggap "tidak diatur" dan kembali ke 350.000).

## Delegasi DNS (Cloudflare, contoh taobot.org)

Urutan penting — glue dulu, baru delegasi:

```text
1. ns1.taobot.org.    IN  A   204.13.232.148   (proxy OFF)
2. Hapus record A seednx.taobot.org (bila ada)
3. seednx.taobot.org. IN  NS  ns1.taobot.org.  (proxy OFF)
```

Record A glue wajib karena nameserver berada di dalam domain yang
didelegasikan. Proxy wajib OFF untuk record delegasi dan glue.
Jangan flip sebelum seeder terbukti menjawab lewat query langsung
(lihat Verifikasi) — flip duluan = seed rusak selama seeder belum jalan.

## Verifikasi

```bash
# 1. Listener hidup?
ss -tulpn | grep dnsseed

# 2. Query langsung ke seeder (bypass delegasi, untuk tes awal)
dig @204.13.232.148 seednx.taobot.org
# Harapan: status NOERROR, flag aa, ANSWER SECTION berisi IP node,
# TTL jawaban 3600. ANSWER 0 + SOA = lihat gotcha 2.
# connection refused = lihat gotcha 1.

# 3. Setelah delegasi aktif, query lewat resolver publik
# (contoh via DNS-over-HTTPS Cloudflare)
curl -s "https://cloudflare-dns.com/dns-query?name=seednx.taobot.org&type=NS" \
  -H "accept: application/dns-json"
curl -s "https://cloudflare-dns.com/dns-query?name=seednx.taobot.org&type=A" \
  -H "accept: application/dns-json"
# Harapan: NS menjawab ns1.taobot.org; A menjawab IP node dengan TTL 3600
# (TTL 3600 = jawaban seeder; TTL 300 = sisa cache record A lama).
```

Dari Windows juga bisa: `nslookup seednx.taobot.org` di Command Prompt.

## Mendaftarkan seed ke nusacoind

Hostname didaftarkan di `src/chainparams.cpp` (fungsi `CreateMain`) pada
repo [TaobotX11/nusacoin](https://github.com/TaobotX11/nusacoin):

```cpp
vSeeds.emplace_back("seed.nusachain.org");
vSeeds.emplace_back("seednx.taobot.org");
vSeeds.emplace_back("seedns.taobot.org");
```

Perubahan ini masuk lewat pull request seperti biasa. Sampai saat itu,
node tetap bisa bootstrap manual dengan `-addnode` / `-seednode`.

## Operasional

- **Log**: `journalctl -u nusacoin-seeder -f`. Statistik crawl juga
  tertulis di `dnsstats.log` (format: timestamp + fraksi uptime node
  untuk jendela 2j/8j/1h/7h/30h — bukan jumlah node).
- **Mesin seeder offline**: node yang sudah berjalan tidak terpengaruh
  (mereka sudah punya peer). Seeder yang kembali online langsung lanjut
  dari `dnsseed.dat` — tanpa `-s` lagi, kecuali databasenya hilang.
- **Pindah ke VPS baru**: delegasi NS tidak perlu diubah (seednx tetap
  menunjuk ns1). Yang diubah hanya glue: record A `ns1` → IP VPS baru
  (propagasi ±5 menit karena TTL 300). Di VPS baru, pasang binary +
  service file dengan `-a` = IP baru, lalu start — crawler membangun
  database dari nol lewat node `-s`; jawaban pertama muncul dalam
  hitungan menit, daftarnya makin lengkap dalam hitungan jam. Selama
  jeda, seed statis tetap menjawab, jadi jaringan tidak kosong.
  (Kalau VPS baru mendapat IP yang sama dengan IP lama, tidak ada
  perubahan DNS sama sekali.)
- **Sinkronisasi file**: file service di repo adalah sumber kebenaran.
  Edit lokal di VPS (mis. via `sed`) WAJIB digabung kembali ke repo
  sebelum push berikutnya — kalau tidak, tarik-ulang berikutnya akan
  menghapus edit lokal tersebut.
- **Keamanan**: seeder hanya melakukan koneksi P2P keluar dan menjawab DNS;
  ia tidak memegang key/wallet apa pun.

## Lisensi

Mengikuti lisensi MIT dari `sipa/bitcoin-seeder`. File konfigurasi dan
panduan di repo ini juga MIT.
