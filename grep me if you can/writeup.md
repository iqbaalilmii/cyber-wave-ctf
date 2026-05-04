# CTF Writeup: Grep Me If You Can

**Category:** Forensics  
**Challenge:** 20GB of logs, 30 questions, 1 very annoying flag.  
**Flag:** `CW2026{l0gny4_g4j3l4s_sk1llny4_jug4}`

---

## Rules

- Submit answers without quotes
- Timestamps must be in Asia/Jakarta timezone
- Timestamp format: `YYYY-MM-DD HH:MM:SS`
- Paths and filenames are case-sensitive
- Domains are case-insensitive
- 5 attempts per question

---

## Overview

Serangan dimulai dari IP **45.148.10.73** yang melakukan reconnaissance pada endpoint upload avatar. Attacker berhasil mengupload webshell dengan ekstensi ganda `.php.jpg` untuk bypass validasi, kemudian mengeksekusi command, membaca konfigurasi aplikasi, melakukan dump database, dan mengeksfiltrasi data ke server eksternal sebelum menghapus jejak.

---

## Log Structure

```
evidence/
├── web/
│   ├── edge-nginx-01/access.log
│   └── app-web-01/
│       ├── laravel.log
│       ├── upload_handler.log
│       ├── app_debug.log
│       └── php-fpm.access.log
├── endpoint/
│   └── app-web-01/
│       ├── process_creation.jsonl
│       ├── file_events.jsonl
│       └── bash_history.log
├── proxy/
│   └── proxy_access.log
├── database/
│   └── db-main-01/
│       ├── mysql_audit.log
│       └── mysql_general.log
└── dns/
    └── dns_queries.log
```

---

## Attack Timeline

```
02:13:37 - Attacker (45.148.10.73) first probes upload endpoint (GET → 405)
02:15:08 - Successful webshell upload via POST (req-7f91ac2e)
02:16:02 - First webshell execution: ?z=id (req-c29bd880)
02:17:41 - Reads /srv/sagaramart/current/.env.production
02:21:36 - mysqldump partner_users → /dev/shm/.cache/.p_users.sql (PID 32177)
02:26:50 - tar -czf .sg_update_0421.tgz (PID 32331, 7348921 bytes)
02:28:17 - POST exfiltration → telemetry-cache-sync.com/api/v1/collect (167.99.182.44)
02:29:03 - rm -rf /dev/shm/.cache (cleanup)
```

---

## Analysis

### Phase 1: Initial Access

Attacker melakukan probe awal dengan GET request ke endpoint upload yang menghasilkan 405 (Method Not Allowed), kemudian berhasil mengupload webshell via POST.

```bash
grep -i "upload" web/edge-nginx-01/access.log | sort -k1 | head -10
```

File webshell menggunakan ekstensi ganda `ava_2b7c91.php.jpg` untuk bypass validasi tipe file. File disimpan di path penuh `/srv/sagaramart/current/public/uploads/avatars/ava_2b7c91.php.jpg`.

### Phase 2: Command Execution

Webshell diakses via HTTP GET dengan parameter `z` untuk mengeksekusi command. Command pertama adalah `id` untuk verifikasi privilege, berjalan sebagai `www-data` yang di-spawn oleh `php-fpm8.1`.

```bash
grep "ava_2b7c91\|z=" web/edge-nginx-01/access.log | head -20
```

### Phase 3: Credential Harvesting

Attacker membaca file `.env.production` yang berisi kredensial database: user `svc_partner_report` pada server `172.16.30.22`.

### Phase 4: Data Exfiltration

Attacker melakukan dump dua tabel database `partner_portal`, dikompres menjadi archive `.tgz`, lalu dieksfiltrasi ke domain `telemetry-cache-sync.com` (IP: `167.99.182.44`) menggunakan `aws-sdk-go/1.44` sebagai User-Agent untuk menyamarkan traffic sebagai AWS SDK.

```bash
# Temukan domain exfiltration dari proxy log
grep "2026-04-21T02:2[6-9]" proxy/proxy_access.log | grep -v "noise_blob" | cut -d' ' -f1-8

# Resolve IP dari DNS log
grep "telemetry-cache-sync" dns/dns_queries.log | head -10
```

