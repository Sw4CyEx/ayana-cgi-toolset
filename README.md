# CGI Toolset — `*.ayana` scripts (bash / perl / python) + .htaccess helper

**Ringkasan:**
Kumpulan skrip CGI kecil (`bash.ayana`, `perl.ayana`, `py.ayana`) dan aturan `.htaccess` pendukung. Di repository ini tersedia dokumentasi lengkap: tujuan tiap file, cara instalasi ringan untuk environment berbasis Apache CGI, contoh penggunaan, langkah keamanan wajib, serta saran mitigasi jika ingin tetap menjalankan `UNSAFE` behavior (eval / system / popen) secara lebih aman.

> ⚠️ **PERINGATAN PENTING:** skrip-skrip ini menjalankan perintah yang dikirim klien (setelah decode base64). Itu adalah **REMOTE CODE EXECUTION** (RCE). Jangan taruh di server publik tanpa proteksi sangat kuat (token rahasia, IP allowlist, TLS, akses lokal saja, sandbox/container disposable). README ini menekankan mitigasi dan opsi yang lebih aman.

---

## Daftar isi
- [Isi repo](#isi-repo)
- [Deskripsi singkat file](#deskripsi-singkat-file)
- [Persyaratan sistem](#persyaratan-sistem)
- [Cara instalasi cepat (local/test)](#cara-instalasi-cepat-localtest)
- [Contoh penggunaan (client)](#contoh-penggunaan-client)
- [Konfigurasi Apache / .htaccess](#konfigurasi-apache--htaccess)
- [Keamanan — mitigasi lengkap](#keamanan----mitigasi-lengkap)
- [Debugging & logging](#debugging--logging)
- [Opsi safer (whitelist) — rekomendasi](#opsi-safer-whitelist----rekomendasi)
- [Lisensi & kredit](#lisensi--kredit)

---

## Isi repo
```
/ (root)
├─ bash.ayana             # original (UNSAFE) — Bash CGI
├─ perl.ayana             # original (UNSAFE) — Perl CGI
├─ py.ayana               # original (UNSAFE) — Python2 CGI
├─ .htaccess             # aturan contoh agar .ayana dianggap CGI
└─ README.md             # (file ini)
```

> Catatan: nama file `*.ayana` dipakai agar Apache/CGI menjalankan file ini (sesuai `AddHandler cgi-script .ayana`). Jangan letakkan file ini di direktori publik tanpa proteksi.

---

## Deskripsi singkat file

### `bash.ayana`
- Bash CGI yang membaca POST body, men-decode URL-encoding, men-decode base64 `cmd` dan `check`, menampilkan `check` dan kemudian `eval` hasil decode `cmd`.
- Behavior: **UNSAFE** — `eval` terhadap input klien.

### `perl.ayana`
- Perl CGI (one-liner di versi asli) yang membaca POST, meng-handle percent-decoding, menampilkan `check` (base64-decode) lalu `system` menjalankan `cmd` yang telah di-decode dari base64.
- Behavior: **UNSAFE** — `system()` terhadap input klien.

### `py.ayana`
- Python (2.x) CGI — memakai `cgi.FieldStorage` untuk POST, decode `check` & `cmd` via base64, lalu menjalankan command dengan `os.popen2`.
- Behavior: **UNSAFE** — menjalankan perintah pada sistem.

### `.htaccess`
- Contoh konfigurasi Apache untuk mengizinkan eksekusi CGI pada file berekstensi `.ayana`:

```apache
Options +FollowSymLinks +MultiViews +Indexes +ExecCGI
AddType application/x-httpd-cgi .ayana
AddHandler cgi-script .ayana
```

> Disarankan untuk menghapus `+Indexes` di server produksi dan membatasi direktori ini dengan `.htaccess` tambahan (auth, IP allowlist, dsb).

---

## Persyaratan sistem
- Web server: Apache dengan modul CGI/`mod_cgi` atau environment CGI compatible.
- Interpreter:
  - `bash` (umumnya di `/bin/bash` atau `/usr/bin/env bash`)
  - `perl`
  - `python` (catatan: versi skrip Python ditulis untuk Python 2 — gunakan environment Python 2 jika ingin menjalankan script asli)
- Perizinan file: executable bit (`chmod +x file.ayana`) dan owner yang sesuai (user CGI atau www-data) agar dapat dieksekusi.
- Jika Anda menyalin script beautified: editor/akses shell untuk membuat file.

---

## Cara instalasi cepat (local/test)
> **JANGAN** gunakan ini di host publik tanpa mitigasi. Untuk uji lokal di mesin development:

1. Tempatkan file `.ayana` di direktori CGI yang diizinkan (misal `/var/www/cgi-bin/` atau direktori publik yang diizinkan ExecCGI).
2. Pastikan `.ayana` executable:
```bash
chmod 750 bash.ayana perl.ayana py.ayana
chown www-data:www-data bash.ayana perl.ayana py.ayana   # sesuaikan user/group server
```
3. Jika menggunakan folder non-standard, tambahkan `.htaccess` sesuai contoh (hanya pada environment yang mengizinkan `.htaccess`).
4. Atur environment variabel bila diperlukan (mis. `CGI_SECRET`, `ENABLE_UNSAFE`, dsb.) jika kamu menambahkan mitigasi token di wrapper.

---

## Contoh penggunaan (client)
Contoh mengirim POST dengan `curl`. Perhatikan: `cmd` dan `check` harus base64 encoded (sesuai skrip).

**Contoh (bash) — hanya untuk lingkungan pengujian lokal, asumsi tidak ada token/allowlist:**

```bash
# contoh menjalankan `whoami`
CMD_B64=$(printf 'whoami' | base64)
CHECK_B64=$(printf 'cek-saya' | base64)
curl -X POST -d "cmd=${CMD_B64}&check=${CHECK_B64}" http://localhost/cgi-bin/bash.ayana
```

Response akan memuat isi `check` decode + output `whoami` di dalam `<pre>`.

---

## Konfigurasi Apache / .htaccess
Contoh `.htaccess` (beautified) untuk mengizinkan `.ayana` dijalankan:

```apache
# ============================================================
# .htaccess by Ayana
# ============================================================
Options +FollowSymLinks +ExecCGI
AddType application/x-httpd-cgi .ayana
AddHandler cgi-script .ayana
```

> Hapus `+Indexes` jika tidak ingin menampilkan listing folder. Untuk keamanan lebih ketat, tambahkan pembatasan IP atau Basic Auth di `.htaccess`.

Contoh pembatasan IP (tambahan) — ganti `1.2.3.4` dengan IP yang diizinkan:

```apache
Require ip 1.2.3.4
Require all denied
```
atau basic auth (sesuaikan user/pass file `.htpasswd`).

---

## Keamanan — mitigasi lengkap (WAJIB dibaca)
Skrip ini berbahaya jika diletakkan di server yang dapat diakses publik. Berikut langkah mitigasi yang harus dipertimbangkan:

1. **JANGAN** nyalakan endpoint ini di server publik tanpa proteksi.
2. Batasi akses:
   - IP allowlist (hanya 127.0.0.1 atau admin IP).
   - Basic Auth (htpasswd) + TLS.
   - Firewall (ufw/iptables) untuk menutup akses dari luar.
3. **Token rahasia**: skrip harus divalidasi dengan secret token (CGI param `token`) yang di-compare dengan env var `CGI_SECRET`. Token harus panjang & acak (>32 chars).
4. **ENABLE_UNSAFE flag**: gunakan environment variable seperti `ENABLE_UNSAFE=1` yang harus di-set di environment server agar `eval` / `system` boleh berjalan — default `0`.
5. **Logging & audit**: catat semua request (timestamp, remote IP, first line of command, token mask). Simpan log di lokasi yang aman.
6. **Run in sandbox**: jalankan skrip hanya di container (Docker) atau VM disposable sehingga kompromi tidak menyebar.
7. **Whitelist commands (direkomendasikan)**: alih-alih `eval`, buat routing perintah terbatas:
   - terima `action` bukan raw command,
   - map `action` ke skrip lokal atau command yang aman,
   - jangan `eval` input mentah.
8. **Gunakan user terbatas**: jalankan proses CGI sebagai user yang tidak punya akses sensitif (bukan root).
9. **Hapus atau nonaktif di production**: jika tidak perlu, jangan taruh endpoint ini di server production.

---

## Debugging & logging
- Periksa log Apache (`/var/log/apache2/error.log` atau `access.log`) jika CGI tidak berjalan.
- Pastikan file executable dan owner/group benar.
- Jika output kosong, cek `CONTENT_LENGTH` dan payload POST sesuai.
- Untuk debugging lokal, gunakan `curl` dan lihat output HTML yang berisi `<pre>` output script.

---

## Opsi safer (whitelist) — rekomendasi
Daripada `eval` atau `system` langsung, saya sangat menyarankan model *whitelist*:

1. Klien kirim `action` (ex: `action=backup`), bukan `cmd`.
2. Server memetakan `action` ke script lokal:
```bash
case "$action" in
  backup) /usr/local/bin/do_backup.sh ;;
  status) /usr/local/bin/show_status.sh ;;
  restart_service) /usr/local/bin/restart_service.sh ;;
  *) echo "Unknown action"; exit 1 ;;
esac
```
3. Validasi tambahan: HMAC/timestamp/signature token untuk mencegah replay.

Jika kamu mau, aku bisa bantu ubah salah satu file menjadi versi whitelist-safe.

---

## Lisensi & kredit
