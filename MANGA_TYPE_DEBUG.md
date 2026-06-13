# Debug: mangaType Tidak Terdeteksi

**Komik:** Magic Emperor (id WP: 39810)  
**Seharusnya:** Manhua 🇨🇳  
**Yang tampil:** Manga 🇯🇵 (default fallback)

---

## Semua Pendekatan yang Sudah Dicoba (GAGAL)

### ❌ Percobaan 1–5: Regex HTML
- `[?&]type=` — match `post_type=manga` lebih dulu
- `/manga/?order=title&type=` — link ini hanya ada di halaman LIST, bukan detail
- Label "Tipe" di tabel — komik7 tidak pakai label ini
- Class `type`/`tipe` — tidak ada di HTML komik7
- Scan 8000 char pertama — kata "Manga" muncul di head/meta sebelum "Manhua"

### ❌ Percobaan 6–7: WP Tags API
- Via `categories?slug={slug}` → `posts?categories={catId}` → `tags` → cek nama
- Gagal: post tidak punya tag Manhua/Manhwa/Manga, atau cache Redis lama

---

## Solusi Baru (v4)

### Perubahan di `scrape-detail/route.ts`

**Cache key diubah ke `v4`** — wajib agar bypass semua cache lama.

### Strategi Deteksi (berurutan, berhenti saat berhasil):

#### 1. WP custom taxonomy `ero_type` (paling reliable)
```
GET /wp-json/wp/v2/ero_type?post={wpPostId}&per_page=5&_fields=name,slug
```
Plugin MangaStream/Madara menyimpan tipe (Manhua/Manhwa/Manga) di custom taxonomy `ero_type`.
WP Post ID diekstrak dari HTML via `postid-XXXXX` di body class, atau `?p=XXXXX` di shortlink.

#### 2. WP categories per post (jika ero_type tidak ada)
```
GET /wp-json/wp/v2/categories?post={wpPostId}&per_page=20&_fields=name,slug
```
Cek apakah ada category dengan slug/name mengandung "manhua"/"manhwa".

#### 3. Regex HTML — blok info (Type/Tipe label)
Pola Madara: `Type : </span> <a>Manhwa</a>`  
Pola span class: `<span class="type">Manhua</span>`  
Pola dt/th tabel: `<dt>Type</dt><dd>Manhua</dd>`

#### 4. Link ?type= di dalam HTML detail
Breadcrumb atau related section mungkin punya link filter.

#### 5. Scan blok setelah `<h1>` judul
Ambil 1500 char setelah heading judul — lebih presisi dari scan 8000 char awal.

#### 6. Scan HTML char 5000–8000 (skip head/meta)
Melewati bagian `<head>` yang mengandung kata "manga" di URL meta/description site.

#### 7. Default: "Manga"

---

## Cara Test

Clear cache Redis dulu (key lama `v3` sudah otomatis tidak dipakai karena key baru `v4`):
```
# Tidak perlu manual clear — cache key v4 baru otomatis miss
GET /api/scrape-detail?slug=magic-emperor
```

Cek di response JSON:
```json
{ "mangaType": "Manhua", ... }
```

---

## Jika Masih Gagal

Inspect HTML asli:
```
GET https://komik7.my.id/manga/magic-emperor/
```
Cari di HTML untuk:
1. `postid-` di `<body class="...">` — untuk dapat WP Post ID
2. `/wp-json/wp/v2/ero_type?post={id}` — apakah endpoint ini ada
3. Kata "Manhua" muncul di mana tepatnya di HTML
