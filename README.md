# Profil Mahasiswa & Catatan Kuliah

Aplikasi Flutter untuk menyimpan **profil mahasiswa** dan **catatan kuliah** secara lokal, dengan isi catatan yang **dienkripsi**, serta menampilkan data dari **API publik** (JSONPlaceholder). Proyek ini dibuat sebagai tugas Responsi 01.

![Flutter](https://img.shields.io/badge/Flutter-3.41-02569B?logo=flutter)
![Dart](https://img.shields.io/badge/Dart-%5E3.11-0175C2?logo=dart)
![Material 3](https://img.shields.io/badge/UI-Material%203-757575)

---

## Daftar Isi

- [Fitur](#fitur)
- [Teknologi & Dependency](#teknologi--dependency)
- [Arsitektur](#arsitektur)
- [Struktur Folder](#struktur-folder)
- [Penyimpanan Data](#penyimpanan-data)
- [Aspek Keamanan](#aspek-keamanan)
- [Validasi Input](#validasi-input)
- [Instalasi & Menjalankan](#instalasi--menjalankan)
- [Konfigurasi `.env`](#konfigurasi-env)
- [Pengujian](#pengujian)
- [Alur Penggunaan](#alur-penggunaan)

---

## Fitur

| Fitur | Keterangan |
|---|---|
| **Profil mahasiswa** | Menampilkan nama, NIM, program studi, dan email beserta avatar inisial. Profil dapat diedit lewat *bottom sheet*. |
| **Catatan kuliah (CRUD)** | Tambah, lihat detail, dan hapus catatan (dengan dialog konfirmasi). Daftar diurutkan dari yang terbaru. |
| **Enkripsi isi catatan** | Isi catatan disimpan di `flutter_secure_storage` (Android Keystore / iOS Keychain); hanya metadata yang disimpan di SharedPreferences. |
| **Data API publik** | Halaman *Data API Publik* mengambil daftar pengguna dari `GET /users` dengan *pull-to-refresh*. |
| **Penanganan error** | Loading, empty state, dan error state dengan tombol **Coba Lagi** — termasuk tidak ada koneksi, timeout, status ≠ 200, dan format JSON tidak valid. |
| **Validasi & sanitasi** | Semua form divalidasi secara *real-time* dan input dibersihkan dari karakter berbahaya. |
| **Konfigurasi via `.env`** | `BASE_URL` dan `API_KEY` tidak ditulis di kode. |

## Teknologi & Dependency

| Paket | Versi | Kegunaan |
|---|---|---|
| `flutter` (Material 3) | SDK | Framework UI |
| `shared_preferences` | ^2.5.6 | Menyimpan profil & metadata catatan |
| `flutter_secure_storage` | ^11.2.0 | Menyimpan isi catatan secara terenkripsi |
| `flutter_dotenv` | ^6.0.1 | Membaca konfigurasi dari `.env` |
| `http` | ^1.6.0 | Request HTTP ke API publik |
| `intl` | ^0.20.3 | Format tanggal catatan |
| `flutter_lints` | ^6.0.0 | Aturan *lint* (dev) |

## Arsitektur

Aplikasi memakai pola berlapis sederhana. UI tidak pernah mengakses storage secara langsung.

```
┌────────────────────────── UI (screens / widgets) ──────────────────────────┐
│  HomeScreen · NoteFormScreen · ProfileScreen · UsersScreen                 │
└──────────────┬───────────────────────────────────────────────┬─────────────┘
               │ AppProvider.of(context)                       │ FutureBuilder
               ▼                                               ▼
┌──── State: AppProvider (InheritedWidget) ────┐     ┌──── ApiService ────┐
│  student · notes · isLoading · errorMessage  │     │  fetchUsers()      │
│  updateStudent · addNote · deleteNote        │     │  → ApiException    │
└──────────────┬───────────────────────────────┘     └─────────┬──────────┘
               ▼                                               ▼
┌──── NoteRepository ──────────────────────────┐        JSONPlaceholder
│  getNotes · addNote · deleteNote             │        /users (HTTP GET)
│  getStudent · saveStudent                    │
└──────┬──────────────────────────┬────────────┘
       ▼                          ▼
 SharedPreferences        SecureStorageService
 (profil, metadata)       (isi catatan, terenkripsi)
```

- **State management**: `InheritedWidget` (`AppProvider`) yang state-nya dipegang `AppProviderWrapper` (StatefulWidget) — tanpa paket pihak ketiga.
- **Repository pattern**: `NoteRepository` menjadi satu-satunya pintu akses data lokal.
- **Dependency injection**: `NoteRepository`, `SecureStorageService`, `ApiService`, dan `http.Client` dapat diganti lewat konstruktor untuk pengujian.

## Struktur Folder

```
lib/
├── main.dart                          # Entry point: load .env lalu runApp
├── config/
│   └── app_config.dart                # BASE_URL, API_KEY, timeout, tema
├── models/
│   ├── api_user.dart                  # Model pengguna dari API
│   ├── note.dart                      # Model catatan (metadata + isi)
│   └── student.dart                   # Model profil mahasiswa
├── providers/
│   └── app_provider.dart              # State aplikasi (InheritedWidget)
├── repositories/
│   └── note_repository.dart           # Akses SharedPreferences + secure storage
├── services/
│   ├── api_service.dart               # HTTP GET /users + ApiException
│   └── secure_storage_service.dart    # Wrapper flutter_secure_storage
├── screens/
│   ├── home_screen.dart               # Ringkasan profil + daftar catatan
│   ├── note_form_screen.dart          # Form tambah catatan
│   ├── profile_screen.dart            # Profil lengkap + form edit
│   └── users_screen.dart              # Data dari API publik
├── utils/
│   └── validators.dart                # Validasi & sanitasi input
└── widgets/
    ├── empty_state.dart               # Tampilan kosong / error
    ├── note_card.dart                 # Kartu catatan
    └── profile_card.dart              # Kartu profil
test/
└── widget_test.dart                   # Unit test validator & model
```

## Penyimpanan Data

| Data | Lokasi | Key |
|---|---|---|
| Profil mahasiswa | SharedPreferences (JSON) | `student_profile` |
| Metadata catatan (`id`, `title`, `createdAt`) | SharedPreferences (JSON list) | `notes_metadata` |
| Isi catatan | Secure storage (terenkripsi) | `note_content_<id>` |

Saat menambah catatan, **isi disimpan lebih dulu** baru metadata, sehingga metadata tidak pernah menunjuk ke isi yang belum ada. Data yang rusak (JSON tidak valid) diabaikan dengan aman, bukan membuat aplikasi *crash*.

## Aspek Keamanan

- **Tidak ada credential di kode** — `BASE_URL` dan `API_KEY` dibaca dari `.env`, dan `.env` masuk `.gitignore`. Yang di-commit hanya `.env.example`.
- **Enkripsi data sensitif** — isi catatan hanya disimpan di secure storage; `Note.toMetadataJson()` sengaja tidak menyertakan `content`.
- **Log aman** — log hanya berisi ID, jumlah, path, dan status code. Isi catatan, header, API key, dan body respons tidak pernah dicetak. `Note.toString()` juga tidak menampilkan isi.
- **Sanitasi input** — karakter `<`, `>`, `"`, dan `'` dibuang, lalu input divalidasi ulang setelah sanitasi.
- **Pesan error ramah pengguna** — detail teknis tidak ditampilkan; error dibungkus `ApiException`.
- **Android** — `android:allowBackup="false"` agar data aplikasi tidak ikut backup otomatis, dan izin `INTERNET` ditambahkan untuk akses API.
- **Timeout request** — 10 detik (`AppConfig.requestTimeout`).

## Validasi Input

| Field | Aturan |
|---|---|
| Judul catatan | Wajib, 3–50 karakter |
| Isi catatan | Wajib, 10–1000 karakter |
| Nama | Wajib, 3–100 karakter |
| Program studi | Wajib, 2–100 karakter |
| NIM | Wajib, hanya angka dan titik (mis. `22.11.1234`), maks. 20 karakter |
| Email | Wajib, format `nama@domain.tld` |

## Instalasi & Menjalankan

### Prasyarat

- Flutter SDK 3.41 atau lebih baru (Dart ^3.11.5)
- Android Studio / VS Code dengan ekstensi Flutter
- Emulator Android, perangkat fisik, atau Chrome (web)

### Langkah

```bash
# 1. Clone repositori
git clone https://github.com/anugrahhhhh/responsi_profil_mahasiswa.git
cd responsi_profil_mahasiswa

# 2. Buat file .env dari contoh (WAJIB — .env terdaftar sebagai asset)
cp .env.example .env          # Windows PowerShell: Copy-Item .env.example .env

# 3. Install dependency
flutter pub get

# 4. Jalankan aplikasi
flutter run                   # atau: flutter run -d chrome
```

> **Penting:** file `.env` terdaftar di bagian `assets` pada `pubspec.yaml`. Jika file ini belum dibuat, proses build akan gagal karena asset tidak ditemukan.

### Build APK

```bash
flutter build apk --release
# hasil: build/app/outputs/flutter-apk/app-release.apk
```

## Konfigurasi `.env`

```env
BASE_URL=https://jsonplaceholder.typicode.com
API_KEY=isi-api-key-di-sini
```

| Variabel | Keterangan |
|---|---|
| `BASE_URL` | URL dasar API. Endpoint yang dipanggil: `${BASE_URL}/users`. Jika kosong, halaman API menampilkan pesan error. |
| `API_KEY` | API key (dummy) yang dikirim lewat header `x-api-key`. JSONPlaceholder tidak memerlukannya. |

## Pengujian

```bash
flutter test
flutter analyze
```

Unit test di `test/widget_test.dart` mencakup:

- Validasi judul (kosong, batas 3–50 karakter) dan isi catatan (batas 10–1000 karakter)
- Sanitasi karakter berbahaya
- Validasi email dan NIM
- Memastikan metadata catatan **tidak** menyimpan isi dan `toString()` tidak membocorkan isi
- Pembentukan inisial nama mahasiswa

## Alur Penggunaan

1. Buka aplikasi → halaman **Catatan Kuliah** menampilkan ringkasan profil (nama & NIM) dan daftar catatan.
2. Tekan **+ Catatan** → isi judul dan isi → **Simpan**.
3. Ketuk sebuah catatan untuk melihat isinya; tekan ikon tempat sampah untuk menghapus.
4. Ikon **orang** di AppBar → halaman **Profil Mahasiswa** → **Edit Profil** untuk mengubah data diri.
5. Ikon **awan** di AppBar → halaman **Data API Publik** untuk melihat daftar pengguna dari API; tarik ke bawah untuk memuat ulang.
