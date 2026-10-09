# BI Lineage Contract

| Item | Nilai |
|---|---|
| Versi kontrak | `1.0.0` |
| Status | Disepakati (dirilis 2026-10-08) |
| Produsen | TMDL Explorer (fitur **Batch Export**) |
| Konsumen | DAG Lineage Explorer (fitur **Load BI Lineage CSV**) |
| Lokasi dokumen | `docs/bi-lineage-contract.md` (salinan identik di kedua repo) |

> Analogi singkat: dokumen ini adalah **standar colokan**. TMDL Explorer membuat "steker", DAG Explorer menyediakan "stopkontak". Selama keduanya patuh pada standar ini, kedua repo boleh berkembang sendiri-sendiri tanpa saling mengubah kode.

---

## 1. Tujuan dan ruang lingkup

Menyambungkan lineage Airflow DAG (tabel target di DWH/BigQuery/Hive) dengan lineage Power BI (tabel model di semantic model) memakai file CSV sebagai perantara.

**Termasuk:** report Power BI di workspace Fabric, folder Power BI Report Server (PBIRS) on-prem, dan folder lokal hasil `pbi-tools extract`.

**Tidak termasuk:** paginated report, Looker / Looker Studio (akan menyusul sebagai `Platform` baru tanpa mengubah kolom), lineage level kolom.

## 2. Prinsip yang mengikat

1. **Additive.** Fitur baru tidak mengubah perilaku fitur lama di kedua tools. Batch Export hanya membaca hasil parser yang sudah ada (`reach()`, `M.sources`).
2. **Client-side, tanpa penyimpanan.** Semua file dibuat lewat download di browser dan dibaca lewat file picker. Tidak ada server, database, atau cache permanen.
3. **Tanpa hak admin tenant.** Tidak ada langkah yang butuh admin Fabric/Azure. PBIRS hanya diakses dari intranet lewat helper yang sudah ada.
4. **Tanpa kredensial di file.** Tidak ada token, password, atau connection string berisi kredensial di file mana pun dalam kontrak ini.
5. **`.pbix` tidak pernah disimpan.** File `.pbix` hanya transit di folder temp helper.
6. **Satu nilai per sel.** Banyak sumber = banyak baris. Tidak ada sel berisi daftar.

## 3. Artefak

| File | Format | Dibuat oleh | Dibaca oleh | Wajib |
|---|---|---|---|---|
| `bi_lineage_<YYYYMMDD-HHmm>.csv` | CSV | TMDL Explorer | DAG Explorer, TMDL Explorer (run berikutnya) | Ya |
| `bi_lineage_manifest.json` | JSON | TMDL Explorer | TMDL Explorer (run berikutnya) | Ya, untuk incremental |
| `bi_lineage_log_<YYYYMMDD-HHmm>.csv` | CSV | TMDL Explorer | Manusia | Ya |
| `bi_lineage_alias.csv` | CSV | Diisi manual oleh tim | DAG Explorer | Opsional |

## 4. Konvensi umum semua file CSV

| Aturan | Nilai |
|---|---|
| Encoding | UTF-8 **dengan BOM** (agar Excel membaca karakter dengan benar; sama dengan `Lineage_DAG_PPN.csv`) |
| Delimiter | Koma `,` |
| Quoting | RFC 4180: sel yang mengandung koma, kutip, atau baris baru dibungkus `"…"`; kutip di dalam sel ditulis `""` |
| Baris baru | Produsen menulis `CRLF`; konsumen wajib menerima `CRLF` maupun `LF` |
| Header | Baris pertama, nama dan urutan kolom **persis** seperti di dokumen ini |
| Nilai kosong | Sel benar-benar kosong. Jangan menulis `null`, `-`, `N/A` |
| Tanggal-waktu | ISO 8601 dengan offset, contoh `2026-10-08T14:05:00+07:00` |
| Boolean | `Ya` / `Tidak` |
| Kolom tambahan | Konsumen mengabaikan kolom tak dikenal yang ada **di ujung kanan** (lihat §11) |

