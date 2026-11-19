# 📱 Template 02 — DummyJSON API
### Panduan Lengkap Membangun Aplikasi Flutter dari Nol

> **Branch ini menggunakan:** DummyJSON (`https://dummyjson.com`)  
> **Cocok untuk soal dengan:** API e-commerce/produk, response dibungkus key `products`, field seperti `price`, `rating`, `stock`, `brand`  
> **Gunakan branch ini jika:** `template-01` gagal karena struktur API soal berbeda (tidak ada `results`, pakai key lain, tidak ada trailing slash)

---

## 🗂️ Struktur Akhir Folder

```
lib/
├── main.dart
├── models/
│   └── product_model.dart
├── services/
│   └── api_service.dart
└── screens/
    ├── login_screen.dart
    ├── register_screen.dart
    ├── home_screen.dart
    ├── list_screen.dart
    ├── detail_screen.dart
    └── favorite_screen.dart
```

> ⚠️ **Perbedaan dari template-01:** Model bernama `ProductModel` (bukan `ArticleModel`), method API bernama `getProducts` / `getSingleProduct`, dan field di semua screen sudah disesuaikan.

---

## LANGKAH 0 — Buat Project Flutter Baru

```bash
flutter create nama_project_kamu
cd nama_project_kamu
```

---

## LANGKAH 1 — Tambah Dependency di `pubspec.yaml`

Ganti bagian `dependencies` dan `dev_dependencies`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.0
  shared_preferences: ^2.2.2
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  url_launcher: ^6.2.5
  intl: ^0.19.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0
```

```bash
flutter pub get
```

---

## LANGKAH 2 — Tambah Permission di `AndroidManifest.xml`

Buka `android/app/src/main/AndroidManifest.xml`:

Tambahkan **sebelum `<application>`**:
```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

Tambahkan **setelah `</application>`**:
```xml
<queries>
    <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="https" />
    </intent>
    <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="http" />
    </intent>
</queries>
```

---

## LANGKAH 3 — Buat Folder Structure

```
lib/models/
lib/services/
lib/screens/
```

---

## LANGKAH 4 — Buat `lib/models/product_model.dart`

```dart
// =========================================================================
// [SIMULASI MODUL 5: MODEL DATA PRODUK]
// SEBELUMNYA: ArticleModel (berisi title, news_site, summary, published_at, dsb)
// SEKARANG: ProductModel (menyesuaikan atribut API DummyJSON: price, rating, stock, brand, dsb)
//
// CARA ADAPTASI KE API LAIN:
// 1. Tentukan atribut yang dimiliki API baru.
// 2. Pada fromJson(), petakan key JSON dengan fallback (??) agar tidak null pointer.
// 3. Pada toJson(), simpan data yang ingin dicatat di Hive Favorite.
// =========================================================================

class ProductModel {
  final int id;
  final String title;
  final String description;
  final String category;
  final double price;
  final double rating;
  final int stock;
  final String brand;
  final String imageUrl;
  final String webUrl; // URL web untuk url_launcher

  ProductModel({
    required this.id,
    required this.title,
    required this.description,
    required this.category,
    required this.price,
    required this.rating,
    required this.stock,
    required this.brand,
    required this.imageUrl,
    required this.webUrl,
  });

  factory ProductModel.fromJson(Map<String, dynamic> json) {
    // Menangani gambar: DummyJSON menyediakan array 'images' dan string 'thumbnail'
    String selectedImage = '';
    if (json['thumbnail'] != null && json['thumbnail'].toString().isNotEmpty) {
      selectedImage = json['thumbnail'].toString();
    } else if (json['images'] is List && (json['images'] as List).isNotEmpty) {
      selectedImage = json['images'][0].toString();
    }

    return ProductModel(
      id: json['id'] is int ? json['id'] : int.tryParse(json['id']?.toString() ?? '0') ?? 0,
      title: json['title']?.toString() ?? 'Tanpa Nama Produk',
      description: json['description']?.toString() ?? 'Tidak ada deskripsi tersedia.',
      category: json['category']?.toString() ?? 'Umum',
      price: json['price'] != null ? (json['price'] as num).toDouble() : 0.0,
      rating: json['rating'] != null ? (json['rating'] as num).toDouble() : 0.0,
      stock: json['stock'] is int ? json['stock'] : int.tryParse(json['stock']?.toString() ?? '0') ?? 0,
      brand: json['brand']?.toString() ?? 'No Brand',
      imageUrl: selectedImage,
      webUrl: 'https://dummyjson.com/products/${json['id']}',
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'title': title,
      'description': description,
      'category': category,
      'price': price,
      'rating': rating,
      'stock': stock,
      'brand': brand,
      'imageUrl': imageUrl,
      'webUrl': webUrl,
    };
  }
}
```

