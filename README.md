# Ujian HOTS Rumus Excel

Aplikasi ujian interaktif untuk SMP ABBS: **5 project, 20 soal** rumus Excel (fungsi dasar, IF, tanda `$`, VLOOKUP/HLOOKUP, dan gabungan/error).

Semua soal dikerjakan di browser. Hasil tiap siswa otomatis dikirim dan disimpan di database, jadi guru bisa melihat seluruh jawaban tanpa perlu mengumpulkan file.

## Isi repositori

| Berkas | Keterangan |
| --- | --- |
| `index.html` | Seluruh aplikasi ujian (HTML + CSS + JavaScript dalam satu berkas). Tidak perlu build, tidak memakai framework. |
| `guru.html` | Halaman pemantauan untuk guru: progres tiap siswa secara langsung. Dilindungi kunci guru. |

## Cara memakai

1. Buka halaman ujian (GitHub Pages).
2. Siswa menulis nama, mencentang pernyataan sudah membaca aturan, lalu menekan **Mulai ujian**.
3. Soal dikerjakan berurutan. Tiap soal maksimal 10 kali percobaan; nilai berkurang 10% setiap percobaan salah.
4. Setiap kali satu soal selesai, jawabannya **langsung dikirim ke server**, jadi jawaban tetap aman meskipun koneksi putus atau halaman tertutup di tengah ujian.
5. Setelah soal ke-20, nilai akhir tampil dan ujian ditandai **selesai**.

## Cara penilaian

Nilai akhir = **80% ketepatan + 20% waktu**.

**Ketepatan** dihitung dari rata-rata **berbobot** skor 20 soal. Skor tiap soal ditentukan percobaan dan petunjuk:

- percobaan ke-1 = 100%, ke-2 = 90%, … ke-10 = 10%; kehabisan kesempatan = 0
- dikalikan pengali petunjuk: 0 petunjuk ×1 · 1 petunjuk ×0,85 · 2 petunjuk ×0,7

**Bobot soal mengikuti tingkat kesulitan** (tersimpan di kolom `bobot` pada `excel_hots_jawaban`):

| Tingkat | Jumlah soal | Bobot | Nilai akhir per soal |
| --- | --- | --- | --- |
| Mudah | 6 | 1× | 2,3 poin |
| HOTS · Menganalisis | 4 | 1,5× | 3,5 poin |
| HOTS · Mengevaluasi | 5 | 2× | 4,6 poin |
| HOTS · Mencipta | 5 | 2,5× | 5,8 poin |

Total 20 soal = **80 poin** ketepatan.

**Waktu** bernilai **20 poin**. Selesai dalam patokan 60 menit (3 menit × 20 soal) bernilai penuh; lewat patokan turun bertahap, paling rendah 10 poin.

Bobot disimpan **per baris jawaban**, bukan sebagai angka tetap di dalam program. Jadi bobot bisa diubah tanpa merusak nilai sesi yang sudah berjalan: jawaban lama tetap bernilai 1×, jawaban yang dikirim setelah pembobotan memakai bobot barunya.

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
| `bobot` | Bobot soal menurut tingkat kesulitan (1 / 1,5 / 2 / 2,5) |
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

## Halaman pemantauan guru

`guru.html` — dasbor untuk mengikuti progres siswa selagi ujian berlangsung.

