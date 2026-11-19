# 📱 Template 01 — Spaceflight News API
### Panduan Lengkap Membangun Aplikasi Flutter dari Nol

> **Branch ini menggunakan:** Spaceflight News API (`https://api.spaceflightnewsapi.net/v4`)  
> **Cocok untuk soal dengan:** API yang response-nya dibungkus key `results`, data berupa list artikel/berita/blog

---

## 🗂️ Struktur Akhir Folder

```
lib/
├── main.dart
├── models/
│   └── article_model.dart
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

---

## LANGKAH 0 — Buat Project Flutter Baru

```bash
flutter create nama_project_kamu
cd nama_project_kamu
```

> Ganti `nama_project_kamu` dengan nama yang diminta soal.

---

## LANGKAH 1 — Tambah Dependency di `pubspec.yaml`

Buka `pubspec.yaml` di root project, **ganti seluruh bagian `dependencies` dan `dev_dependencies`** menjadi:

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

Lalu jalankan:

```bash
flutter pub get
```

---

## LANGKAH 2 — Tambah Permission di `AndroidManifest.xml`

Buka `android/app/src/main/AndroidManifest.xml`, tambahkan **sebelum tag `<application>`**:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

Tambahkan **setelah tag `</application>`** (untuk url_launcher):

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

Di dalam folder `lib/`, buat folder-folder berikut:

```bash
# Buat manual lewat IDE, atau via terminal:
mkdir lib\models
mkdir lib\services
mkdir lib\screens
```

---

## LANGKAH 4 — Buat `lib/models/article_model.dart`

Buat file baru: `lib/models/article_model.dart`, lalu **copy semua kode berikut**:

```dart
// =========================================================================
// [RESPONSI CHECKPOINT 1: MODEL DATA]
// File ini merepresentasikan struktur data dari API.
// JIKA STUDI KASUS BERUBAH:
// 1. Sesuaikan nama atribut sesuai JSON dari API baru (misal: name, price, category, dsb).
// 2. Sesuaikan tipe datanya (int, String, double, List).
// 3. Pada fromJson(): petakan key JSON ke variabel (selalu gunakan fallback "??" untuk mencegah error null).
// 4. Pada toJson(): sesuaikan agar data yang disimpan ke Hive Favorite tetap lengkap.
// =========================================================================

class ArticleModel {
  // [RESPONSI]: Ganti/tambah atribut sesuai data baru
  final int id;
  final String title;
  final String url;
  final String imageUrl;
  final String newsSite;
  final String summary;
  final String publishedAt;
  final String author;
  final String type; // Kategori/endpoint menu (misal: 'articles', 'blogs', 'reports')

  ArticleModel({
    required this.id,
    required this.title,
    required this.url,
    required this.imageUrl,
    required this.newsSite,
    required this.summary,
    required this.publishedAt,
    required this.author,
    required this.type,
  });

  // [RESPONSI]: Factory constructor untuk konversi JSON dari API ke objek Dart
  factory ArticleModel.fromJson(Map<String, dynamic> json, {String type = 'articles'}) {
    // Ekstraksi nama author / kategori jika bersarang (nested)
    String authorName = '';
    if (json['authors'] != null && (json['authors'] as List).isNotEmpty) {
      final firstAuthor = json['authors'][0];
      if (firstAuthor is Map && firstAuthor['name'] != null) {
        authorName = firstAuthor['name'].toString();
      }
    }
    final site = json['news_site']?.toString() ?? '';
    if (authorName.isEmpty) {
      authorName = site.isNotEmpty ? site : 'Unknown';
    }

    return ArticleModel(
      // [RESPONSI]: Sesuaikan key JSON di bawah ini dengan nama key dari API baru
      id: json['id'] is int ? json['id'] : int.tryParse(json['id']?.toString() ?? '0') ?? 0,
      title: json['title']?.toString() ?? json['name']?.toString() ?? 'Tanpa Judul',
      url: json['url']?.toString() ?? '',
      // Jika image berupa List string (seperti DummyJSON json['images'][0]), sesuaikan di sini
      imageUrl: json['image_url']?.toString() ??
          (json['images'] is List && (json['images'] as List).isNotEmpty
              ? json['images'][0].toString()
              : json['image']?.toString() ?? ''),
      newsSite: site.isNotEmpty ? site : authorName,
      summary: json['summary']?.toString() ?? json['description']?.toString() ?? '',
      publishedAt: json['published_at']?.toString() ?? '',
      author: authorName,
      type: json['type']?.toString() ?? type,
    );
  }

