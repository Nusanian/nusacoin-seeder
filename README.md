# Nusacoin DNS Seeder

DNS seeder / network crawler untuk jaringan Nusacoin (NUX).

Node `nusacoind` yang baru pertama kali dijalankan butuh cara menemukan peer.
Seeder ini yang menjawabnya: ia rutin merangkak (crawl) jaringan P2P Nusacoin,
mencatat node mana yang aktif dan sehat, lalu menjawab query DNS untuk
`seed.nusachain.org` dengan daftar IP node sehat tersebut.

Software inti memakai [sipa/bitcoin-seeder](https://github.com/sipa/bitcoin-seeder)
(implementasi referensi open-source, lisensi MIT) yang dijalankan dengan
parameter jaringan Nusacoin — tanpa perubahan kode.

## Cara kerja singkat

1. Seeder konek ke node bootstrap, jabat tangan P2P (`version`/`verack`),
   lalu minta daftar peer (`getaddr`) — berulang secara paralel.
2. Node yang lolos uji (bisa dihubungi, bicara protokol Nusacoin di port 28573)
   masuk database; yang mati di-ban sementara.
3. Server DNS built-in (port 53) menjawab setiap query `seed.nusachain.org`
   dengan sampel acak ~25 alamat node sehat.

## Prasyarat

- VPS Ubuntu 22.04/24.04 dengan IP publik tetap (saat ini: `202.10.38.22`)
- Port 53 UDP **dan** TCP terbuka di firewall
- Delegasi DNS: `NS` record untuk `seed.nusachain.org` menunjuk ke server ini
  (diatur oleh pemegang admin DNS `nusachain.org` — lihat bagian Delegasi DNS)
- Node bootstrap awal (port P2P 28573):
  - `163.245.193.102`
  - `204.13.232.148`
  - `202.10.38.22`

## Instalasi

### 1. Dependensi dan build

```bash
sudo apt-get update
sudo apt-get install -y build-essential libboost-all-dev libssl-dev git

git clone https://github.com/sipa/bitcoin-seeder.git
cd bitcoin-seeder
make
```

Hasilnya satu binary: `./dnsseed`.

### 2. Bebaskan port 53

Ubuntu menjalankan stub resolver `systemd-resolved` di port 53.
Seeder butuh port itu, jadi:

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

### 3. Jalankan seeder

```bash
./dnsseed \
  -h seed.nusachain.org \
  -n ns1.nusachain.org \
  -m admin.nusachain.org \
  --p2port 28573 \
  --magic 4e555341 \
  -s 163.245.193.102 \
  -s 204.13.232.148 \
  -s 202.10.38.22
```

Keterangan flag:

| Flag | Arti |
|---|---|
| `-h` | Hostname seed yang dijawab (`seed.nusachain.org`) |
| `-n` | Hostname nameserver (harus resolve ke IP VPS ini) |
| `-m` | Email admin (ganti `@` dengan `.`) untuk SOA record |
| `--p2port 28573` | Port P2P Nusacoin |
| `--magic 4e555341` | Magic bytes jaringan = ASCII `"NUSA"` |
| `-s` | Node bootstrap (boleh IP, boleh diulang beberapa kali) |

Database (`dnsseed.dat`) tersimpan di working directory — jangan dihapus,
isinya hasil crawl yang terakumulasi.

### 4. Jalankan sebagai service (systemd)

Sesuaikan `User`, path binary, dan flag di `nusacoin-seeder.service`,
lalu:

```bash
sudo cp nusacoin-seeder.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now nusacoin-seeder
sudo journalctl -u nusacoin-seeder -f
```

## Delegasi DNS (untuk pemegang admin `nusachain.org`)

Di zone `nusachain.org`, tambahkan:

```
seed.nusachain.org.   IN  NS  ns1.nusachain.org.
ns1.nusachain.org.    IN  A   202.10.38.22
```

Record kedua (glue) wajib karena nameserver berada di dalam domain yang
didelegasikan. Propagasi biasanya beberapa menit hingga beberapa jam.

## Verifikasi

Setelah seeder berjalan dan merangkak beberapa menit:

```bash
# Query langsung ke seeder (bypass delegasi, untuk tes awal)
dig @202.10.38.22 seed.nusachain.org +short

# Setelah delegasi DNS aktif, query normal
dig seed.nusachain.org +short
nslookup seed.nusachain.org
```

Respons normal: daftar 20–25 alamat IPv4. Dari Windows juga bisa:
`nslookup seed.nusachain.org` di Command Prompt.

## Mendaftarkan seed ke nusacoind

Setelah `seed.nusachain.org` live dan menjawab, daftarkan di
`src/chainparams.cpp` (fungsi `CreateMain`) pada repo
[TaobotX11/nusacoin](https://github.com/TaobotX11/nusacoin):

```cpp
vSeeds.emplace_back("seed.nusachain.org");
```

Perubahan ini masuk lewat pull request seperti biasa. Sampai saat itu,
node tetap bisa bootstrap manual dengan `-addnode` / `-seednode`.

## Operasional

- **Log**: seeder mencetak statistik tiap interval ke stdout/journal.
- **Filter kualitas opsional**: `--minheight <n>` menolak node di bawah tinggi
  block tertentu; `--knownblock <hash>` mewajibkan node punya block tertentu
  di chain-nya (mis. hash checkpoint terakhir).
- **Mulai ulang bersih**: hapus `dnsseed.dat` bila database korup; ia akan
  dibangun ulang dari node bootstrap.
- **Keamanan**: seeder hanya melakukan koneksi P2P keluar dan menjawab DNS;
  ia tidak memegang key/wallet apa pun.

## Lisensi

Mengikuti lisensi MIT dari `sipa/bitcoin-seeder`. File konfigurasi dan
panduan di repo ini juga MIT.