---

## 5. Lineage CSV (`bi_lineage_<YYYYMMDD-HHmm>.csv`)

### 5.1 Grain (satuan baris)

Satu baris = **satu kombinasi unik** dari:

`Platform` + `Report ID` + `Tabel di Power BI` + `Query perantara` + `Objek Sumber`

Jika satu tabel model membaca 3 sumber, hasilnya 3 baris. Jika satu sumber dicapai lewat 2 query perantara berbeda, hasilnya 2 baris.

### 5.2 Kolom

| # | Kolom | Wajib | Nilai yang diizinkan | Keterangan |
|---|---|---|---|---|
| 1 | `Platform` | Ya | `Fabric` \| `PBIRS` \| `Lokal` | Asal report |
| 2 | `Lokasi` | Ya | teks | Fabric: nama workspace. PBIRS: path folder (mis. `/Subholding/Finance`). Lokal: path relatif folder ekstrak |
| 3 | `Report ID` | Ya | teks | Fabric: report GUID. PBIRS: CatalogItem `Id`. Lokal: path relatif folder ekstrak (sama dengan `Lokasi`) |
| 4 | `Report Name` | Ya | teks | Nama tampil report |
| 5 | `Model ID` | Tidak | teks | Fabric: semantic model GUID (`datasetId`). PBIRS: sama dengan `Report ID`. Lokal: kosong |
| 6 | `Versi Sumber` | Ya | teks | Nilai yang dipakai manifest untuk incremental (lihat §8.3) |
| 7 | `Tabel di Power BI` | Ya | teks | **Tabel model akhir** (yang muncul di Model view), bukan query staging |
| 8 | `Mode Tabel` | Ya | `import` \| `directQuery` \| `dual` \| `calculated` \| `calcGroup` \| `lainnya` | Diambil dari `Kind` parser |
| 9 | `Query perantara` | Kondisional | teks | Query yang **langsung memanggil connector**; query di antara tabel model dan query ini tidak dicatat (rantai lengkap bisa ditelusuri lewat `reach()` di TMDL Explorer). Kosong jika `Jalur` = `Langsung` |
| 10 | `Kedalaman` | Kondisional | bilangan bulat ≥ 0 | Jumlah lompatan dari tabel model ke query yang memanggil connector. `0` = tabel itu sendiri yang memanggil connector |
| 11 | `Jalur` | Ya | `Langsung` \| `Via query` \| `Tanpa sumber eksternal` \| `Tidak terbaca` | Lihat §5.3 |
| 12 | `Connector` | Kondisional | teks | Nama connector mentah dari parser (mis. `BigQuery`, `SQL Server`, `SharePoint`, `Folder`, `File`, `Web`) |
| 13 | `Source Type` | Kondisional | lihat §5.4 | Kosakata yang **sama** dengan kolom `Source Type` di Lineage DAG CSV |
| 14 | `Lokasi Sumber` | Kondisional | teks | Server/host, project BigQuery, URL site SharePoint, atau **full path** UNC untuk file/folder jaringan |
| 15 | `Objek Sumber` | Kondisional | teks | Nama objek **apa adanya** dari M/SQL (sebelum normalisasi). Untuk file: nama file atau pola filter |
| 16 | `Kunci Sumber` | Kondisional | teks lowercase | Kunci penyambung hasil normalisasi (lihat §6). Kosong jika bukan tabel database |
| 17 | `Status Kunci` | Ya | `Lengkap` \| `Dilengkapi dari koneksi` \| `Tidak lengkap` \| `Bukan tabel` | Kualitas `Kunci Sumber` |
| 18 | `Dari Native SQL` | Ya | `Ya` \| `Tidak` | `Ya` jika objek diambil dari teks SQL di `Value.NativeQuery` / opsi `Query=` |
| 19 | `Waktu Harvest` | Ya | ISO 8601 | Kapan baris ini dibuat |
| 20 | `Catatan` | Kondisional | teks | Wajib diisi jika `Jalur` = `Tidak terbaca`. Juga dipakai untuk penanda seperti `parameter: SourceDataPath` atau `path pribadi` |
| 21 | `Versi Kontrak` | Ya | `MAJOR.MINOR.PATCH` | Versi kontrak yang diimplementasikan produsen saat baris ditulis, mis. `1.0.0`. Baris yang disalin dari run sebelumnya (§8.4) mempertahankan nilainya |