- Alamat: `https://smpabbs.github.io/ujian-hots-excel/guru.html`
- Dilindungi **kunci guru**: diminta sekali saat dibuka, lalu disimpan di browser (tombol **Keluar** untuk melupakannya).
- Isinya: ringkasan (jumlah siswa, sedang mengerjakan, sudah selesai, rata-rata nilai, soal terjawab), daftar siswa yang bisa dicari dan diurutkan, serta rincian tiap soal beserta seluruh riwayat percobaan siswa.
- Status `berjalan` menandai siswa yang belum menekan **Kumpulkan ujian**, sehingga ujian yang terputus langsung terlihat.
- Untuk siswa yang belum selesai ditampilkan **nilai berjalan** (tanda `~`), yaitu perkiraan nilai **bila ujian berhenti sekarang**: soal yang belum dikerjakan dihitung 0 dan komponen waktu dihitung dari waktu berjalan. Karena memakai rumus yang sama dengan nilai akhir, angkanya hanya naik dan akhirnya **menyamai nilai akhir** saat siswa menekan Kumpulkan. (Angka "rata-rata soal terjawab" yang dulu dipakai bisa tampak tinggi di awal — misalnya `~100` padahal baru 3 soal dari 20 — karena hanya menghitung soal yang sudah dikerjakan; angka itu kini hanya ditampilkan di panel rincian sebagai keterangan tambahan.)
- Daftar dimuat ulang otomatis setiap 20 detik (bisa dimatikan).

### Ekspor Excel

Tombol **⬇ Ekspor Excel** mengunduh berkas `.xlsx` berisi dua lembar:

| Lembar | Isi |
| --- | --- |
| **Rekap** | satu baris per siswa: status, soal terjawab, jawaban benar, nilai akhir, nilai berjalan, bintang, durasi, waktu mulai dan selesai |
| **Jawaban** | satu baris per soal per siswa: project, tingkat, bobot, percobaan, petunjuk, hasil, skor, waktu, jawaban terakhir, dan seluruh riwayat percobaan |

Baris judul dibekukan dan dilengkapi penyaring otomatis (AutoFilter), jadi langsung bisa diurutkan dan disaring di Excel.

Nilai akhir dan nilai berjalan ditaruh di **kolom terpisah**, supaya perkiraan nilai siswa yang belum selesai tidak keliru dipakai sebagai nilai final.

Berkasnya dibuat langsung di dalam browser **tanpa pustaka luar**, sehingga tidak ada masalah pemisah koma/titik koma seperti pada berkas CSV.

### Mengapa aman dibuka publik

Anon key di halaman ini **tidak dapat membaca tabel** — peran `anon` memang hanya diberi izin INSERT. Pembacaan hanya mungkin lewat tiga fungsi database `security definer`:

| Fungsi | Kegunaan |
| --- | --- |
| `excel_hots_cek_kunci(p_kunci)` | memeriksa kunci guru, mengembalikan benar/salah |
| `excel_hots_rekap_guru(p_kunci)` | daftar siswa beserta progres dan nilainya |
| `excel_hots_jawaban_guru(p_kunci, p_sesi)` | rincian jawaban satu siswa |

Ketiganya hanya mengembalikan data bila `p_kunci` cocok dengan hash bcrypt di tabel `excel_hots_guru`. Tabel itu tertutup rapat: RLS aktif dan tidak ada izin apa pun untuk `anon`, sehingga kunci hanya bisa diperiksa lewat fungsi, tidak pernah bisa dibaca.

### Mengganti kunci guru

Jalankan di SQL Editor Supabase (ganti `KUNCI_BARU` dengan kunci baru):

```sql
update excel_hots_guru
set kunci_hash = extensions.crypt('KUNCI_BARU', extensions.gen_salt('bf', 10))
where id = 1;
```

## Catatan keamanan

Kunci yang tertanam di `index.html` adalah **anon key**, yang memang dirancang untuk dipakai di sisi pengunjung. Pada ketiga tabel, peran `anon` hanya diberi izin **INSERT** — tanpa SELECT, UPDATE, maupun DELETE, dan tanpa policy SELECT sama sekali. Artinya pengunjung hanya bisa mengirim jawaban, dan **tidak dapat membaca jawaban siswa lain maupun daftar nilainya** (setiap percobaan membaca ditolak dengan `permission denied`). Row Level Security (RLS) aktif pada semua tabel, dan view `excel_hots_rekap` dibuat dengan `security_invoker` sehingga tetap tunduk pada aturan yang sama.

## Hosting

Repositori ini dihosting gratis melalui **GitHub Pages**.
