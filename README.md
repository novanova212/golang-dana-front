# Dana Clone — Frontend (Vue.js)

Frontend untuk aplikasi Dana Clone, dibangun dengan Vue 3 + Vite. Terhubung ke [backend Go](../danaclone-back) lewat REST API.

## ✨ Fitur

- Sign in / Sign up dengan desain mobile-style
- Dashboard dengan saldo, menu ikon (Top Up, Transfer, Split Bill, Minta Uang, Riwayat)
- Navigasi multi-halaman dengan bottom tab bar
- Top Up & Transfer saldo
- Split Bill — bagi rata atau custom per orang, dengan progress bar pelunasan
- Minta Uang / Tagih Teman — lengkap dengan aksi bayar/tolak
- Riwayat transaksi dengan pencarian & filter tipe
- Edit profil & ganti password

## 🛠️ Tech Stack

- **Vue 3** (Options API)
- **Vite** — build tool & dev server

## 🚀 Cara Menjalankan

### Prasyarat
- Node.js & npm
- [Backend](../danaclone-back) harus sudah jalan di `http://localhost:8080`

### Setup

```
npm install
npm run dev
```

Buka alamat yang muncul di terminal (biasanya `http://localhost:5173`).

## 📁 Struktur

Aplikasi ini sengaja dibuat sebagai **single component** (`src/App.vue`) untuk tahap belajar — semua state dan logic ada di satu file, navigasi antar "halaman" dikelola lewat state `currentPage` (bukan Vue Router), supaya konsep reactivity Vue lebih mudah dipahami sebelum menambah kompleksitas routing.

## 🎨 Desain

Tema visual (warna ungu-navy, tombol pil, kartu gradient) terinspirasi dari UI kit fintech mobile app.

## 📝 Pengembangan Selanjutnya

- Migrasi ke Vue Router untuk navigasi berbasis URL
- Pecah `App.vue` jadi komponen-komponen terpisah per halaman
- State management (Pinia) kalau aplikasi makin kompleks