### 5.3 Aturan konsistensi per `Jalur`

| `Jalur` | `Query perantara` | `Kedalaman` | `Connector` / `Source Type` / `Lokasi Sumber` / `Objek Sumber` | `Kunci Sumber` | `Catatan` |
|---|---|---|---|---|---|
| `Langsung` | kosong | `0` | terisi | terisi jika tabel database | opsional |
| `Via query` | terisi | ≥ `1` | terisi | terisi jika tabel database | opsional |
| `Tanpa sumber eksternal` | kosong | kosong | kosong | kosong | opsional (mis. `Enter Data`, `DAX calculated`) |
| `Tidak terbaca` | boleh terisi | boleh terisi | boleh terisi sebagian | kosong | **wajib** berisi alasan |

Contoh `Tanpa sumber eksternal`: tabel `Calendar` (DAX), `Measures Table` (Enter Data / `Binary.Decompress`), calculation group.

Contoh `Tidak terbaca`: connector dinamis yang path-nya dirakit saat runtime dan tidak bisa di-resolve, atau M yang gagal di-parse.

### 5.4 Kosakata `Source Type`

Disamakan dengan konvensi Lineage DAG CSV agar join dan filter konsisten.

| Kondisi di Power BI | `Source Type` | Isi `Lokasi Sumber` |
|---|---|---|
| `GoogleBigQuery.Database` | `BigQuery` | project ID |
| `Sql.Database` ke server on-prem | `SQL Server` | nama server |
| `Sql.Database` ke `*.database.windows.net` | `Azure SQL` | nama server |
| `SharePoint.Files` / `SharePoint.Contents` | `SharePoint` | URL site |
| `Folder.Files` / `File.Contents` berisi Excel di folder jaringan | `Excel` | **full path** UNC |
| `Csv.Document` dari folder jaringan | `CSV` | **full path** UNC |
| `Web.Contents` | `Website` | URL lengkap |
| `SapBusinessWarehouse.Cubes` | `SAP BW` | nama server/host SAP |
| Hive / Spark via ODBC | `Hive` | DSN atau host (tanpa kredensial) |
| Connector lain | nama connector mentah | sesuai connector |

### 5.5 Parameter dan path pribadi

- Jika lokasi dirakit dari parameter (mis. `SourceDataPath & "\Tax"`), tulis hasil resolve memakai **Current value** parameter, dan isi `Catatan` dengan `parameter: <NamaParameter>`.
- Jika hasil resolve berupa path pribadi (`C:\Users\…`, OneDrive pribadi), baris tetap ditulis dan `Catatan` diisi `path pribadi`.

---

## 6. Kunci penyambung dan normalisasi

### 6.1 Bentuk kunci

| Sistem | Bentuk | Contoh |
|---|---|---|
| SQL Server / Azure SQL | `database.schema.table` | `cntdimensional.sap.credit_limit_customer_cnt` |
| BigQuery | `project.dataset.table` | `data-governance-ppn.data_analytics_bronze.aviasi_fact_trx_sales_sap_zsddhavi_q001` |
| Hive | `database.table` | `rz_prd_cnt_ppn.tbl_digicracker_md_assignments` |

Nama server/host **tidak** termasuk dalam kunci.

### 6.2 Langkah normalisasi (urutan wajib)

Diterapkan **identik** di kedua tools, baik pada `Objek Sumber` (sisi BI) maupun `Target Table Path` dan `Source` (sisi DAG).