  // [RESPONSI]: toJson() digunakan untuk menyimpan data ke Hive Favorite
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'title': title,
      'url': url,
      'image_url': imageUrl,
      'news_site': newsSite,
      'summary': summary,
      'published_at': publishedAt,
      'author': author,
      'type': type,
    };
  }
}
```

---

## LANGKAH 5 — Buat `lib/services/api_service.dart`

Buat file baru: `lib/services/api_service.dart`, lalu **copy semua kode berikut**:

```dart
// =========================================================================
// [RESPONSI CHECKPOINT 2: SERVICE & ENDPOINTS API]
// File ini mengatur pemanggilan HTTP Request ke server API.
// JIKA STUDI KASUS BERUBAH:
// 1. Ganti baseUrl dengan URL API baru pada soal.                      ← PALING PENTING
// 2. Periksa format root response API:
//    - Apakah dibungkus Map: data['results'] / data['products'] / data['data']?
//    - Ataukah langsung berupa List: data as List?
// 3. Sesuaikan method getArticles() dan getDetail() di bawah ini.
// =========================================================================

import 'dart:convert';
import 'package:http/http.dart' as http;
import '../models/article_model.dart';

class ApiService {
  // [RESPONSI]: Ganti baseUrl sesuai domain API yang diberikan di soal
  static const String baseUrl = 'https://api.spaceflightnewsapi.net/v4';

  // [RESPONSI]: Mengambil list data dari API
  static Future<List<ArticleModel>> getArticles(String endpoint) async {
    // Sesuaikan format URL jika API baru tidak memerlukan garis miring di akhir
    final url = Uri.parse('$baseUrl/$endpoint/');
    final response = await http.get(url);

    if (response.statusCode == 200) {
      final data = jsonDecode(response.body);

      // [RESPONSI TIP]:
      // Kasus A: Jika data API dibungkus key tertentu, contoh: data['results'] atau data['products']
      List results = [];
      if (data is Map<String, dynamic>) {
        results = data['results'] ?? data['products'] ?? data['data'] ?? [];
      } else if (data is List) {
        // Kasus B: Jika response API langsung berupa List array murni [...]
        results = data;
      }

      return results
          .map((item) => ArticleModel.fromJson(item, type: endpoint))
          .toList();
    } else {
      throw Exception('Gagal memuat data dari API (Status: ${response.statusCode})');
    }
  }

  // [RESPONSI]: Mengambil detail satu item berdasarkan id
  static Future<ArticleModel> getDetail(String endpoint, int id) async {
    final url = Uri.parse('$baseUrl/$endpoint/$id/');
    final response = await http.get(url);

    if (response.statusCode == 200) {
      final data = jsonDecode(response.body);
      return ArticleModel.fromJson(data, type: endpoint);
    } else {
      throw Exception('Gagal memuat detail data (id: $id)');
    }
  }
}
```

---

## LANGKAH 6 — Buat `lib/screens/login_screen.dart`

Buat file baru: `lib/screens/login_screen.dart`, lalu **copy semua kode berikut**:

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
  final TextEditingController _usernameController = TextEditingController();
  final TextEditingController _passwordController = TextEditingController();
  bool _isLoading = false;

  @override
  void initState() {
    super.initState();
    _checkLogin();
  }

  // Cek apakah pengguna sudah memiliki sesi login aktif (Modul 6)
  Future<void> _checkLogin() async {
    final prefs = await SharedPreferences.getInstance();
    final bool isLogin = prefs.getBool('isLogin') ?? false;
    final String? username = prefs.getString('username');

    if (isLogin && username != null && mounted) {
      Navigator.pushReplacement(
        context,
        MaterialPageRoute(
          builder: (context) => HomeScreen(username: username),
        ),
      );
    }
  }

  // Fungsi proses login dengan validasi SharedPreferences
  Future<void> _login() async {
    final username = _usernameController.text.trim();
    final password = _passwordController.text.trim();

    if (username.isEmpty || password.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(
          content: Text('Login Gagal! Pastikan sudah Register atau cek password.'),
          backgroundColor: Colors.red,
        ),
      );
      return;
    }

    setState(() => _isLoading = true);
    final prefs = await SharedPreferences.getInstance();
    final registeredPassword = prefs.getString('user_$username');

    // Login berhasil jika user terdaftar dengan password cocok
    if (registeredPassword != null && registeredPassword == password) {
      await prefs.setBool('isLogin', true);
      await prefs.setString('username', username);

      if (mounted) {
        Navigator.pushReplacement(
          context,
          MaterialPageRoute(
            builder: (context) => HomeScreen(username: username),
          ),
        );
      }
    } else {
      setState(() => _isLoading = false);
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('Login Gagal! Pastikan sudah Register atau cek password.'),
            backgroundColor: Colors.red,
          ),
        );
      }
    }
  }

  @override
  void dispose() {
    _usernameController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFFF7F8FC),
      appBar: AppBar(
        title: const Text(
          'Login',
          style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold),
        ),
        centerTitle: true,
        backgroundColor: const Color(0xFF4C6EF5),
        elevation: 0,
      ),
      body: Center(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24.0),
          child: Card(
            elevation: 3,
            shape: RoundedRectangleBorder(
              borderRadius: BorderRadius.circular(16),
            ),
            child: Padding(
              padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 32),
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  const Text(
                    'Silakan Login',
                    style: TextStyle(
                      fontSize: 20,
                      fontWeight: FontWeight.bold,
                      color: Colors.black87,
                    ),
                  ),
                  const SizedBox(height: 24),
                  TextField(
                    controller: _usernameController,
                    decoration: InputDecoration(
                      labelText: 'Username',
                      prefixIcon: const Icon(Icons.person_outline),
                      border: OutlineInputBorder(
                        borderRadius: BorderRadius.circular(12),
                      ),
                    ),
                  ),
                  const SizedBox(height: 16),
                  TextField(
                    controller: _passwordController,
                    obscureText: true,
                    decoration: InputDecoration(
                      labelText: 'Password',
                      prefixIcon: const Icon(Icons.lock_outline),
                      border: OutlineInputBorder(
                        borderRadius: BorderRadius.circular(12),
                      ),
                    ),
                  ),
                  const SizedBox(height: 24),
                  SizedBox(
                    width: 140,
                    height: 44,
                    child: ElevatedButton(
                      style: ElevatedButton.styleFrom(
                        backgroundColor: const Color(0xFF4C6EF5),
                        foregroundColor: Colors.white,
                        shape: RoundedRectangleBorder(
                          borderRadius: BorderRadius.circular(22),
                        ),
                      ),
                      onPressed: _isLoading ? null : _login,
                      child: _isLoading
                          ? const SizedBox(
                              width: 20, height: 20,
                              child: CircularProgressIndicator(
                                color: Colors.white, strokeWidth: 2,
                              ),
                            )
                          : const Text('Login',
                              style: TextStyle(fontSize: 16, fontWeight: FontWeight.w600)),
                    ),
                  ),
                  const SizedBox(height: 16),
                  TextButton(
                    onPressed: () {
                      Navigator.push(context,
                        MaterialPageRoute(builder: (context) => const RegisterScreen()));
                    },
                    child: const Text('Belum punya akun? Daftar di sini',
                      style: TextStyle(color: Colors.grey, fontSize: 13)),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

---

## LANGKAH 7 — Buat `lib/screens/register_screen.dart`

Buat file baru: `lib/screens/register_screen.dart`, lalu **copy semua kode berikut**:

```dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