---

## LANGKAH 5 — Buat `lib/services/api_service.dart`

```dart
// =========================================================================
// [SIMULASI MODUL 5: SERVICE API DUMMYJSON]
// PERBEDAAN DARI TEMPLATE-01:
// 1. baseUrl: 'https://dummyjson.com' (tanpa versi /v4)
// 2. URL endpoint TANPA trailing slash: '$baseUrl/$endpoint' bukan '$endpoint/'
// 3. Key pembungkus JSON: data['products'] bukan data['results']
// 4. Method: getProducts() & getSingleProduct() bukan getArticles() & getDetail()
// =========================================================================

import 'dart:convert';
import 'package:http/http.dart' as http;
import '../models/product_model.dart';

class ApiService {
  // [PERUBAHAN 1]: Ganti URL dasar API
  static const String baseUrl = 'https://dummyjson.com';

  // [PERUBAHAN 2]: Mengambil daftar produk
  static Future<List<ProductModel>> getProducts(String endpoint) async {
    // PERHATIAN: DummyJSON tidak pakai trailing slash!
    // Contoh URL: https://dummyjson.com/products
    //             https://dummyjson.com/products/category/beauty
    final url = Uri.parse('$baseUrl/$endpoint');     // <-- Tanpa '/' di akhir
    final response = await http.get(url);

    if (response.statusCode == 200) {
      final data = jsonDecode(response.body);

      // [PERUBAHAN 3]: DummyJSON membungkus data dalam key "products"
      List results = [];
      if (data is Map<String, dynamic> && data.containsKey('products')) {
        results = data['products'] as List;
      } else if (data is List) {
        results = data;
      }

      return results
          .map((item) => ProductModel.fromJson(item as Map<String, dynamic>))
          .toList();
    } else {
      throw Exception('Gagal memuat produk dari API (Status: ${response.statusCode})');
    }
  }

  // [PERUBAHAN 4]: Mengambil detail satu produk berdasarkan id
  static Future<ProductModel> getSingleProduct(int id) async {
    final url = Uri.parse('$baseUrl/products/$id');  // <-- Endpoint detail fixed
    final response = await http.get(url);

    if (response.statusCode == 200) {
      final data = jsonDecode(response.body);
      return ProductModel.fromJson(data as Map<String, dynamic>);
    } else {
      throw Exception('Gagal memuat detail produk (id: $id)');
    }
  }
}
```

---

## LANGKAH 6 — Buat `lib/screens/login_screen.dart`

> Sama persis dengan template-01. Copy dari sana, atau gunakan kode berikut:

```dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'home_screen.dart';
import 'register_screen.dart';

class LoginScreen extends StatefulWidget {
  const LoginScreen({super.key});
  @override
  State<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _usernameCtrl = TextEditingController();
  final _passwordCtrl = TextEditingController();
  bool _isLoading = false;

  @override
  void initState() { super.initState(); _checkLogin(); }

  Future<void> _checkLogin() async {
    final prefs = await SharedPreferences.getInstance();
    if ((prefs.getBool('isLogin') ?? false) && prefs.getString('username') != null && mounted) {
      Navigator.pushReplacement(context,
        MaterialPageRoute(builder: (_) => HomeScreen(username: prefs.getString('username')!)));
    }
  }

  Future<void> _login() async {
    final u = _usernameCtrl.text.trim(), p = _passwordCtrl.text.trim();
    if (u.isEmpty || p.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Login Gagal! Pastikan sudah Register.'), backgroundColor: Colors.red));
      return;
    }
    setState(() => _isLoading = true);
    final prefs = await SharedPreferences.getInstance();
    if (prefs.getString('user_$u') == p) {
      await prefs.setBool('isLogin', true);
      await prefs.setString('username', u);
      if (mounted) Navigator.pushReplacement(context,
        MaterialPageRoute(builder: (_) => HomeScreen(username: u)));
    } else {
      setState(() => _isLoading = false);
      if (mounted) ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Login Gagal! Cek username dan password.'), backgroundColor: Colors.red));
    }
  }

  @override
  void dispose() { _usernameCtrl.dispose(); _passwordCtrl.dispose(); super.dispose(); }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFFF7F8FC),
      appBar: AppBar(title: const Text('Login',
        style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        centerTitle: true, backgroundColor: const Color(0xFF4C6EF5), elevation: 0),
      body: Center(child: SingleChildScrollView(
        padding: const EdgeInsets.all(24),
        child: Card(elevation: 3, shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
          child: Padding(padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 32),
            child: Column(mainAxisSize: MainAxisSize.min, children: [
              const Text('Silakan Login', style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold)),
              const SizedBox(height: 24),
              TextField(controller: _usernameCtrl,
                decoration: InputDecoration(labelText: 'Username', prefixIcon: const Icon(Icons.person_outline),
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)))),
              const SizedBox(height: 16),
              TextField(controller: _passwordCtrl, obscureText: true,
                decoration: InputDecoration(labelText: 'Password', prefixIcon: const Icon(Icons.lock_outline),
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)))),
              const SizedBox(height: 24),
              SizedBox(width: 140, height: 44,
                child: ElevatedButton(
                  style: ElevatedButton.styleFrom(backgroundColor: const Color(0xFF4C6EF5),
                    foregroundColor: Colors.white, shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(22))),
                  onPressed: _isLoading ? null : _login,
                  child: _isLoading
                    ? const SizedBox(width: 20, height: 20, child: CircularProgressIndicator(color: Colors.white, strokeWidth: 2))
                    : const Text('Login', style: TextStyle(fontSize: 16, fontWeight: FontWeight.w600)))),
              const SizedBox(height: 16),
              TextButton(
                onPressed: () => Navigator.push(context, MaterialPageRoute(builder: (_) => const RegisterScreen())),
                child: const Text('Belum punya akun? Daftar di sini', style: TextStyle(color: Colors.grey, fontSize: 13))),
            ]))),
      )),
    );
  }
}
```

---

## LANGKAH 7 — Buat `lib/screens/register_screen.dart`

> Sama persis dengan template-01. Salin dari template-01 Langkah 7.

---

## LANGKAH 8 — Buat `lib/screens/home_screen.dart`

