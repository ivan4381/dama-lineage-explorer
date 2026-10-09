# DAMA ETL Lineage Explorer

Aplikasi untuk menelusuri alur data (lineage) Airflow: **DAG → task → script → tabel**.
Semuanya ada dalam satu file HTML — cukup buka `dama_lineage_explorer.html` di
browser (Chrome/Edge). Tidak perlu server, tidak perlu install apa pun.

**[Buka aplikasi](./dama_lineage_explorer.html)** (atau klik dua kali file-nya di komputer Anda).

## Fitur

- **Explore (panel kiri)**: cari DAG, task, bash runner, script, tabel, **atau nama
  owner** (cukup sebagian, misalnya `izuardo`). Hasil bisa disaring berdasarkan jenis
  entity, teknologi (SQL Server, Hive, BigQuery, Excel, MySQL, CSV, Azure SQL, SAP BW,
  Website, GCS, …) dan DAG tertentu. Tekan `/` untuk langsung ke kotak pencarian.
- **Lineage graph (tengah)**: pilih entity mana pun untuk melihat alur data
  hulu (upstream) dan hilirnya (downstream).
  - **Depth**: Direct, 2 hops, 3 hops, atau Full chain.
  - Zoom (`−` `+`, atau Ctrl + scroll), geser dengan drag, tombol **Back** untuk
    kembali ke entity sebelumnya.
  - **Full screen**: tampilkan graph satu layar penuh (tekan Esc untuk keluar).
  - **Fit to view**: pas-kan graph ke layar. Otomatis juga saat panel Details
    dibuka/ditutup.
- **Rows**: tabel data lineage mentah, bisa dicari dan per halaman.
- **Details (panel kanan)**: info entity yang sedang dipilih — DAG, task, script,
  tabel terkait, serta **Owner** dan **Task description**. Semua daftar bisa
  dibuka/ditutup (Expand all / Collapse all). Panel bisa disembunyikan dengan
  tombol `>` atau Ctrl + →.
- **Link yang bisa dibagikan**: entity yang dipilih tersimpan di alamat (URL), jadi
  link-nya bisa dikirim ke rekan untuk langsung membuka tabel/DAG yang sama.
- **Ask AI**: tanya jawab tentang lineage dengan Google Gemini (lihat bagian di bawah).
- Tema terang/gelap (mengikuti Windows, bisa diganti dengan tombol bulan di kanan atas).

## Ask AI (asisten AI)

Asisten ini memakai **API key Gemini milik Anda sendiri** — aplikasi tidak
menyediakan key.

**Cara mulai:**

1. Buka tab **Ask AI** (di sebelah tab *Rows*).
2. Di **AI settings**, tempel **Gemini API key** Anda (diawali `AIza…`).
3. Klik **Load models**, lalu pilih model (disarankan **gemini-2.5-flash**; versi
   *lite* lebih murah tapi sering kurang tepat).
4. Pilih entity di graph, lalu ketik pertanyaan. Contoh:
   - "Jelaskan flow DAG ini"
   - "Kalau tabel ini berubah, apa saja yang terdampak?"
   - "DAG mana saja yang menulis ke BigQuery?"
   - "Tampilkan DAG milik zaldi"

**Yang perlu diketahui:**

- AI otomatis melihat entity yang sedang dipilih. Untuk pertanyaan di luar entity
  itu, AI mencari sendiri di seluruh dataset.
- Data yang dikirim ke Google hanya **potongan yang relevan** (lineage entity yang
  dipilih dan hasil pencarian), bukan seluruh file.
- Setiap jawaban menampilkan jumlah token, misalnya `(2,261 tokens)`. Arahkan
  kursor ke angka itu untuk melihat rinciannya dan pencarian apa yang dilakukan AI.
- Klik **Clear** saat ganti topik — riwayat chat ikut terkirim di setiap
  pertanyaan, jadi membersihkannya menghemat token dan mencegah AI terbawa jawaban
  sebelumnya.

**Keamanan API key:**

- Key **tidak pernah disimpan di dalam file HTML**. Kalau file HTML dikirim ke orang
  lain, key Anda tidak ikut — mereka harus memakai key sendiri.