### Phase 5: Cleanup

Setelah exfiltration berhasil, attacker menghapus seluruh staging directory dengan `rm -rf /dev/shm/.cache`.

---

## Answers

| # | Question | Answer |
|---|----------|--------|
| 1 | External IP that first accessed the vulnerable upload endpoint | `45.148.10.73` |
| 2 | First timestamp when that IP accessed the vulnerable upload endpoint | `2026-04-21 02:13:37` |
| 3 | Vulnerable endpoint path | `/api/v2/profile/avatar/upload` |
| 4 | request_id assigned to the successful malicious upload | `req-7f91ac2e` |
| 5 | Uploaded webshell filename | `ava_2b7c91.php.jpg` |
| 6 | SHA256 hash of the uploaded webshell | `8d6f5b7a4e3b0c8f19a01ab7e13d9dfeaf8d84f0ab6a2c3d6c99dbe119e4cc71` |
| 7 | HTTP parameter used to execute commands through the webshell | `z` |
| 8 | First command executed through the webshell | `id` |
| 9 | Linux user that executed the webshell commands | `www-data` |
| 10 | Application configuration file read by the attacker | `/srv/sagaramart/current/.env.production` |
| 11 | Database username used by attacker | `svc_partner_report` |
| 12 | Internal IP address of database server | `172.16.30.22` |
| 13 | Database name accessed by attacker | `partner_portal` |
| 14 | First table dumped by attacker | `partner_users` |
| 15 | PID of first mysqldump process | `32177` |
| 16 | Full path of first SQL dump file | `/dev/shm/.cache/.p_users.sql` |
| 17 | Full path of staging archive before exfiltration | `/dev/shm/.cache/.sg_update_0421.tgz` |
| 18 | Size in bytes of staging archive | `7348921` |
| 19 | Domain used for data exfiltration | `telemetry-cache-sync.com` |
| 20 | Proxy request_id for successful exfiltration | `pxy-62df7a11` |
| 21 | request_id for first confirmed webshell command execution | `req-c29bd880` |
| 22 | Full stored path of uploaded webshell | `/srv/sagaramart/current/public/uploads/avatars/ava_2b7c91.php.jpg` |
| 23 | Process that spawned attacker's command execution process | `php-fpm8.1` |
| 24 | MySQL connection_id of attacker's database session | `884213` |
| 25 | Process that created the staging archive | `tar` |
| 26 | PID of tar process that created staging archive | `32331` |
| 27 | Command that removed attacker's staging directory | `rm -rf /dev/shm/.cache` |
| 28 | Destination IP that received exfiltrated data | `167.99.182.44` |
| 29 | URI used for exfiltration POST request | `/api/v1/collect` |
| 30 | User-Agent on successful exfiltration request | `aws-sdk-go/1.44` |

---

## Key Commands Reference

```bash
# Q1-Q3: Find initial access on upload endpoint
grep -i "upload" web/edge-nginx-01/access.log | sort -k1 | head -10

# Q4-Q6: Webshell upload details
grep "req-7f91ac2e" web/app-web-01/upload_handler.log
grep "req-7f91ac2e" web/app-web-01/laravel.log

# Q7-Q8: Webshell execution parameter and first command
grep "ava_2b7c91\|z=" web/edge-nginx-01/access.log | head -20

# Q11-Q16: Database dump activity
grep -v "noise_blob" endpoint/app-web-01/process_creation.jsonl \
  | grep -i "mysqldump\|shm\|tgz"

# Q17-Q18: Staging archive metadata
grep "sg_update_0421" endpoint/app-web-01/file_events.jsonl

# Q19: Exfiltration domain discovery
grep "2026-04-21T02:2[6-9]" proxy/proxy_access.log \
  | grep -v "noise_blob" | cut -d' ' -f1-8

# Q20: Proxy request_id for exfiltration
grep "telemetry-cache-sync.com" proxy/proxy_access.log \
  | grep -o "request_id=[^ ]*"

# Q28: Resolve exfiltration destination IP
grep "telemetry-cache-sync" dns/dns_queries.log | head -10
```

---

## Flag

```
CW2026{l0gny4_g4j3l4s_sk1llny4_jug4}
```