```dart
// =========================================================================
// [SIMULASI MODUL 5: HALAMAN UTAMA PRODUK]
// PERBEDAAN DARI TEMPLATE-01:
// - Judul menu: kategori produk e-commerce (All Products, Beauty, Fragrances)
// - Endpoint: 'products', 'products/category/beauty', 'products/category/fragrances'
//
// CARA ADAPTASI:
// 1. Ganti title dan subtitle tiap menu.
// 2. Ganti string endpoint — perhatikan format DummyJSON: 'products/category/namaKategori'
// =========================================================================

import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'list_screen.dart';
import 'login_screen.dart';

class HomeScreen extends StatelessWidget {
  final String username;
  const HomeScreen({super.key, required this.username});

  Future<void> _logout(BuildContext context) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setBool('isLogin', false);
    if (context.mounted) {
      Navigator.pushAndRemoveUntil(context,
        MaterialPageRoute(builder: (_) => const LoginScreen()), (r) => false);
    }
  }

  Widget _buildMenuCard({required BuildContext context, required IconData icon,
    required String title, required String subtitle, required VoidCallback onTap}) {
    return Card(
      elevation: 2, margin: const EdgeInsets.only(bottom: 16),
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      color: const Color(0xFFFBFBFE),
      child: InkWell(
        borderRadius: BorderRadius.circular(16), onTap: onTap,
        child: Padding(padding: const EdgeInsets.all(20),
          child: Row(crossAxisAlignment: CrossAxisAlignment.start, children: [
            Container(padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(color: const Color(0xFFECEEFE),
                borderRadius: BorderRadius.circular(12)),
              child: Icon(icon, color: const Color(0xFF4C6EF5), size: 28)),
            const SizedBox(width: 16),
            Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
              Text(title, style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold, color: Colors.black87)),
              const SizedBox(height: 6),
              Text(subtitle, style: TextStyle(fontSize: 12, color: Colors.grey.shade600, height: 1.4)),
            ])),
          ])),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFFF7F8FC),
      appBar: AppBar(
        title: Text('Hai, $username!',
          style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        centerTitle: true, backgroundColor: const Color(0xFF4C6EF5),
        elevation: 0, automaticallyImplyLeading: false),
      body: SafeArea(child: Padding(
        padding: const EdgeInsets.all(20.0),
        child: Column(children: [
          Expanded(child: ListView(children: [
            // [RESPONSI]: Menu 1 - Ganti title, subtitle, dan endpoint
            _buildMenuCard(context: context, icon: Icons.shopping_bag_outlined,
              title: 'All Products',                          // <-- Ganti nama menu
              subtitle: 'Lihat semua produk tersedia.',       // <-- Ganti deskripsi
              onTap: () => Navigator.push(context, MaterialPageRoute(
                builder: (_) => const ListScreen(
                  menuTitle: 'All Products',
                  endpoint: 'products',                       // <-- GANTI endpoint
                )))),

            // [RESPONSI]: Menu 2 - Ganti title, subtitle, dan endpoint
            _buildMenuCard(context: context, icon: Icons.spa_outlined,
              title: 'Beauty',                                // <-- Ganti nama menu
              subtitle: 'Produk perawatan kecantikan.',       // <-- Ganti deskripsi
              onTap: () => Navigator.push(context, MaterialPageRoute(
                builder: (_) => const ListScreen(
                  menuTitle: 'Beauty',
                  endpoint: 'products/category/beauty',       // <-- GANTI endpoint
                )))),

            // [RESPONSI]: Menu 3 - Ganti title, subtitle, dan endpoint
            _buildMenuCard(context: context, icon: Icons.local_florist_outlined,
              title: 'Fragrances',                           // <-- Ganti nama menu
              subtitle: 'Koleksi wewangian premium.',         // <-- Ganti deskripsi
              onTap: () => Navigator.push(context, MaterialPageRoute(
                builder: (_) => const ListScreen(
                  menuTitle: 'Fragrances',
                  endpoint: 'products/category/fragrances',   // <-- GANTI endpoint
                )))),
          ])),
          const SizedBox(height: 12),
          SizedBox(width: 140, height: 42,
            child: ElevatedButton(
              style: ElevatedButton.styleFrom(backgroundColor: Colors.red,
                foregroundColor: Colors.white,
                shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(22))),
              onPressed: () => _logout(context),
              child: const Text('Log Out', style: TextStyle(fontWeight: FontWeight.bold, fontSize: 15)))),
          const SizedBox(height: 12),
        ]),
      )),
    );
  }
}
```

---

## LANGKAH 9 — Buat `lib/screens/list_screen.dart`

