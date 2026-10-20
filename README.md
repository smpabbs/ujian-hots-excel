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
4. Setiap kali satu soal selesai, jawabannya **langsung dikirim ke server**, jadi jawaban tetap aman meskipun koneksi putus atau halaman tertutup di tengah ujian.
5. Setelah soal ke-20, nilai akhir tampil dan ujian ditandai **selesai**.

## Penyimpanan hasil

Hasil disimpan di **Supabase** (project `fsrfvkaqbhfuptsooxiv`, berbagi dengan project AVATAR). Tabel dan view berikut dibuat khusus untuk ujian ini dan **tidak mengganggu tabel AVATAR** yang sudah ada.

Jawaban dikirim **bertahap**, bukan sekaligus di akhir:

| Saat | Yang dikirim |
| --- | --- |
| Siswa menekan **Mulai ujian** | satu baris di `excel_hots_sesi` |
| Satu soal selesai (benar, atau kesempatan habis) | satu baris di `excel_hots_jawaban` |
| Siswa menekan **Kumpulkan ujian** | satu baris di `excel_hots_selesai` |

**`excel_hots_sesi`** — satu baris per siswa, ditulis saat ujian dimulai

| Kolom | Keterangan |
| --- | --- |
| `id` | Kode unik sesi ujian |
| `nama` | Nama siswa |
| `mulai_at` | Waktu mulai |
| `total_soal`, `patokan_detik` | Jumlah soal dan patokan waktu (60 menit) |
| `perangkat` | Info browser siswa |

**`excel_hots_jawaban`** — satu baris per soal (paling banyak 20 baris per siswa)

| Kolom | Keterangan |
| --- | --- |
| `sesi_id`, `soal_no` | Penghubung ke sesi dan nomor soal |
| `project_no`, `project_nama`, `tingkat` | Project dan tingkat kesulitan soal |
| `percobaan`, `petunjuk` | Berapa kali mencoba dan berapa petunjuk dipakai |
| `benar`, `habis` | Benar atau kehabisan kesempatan |
| `waktu_detik`, `skor` | Waktu dan nilai soal tersebut |
| `jawaban_terakhir` | Jawaban terakhir yang ditulis siswa |
| `riwayat` | **Semua** jawaban yang pernah dicoba, berurutan (JSON) |

**`excel_hots_selesai`** — satu baris per siswa, ditulis saat menekan Kumpulkan ujian

| Kolom | Keterangan |
| --- | --- |
| `sesi_id` | Penghubung ke sesi |
| `selesai_at`, `durasi_detik` | Waktu selesai dan lama pengerjaan |
| `jumlah_benar`, `jumlah_habis` | Rekap jumlah soal |
| `ketepatan`, `nilai_waktu`, `nilai_akhir` | Nilai (ketepatan 80%, waktu 20%) |
| `bintang` | 1–3 bintang |

### Melihat hasil

Cara paling mudah: buka **`excel_hots_rekap`** (view) di Supabase Table Editor — satu baris per siswa, gabungan dari ketiga tabel:

```sql
select nama, status, soal_terjawab, nilai_akhir, bintang, durasi_detik
from excel_hots_rekap
order by mulai_at;
```

Kolom `status` bernilai `selesai` bila siswa sudah mengumpulkan, dan `berjalan` bila ujian belum tuntas. Jadi guru bisa **langsung melihat siswa mana yang ujiannya terputus**, lengkap dengan berapa soal yang sempat terjawab.

Untuk melihat semua jawaban beserta riwayat percobaannya:

```sql
select s.nama, j.soal_no, j.project_nama, j.percobaan, j.skor, j.jawaban_terakhir
from excel_hots_jawaban j
join excel_hots_sesi s on s.id = j.sesi_id
order by s.mulai_at desc, j.soal_no;
```

## Catatan keamanan

Kunci yang tertanam di `index.html` adalah **anon key**, yang memang dirancang untuk dipakai di sisi pengunjung. Pada ketiga tabel, peran `anon` hanya diberi izin **INSERT** — tanpa SELECT, UPDATE, maupun DELETE, dan tanpa policy SELECT sama sekali. Artinya pengunjung hanya bisa mengirim jawaban, dan **tidak dapat membaca jawaban siswa lain maupun daftar nilainya** (setiap percobaan membaca ditolak dengan `permission denied`). Row Level Security (RLS) aktif pada semua tabel, dan view `excel_hots_rekap` dibuat dengan `security_invoker` sehingga tetap tunduk pada aturan yang sama.

## Hosting

Repositori ini dihosting gratis melalui **GitHub Pages**.
