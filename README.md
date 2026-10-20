# Ujian HOTS Rumus Excel

Aplikasi ujian interaktif untuk SMP ABBS: **5 project, 20 soal** rumus Excel (fungsi dasar, IF, tanda `$`, VLOOKUP/HLOOKUP, dan gabungan/error).

Semua soal dikerjakan di browser. Hasil tiap siswa otomatis dikirim dan disimpan di database, jadi guru bisa melihat seluruh jawaban tanpa perlu mengumpulkan file.

## Isi repositori

| Berkas | Keterangan |
| --- | --- |
| `index.html` | Seluruh aplikasi ujian (HTML + CSS + JavaScript dalam satu berkas). Tidak perlu build, tidak memakai framework. |

## Cara memakai

1. Buka halaman ujian (GitHub Pages).
2. Siswa menulis nama, mencentang pernyataan sudah membaca aturan, lalu menekan **Mulai ujian**.
3. Soal dikerjakan berurutan. Tiap soal maksimal 10 kali percobaan; nilai berkurang 10% setiap percobaan salah.
4. Setelah soal ke-20, nilai akhir tampil dan jawaban otomatis tersimpan di server.

## Penyimpanan hasil

Hasil disimpan di **Supabase** (project `fsrfvkaqbhfuptsooxiv`, berbagi dengan project AVATAR). Dua tabel berikut dibuat khusus untuk ujian ini dan **tidak mengganggu tabel AVATAR** yang sudah ada:

**`excel_hots_sesi`** — satu baris per siswa (hasil akhir)

| Kolom | Keterangan |
| --- | --- |
| `id` | Kode unik sesi ujian |
| `nama` | Nama siswa |
| `mulai_at`, `selesai_at` | Waktu mulai dan selesai |
| `durasi_detik`, `patokan_detik` | Waktu pengerjaan dan patokan (60 menit) |
| `total_soal`, `jumlah_benar`, `jumlah_habis` | Rekap jumlah soal |
| `ketepatan`, `nilai_waktu`, `nilai_akhir` | Nilai (ketepatan 80%, waktu 20%) |
| `bintang` | 1–3 bintang |
| `perangkat` | Info browser siswa |

**`excel_hots_jawaban`** — satu baris per soal (20 baris per siswa)

| Kolom | Keterangan |
| --- | --- |
| `sesi_id`, `soal_no` | Penghubung ke sesi dan nomor soal |
| `project_no`, `project_nama`, `tingkat` | Project dan tingkat kesulitan soal |
| `percobaan`, `petunjuk` | Berapa kali mencoba dan berapa petunjuk dipakai |
| `benar`, `habis` | Benar atau kehabisan kesempatan |
| `waktu_detik`, `skor` | Waktu dan nilai soal tersebut |
| `jawaban_terakhir` | Jawaban terakhir yang ditulis siswa |
| `riwayat` | **Semua** jawaban yang pernah dicoba, berurutan (JSON) |

### Melihat hasil

Buka Supabase Dashboard → **Table Editor** → `excel_hots_sesi`, atau jalankan:

```sql
select nama, nilai_akhir, jumlah_benar, durasi_detik, selesai_at
from excel_hots_sesi
order by selesai_at desc;
```

Untuk melihat semua jawaban beserta riwayat percobaannya:

```sql
select s.nama, j.soal_no, j.project_nama, j.percobaan, j.skor, j.jawaban_terakhir
from excel_hots_jawaban j
join excel_hots_sesi s on s.id = j.sesi_id
order by s.selesai_at desc, j.soal_no;
```

## Catatan keamanan

Kunci yang tertanam di `index.html` adalah **anon key**, yang memang dirancang untuk dipakai di sisi pengunjung. Pada kedua tabel, peran `anon` hanya diberi izin **INSERT** — tanpa izin SELECT. Artinya pengunjung hanya bisa mengirim jawaban, dan **tidak dapat membaca jawaban siswa lain atau daftar nilainya**. Row Level Security (RLS) aktif pada kedua tabel.

## Hosting

Repositori ini dihosting gratis melalui **GitHub Pages**.