```dart
// =========================================================================
// [SIMULASI MODUL 5: HALAMAN LIST PRODUK]
// PERBEDAAN DARI TEMPLATE-01:
// - Pakai ProductModel bukan ArticleModel
// - Filter berdasarkan item.category (bukan item.author)
// - Field tampilan: item.title, item.brand, item.price
// - Method API: ApiService.getProducts() bukan getArticles()
// =========================================================================

import 'package:flutter/material.dart';
import '../models/product_model.dart';
import '../services/api_service.dart';
import 'detail_screen.dart';
import 'favorite_screen.dart';

class ListScreen extends StatefulWidget {
  final String menuTitle;
  final String endpoint;
  const ListScreen({super.key, required this.menuTitle, required this.endpoint});
  @override
  State<ListScreen> createState() => _ListScreenState();
}

class _ListScreenState extends State<ListScreen> {
  List<ProductModel> _allProducts = [];
  List<String> _filters = [];
  String _selectedFilter = 'All';
  bool _isLoading = true;
  String? _errorMessage;

  @override
  void initState() { super.initState(); _fetchData(); }

  Future<void> _fetchData() async {
    setState(() { _isLoading = true; _errorMessage = null; });
    try {
      final data = await ApiService.getProducts(widget.endpoint);

      // [PERUBAHAN]: Filter berdasarkan category (bukan author)
      final Set<String> filterSet = {};
      for (final item in data) {
        if (item.category.trim().isNotEmpty) filterSet.add(item.category.trim());
      }

      setState(() {
        _allProducts = data;
        _filters = ['All', ...filterSet];
        _isLoading = false;
      });
    } catch (e) {
      setState(() { _isLoading = false; _errorMessage = e.toString(); });
    }
  }

  @override
  Widget build(BuildContext context) {
    // [PERUBAHAN]: Filter berdasarkan category
    final displayed = _selectedFilter == 'All'
        ? _allProducts
        : _allProducts.where((p) => p.category == _selectedFilter).toList();

    return Scaffold(
      backgroundColor: const Color(0xFFF7F8FC),
      appBar: AppBar(
        title: Text(widget.menuTitle, style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        backgroundColor: const Color(0xFF4C6EF5), elevation: 0,
        iconTheme: const IconThemeData(color: Colors.white),
        actions: [
          IconButton(icon: const Icon(Icons.favorite, color: Colors.white),
            onPressed: () => Navigator.push(context, MaterialPageRoute(builder: (_) => const FavoriteScreen()))),
        ],
      ),
      body: Column(children: [
        if (!_isLoading && _filters.isNotEmpty)
          Container(color: Colors.white, padding: const EdgeInsets.symmetric(vertical: 8),
            child: SingleChildScrollView(scrollDirection: Axis.horizontal,
              padding: const EdgeInsets.symmetric(horizontal: 16),
              child: Row(children: _filters.map((f) {
                final sel = _selectedFilter == f;
                return Padding(padding: const EdgeInsets.only(right: 8),
                  child: FilterChip(selected: sel, label: Text(f),
                    labelStyle: TextStyle(color: sel ? const Color(0xFF4C6EF5) : Colors.black87,
                      fontWeight: sel ? FontWeight.bold : FontWeight.normal, fontSize: 13),
                    backgroundColor: const Color(0xFFF1F3F9), selectedColor: const Color(0xFFE4E9FE),
                    onSelected: (_) => setState(() => _selectedFilter = f),
                    shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(10),
                      side: BorderSide(color: sel ? const Color(0xFF4C6EF5) : Colors.grey.shade300))));
              }).toList()))),

        Expanded(child: _isLoading
          ? const Center(child: CircularProgressIndicator())
          : _errorMessage != null
            ? Center(child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                const Icon(Icons.error_outline, size: 64, color: Colors.red),
                const SizedBox(height: 12),
                Text(_errorMessage!, textAlign: TextAlign.center),
                const SizedBox(height: 16),
                ElevatedButton(onPressed: _fetchData, child: const Text('Coba Lagi'))]))
            : ListView.builder(
                padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
                itemCount: displayed.length,
                itemBuilder: (ctx, i) {
                  final item = displayed[i];
                  return Card(elevation: 1.5, margin: const EdgeInsets.only(bottom: 12),
                    shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(14)),
                    color: const Color(0xFFFBFBFE),
                    child: InkWell(
                      borderRadius: BorderRadius.circular(14),
                      onTap: () => Navigator.push(ctx, MaterialPageRoute(
                        builder: (_) => DetailScreen(item: item))),
                      child: Padding(padding: const EdgeInsets.all(12),
                        child: Row(children: [
                          ClipRRect(borderRadius: BorderRadius.circular(8),
                            child: Image.network(item.imageUrl, width: 80, height: 80, fit: BoxFit.cover,
                              errorBuilder: (c, e, s) => Container(width: 80, height: 80,
                                color: Colors.grey.shade200,
                                child: const Icon(Icons.image_not_supported_outlined, color: Colors.grey)),
                              loadingBuilder: (c, child, p) => p == null ? child
                                : Container(width: 80, height: 80, color: Colors.grey.shade100,
                                    child: const Center(child: SizedBox(width: 18, height: 18,
                                      child: CircularProgressIndicator(strokeWidth: 2)))))),
                          const SizedBox(width: 14),
                          Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                            // [PERUBAHAN]: Judul produk
                            Text(item.title, maxLines: 2, overflow: TextOverflow.ellipsis,
                              style: const TextStyle(fontSize: 14, fontWeight: FontWeight.bold, color: Colors.black87)),
                            const SizedBox(height: 4),
                            // [PERUBAHAN]: Brand sebagai subtitle
                            Text(item.brand, style: TextStyle(fontSize: 12, color: Colors.grey.shade600)),
                            const SizedBox(height: 4),
                            // [PERUBAHAN]: Harga
                            Text('Rp ${item.price.toStringAsFixed(0)}',
                              style: const TextStyle(fontSize: 13, fontWeight: FontWeight.bold,
                                color: Color(0xFF4C6EF5))),
                          ])),
                        ])),
                    ));
                })),
      ]),
    );
  }
}
```