1. Pecah per titik `.` dengan menghormati pembungkus `[ ]`, backtick, dan kutip ganda `"` (titik di dalam pembungkus bukan pemisah).
2. Buang karakter `[`, `]`, backtick, dan `"` dari setiap bagian.
3. Trim spasi di awal dan akhir setiap bagian.
4. Lowercase setiap bagian.
5. Gabungkan kembali dengan `.`.

| Mentah | Hasil normalisasi |
|---|---|
| `[CntDimensional].[sap].[Credit_Limit_Customer_CNT]` | `cntdimensional.sap.credit_limit_customer_cnt` |
| `"CntDimensional"."sap"."Credit_Limit_Customer_CNT"` | `cntdimensional.sap.credit_limit_customer_cnt` |
| `` `data-governance-ppn.data_analytics_silver.GOLDEN_RECORD_SAP_FACT_lpg_daily` `` | `data-governance-ppn.data_analytics_silver.golden_record_sap_fact_lpg_daily` |
| ` sap . credit_limit_customer_cnt ` (DB dari koneksi = `CntDimensional`) | `cntdimensional.sap.credit_limit_customer_cnt` |

### 6.3 Melengkapi kunci yang kurang bagian

| Kondisi | Tindakan | `Status Kunci` |
|---|---|---|
| Sudah 3 bagian (atau 2 bagian untuk Hive) | Pakai apa adanya | `Lengkap` |
| 2 bagian, database diketahui dari koneksi (`Sql.Database(server, db)`) | Tambahkan `db.` di depan | `Dilengkapi dari koneksi` |
| 1–2 bagian, database tidak diketahui | Kunci tetap ditulis sebagian | `Tidak lengkap` |
| File, folder, URL, SharePoint | `Kunci Sumber` kosong | `Bukan tabel` |

Baris `Tidak lengkap` tetap ditampilkan di DAG Explorer, tetapi tidak ikut dicocokkan otomatis kecuali ada alias (§7).

Aturan yang sama berlaku di **sisi DAG**: kunci dari `Target Table Path` yang jumlah bagiannya kurang dari yang dibutuhkan `Target Table Type`-nya (3 bagian untuk `SQL Server`, `Azure SQL`, `BigQuery`; 2 bagian untuk `Hive`) dianggap tidak lengkap, tidak dicocokkan otomatis, dan ditandai "tidak lengkap" di daftar tidak tersambung. Konsumen tidak boleh menebak atau melengkapinya. Tipe target lain tidak dicocokkan lewat kunci tabel.

---

## 7. File alias (`bi_lineage_alias.csv`) — opsional

Dipakai saat nama di Power BI berbeda dengan nama di DAG walaupun objeknya sama, misalnya report membaca **view** yang dibangun di atas tabel target DAG, atau nama database berbeda karena synonym.

### 7.1 Kolom

| # | Kolom | Wajib | Keterangan |
|---|---|---|---|
| 1 | `Kunci Asal` | Ya | Kunci yang muncul (akan dinormalisasi ulang saat dibaca) |
| 2 | `Kunci Tujuan` | Ya | Kunci pengganti (akan dinormalisasi ulang saat dibaca) |
| 3 | `Jenis` | Ya | `View` \| `Synonym` \| `Rename` \| `Lainnya` |
| 4 | `Sisi` | Ya | `BI` \| `DAG` \| `Keduanya` (sisi mana yang kuncinya diganti) |
| 5 | `Catatan` | Tidak | Alasan / bukti |
| 6 | `Diubah Oleh` | Ya | Nama / akun |
| 7 | `Tanggal` | Ya | ISO 8601 (tanggal saja boleh: `2026-10-08`) |

### 7.2 Aturan

1. Alias diterapkan **setelah** normalisasi, **satu lompatan saja** (alias tidak berantai), sehingga tidak mungkin terjadi loop.
2. Jika ada dua baris dengan `Kunci Asal` dan `Sisi` yang sama, baris pertama dipakai dan konsumen menampilkan peringatan.
3. Alias tidak mengubah data mentah. Di tampilan, kunci hasil alias diberi penanda (mis. ikon `≈`) agar jelas bahwa sambungannya buatan manusia.

