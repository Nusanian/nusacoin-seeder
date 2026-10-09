# Nusacoin DNS Seeder

DNS seeder / network crawler untuk jaringan Nusacoin (NUX).

Node `nusacoind` yang baru pertama kali dijalankan butuh cara menemukan peer.
Seeder ini yang menjawabnya: ia rutin merangkak (crawl) jaringan P2P Nusacoin,
mencatat node mana yang aktif dan sehat, lalu menjawab query DNS dengan daftar
IP node sehat tersebut.

Software inti memakai [sipa/bitcoin-seeder](https://github.com/sipa/bitcoin-seeder)
(implementasi referensi open-source, lisensi MIT) yang dijalankan dengan
parameter jaringan Nusacoin — tanpa perubahan kode.

## Daftar seeder

| Hostname | IP VPS | Keterangan |
|---|---|---|
| `seed.nusachain.org` | `202.10.38.22` | Delegasi DNS via member komunitas |
| `seed.nusacoin.org` | `204.13.232.148` | VPS baru |

Dua seeder = redundansi: kalau satu mati, yang lain tetap menjawab.
Keduanya didaftarkan di `vSeeds` chainparams.

## Cara kerja singkat

1. Seeder konek ke node bootstrap, jabat tangan P2P (`version`/`verack`),
   lalu minta daftar peer (`getaddr`) — berulang secara paralel.
2. Node yang lolos uji (bisa dihubungi, bicara protokol Nusacoin di port 28573)
   masuk database; yang mati di-ban sementara.
3. Server DNS built-in (port 53) menjawab setiap query hostname seed
   dengan sampel acak ~25 alamat node sehat.

## Prasyarat (per seeder)

- VPS Ubuntu 22.04/24.04 dengan IP publik tetap
- Port 53 UDP **dan** TCP terbuka di firewall
- Delegasi DNS: `NS` record untuk hostname seed menunjuk ke server tersebut
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

Sesuaikan `-h`/`-n` dengan hostname mesin ini:

```bash
# Di 204.13.232.148:
./dnsseed \
  -h seed.nusacoin.org \
  -n ns1.nusacoin.org \
  -m admin.nusacoin.org \
  --p2port 28573 \
  --magic 4e555341 \
  -s 163.245.193.102 \
  -s 204.13.232.148 \
  -s 202.10.38.22

# Di 202.10.38.22 (ganti domainnya):
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
| `-h` | Hostname seed yang dijawab |
| `-n` | Hostname nameserver (harus resolve ke IP VPS ini) |
| `-m` | Email admin (ganti `@` dengan `.`) untuk SOA record |
| `--p2port 28573` | Port P2P Nusacoin |
| `--magic 4e555341` | Magic bytes jaringan = ASCII `"NUSA"` |
| `-s` | Node bootstrap (boleh IP, boleh diulang beberapa kali) |

Database (`dnsseed.dat`) tersimpan di working directory — jangan dihapus,
isinya hasil crawl yang terakumulasi. Flag `-s` hanya dibutuhkan saat
database masih kosong (pertama kali / setelah dihapus).

### 4. Jalankan sebagai service (systemd)

Sesuaikan `User`, path binary, dan flag `-h`/`-n`/`-m` di
`nusacoin-seeder.service` dengan mesin ini, lalu:

```bash
sudo cp nusacoin-seeder.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now nusacoin-seeder
sudo journalctl -u nusacoin-seeder -f
```

## Delegasi DNS

Di zone induk masing-masing, tambahkan:

```
# Zone nusacoin.org (untuk seeder di 204.13.232.148):
seed.nusacoin.org.   IN  NS  ns1.nusacoin.org.
ns1.nusacoin.org.    IN  A   204.13.232.148

# Zone nusachain.org (untuk seeder di 202.10.38.22):
seed.nusachain.org.  IN  NS  ns1.nusachain.org.
ns1.nusachain.org.   IN  A   202.10.38.22
```

Record A kedua (glue) wajib karena nameserver berada di dalam domain yang
didelegasikan. Propagasi biasanya beberapa menit hingga beberapa jam.

## Verifikasi

Setelah seeder berjalan dan merangkak beberapa menit:

```bash
# Query langsung ke seeder (bypass delegasi, untuk tes awal)
dig @204.13.232.148 seed.nusacoin.org +short
dig @202.10.38.22 seed.nusachain.org +short

# Setelah delegasi DNS aktif, query normal
dig seed.nusacoin.org +short
dig seed.nusachain.org NS +short
```

Respons normal: daftar 20–25 alamat IPv4. Dari Windows juga bisa:
`nslookup seed.nusacoin.org` di Command Prompt.

## Mendaftarkan seed ke nusacoind

Setelah kedua hostname live dan menjawab, daftarkan di
`src/chainparams.cpp` (fungsi `CreateMain`) pada repo
[TaobotX11/nusacoin](https://github.com/TaobotX11/nusacoin):

```cpp
vSeeds.emplace_back("seed.nusacoin.org");
vSeeds.emplace_back("seed.nusachain.org");
```

Perubahan ini masuk lewat pull request seperti biasa. Sampai saat itu,
node tetap bisa bootstrap manual dengan `-addnode` / `-seednode`.

## Operasional

- **Log**: seeder mencetak statistik tiap interval ke stdout/journal.
- **Mesin seeder offline**: node yang sudah berjalan tidak terpengaruh
  (mereka sudah punya peer). Seeder yang kembali online langsung lanjut
  dari `dnsseed.dat` — tanpa `-s` lagi, kecuali databasenya hilang.
- **Filter kualitas opsional**: `--minheight <n>` menolak node di bawah tinggi
  block tertentu; `--knownblock <hash>` mewajibkan node punya block tertentu
  di chain-nya.
- **Keamanan**: seeder hanya melakukan koneksi P2P keluar dan menjawab DNS;
  ia tidak memegang key/wallet apa pun.

## Lisensi

Mengikuti lisensi MIT dari `sipa/bitcoin-seeder`. File konfigurasi dan
panduan di repo ini juga MIT.