---

## LANGKAH 10 — Buat `lib/screens/detail_screen.dart`

```dart
// =========================================================================
// [SIMULASI MODUL 5: HALAMAN DETAIL PRODUK]
// PERBEDAAN DARI TEMPLATE-01:
// - Pakai ProductModel
// - Field tampil: title, brand, category, price, rating, stock, description
// - Url launcher: item.webUrl
// - Tombol Favorite menyimpan ke Hive
// =========================================================================

import 'package:flutter/material.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'package:url_launcher/url_launcher.dart';
import '../models/product_model.dart';

class DetailScreen extends StatefulWidget {
  final ProductModel item;
  const DetailScreen({super.key, required this.item});
  @override
  State<DetailScreen> createState() => _DetailScreenState();
}

class _DetailScreenState extends State<DetailScreen> {
  late Box _box;
  bool _isFavorite = false;

  @override
  void initState() {
    super.initState();
    _box = Hive.box('favorites');
    setState(() => _isFavorite = _box.containsKey(widget.item.id.toString()));
  }

  Future<void> _toggleFavorite() async {
    final key = widget.item.id.toString();
    if (_isFavorite) {
      await _box.delete(key);
      setState(() => _isFavorite = false);
      if (mounted) ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Dihapus dari favorite'), duration: Duration(seconds: 1)));
    } else {
      await _box.put(key, widget.item.toJson());
      setState(() => _isFavorite = true);
      if (mounted) ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Ditambahkan ke favorite'), duration: Duration(seconds: 1)));
    }
  }

  Future<void> _openWeb() async {
    final uri = Uri.parse(widget.item.webUrl);
    try { await launchUrl(uri, mode: LaunchMode.externalApplication); }
    catch (e) { if (mounted) ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Error: $e'), backgroundColor: Colors.red)); }
  }

  Widget _infoRow(String label, String value) => Padding(
    padding: const EdgeInsets.only(bottom: 8),
    child: Row(children: [
      SizedBox(width: 90, child: Text('$label:', style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 14))),
      Expanded(child: Text(value, style: const TextStyle(fontSize: 14))),
    ]),
  );

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.white,
      appBar: AppBar(
        title: const Text('Detail Produk',
          style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        backgroundColor: const Color(0xFF4C6EF5), elevation: 0,
        iconTheme: const IconThemeData(color: Colors.white),
        actions: [
          IconButton(
            icon: Icon(_isFavorite ? Icons.favorite : Icons.favorite_border,
              color: _isFavorite ? Colors.red : Colors.white),
            onPressed: _toggleFavorite),
        ],
      ),
      body: Stack(children: [
        SingleChildScrollView(
          padding: const EdgeInsets.only(bottom: 80),
          child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
            // Gambar Produk
            SizedBox(width: double.infinity, height: 240,
              child: Image.network(widget.item.imageUrl, fit: BoxFit.cover,
                errorBuilder: (c, e, s) => Container(color: Colors.grey.shade200,
                  child: const Center(child: Icon(Icons.broken_image_outlined, size: 64, color: Colors.grey))),
                loadingBuilder: (c, child, p) => p == null ? child
                  : Container(color: Colors.grey.shade100, child: const Center(child: CircularProgressIndicator(strokeWidth: 2))))),

            Padding(padding: const EdgeInsets.all(16), child: Column(
              crossAxisAlignment: CrossAxisAlignment.start, children: [
                // Judul & Harga
                Text(widget.item.title,
                  style: const TextStyle(fontSize: 20, fontWeight: FontWeight.bold, color: Colors.black87, height: 1.3)),
                const SizedBox(height: 8),
                Text('Rp ${widget.item.price.toStringAsFixed(0)}',
                  style: const TextStyle(fontSize: 22, fontWeight: FontWeight.bold, color: Color(0xFF4C6EF5))),
                const SizedBox(height: 16),
                const Divider(height: 1),
                const SizedBox(height: 16),
                // Info Produk
                _infoRow('Brand', widget.item.brand),
                _infoRow('Kategori', widget.item.category),
                _infoRow('Rating', '${widget.item.rating} ⭐'),
                _infoRow('Stok', '${widget.item.stock} tersisa'),
                const SizedBox(height: 16),
                const Divider(height: 1),
                const SizedBox(height: 16),
                // Deskripsi
                const Text('Deskripsi',
                  style: TextStyle(fontSize: 15, fontWeight: FontWeight.bold, color: Color(0xFF4C6EF5))),
                const SizedBox(height: 8),
                Text(widget.item.description,
                  textAlign: TextAlign.justify,
                  style: const TextStyle(fontSize: 14, color: Colors.black87, height: 1.6)),
              ],
            )),
          ]),
        ),
        Positioned(bottom: 20, right: 20,
          child: FloatingActionButton.extended(
            backgroundColor: const Color(0xFFECEEFE),
            foregroundColor: const Color(0xFF4C6EF5),
            elevation: 3,
            icon: const Icon(Icons.open_in_new, size: 20),
            label: const Text('Buka Web', style: TextStyle(fontWeight: FontWeight.bold)),
            onPressed: _openWeb)),
      ]),
    );
  }
}
```