Contoh:

```csv
Kunci Asal,Kunci Tujuan,Jenis,Sisi,Catatan,Diubah Oleh,Tanggal
cntdimensional.sap.vw_credit_limit,cntdimensional.sap.credit_limit_customer_cnt,View,BI,View select * dari tabel target DAG,metadata.team,2026-10-08
```

---

## 8. Manifest (`bi_lineage_manifest.json`)

### 8.1 Fungsi

Analogi: manifest adalah **buku tamu**. Setiap report yang sudah "berkunjung" dicatat beserta versi terakhirnya, sehingga run berikutnya cukup memproses report yang baru atau berubah.

### 8.2 Struktur

```json
{
  "contractVersion": "1.0.0",
  "generatedAt": "2026-10-08T14:05:00+07:00",
  "tool": { "name": "TMDL Explorer", "version": "x.y.z" },
  "sources": [
    { "platform": "Fabric", "scope": "<workspace id>" },
    { "platform": "PBIRS",  "scope": "/Subholding/Finance" },
    { "platform": "Lokal",  "scope": "<nama folder root>" }
  ],
  "reports": [
    {
      "platform": "PBIRS",
      "reportId": "<guid>",
      "reportName": "Report B",
      "location": "/Subholding/Finance",
      "modelId": "<guid>",
      "sourceVersion": "2026-10-01T09:12:44.523+07:00",
      "versionKind": "ModifiedDate",
      "lastHarvestedAt": "2026-10-08T14:03:10+07:00",
      "lastStatus": "Sukses",
      "rowCount": 41,
      "message": ""
    }
  ]
}
```

| Field | Nilai |
|---|---|
| `versionKind` | `ModifiedDate` \| `ContentHash` \| `FileMtime` |
| `lastStatus` | `Sukses` \| `Gagal` \| `Dilewati` \| `Hilang dari sumber` |

### 8.3 `Versi Sumber` per platform

| Platform | Sumber versi | `versionKind` | Catatan |
|---|---|---|---|
| PBIRS | `ModifiedDate` dari `CatalogItems` (sudah tersedia di helper `pbirs.py`) | `ModifiedDate` | Murah: tidak perlu download `.pbix` untuk mengecek |
| Lokal | `lastModified` terbaru dari semua file di folder ekstrak + jumlah file, mis. `2026-10-01T09:12:44+07:00#57` | `FileMtime` | Dari File System Access API |
| Fabric | SHA-256 dari **baris lineage** report (lihat di bawah tabel) | `ContentHash` | API non-admin (`Get Reports In Group`, `Get Item`) tidak mengembalikan tanggal ubah; field `modifiedDateTime` hanya ada di API admin, yang di luar lingkup (§2 poin 3). Definisi model **selalu diambil penuh** di setiap run |

**Cara menghitung hash Fabric:**

1. Hasilkan baris lineage report seperti biasa.
2. Ambil semua kolom **kecuali** `Versi Sumber`, `Waktu Harvest`, dan `Versi Kontrak`. Dua yang pertama berubah tiap run (dan `Versi Sumber` sendiri berisi hash ini); `Versi Kontrak` dikecualikan agar kenaikan versi kontrak saja tidak dianggap perubahan lineage.
3. Urutkan baris secara leksikografis, gabungkan dengan pemisah baris baru, lalu hitung SHA-256 (hex lowercase).

Hash dihitung dari baris lineage, bukan dari TMDL mentah, karena TMDL memuat hal yang berubah tanpa mengubah lineage (mis. `lineageTag`, annotation). Hash berubah berarti lineage-nya memang berubah.

Konsekuensi: incremental yang benar-benar menghemat waktu hanya berlaku untuk **PBIRS** dan **Lokal**. Penghematan di Fabric hanya dari model yang dipakai banyak report (diambil sekali per run).

