`grep me if u can` 

oke jadi gw dapet first blood di chall ini somehow, mungkin karna filenya 20gb kali ya, makanya gaada yang mau ngerjain wkkwkw


pertama-tama, gw coba tree dulu di direktorinya, dan ini outputnya
<img width="1920" height="1128" alt="image" src="https://github.com/user-attachments/assets/823cdff4-b15b-4662-b42d-d2131e2ca543" />

okeh, langsung aja jawab pertanyaan-pertanyaannya, ada 30 soal anjay

`Question 1/30:
What external IP address first accessed the vulnerable upload endpoint?`

karena vulnnya berada di endpoint upload, saya grep "upload" ke file access log, dan ini outputnya
<img width="1920" height="1128" alt="image" src="https://github.com/user-attachments/assets/946b8490-a45c-498b-aab9-bac59cee012a" />

dari output ini, langsung keliatan jelas ip nya adalah 45.148.10.73

**Answer:** `45.148.10.73`

`Question 2/30:
What was the first timestamp when that IP accessed the vulnerable upload endpoint?`

pertanyaan ini juga bisa langsung dijawab dari output command diatas

**Answer:** `026-04-21 02:13:37`

`Question 3/30:
What was the vulnerable endpoint path?`

ini juga, endpoint pathnya adalah /api/v2/profile/avatar/upload

**Answer:** `/api/v2/profile/avatar/upload`

`Question 4/30:
What request_id was assigned to the successful malicious upload?`

dari output command diatas, request_idnya adalah req-7f91ac2e

**Answer:** `req-7f91ac2e`

`Question 5/30:
What was the uploaded webshell filename?`

nama file yang diupload attacker adalah ava_2b7c91.php.jpg

**Answer:** `ava_2b7c91.php.jpg`

`Question 6/30:
What is the SHA256 hash of the uploaded webshell?`

nah untuk pertanyaan yang ini, kita perlu grep dengan request ID dari POST yang tadi buat liat detail filenya dari sisi aplikasi

<img width="1920" height="1128" alt="image" src="https://github.com/user-attachments/assets/82c40995-9af0-40bd-9442-97984e0410ec" />

dari outputnya keliatan, sha256 hash dari file ava_2b7c91.php.jpg itu adalah 8d6f5b7a4e3b0c8f19a01ab7e13d9dfeaf8d84f0ab6a2c3d6c99dbe119e4cc71

**Answer:** `8d6f5b7a4e3b0c8f19a01ab7e13d9dfeaf8d84f0ab6a2c3d6c99dbe119e4cc71`

`Question 7/30:
What HTTP parameter was used by the attacker to execute commands through the webshell?`

dari output grep file access.log tadi, parameter http yang dipakai untuk execute commands adalah z (karna ?z=id)

**Answer:** `z`

`Question 8/30:
What was the first command executed through the webshell?`

ini juga, command pertama yang dijalanin attacker adalah id

**Answer:** `id`

`Question 9/30:
What Linux user executed the webshell commands?`

Untuk menemuin Linux user yang mengeksekusi webshell, saya mencoba grep berdasarkan timestamp setelah webshell diakses, tapi hasilnya noise semua. Saya akhirnya coba educated guess berdasarkann PHP-FPM pada server Linux umumnya berjalan di bawah beberapa user standar. Saya mencoba satu per satu, nginx, apache, lalu www-data dan www-data terbukti benar, yang kemudian dikonfirmasi oleh entry actor=www-data parent=php-fpm8.1 pada app_debug.log
<img width="1920" height="1128" alt="image" src="https://github.com/user-attachments/assets/502c92ba-7713-44fe-8897-31fa9492d01c" />

**Answer:** `www-data`

`Question 10/30:
What application configuration file was read by the attacker?`

dari output command diatas, path file configuration adalah /srv/sagaramart/current/.env.production

**Answer:** `/srv/sagaramart/current/.env.production`

`Question 11/30:
What database username did the attacker use?`

Dari struktur direktori evidence, keliatan folder database/db-main-01/ isinya file-file seperti mysql_audit.log, mysql_general.log, dan mysql_error.log, yang menunjukkan database yang digunakan adalah MySQL. Maka saya mencari jejak aktivitas MySQL dengan grep "mysql" di process_creation.jsonl, saya juga menggunakan argumen grep -v noise_blob agar tidak ada noise

<img width="1920" height="1128" alt="image" src="https://github.com/user-attachments/assets/82c84024-7a83-46a7-8e9d-1ac0c0077c08" />

terlihat www-data via php-fpm8.1 dump user database svc_partner_report

**Answer:** `svc_partner_report`

`Question 12/30:
What was the internal IP address of the database server?`

jawabannya ada di cmdline output diatas, yaitu, 
```
mysqldump -h 172.16.30.22 -u svc_partner_report partner_portal partner_users > /dev/shm/.cache/.p_users.sql
```