---

## LANGKAH 11 — Buat `lib/screens/favorite_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:hive_flutter/hive_flutter.dart';
import '../models/product_model.dart';
import 'detail_screen.dart';

class FavoriteScreen extends StatelessWidget {
  const FavoriteScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final Box box = Hive.box('favorites');
    return Scaffold(
      backgroundColor: const Color(0xFFF7F8FC),
      appBar: AppBar(
        title: const Text('Favorite', style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        backgroundColor: const Color(0xFF4C6EF5), elevation: 0,
        iconTheme: const IconThemeData(color: Colors.white)),
      body: ValueListenableBuilder<Box>(
        valueListenable: box.listenable(),
        builder: (context, b, _) {
          if (b.isEmpty) return Center(child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
            Icon(Icons.favorite_border, size: 72, color: Colors.grey.shade400),
            const SizedBox(height: 12),
            Text('Belum ada produk favorit',
              style: TextStyle(fontSize: 16, color: Colors.grey.shade600, fontWeight: FontWeight.w500)),
          ]));

          final List<ProductModel> favs = b.values.map((v) =>
            ProductModel.fromJson(Map<String, dynamic>.from(v as Map))).toList();

          return ListView.builder(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
            itemCount: favs.length,
            itemBuilder: (ctx, i) {
              final item = favs[i];
              return Card(elevation: 1.5, margin: const EdgeInsets.only(bottom: 12),
                shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(14)),
                color: const Color(0xFFFBFBFE),
                child: InkWell(
                  borderRadius: BorderRadius.circular(14),
                  onTap: () => Navigator.push(ctx, MaterialPageRoute(
                    builder: (_) => DetailScreen(item: item))),
                  child: Padding(padding: const EdgeInsets.all(12),
                    child: Row(children: [
                      ClipRRect(borderRadius: BorderRadius.circular(8),
                        child: Image.network(item.imageUrl, width: 80, height: 60, fit: BoxFit.cover,
                          errorBuilder: (c, e, s) => Container(width: 80, height: 60,
                            color: Colors.grey.shade200,
                            child: const Icon(Icons.image_not_supported_outlined, color: Colors.grey, size: 28)))),
                      const SizedBox(width: 14),
                      Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                        Text(item.title, maxLines: 2, overflow: TextOverflow.ellipsis,
                          style: const TextStyle(fontSize: 14, fontWeight: FontWeight.bold, color: Colors.black87)),
                        const SizedBox(height: 4),
                        Text('Rp ${item.price.toStringAsFixed(0)}',
                          style: const TextStyle(fontSize: 13, fontWeight: FontWeight.bold, color: Color(0xFF4C6EF5))),
                      ])),
                      IconButton(
                        icon: const Icon(Icons.favorite, color: Colors.red),
                        onPressed: () async {
                          await b.delete(item.id.toString());
                          if (ctx.mounted) ScaffoldMessenger.of(ctx).showSnackBar(
                            const SnackBar(content: Text('Dihapus dari favorite'), duration: Duration(seconds: 1)));
                        }),
                    ])),
                ));
            });
        }),
    );
  }
}
```

---

## LANGKAH 12 — Edit `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'screens/login_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  await Hive.openBox('favorites');
  runApp(const MainApp());
}

class MainApp extends StatelessWidget {
  const MainApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Product Store App',           // <-- Ganti sesuai soal
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF4C6EF5)),
        useMaterial3: true,
        scaffoldBackgroundColor: const Color(0xFFF7F8FC),
      ),
      home: const LoginScreen(),
    );
  }
}
```

---

## LANGKAH 13 — Jalankan Aplikasi

```bash
flutter run
```

---

## ⚡ Perbedaan Utama template-02 vs template-01

| Aspek | template-01 | template-02 |
|-------|-------------|-------------|
| API | Spaceflight News | DummyJSON |
| `baseUrl` | `.../v4` | `https://dummyjson.com` |
| URL endpoint | `$baseUrl/$endpoint/` (ada `/`) | `$baseUrl/$endpoint` (tanpa `/`) |
| Key JSON | `data['results']` | `data['products']` |
| Model | `ArticleModel` | `ProductModel` |
| Field model | `title, url, imageUrl, newsSite, summary, author` | `title, price, rating, stock, brand, imageUrl, webUrl` |
| Field filter | `item.author` | `item.category` |
| Detail URL | `item.url` | `item.webUrl` |

> Lihat `PANDUAN_GANTI_API.md` di branch ini untuk panduan adaptasi ke API soal lain.