Catatan Fabric: beberapa report bisa memakai **satu semantic model yang sama**. Lineage dihitung per model, lalu ditulis per report. Model yang sama cukup diambil sekali per run.

### 8.4 Logika incremental

| Kondisi | Tindakan | Status di log |
|---|---|---|
| Report belum ada di manifest | Proses | `Sukses` / `Gagal` |
| PBIRS / Lokal: `sourceVersion` sama **dan** `lastStatus` = `Sukses` | Lewati (tanpa fetch), salin baris lama dari CSV sebelumnya | `Dilewati` |
| Fabric: hash baru **sama** dengan `sourceVersion` di manifest | Fetch tetap dilakukan; baris lama dipertahankan apa adanya (termasuk `Waktu Harvest` lama) | `Sukses`, `Alasan` = `lineage tidak berubah` |
| `sourceVersion` berbeda (Fabric: hash berbeda) | Proses ulang, **ganti semua** baris lama milik `Platform` + `Report ID` itu | `Sukses` / `Gagal` |
| `lastStatus` sebelumnya `Gagal` | Proses ulang | `Sukses` / `Gagal` |
| Ada di manifest tetapi tidak ditemukan lagi di sumber yang dipindai | Baris lama tetap disalin, manifest ditandai | `Hilang dari sumber` |
| Gagal di run ini | Baris lama (jika ada) tetap disalin, tidak dihapus | `Gagal` |

Untuk Fabric, status `Dilewati` **tidak pernah dipakai**, karena report tetap diambil di setiap run.

Jika Lineage CSV atau manifest sebelumnya berversi **mayor** berbeda dari yang diimplementasikan produsen, riwayat itu diabaikan: semua report diproses ulang, tidak ada baris yang disalin. Untuk versi mayor yang sama, baris lama disalin apa adanya (kolom di kanan yang belum ada diisi kosong) dan mempertahankan `Versi Kontrak` aslinya.

Konsekuensinya, setiap run menghasilkan **Lineage CSV lengkap** (snapshot penuh), bukan potongan. Input run berikutnya = Lineage CSV terakhir + manifest terakhir.

### 8.5 Log run (`bi_lineage_log_<YYYYMMDD-HHmm>.csv`)

| Kolom | Keterangan |
|---|---|
| `Waktu` | ISO 8601 |
| `Platform` | `Fabric` \| `PBIRS` \| `Lokal` |
| `Lokasi` | Workspace / folder |
| `Report ID` | |
| `Report Name` | |
| `Status` | `Sukses` \| `Gagal` \| `Dilewati` \| `Hilang dari sumber` |
| `Jumlah Baris` | Baris lineage yang dihasilkan |
| `Alasan` | Wajib jika `Gagal` (pesan error tanpa token/URL bertanda tangan). Untuk Fabric yang hash-nya sama: `lineage tidak berubah` |

---

## 9. Perilaku konsumen (DAG Lineage Explorer)

1. **Tanpa file BI**, explorer berjalan persis seperti sekarang. Satu-satunya elemen UI baru yang terlihat adalah tombol **Load BI Lineage CSV**; elemen lain (banner, tampilan downstream, impact analysis, daftar tidak tersambung, Unload) baru muncul setelah data BI dimuat.
2. **Join**: `normalize(Target Table Path)` di Lineage DAG CSV dicocokkan dengan `Kunci Sumber` (setelah alias) di Lineage CSV.
3. **Tampilan yang diturunkan**:
   - *Downstream report per target table*: daftar `Report Name` + `Tabel di Power BI` + `Jalur` untuk setiap tabel target DAG.
   - *Impact analysis*: dari satu DAG/task, semua report yang terdampak (termasuk lewat tabel target yang menjadi `Source` DAG lain).
   - *Tidak tersambung*, dua arah: (a) `Kunci Sumber` BI yang tidak punya pasangan DAG; (b) tabel target DAG yang tidak dipakai report mana pun.
