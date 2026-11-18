# 🗺️ Panduan Ganti API saat Responsi — `piw_latresp_mobile`

> Repo: [primaryptr/piw_latresp_mobile](https://github.com/primaryptr/piw_latresp_mobile)  
> Ada **2 template** siap pakai: `template-01` (Spaceflight API) dan `template-02` (DummyJSON).  
> Gunakan `template-01` dulu. Kalau API soal tidak mirip strukturnya, lihat `template-02` sebagai contoh adaptasi.

---

## ⚡ 5 Menit Pertama — Urutan Wajib

```
Baca soal → Catat: baseUrl, nama endpoint(s), struktur JSON response
     ↓
Edit 4 file berikut (urutan ini)
     ↓
flutter run
```

---

## 📁 Branch: `template-01` — Spaceflight News API

### 1️⃣ `lib/services/api_service.dart` — **MULAI DI SINI**

| Baris | Yang diganti | Contoh |
|-------|-------------|--------|
| `static const String baseUrl = '...'` | URL dasar API soal | `'https://api.soal.com/v1'` |
| `Uri.parse('$baseUrl/$endpoint/')` | Tambah/hapus trailing slash `/` tergantung API | Coba tanpa `/` kalau 404 |
| `data['results']` | Key pembungkus JSON | Ganti ke `data['products']`, `data['data']`, `data['items']`, atau hapus jika langsung List |

**Lokasi komentar penanda di file:**
```dart
// [RESPONSI]: Ganti baseUrl sesuai domain API yang diberikan di soal
static const String baseUrl = 'https://api.spaceflightnewsapi.net/v4';
```

> [!TIP] Kalau API response langsung `[{...}, {...}]` (array mentah), pakai cabang `else if (data is List)` yang sudah ada — tidak perlu ubah apapun selain `baseUrl`.

---

### 2️⃣ `lib/models/article_model.dart` — Sesuaikan Field JSON

**Yang perlu diubah:** nama atribut + key `fromJson()`.

| Field saat ini | Key `fromJson` default | Contoh key alternatif |
|---|---|---|
| `title` | `json['title']` atau `json['name']` | `name`, `judul` |
| `imageUrl` | `json['image_url']` atau `json['images'][0]` | `thumbnail`, `photo`, `cover` |
| `newsSite` | `json['news_site']` | `source`, `author`, `brand`, `category` |
| `summary` | `json['summary']` atau `json['description']` | `body`, `content`, `desc` |
| `publishedAt` | `json['published_at']` | `date`, `created_at`, `price` |
| `url` | `json['url']` | `link`, `website`, `webUrl` |

**Lokasi komentar penanda di file:**
```dart
// [RESPONSI]: Sesuaikan key JSON di bawah ini dengan nama key dari API baru
id: json['id'] ...
title: json['title']?.toString() ?? json['name']?.toString() ...
```

> [!IMPORTANT] Selalu pakai `?.toString() ?? 'fallback'` agar tidak crash saat field kosong/null.

---

### 3️⃣ `lib/screens/home_screen.dart` — Ganti Nama Menu & Endpoint

Cari ketiga blok `_buildMenuCard(...)` dan ganti bagian ini:

```dart
// [RESPONSI]: Menu 1 - Sesuaikan title, deskripsi, dan endpoint API
title: 'News',                    // <-- Ganti nama menu
subtitle: 'Deskripsi menu...',    // <-- Ganti deskripsi
endpoint: 'articles',             // <-- GANTI endpoint API (paling penting!)
```

Jika soal hanya punya **1 menu**, hapus 2 blok `_buildMenuCard` yang tidak terpakai.  
Jika punya **lebih dari 3**, duplikasi blok yang ada.

---

### 4️⃣ `lib/screens/list_screen.dart` — Ganti Field Filter

**Lokasi komentar penanda:**
```dart
// [RESPONSI CHECKPOINT 4: HALAMAN LIST & FITUR FILTER]
// 1. Pada _fetchData(): tentukan atribut apa yang dijadikan filter
// 2. Pada FilterChip: sesuaikan logika filter
```

Cari baris yang menentukan filter, biasanya seperti:
```dart
item.newsSite   // <-- Ganti dengan field yang ingin dijadikan filter
                //     misal: item.category, item.brand, item.author
```

---

## 📁 Branch: `template-02` — DummyJSON (Contoh Adaptasi API Lain)

> Gunakan ini sebagai **referensi/inspirasi** saat API soal berbeda struktur dari `template-01`.

### Perbedaan utama vs template-01:

| Hal | template-01 | template-02 |
|-----|-------------|-------------|
| `baseUrl` | `https://api.spaceflightnewsapi.net/v4` | `https://dummyjson.com` |
| Trailing slash URL | Ada (`/articles/`) | Tidak ada (`/products`) |
| Key pembungkus JSON | `data['results']` | `data['products']` |
| Model | `ArticleModel` | `ProductModel` |
| Field model | `title, url, imageUrl, newsSite, summary` | `title, price, rating, stock, brand, imageUrl` |
| Field gambar | `json['image_url']` | `json['thumbnail']` |

### File yang berubah di template-02:

```
lib/
├── services/
│   └── api_service.dart       ← baseUrl + key pembungkus + nama method
├── models/
│   └── product_model.dart     ← Semua field disesuaikan ke DummyJSON
└── screens/
    ├── home_screen.dart       ← Nama menu + endpoint (products, beauty, dsb)
    ├── list_screen.dart       ← Filter berdasarkan category (bukan newsSite)
    └── detail_screen.dart     ← Field tampilan: price, rating, stock, brand
```

---

## 🔥 Skenario Soal & Solusi Cepat

### Skenario A: API langsung return List `[{...}]`
```dart
// Di api_service.dart — tidak perlu ubah, cabang else if sudah ada:
} else if (data is List) {
  results = data;   // ← Sudah handle otomatis
}
```

### Skenario B: API pakai key pembungkus berbeda (misal `data['items']`)
```dart
// Di api_service.dart — tambah key baru:
results = data['results'] ?? data['products'] ?? data['data'] ?? data['items'] ?? [];
//                                                                        ^^ tambah ini
```

### Skenario C: API tidak pakai trailing slash
```dart
// Di api_service.dart — hapus '/' di akhir:
final url = Uri.parse('$baseUrl/$endpoint');   // bukan '$endpoint/'
```

### Skenario D: API pakai API Key / Authorization header
```dart
// Di api_service.dart — tambah headers:
final response = await http.get(url, headers: {
  'Authorization': 'Bearer YOUR_API_KEY',   // atau
  'x-api-key': 'YOUR_API_KEY',
});
```

### Skenario E: Gambar dari array `images[0]`
```dart
// Di model fromJson() — sudah ada fallback-nya di template-01:
imageUrl: json['image_url']?.toString() ??
    (json['images'] is List && (json['images'] as List).isNotEmpty
        ? json['images'][0].toString()
        : ''),
```

---

## 📋 Checklist Verifikasi (sebelum flutter run)

- [ ] `baseUrl` sudah diganti
- [ ] Trailing slash URL sudah disesuaikan (ada/tidak)
- [ ] Key pembungkus JSON sudah benar (`results`/`products`/`data`/langsung List)
- [ ] `fromJson()` semua key sudah dipetakan ke field model
- [ ] Nama menu & endpoint di `home_screen.dart` sudah sesuai soal
- [ ] Field filter di `list_screen.dart` sudah benar

---

## 🚨 Kalau Masih Gagal (Error Checklist)

| Error | Penyebab | Solusi |
|-------|----------|--------|
| `type 'Null' is not a subtype of type 'String'` | Key JSON salah/null | Cek nama key di Postman/browser, tambah `?? ''` |
| `type 'List<dynamic>' is not a subtype` | Parsing wrapper salah | Cek struktur JSON: Map atau List langsung? |
| `SocketException` / `Connection refused` | API tidak bisa diakses | Cek INTERNET permission di `AndroidManifest.xml` |
| `404 Not Found` | URL/endpoint salah | Coba hapus/tambah trailing slash |
| `Null check operator used on null` | `fromJson` tanpa fallback | Tambah `?.toString() ?? 'default'` |