class RegisterScreen extends StatefulWidget {
  const RegisterScreen({super.key});

  @override
  State<RegisterScreen> createState() => _RegisterScreenState();
}

class _RegisterScreenState extends State<RegisterScreen> {
  final TextEditingController _usernameController = TextEditingController();
  final TextEditingController _passwordController = TextEditingController();
  bool _isLoading = false;

  Future<void> _register() async {
    final username = _usernameController.text.trim();
    final password = _passwordController.text.trim();

    if (username.isEmpty || password.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Username dan Password tidak boleh kosong!'),
          backgroundColor: Colors.red),
      );
      return;
    }

    setState(() => _isLoading = true);
    final prefs = await SharedPreferences.getInstance();

    if (prefs.containsKey('user_$username')) {
      setState(() => _isLoading = false);
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Username sudah terdaftar! Silakan gunakan username lain.'),
            backgroundColor: Colors.red),
        );
      }
      return;
    }

    await prefs.setString('user_$username', password);
    setState(() => _isLoading = false);

    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Registrasi berhasil! Silakan login.'),
          backgroundColor: Colors.green),
      );
      Navigator.pop(context);
    }
  }

  @override
  void dispose() {
    _usernameController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFFF7F8FC),
      appBar: AppBar(
        title: const Text('Register',
          style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        centerTitle: true,
        backgroundColor: const Color(0xFF4C6EF5),
        elevation: 0,
        iconTheme: const IconThemeData(color: Colors.white),
      ),
      body: Center(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24.0),
          child: Card(
            elevation: 3,
            shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
            child: Padding(
              padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 32),
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  const Text('Daftar Akun Baru',
                    style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold, color: Colors.black87)),
                  const SizedBox(height: 24),
                  TextField(
                    controller: _usernameController,
                    decoration: InputDecoration(
                      labelText: 'Username',
                      prefixIcon: const Icon(Icons.person_outline),
                      border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
                    ),
                  ),
                  const SizedBox(height: 16),
                  TextField(
                    controller: _passwordController,
                    obscureText: true,
                    decoration: InputDecoration(
                      labelText: 'Password',
                      prefixIcon: const Icon(Icons.lock_outline),
                      border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
                    ),
                  ),
                  const SizedBox(height: 24),
                  SizedBox(
                    width: 140, height: 44,
                    child: ElevatedButton(
                      style: ElevatedButton.styleFrom(
                        backgroundColor: const Color(0xFF4C6EF5),
                        foregroundColor: Colors.white,
                        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(22)),
                      ),
                      onPressed: _isLoading ? null : _register,
                      child: _isLoading
                          ? const SizedBox(width: 20, height: 20,
                              child: CircularProgressIndicator(color: Colors.white, strokeWidth: 2))
                          : const Text('Daftar',
                              style: TextStyle(fontSize: 16, fontWeight: FontWeight.w600)),
                    ),
                  ),
                  const SizedBox(height: 16),
                  TextButton(
                    onPressed: () => Navigator.pop(context),
                    child: const Text('Sudah punya akun? Login di sini',
                      style: TextStyle(color: Colors.grey, fontSize: 13)),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

---

## LANGKAH 8 — Buat `lib/screens/home_screen.dart`

Buat file baru: `lib/screens/home_screen.dart`, lalu **copy semua kode berikut**:

```dart
// =========================================================================
// [RESPONSI CHECKPOINT 3: HALAMAN UTAMA / MENU PILIHAN]
// JIKA STUDI KASUS BERUBAH:
// 1. Sesuaikan nama-nama menu (judul, deskripsi, icon) pada _buildMenuCard.
// 2. Sesuaikan endpoint yang dikirimkan ke ListScreen.
//    Contoh: 'articles', 'blogs', 'reports', 'characters', 'products', dsb
// 3. Tombol Logout sudah baku — tidak perlu diubah.
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
      Navigator.pushAndRemoveUntil(
        context,
        MaterialPageRoute(builder: (context) => const LoginScreen()),
        (route) => false,
      );
    }
  }

  Widget _buildMenuCard({
    required BuildContext context,
    required IconData icon,
    required String title,
    required String subtitle,
    required VoidCallback onTap,
  }) {
    return Card(
      elevation: 2,
      margin: const EdgeInsets.only(bottom: 16),
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      color: const Color(0xFFFBFBFE),
      child: InkWell(
        borderRadius: BorderRadius.circular(16),
        onTap: onTap,
        child: Padding(
          padding: const EdgeInsets.all(20),
          child: Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Container(
                padding: const EdgeInsets.all(12),
                decoration: BoxDecoration(
                  color: const Color(0xFFECEEFE),
                  borderRadius: BorderRadius.circular(12),
                ),
                child: Icon(icon, color: const Color(0xFF4C6EF5), size: 28),
              ),
              const SizedBox(width: 16),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(title,
                      style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold, color: Colors.black87)),
                    const SizedBox(height: 6),
                    Text(subtitle,
                      style: TextStyle(fontSize: 12, color: Colors.grey.shade600, height: 1.4)),
                  ],
                ),
              ),
            ],
          ),
        ),
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
        centerTitle: true,
        backgroundColor: const Color(0xFF4C6EF5),
        elevation: 0,
        automaticallyImplyLeading: false,
      ),
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(20.0),
          child: Column(
            children: [
              Expanded(
                child: ListView(
                  children: [
                    // [RESPONSI]: Menu 1 - Sesuaikan title, deskripsi, dan endpoint API
                    _buildMenuCard(
                      context: context,
                      icon: Icons.newspaper_outlined,
                      title: 'News',                        // <-- Ganti nama menu
                      subtitle: 'Lihat berita terbaru.',    // <-- Ganti deskripsi
                      onTap: () {
                        Navigator.push(context, MaterialPageRoute(
                          builder: (context) => const ListScreen(
                            menuTitle: 'News',
                            endpoint: 'articles',           // <-- GANTI endpoint di sini
                          ),
                        ));
                      },
                    ),

                    // [RESPONSI]: Menu 2 - Sesuaikan title, deskripsi, dan endpoint API
                    _buildMenuCard(
                      context: context,
                      icon: Icons.menu_book_outlined,
                      title: 'Blog',                        // <-- Ganti nama menu
                      subtitle: 'Kumpulan tulisan blog.',   // <-- Ganti deskripsi
                      onTap: () {
                        Navigator.push(context, MaterialPageRoute(
                          builder: (context) => const ListScreen(
                            menuTitle: 'Blog',
                            endpoint: 'blogs',              // <-- GANTI endpoint di sini
                          ),
                        ));
                      },
                    ),

                    // [RESPONSI]: Menu 3 - Sesuaikan title, deskripsi, dan endpoint API
                    _buildMenuCard(
                      context: context,
                      icon: Icons.analytics_outlined,
                      title: 'Reports',                     // <-- Ganti nama menu
                      subtitle: 'Data laporan misi.',       // <-- Ganti deskripsi
                      onTap: () {
                        Navigator.push(context, MaterialPageRoute(
                          builder: (context) => const ListScreen(
                            menuTitle: 'Reports',
                            endpoint: 'reports',            // <-- GANTI endpoint di sini
                          ),
                        ));
                      },
                    ),
                  ],
                ),
              ),
              const SizedBox(height: 12),
              // Tombol Logout (wajib ada, tidak perlu diubah)
              SizedBox(
                width: 140, height: 42,
                child: ElevatedButton(
                  style: ElevatedButton.styleFrom(
                    backgroundColor: Colors.red,
                    foregroundColor: Colors.white,
                    shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(22)),
                  ),
                  onPressed: () => _logout(context),
                  child: const Text('Log Out',
                    style: TextStyle(fontWeight: FontWeight.bold, fontSize: 15)),
                ),
              ),
              const SizedBox(height: 12),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## LANGKAH 9 — Buat `lib/screens/list_screen.dart`

Buat file baru: `lib/screens/list_screen.dart`, lalu **copy semua kode berikut**:

```dart
// =========================================================================
// [RESPONSI CHECKPOINT 4: HALAMAN LIST & FITUR FILTER]
// JIKA STUDI KASUS BERUBAH:
// 1. Pada _fetchData(): tentukan atribut apa yang dijadikan filter
//    Contoh: item.author, item.category, item.brand, item.clan
//    Ganti semua 'item.author' dengan field yang diinginkan.
// 2. Pada logika filter di build(): sesuaikan field perbandingannya.
// 3. Pada ListTile: sesuaikan field gambar, judul, dan subtitle.
// =========================================================================

import 'package:flutter/material.dart';
import '../models/article_model.dart';
import '../services/api_service.dart';
import 'detail_screen.dart';
import 'favorite_screen.dart';

class ListScreen extends StatefulWidget {
  final String menuTitle;
  final String endpoint;

  const ListScreen({
    super.key,
    required this.menuTitle,
    required this.endpoint,
  });

  @override
  State<ListScreen> createState() => _ListScreenState();
}

class _ListScreenState extends State<ListScreen> {
  List<ArticleModel> _allArticles = [];
  List<String> _filters = [];
  String _selectedFilter = 'All';
  bool _isLoading = true;
  String? _errorMessage;

  @override
  void initState() {
    super.initState();
    _fetchData();
  }

  Future<void> _fetchData() async {
    setState(() { _isLoading = true; _errorMessage = null; });

    try {
      final data = await ApiService.getArticles(widget.endpoint);

      // [RESPONSI]: Ekstraksi data unik untuk Filter Chip.
      // Ganti 'item.author' dengan field yang ingin dijadikan filter chip.
      final Set<String> filterSet = {};
      for (final item in data) {
        if (item.author.trim().isNotEmpty) {    // <-- Ganti item.author
          filterSet.add(item.author.trim());    // <-- Ganti item.author
        }
      }

      setState(() {
        _allArticles = data;
        _filters = ['All', ...filterSet];
        _isLoading = false;
      });
    } catch (e) {
      setState(() { _isLoading = false; _errorMessage = e.toString(); });
    }
  }

  @override
  Widget build(BuildContext context) {
    // [RESPONSI]: Logika filter — ganti 'item.author' dengan field yang sesuai
    final displayedArticles = _selectedFilter == 'All'
        ? _allArticles
        : _allArticles.where((item) => item.author == _selectedFilter).toList();
                                                 // ^^ Ganti item.author

    return Scaffold(
      backgroundColor: const Color(0xFFF7F8FC),
      appBar: AppBar(
        title: Text(widget.menuTitle,
          style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        backgroundColor: const Color(0xFF4C6EF5),
        elevation: 0,
        iconTheme: const IconThemeData(color: Colors.white),
        actions: [
          IconButton(
            tooltip: 'Halaman Favourite',
            icon: const Icon(Icons.favorite, color: Colors.white),
            onPressed: () {
              Navigator.push(context,
                MaterialPageRoute(builder: (context) => const FavoriteScreen()));
            },
          ),
        ],
      ),
      body: Column(
        children: [
          // Filter Chip Horizontal
          if (!_isLoading && _filters.isNotEmpty)
            Container(
              color: Colors.white,
              padding: const EdgeInsets.symmetric(vertical: 8),
              child: SingleChildScrollView(
                scrollDirection: Axis.horizontal,
                padding: const EdgeInsets.symmetric(horizontal: 16),
                child: Row(
                  children: _filters.map((filter) {
                    final bool isSelected = _selectedFilter == filter;
                    return Padding(
                      padding: const EdgeInsets.only(right: 8.0),
                      child: FilterChip(
                        selected: isSelected,
                        label: Text(filter),
                        labelStyle: TextStyle(
                          color: isSelected ? const Color(0xFF4C6EF5) : Colors.black87,
                          fontWeight: isSelected ? FontWeight.bold : FontWeight.normal,
                          fontSize: 13,
                        ),
                        backgroundColor: const Color(0xFFF1F3F9),
                        selectedColor: const Color(0xFFE4E9FE),
                        onSelected: (_) => setState(() => _selectedFilter = filter),
                        shape: RoundedRectangleBorder(
                          borderRadius: BorderRadius.circular(10),
                          side: BorderSide(
                            color: isSelected ? const Color(0xFF4C6EF5) : Colors.grey.shade300,
                          ),
                        ),
                      ),
                    );
                  }).toList(),
                ),
              ),
            ),

          Expanded(
            child: _isLoading
              ? const Center(child: CircularProgressIndicator())
              : _errorMessage != null
                ? Center(child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      const Icon(Icons.error_outline, size: 64, color: Colors.red),
                      const SizedBox(height: 12),
                      Text(_errorMessage!, textAlign: TextAlign.center),
                      const SizedBox(height: 16),
                      ElevatedButton(onPressed: _fetchData, child: const Text('Coba Lagi')),
                    ],
                  ))
                : ListView.builder(
                    padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
                    itemCount: displayedArticles.length,
                    itemBuilder: (context, index) {
                      final item = displayedArticles[index];
                      return Card(
                        elevation: 1.5,
                        margin: const EdgeInsets.only(bottom: 12),
                        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(14)),
                        color: const Color(0xFFFBFBFE),
                        child: InkWell(
                          borderRadius: BorderRadius.circular(14),
                          onTap: () {
                            Navigator.push(context, MaterialPageRoute(
                              builder: (context) => DetailScreen(
                                item: item,
                                endpoint: widget.endpoint,
                              ),
                            ));
                          },
                          child: Padding(
                            padding: const EdgeInsets.all(12.0),
                            child: Row(
                              children: [
                                // [RESPONSI]: Gambar thumbnail — field: item.imageUrl
                                ClipRRect(
                                  borderRadius: BorderRadius.circular(8),
                                  child: Image.network(
                                    item.imageUrl,              // <-- field gambar
                                    width: 80, height: 60,
                                    fit: BoxFit.cover,
                                    errorBuilder: (ctx, e, s) => Container(
                                      width: 80, height: 60,
                                      color: Colors.grey.shade200,
                                      child: const Icon(Icons.image_not_supported_outlined,
                                        color: Colors.grey),
                                    ),
                                    loadingBuilder: (ctx, child, prog) {
                                      if (prog == null) return child;
                                      return Container(width: 80, height: 60,
                                        color: Colors.grey.shade100,
                                        child: const Center(child: SizedBox(
                                          width: 18, height: 18,
                                          child: CircularProgressIndicator(strokeWidth: 2))));
                                    },
                                  ),
                                ),
                                const SizedBox(width: 14),
                                Expanded(
                                  child: Column(
                                    crossAxisAlignment: CrossAxisAlignment.start,
                                    children: [
                                      // [RESPONSI]: Judul item — field: item.title
                                      Text(item.title,          // <-- field judul
                                        maxLines: 2,
                                        overflow: TextOverflow.ellipsis,
                                        style: const TextStyle(
                                          fontSize: 14, fontWeight: FontWeight.bold, color: Colors.black87)),
                                      const SizedBox(height: 6),
                                      // [RESPONSI]: Subtitle — field: item.author
                                      Text(item.author,         // <-- field subtitle
                                        style: TextStyle(fontSize: 12, color: Colors.grey.shade600)),
                                    ],
                                  ),
                                ),
                              ],
                            ),
                          ),
                        ),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}
```

---

## LANGKAH 10 — Buat `lib/screens/detail_screen.dart`

Buat file baru: `lib/screens/detail_screen.dart`, lalu **copy semua kode berikut**:

```dart
// =========================================================================
// [RESPONSI CHECKPOINT 5: HALAMAN DETAIL, LAUNCH URL & FAVORITE]
// JIKA STUDI KASUS BERUBAH:
// 1. Sesuaikan field-field yang ditampilkan di body
//    (widget.item.title, .author, .publishedAt, .summary, dsb).
// 2. Tombol Favorite dan Hive sudah otomatis.
// 3. Tombol Buka Web menggunakan url_launcher — ganti widget.item.url jika field berbeda.
// =========================================================================

import 'package:flutter/material.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'package:url_launcher/url_launcher.dart';
import '../models/article_model.dart';

class DetailScreen extends StatefulWidget {
  final ArticleModel item;
  final String endpoint;

  const DetailScreen({super.key, required this.item, required this.endpoint});

  @override
  State<DetailScreen> createState() => _DetailScreenState();
}

class _DetailScreenState extends State<DetailScreen> {
  late Box _favoriteBox;
  bool _isFavorite = false;

  @override
  void initState() {
    super.initState();
    _favoriteBox = Hive.box('favorites');
    _checkFavoriteStatus();
  }

  void _checkFavoriteStatus() {
    setState(() {
      _isFavorite = _favoriteBox.containsKey(widget.item.id.toString());
    });
  }

  Future<void> _toggleFavorite() async {
    final key = widget.item.id.toString();
    if (_isFavorite) {
      await _favoriteBox.delete(key);
      setState(() => _isFavorite = false);
      if (mounted) ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Dihapus dari favorite'), duration: Duration(seconds: 1)));
    } else {
      await _favoriteBox.put(key, widget.item.toJson());
      setState(() => _isFavorite = true);
      if (mounted) ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Ditambahkan ke favorite'), duration: Duration(seconds: 1)));
    }
  }

  Future<void> _openOfficialSite() async {
    // [RESPONSI]: Ganti widget.item.url dengan field URL yang sesuai dari model
    if (widget.item.url.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('URL situs tidak tersedia'), backgroundColor: Colors.red));
      return;
    }

    final Uri uri = Uri.parse(widget.item.url);     // <-- Ganti .url dengan field URL model
    try {
      await launchUrl(uri, mode: LaunchMode.externalApplication);
    } catch (e) {
      if (mounted) ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error: $e'), backgroundColor: Colors.red));
    }
  }

  String _formatDate(String isoString) {
    if (isoString.isEmpty) return '-';
    try {
      final dt = DateTime.parse(isoString).toLocal();
      return '${dt.year}-${dt.month.toString().padLeft(2,'0')}-${dt.day.toString().padLeft(2,'0')} ${dt.hour.toString().padLeft(2,'0')}:${dt.minute.toString().padLeft(2,'0')}';
    } catch (_) { return isoString; }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.white,
      appBar: AppBar(
        title: const Text('Detail',
          style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        backgroundColor: const Color(0xFF4C6EF5),
        elevation: 0,
        iconTheme: const IconThemeData(color: Colors.white),
        actions: [
          IconButton(
            tooltip: _isFavorite ? 'Hapus dari Favorite' : 'Tambah ke Favorite',
            icon: Icon(_isFavorite ? Icons.favorite : Icons.favorite_border,
              color: _isFavorite ? Colors.red : Colors.white),
            onPressed: _toggleFavorite,
          ),
        ],
      ),
      body: Stack(
        children: [
          SingleChildScrollView(
            padding: const EdgeInsets.only(bottom: 80),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // [RESPONSI]: Gambar banner — field: item.imageUrl
                SizedBox(
                  width: double.infinity, height: 240,
                  child: Image.network(
                    widget.item.imageUrl,                 // <-- field gambar
                    fit: BoxFit.cover,
                    errorBuilder: (ctx, e, s) => Container(
                      color: Colors.grey.shade200,
                      child: const Center(child: Icon(Icons.broken_image_outlined,
                        size: 64, color: Colors.grey))),
                    loadingBuilder: (ctx, child, prog) {
                      if (prog == null) return child;
                      return Container(color: Colors.grey.shade100,
                        child: const Center(child: CircularProgressIndicator(strokeWidth: 2)));
                    },
                  ),
                ),
                Padding(
                  padding: const EdgeInsets.all(16.0),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      // [RESPONSI]: Judul — field: item.title
                      Text(widget.item.title,              // <-- field judul
                        style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold,
                          color: Colors.black87, height: 1.3)),
                      const SizedBox(height: 10),
                      // [RESPONSI]: Author / Kategori — field: item.author
                      Text(widget.item.author,             // <-- field author/kategori
                        style: const TextStyle(fontSize: 14, fontWeight: FontWeight.w600,
                          color: Colors.black54)),
                      const SizedBox(height: 4),
                      // [RESPONSI]: Tanggal — field: item.publishedAt
                      Text(_formatDate(widget.item.publishedAt),  // <-- field tanggal
                        style: TextStyle(fontSize: 12, color: Colors.grey.shade600)),
                      const SizedBox(height: 16),
                      const Divider(height: 1),
                      const SizedBox(height: 16),
                      // [RESPONSI]: Deskripsi — field: item.summary
                      Text(widget.item.summary.isNotEmpty   // <-- field deskripsi
                          ? widget.item.summary
                          : 'Tidak ada ringkasan yang tersedia.',
                        textAlign: TextAlign.justify,
                        style: const TextStyle(fontSize: 14, color: Colors.black87, height: 1.6)),
                    ],
                  ),
                ),
              ],
            ),
          ),
          // Tombol Buka Web (url_launcher)
          Positioned(
            bottom: 20, right: 20,
            child: FloatingActionButton.extended(
              backgroundColor: const Color(0xFFECEEFE),
              foregroundColor: const Color(0xFF4C6EF5),
              elevation: 3,
              icon: const Icon(Icons.open_in_new, size: 20),
              label: const Text('Buka Web', style: TextStyle(fontWeight: FontWeight.bold)),
              onPressed: _openOfficialSite,
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## LANGKAH 11 — Buat `lib/screens/favorite_screen.dart`

Buat file baru: `lib/screens/favorite_screen.dart`, lalu **copy semua kode berikut**:

```dart
import 'package:flutter/material.dart';
import 'package:hive_flutter/hive_flutter.dart';
import '../models/article_model.dart';
import 'detail_screen.dart';

class FavoriteScreen extends StatelessWidget {
  const FavoriteScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final Box favoriteBox = Hive.box('favorites');

    return Scaffold(
      backgroundColor: const Color(0xFFF7F8FC),
      appBar: AppBar(
        title: const Text('Favorite',
          style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        backgroundColor: const Color(0xFF4C6EF5),
        elevation: 0,
        iconTheme: const IconThemeData(color: Colors.white),
      ),
      // ValueListenableBuilder agar list reaktif otomatis saat data Hive berubah
      body: ValueListenableBuilder<Box>(
        valueListenable: favoriteBox.listenable(),
        builder: (context, box, child) {
          if (box.isEmpty) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Icon(Icons.favorite_border, size: 72, color: Colors.grey.shade400),
                  const SizedBox(height: 12),
                  Text('Belum ada data favourite',
                    style: TextStyle(fontSize: 16, color: Colors.grey.shade600,
                      fontWeight: FontWeight.w500)),
                ],
              ),
            );
          }

          final List<ArticleModel> favoriteList = box.values.map((val) {
            final map = Map<String, dynamic>.from(val as Map);
            return ArticleModel.fromJson(map);
          }).toList();

          return ListView.builder(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
            itemCount: favoriteList.length,
            itemBuilder: (context, index) {
              final item = favoriteList[index];
              return Card(
                elevation: 1.5,
                margin: const EdgeInsets.only(bottom: 12),
                shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(14)),
                color: const Color(0xFFFBFBFE),
                child: InkWell(
                  borderRadius: BorderRadius.circular(14),
                  onTap: () {
                    Navigator.push(context, MaterialPageRoute(
                      builder: (context) => DetailScreen(item: item, endpoint: item.type)));
                  },
                  child: Padding(
                    padding: const EdgeInsets.all(12.0),
                    child: Row(
                      children: [
                        ClipRRect(
                          borderRadius: BorderRadius.circular(8),
                          child: Image.network(
                            item.imageUrl,
                            width: 80, height: 60, fit: BoxFit.cover,
                            errorBuilder: (ctx, e, s) => Container(
                              width: 80, height: 60, color: Colors.grey.shade200,
                              child: const Icon(Icons.image_not_supported_outlined, color: Colors.grey, size: 28)),
                            loadingBuilder: (ctx, child, prog) {
                              if (prog == null) return child;
                              return Container(width: 80, height: 60, color: Colors.grey.shade100,
                                child: const Center(child: SizedBox(width: 18, height: 18,
                                  child: CircularProgressIndicator(strokeWidth: 2))));
                            },
                          ),
                        ),
                        const SizedBox(width: 14),
                        Expanded(
                          child: Column(
                            crossAxisAlignment: CrossAxisAlignment.start,
                            children: [
                              Text(item.title, maxLines: 2, overflow: TextOverflow.ellipsis,
                                style: const TextStyle(fontSize: 14, fontWeight: FontWeight.bold,
                                  color: Colors.black87)),
                              const SizedBox(height: 6),
                              Text(item.author,
                                style: TextStyle(fontSize: 12, color: Colors.grey.shade600)),
                            ],
                          ),
                        ),
                        IconButton(
                          tooltip: 'Hapus dari Favourite',
                          icon: const Icon(Icons.favorite, color: Colors.red),
                          onPressed: () async {
                            await box.delete(item.id.toString());
                            if (context.mounted) {
                              ScaffoldMessenger.of(context).showSnackBar(
                                const SnackBar(content: Text('Dihapus dari favorite'),
                                  duration: Duration(seconds: 1)));
                            }
                          },
                        ),
                      ],
                    ),
                  ),
                ),
              );
            },
          );
        },
      ),
    );
  }
}
```

---

## LANGKAH 12 — Edit `lib/main.dart`

**Ganti seluruh isi** `lib/main.dart` dengan kode berikut:

```dart
import 'package:flutter/material.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'screens/login_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  // Inisialisasi Hive database (Modul 6)
  await Hive.initFlutter();
  await Hive.openBox('favorites');

  runApp(const MainApp());
}

class MainApp extends StatelessWidget {
  const MainApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      // [RESPONSI]: Ganti judul aplikasi sesuai soal
      title: 'SpaceFlight News App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          // [RESPONSI]: Ganti warna tema sesuai soal
          seedColor: const Color(0xFF4C6EF5),
        ),
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

Aplikasi siap! Alur lengkap:
1. **Register** akun baru di halaman Register
2. **Login** dengan akun tersebut
3. Pilih menu (News / Blog / Reports) di Home
4. Lihat list data dengan Filter Chip
5. Tap item → Detail dengan tombol Buka Web & toggle Favorite
6. Lihat koleksi Favorite di halaman Favourite

---

## ⚡ Kalau Soal Responsi Berbeda API — Yang Harus Diedit

| Prioritas | File | Yang Diubah |
|-----------|------|------------|
| 🔴 Wajib | `lib/services/api_service.dart` | `baseUrl`, trailing slash, key wrapper JSON |
| 🔴 Wajib | `lib/models/article_model.dart` | Key `fromJson()` sesuai JSON API soal |
| 🔴 Wajib | `lib/screens/home_screen.dart` | Nama menu + `endpoint` tiap menu |
| 🟡 Perlu | `lib/screens/list_screen.dart` | Field filter chip (`item.author` → field lain) |
| 🟢 Opsional | `lib/screens/detail_screen.dart` | Field yang ditampilkan di body |

> Lihat file `PANDUAN_GANTI_API.md` di branch ini untuk panduan lengkap beserta skenario A–E.