4. **Unload** menghapus data BI dari memori dan mengembalikan tampilan ke kondisi awal.
5. Konsumen **tidak pernah** menulis balik ke Lineage CSV.
6. Data BI (Lineage CSV + alias) hanya hidup di **memori tab**. Tidak ditulis ke `localStorage` atau penyimpanan browser lain; memuat ulang halaman menghapusnya, kecuali jika di-embed (§9.1).

### 9.1 Embed snapshot BI ke HTML (opsional)

Memakai mekanisme **Save as built-in** yang sudah ada di DAG Explorer, diperluas dengan konstanta tambahan (mis. `EMBEDDED_BI`). Tanpa data BI, tombol menghasilkan file persis seperti sekarang.

1. **Opt-in.** Saat menyimpan, pengguna memilih apakah snapshot BI (dan file alias) ikut ditanam. Default: **tidak ikut**.
2. **Nama file bertanggal**, mis. `DAMA_ETL_Lineage_Explorer_YYYYMMDD.html`, agar HTML lama tidak tertimpa.
3. **Alias ikut ditanam** bersama snapshot, karena join bergantung padanya.
4. **Masking.** Untuk baris yang `Catatan`-nya mengandung `path pribadi`, kolom `Lokasi Sumber` dan `Objek Sumber` diganti `[disamarkan]` pada data yang ditanam. Kolom lain tidak berubah.
5. **Banner usia data.** Jika data BI berasal dari embed, tampilkan banner permanen *"Snapshot BI per [Waktu Harvest terbaru]"*.
6. **Unload** pada data embed hanya menyembunyikannya selama sesi tab; data muncul lagi saat halaman dimuat ulang. Label tombol harus menjelaskan hal ini.
7. Data BI yang dimuat dari file saat runtime menggantikan data embed selama sesi tersebut.

## 10. Validasi saat file dibaca

| Pemeriksaan | Jika gagal |
|---|---|
| Header tidak persis sama (nama/urutan 21 kolom pertama) | **Tolak file**, tampilkan kolom yang berbeda |
| `Versi Kontrak` pada baris mana pun bermayor berbeda dari yang didukung konsumen | **Tolak seluruh file**, sebutkan versi yang ditemukan |
| `Versi Kontrak` kosong atau tidak berformat `MAJOR.MINOR.PATCH` | Lewati baris, catat peringatan |
| Nilai enum di luar daftar (`Platform`, `Jalur`, `Mode Tabel`, `Status Kunci`, `Dari Native SQL`) | Lewati baris, catat peringatan |
| Kombinasi melanggar §5.3 | Lewati baris, catat peringatan |
| `Kedalaman` bukan bilangan bulat ≥ 0 | Lewati baris, catat peringatan |
| Duplikat grain (§5.1) | Ambil baris pertama, catat peringatan |

Ringkasan peringatan ditampilkan setelah load (jumlah baris diterima / dilewati + contoh).

## 11. Versioning kontrak

Mengikuti semantic versioning.

| Perubahan | Versi |
|---|---|
| Tambah kolom **di ujung kanan**, tambah nilai enum baru, tambah `Platform` baru | Minor (`1.1.0`) |
| Ubah nama/urutan kolom, hapus kolom, ubah arti kolom, ubah aturan normalisasi | Mayor (`2.0.0`) |
| Perbaikan redaksi dokumen | Patch (`1.0.1`) |

Konsumen versi `1.x` wajib bisa membaca file `1.y` mana pun dengan mengabaikan kolom tambahan di kanan.

Produsen menulis versi yang ia implementasikan di kolom `Versi Kontrak` pada setiap baris baru. Perubahan makna **tanpa** perubahan header (mis. aturan normalisasi) wajib menaikkan versi mayor, supaya terdeteksi lewat kolom ini dan tidak menghasilkan join yang salah secara diam-diam.

---

## 12. Contoh baris (kasus yang sudah diverifikasi)

Nilai dalam `<…>` adalah placeholder yang belum dikonfirmasi.

