# Tutorial HTB (Hierarchical Token Bucket) MikroTik
---

## Prolog
Assalamualaikum Warahmatullahi Wabarakatuh. Video tutorial ini dibuat oleh tim dari SMK Negeri 1 Cimahi dengan anggota Muhammad Haikal Al Yasir dan Septian Ramdani Solihin.  
Nomor peserta kami adalah 25030026683.
Pada kesempatan kali ini kami akan menjelaskan tentang manajemen bandwidth menggunakan **Hierarchical Token Bucket (HTB)** pada MikroTik.  

Dalam manajemen bandwidth di MikroTik, terdapat beberapa metode yang bisa digunakan. Salah satunya adalah Simple Queue, namun fitur ini hanya efektif jika jumlah pengguna sedikit; ketika user semakin banyak, pembagian bandwidth menjadi tidak konsisten, sulit mengatur prioritas, dan performanya menurun, sehingga pada kasus yang lebih kompleks digunakanlah HTB (Hierarchical Token Bucket) melalui queue tree, karena mampu membagi bandwidth secara adil, menetapkan prioritas trafik, serta menjaga efisiensi meski jumlah pengguna bertambah.  

<img width="1920" height="1080" alt="Mikrotik 2" src="https://github.com/user-attachments/assets/b6a62220-0702-4959-a69d-e78f77c07adc" />

Keunggulan HTB adalah pembagian bandwidth yang lebih fleksibel karena menggunakan **hierarki**. Dalam HTB ada dua istilah penting:  
- **Inner Queue (Parent)** → sebagai wadah utama yang berfungsi membatasi total bandwidth di interface. Parent ini tidak perlu packet mark karena fungsinya hanya menampung trafik.  

- **Leaf Queue (Child)** → antrian anak yang ditempelkan ke parent queue. Fungsinya adalah mengikat trafik tertentu (biasanya berdasarkan packet mark) agar bisa diatur pembagian bandwidthnya. 

  Dalam leaf queue ada dua parameter penting yang berhubungan dengan konsep **CIR** dan **MIR**:

  * **CIR (Committed Information Rate)** → bandwidth **minimum** yang dijamin akan diberikan ke trafik tersebut. Pada konfigurasi MikroTik, ini setara dengan **limit-at**. Jadi meskipun jaringan padat, trafik ini tetap mendapat alokasi sesuai nilai CIR.

  * **MIR (Maximum Information Rate)** → bandwidth **maksimum** yang boleh dipakai trafik tersebut ketika jaringan sedang longgar. Di MikroTik, ini setara dengan **max-limit**. Jadi trafik tidak bisa melebihi batas MIR meskipun kapasitas masih ada.

Dengan kata lain, **Leaf Queue** berfungsi sebagai pengatur bandwidth berbasis prioritas dan batasan. CIR memastikan ada **jaminan minimum**, sementara MIR membatasi agar tidak ada trafik yang **memonopoli seluruh bandwidth**.


---

## Studi Kasus
<img width="1920" height="1080" alt="Mikrotik 3" src="https://github.com/user-attachments/assets/686d01fd-8274-4b47-b848-e90adf0c838c" />
---



## Mangle dalam Studi Kasus

Langkah pertama kita **menandai trafik** dengan membuat rule `mark-connection` di chain **prerouting** — kita pilih `prerouting` karena di situ paket ditangani sedini mungkin, sebelum ada keputusan routing, sehingga setiap koneksi baru langsung bisa dikenali sejak awal.
Selanjutnya pada kolom **Src Address** kita isi range IP untuk jurusan SIJA, misalnya `172.168.10.2-172.168.10.5`; kolom lain seperti Dst Address, Protocol, Port atau Interface dibiarkan kosong supaya rule ini berlaku umum untuk semua trafik dari sumber itu (jangan mempersempit kecuali memang diperlukan).

Di tab **Advanced** sebenarnya tersedia pilihan seperti address-list atau layer7, dan di tab **Extra** ada filter teknis (TTL, packet-size), namun pada kasus pembagian berdasarkan IP sederhana ini kita **tidak mengisinya** — singkatnya: fungsi ada, tapi tidak dipakai karena tidak diperlukan.
Pada tab **Action** kita pilih **mark-connection** dan beri nama misalnya `con-sija`; opsi **Passthrough = Yes** diaktifkan supaya rule berikutnya (yang akan menandai paket) tetap diproses. Alasan mark-connection di level koneksi adalah agar semua paket yang termasuk dalam satu koneksi otomatis mewarisi label yang sama. Setelah itu ulangi prosedur yang sama untuk jurusan lain: buat rule `prerouting` baru. 

### Mark Packet
Setelah selesai membuat dua rule **mark-connection**, tahap berikutnya adalah membuat rule **mark-packet**. Di sini kita menggunakan chain **forward**, bukan prerouting lagi. Alasannya jelas: semua trafik yang berjalan dari LAN menuju internet maupun dari internet ke LAN selalu melewati chain forward. Jadi di titik ini kita bisa membedakan arah paket — apakah sedang upload atau download.

Pada bagian **Connection Mark**, kita isi dengan label yang sudah dibuat sebelumnya, misalnya `con-sija`. Dengan begitu hanya trafik milik koneksi tersebut yang akan diproses oleh rule ini.

Kemudian kita manfaatkan kolom **In. Interface** dan **Out. Interface** untuk menentukan arah trafik:

* Jika trafik berasal dari **bridge1** (LAN) menuju **wlan1** (internet), berarti ini **upload**. Action → `mark-packet`, beri nama misalnya `sija-upload`.
* Jika trafik berasal dari **wlan1** menuju **bridge1**, berarti ini **download**. Action → `mark-packet`, beri nama `sija-download`.

Tab **Advanced** dan **Extra** kembali tidak perlu digunakan karena kita hanya membedakan arah trafik berdasarkan interface, bukan berdasarkan detail lain seperti address list, TTL, atau packet size.

Terakhir, pada tab **Action**, kita set **Passthrough = No**. Alasannya, setelah paket diberi tanda (packet-mark), rule ini selesai bekerja dan tidak perlu dilanjutkan ke rule lain agar tidak terjadi pelabelan ganda yang bisa mengacaukan antrian di Queue Tree.

Langkah ini kemudian diulang dengan cara yang sama untuk koneksi jurusan Elektronika (`con-elektro`). Kita buat dua rule: satu untuk `elektro-upload`, satu lagi untuk `elektro-download`. Dengan begitu, setiap paket dari masing-masing jurusan sudah jelas arahnya dan sudah siap diatur bandwidth-nya di Queue Tree.

---

## Queue Tree dalam Studi Kasus
<img width="1920" height="1080" alt="Mikrotik 1 - Copy" src="https://github.com/user-attachments/assets/ccdfe931-6644-4539-bd0e-d3e28db89ef1" />
Setelah semua trafik berhasil ditandai menggunakan **Mangle**, langkah berikutnya adalah membuat **Queue Tree**. 
Perlu diingat bahwa Queue Tree hanya bekerja berdasarkan **packet mark** yang sudah kita buat sebelumnya. 
Secara umum, Queue Tree memiliki beberapa parameter penting:

- **Name** → Nama queue agar mudah dikenali.
- **Parent** → Menentukan interface atau queue induk (inner queue).
- **Packet Mark** → Mengacu pada hasil marking di Mangle.
- **Queue Type** → Tipe antrean (misalnya `default-small`) yang menentukan cara paket diproses.
- **Priority** → Skala 1–8 (1 tertinggi) untuk menentukan siapa yang lebih diprioritaskan.
- **Bucket Size** → Ukuran buffer sementara untuk burst traffic (umumnya 0.1–1).
- **Limit At** → Bandwidth minimum yang dijamin.
- **Max Limit** → Bandwidth maksimum yang bisa digunakan.
- **Burst** → Parameter tambahan untuk memberi kecepatan ekstra sementara saat bandwidth tidak penuh.
- **Statistics** → Tab untuk mengecek apakah trafik sudah masuk ke queue yang benar.

### Langkah Pertama: Inner Queue (Parent)
Pertama kita buat **inner queue** sebagai induk, yang berfungsi sebagai wadah total bandwidth. 
Inner queue tidak menggunakan **packet mark** karena tugasnya hanya membatasi total kapasitas sesuai bandwidth ISP. 
Misalnya:
- Parent Download di **bridge1** dengan `max-limit = 5M`.
- Parent Upload di **wlan1** dengan `max-limit = 3M`.

### Langkah Kedua: Leaf Queue (Child)
Setelah parent dibuat, kita lanjut ke **leaf queue**. Leaf queue inilah yang memakai **packet mark** dari Mangle untuk membagi bandwidth sesuai jurusan. Contohnya:

- **SIJA Download** → packet mark `sija-download`, limit-at `3M`, max-limit `5M`, priority `1`.
- **Elektronika Download** → packet mark `elektro-download`, limit-at `2M`, max-limit `3M`, priority `8`.
- **SIJA Upload** → packet mark `sija-upload`, limit-at `2M`, max-limit `3M`, priority `1`.
- **Elektronika Upload** → packet mark `elektro-upload`, limit-at `1M`, max-limit `2M`, priority `8`.

### Kenapa Dibedakan?
- **Parent tidak diberi packet mark** karena hanya sebagai wadah total bandwidth.  
- **Leaf queue wajib pakai packet mark** agar trafik bisa dipisahkan sesuai jurusan.  
- **Limit-at dan Max-limit** menjamin pembagian minimum, tetapi tetap fleksibel jika ada bandwidth tersisa.  
- **Priority** mengatur siapa yang lebih dulu dilayani saat bandwidth padat (SIJA lebih tinggi dari Elektronika).  
- **Statistics** akhirnya digunakan untuk memverifikasi apakah paket benar-benar masuk ke queue sesuai label yang diberikan.  

---

## Kesimpulan
Dari studi kasus ini, kita bisa lihat bahwa **HTB lebih unggul daripada simple queue**. Dengan mangle, trafik dipisahkan jelas sesuai jurusan. Dengan queue tree, bandwidth bisa dibagi adil: SIJA mendapat prioritas lebih tinggi, Elektronika tetap mendapat jatah minimum. Inner queue memastikan bandwidth ISP tidak terlewati, sedangkan leaf queue memastikan pembagian detail ke tiap jurusan.  

---

## Penutup
Demikianlah penjelasan mengenai HTB pada MikroTik beserta penerapannya dalam studi kasus di SMK. Dengan memahami **mangle, inner queue, dan leaf queue**, administrator dapat mengatur bandwidth lebih adil dan efisien. Semoga bermanfaat dan bisa langsung dipraktikkan di jaringan nyata.  

Wassalamualaikum Warahmatullahi Wabarakatuh.