- Secara default key hilang saat tab browser ditutup. Centang **Remember key on this
  browser** agar tidak perlu menempel ulang (jangan dicentang di PC bersama).
  Tombol **Forget key** menghapusnya.
- Pastikan key Anda dari project GCP berbayar (paid tier), karena di free tier data
  bisa dipakai Google untuk melatih model.

## Data

Aplikasi sudah berisi **data bawaan (built-in)** di dalam file HTML-nya, diambil dari
[`Lineage DAG PPN.csv`](./Lineage%20DAG%20PPN.csv). Data
langsung tampil saat file dibuka.

Kolom yang dipakai (nama kolom tidak harus persis sama, variasi yang mirip tetap dikenali):

| Kolom | Arti |
|---|---|
| `DAG Name` | Nama DAG Airflow |
| `DAG Task Id` | Task di dalam DAG |
| `Bash File Name` | Bash runner yang menjalankan task |
| `Python Script File Name` | Script Python yang menjalankan proses ETL |
| `Source` / `Source Type` | Tabel/path yang dibaca dan teknologinya |
| `Target Table Path` / `Target Table Type` | Tabel/path yang ditulis dan teknologinya |
| `Owner` | Pemilik task (opsional) |
| `Task Description` | Penjelasan apa yang dilakukan task (opsional) |

### Melihat data lain (hanya untuk Anda)

Klik **Load → ETL Lineage** (atau drag & drop file ke halaman) dan pilih file
CSV/TXT/XLSX/XLS dengan kolom seperti di atas. File dibaca di browser Anda dan akan
diingat saat aplikasi dibuka lagi. Klik **Use built-in** untuk kembali ke data bawaan.

> Cara ini **hanya mengubah tampilan di browser Anda**. File HTML-nya tidak berubah,
> jadi orang lain yang membuka file yang sama tetap melihat data lama.

### Memperbarui data bawaan (untuk semua orang)

Tidak perlu install apa pun, cukup browser:

1. Buka file HTML, klik **Load → ETL Lineage**, lalu pilih file data terbaru.
2. Klik **Save as built-in** (muncul di sebelah nama dataset di bagian atas).
3. Browser akan men-download file `dama_lineage_explorer.html` baru yang sudah
   berisi data terbaru. **Ganti file HTML lama dengan file ini**, lalu bagikan (atau
   commit ke git).

Tips: lakukan dari jendela **Incognito/InPrivate** agar ekstensi browser (misalnya
Grammarly) tidak ikut tersimpan ke file hasil download.

<details>
<summary>Alternatif untuk developer: lewat command line (perlu Node.js)</summary>

Menanamkan CSV langsung ke HTML tanpa membuka browser. Script ini memakai parser CSV
yang sama dengan aplikasi, karena kolom deskripsi berisi koma di dalam tanda kutip.

```js
node -e "
const fs = require('fs');
const CSV = 'Lineage DAG PPN.csv', HTML = 'dama_lineage_explorer.html';
let html = fs.readFileSync(HTML, 'utf8');
const src = html.slice(html.indexOf('function parseCSV('), html.indexOf('function parseWorkbook('));
const parseCSV = new Function(src + '; return parseCSV;')();
const rows = parseCSV(fs.readFileSync(CSV, 'utf8')).filter(r => r.some(c => c.trim()));
const nl = html.includes('\r\n') ? '\r\n' : '\n';
const lines = html.split(/\r?\n/);
const idx = lines.findIndex(l => l.startsWith('const EMBEDDED = ['));
lines[idx] = 'const EMBEDDED = ' + JSON.stringify(rows).split('</').join('<' + String.fromCharCode(92) + '/') + ';'; // never close the <script> early
lines[idx + 1] = 'const EMBEDDED_NAME = ' + JSON.stringify(CSV) + ';';
fs.writeFileSync(HTML, lines.join(nl), 'utf8');
"
```

</details>

## Teknis

HTML/CSS/JavaScript biasa dalam satu file, tanpa proses build. Satu-satunya library
eksternal adalah `xlsx` (dimuat dari CDN, perlu internet) untuk membaca file Excel —
file CSV tetap bisa dibaca tanpa internet. Fitur Ask AI memanggil API Gemini langsung
dari browser, jadi perlu akses internet ke `generativelanguage.googleapis.com`.