```csv
Platform,Lokasi,Report ID,Report Name,Model ID,Versi Sumber,Tabel di Power BI,Mode Tabel,Query perantara,Kedalaman,Jalur,Connector,Source Type,Lokasi Sumber,Objek Sumber,Kunci Sumber,Status Kunci,Dari Native SQL,Waktu Harvest,Catatan,Versi Kontrak
<platform>,<lokasi>,<id>,Report B,<model id>,<versi>,FactCA,import,Credit Limit,1,Via query,SQL Server,SQL Server,<server>,<objek mentah>,cntdimensional.sap.credit_limit_customer_cnt,<status>,<Ya/Tidak>,2026-10-08T14:05:00+07:00,,1.0.0
<platform>,<lokasi>,<id>,Report A,<model id>,<versi>,VL06F+MB51,import,Industri,1,Via query,BigQuery,BigQuery,data-governance-ppn,data-governance-ppn.data_analytics_bronze.aviasi_fact_trx_sales_sap_zsddhavi_q001,data-governance-ppn.data_analytics_bronze.aviasi_fact_trx_sales_sap_zsddhavi_q001,Lengkap,Ya,2026-10-08T14:05:00+07:00,,1.0.0
<platform>,<lokasi>,<id>,Report A,<model id>,<versi>,Calendar,calculated,,,Tanpa sumber eksternal,,,,,,Bukan tabel,Tidak,2026-10-08T14:05:00+07:00,DAX calculated,1.0.0
```

Pasangan di sisi DAG:

| Kunci Sumber (BI) | DAG |
|---|---|
| `cntdimensional.sap.credit_limit_customer_cnt` | `etl_sap_credit_limit_customer` |
| `data-governance-ppn.data_analytics_bronze.aviasi_fact_trx_sales_sap_zsddhavi_q001` | `el_sap_trx_aviation_daily` |

---

## 13. Keputusan desain

- `Query perantara` = query yang langsung memanggil connector (bukan yang paling dekat ke tabel model). `Kedalaman` menunjukkan jumlah lompatan dari tabel model ke query tersebut.
- Versi Fabric = SHA-256 dari baris lineage (di luar `Versi Sumber`, `Waktu Harvest`, `Versi Kontrak`); Fabric selalu fetch penuh karena API non-admin tidak menyediakan tanggal ubah (dicek ke dokumentasi Microsoft Learn: `Reports - Get Reports In Group`, `Items - Get Item`; `modifiedDateTime` hanya ada di API admin).
- Pembaruan data embed DAG Explorer: perluas **Save as built-in** yang sudah ada (§9.1). Snapshot BI di-embed secara **opt-in**, default tidak ikut.
- Normalisasi kunci membuang `[ ]`, backtick, dan kutip ganda `"` (§6.2). Kunci Hive = `database.table` (§6.1).
- Versi kontrak dibawa di setiap baris (kolom `Versi Kontrak`), sehingga konsumen bisa menolak file bermayor berbeda tanpa membaca manifest (§10, §11).

## 14. Riwayat perubahan

| Versi | Tanggal | Perubahan |
|---|---|---|
| `1.0.0` | 2026-10-08 | Rilis pertama |

---

## Lampiran A — Catatan sisi DAG (di luar kontrak, untuk harvester)

- Tambahkan mapping **key config → database** (contoh: `mssql_dwhcntprd` → `cntdimensional`) agar Target Table Path 2 bagian bisa dilengkapi otomatis. Simpan sebagai file mapping terpisah yang hanya berisi nama key dan nama database.
- Harvester **tidak pernah** membaca atau menyalin field kredensial dari YAML; hanya nama database yang diambil.
- Standarisasi penulisan DAG baru: hindari `from pipelines import *` supaya asal fungsi dan konfigurasi bisa dilacak secara statis.
- Sekitar 40 baris `Source` dengan path 2 bagian (non-Hive) di Lineage DAG CSV belum dilengkapi. Ini pekerjaan data, bukan keputusan kontrak. Sampai dilengkapi, konsumen memperlakukannya sebagai kunci tidak lengkap (§6.3).
