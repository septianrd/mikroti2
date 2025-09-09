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

Mari kita ambil contoh di sebuah **SMK** yang memiliki dua jurusan:  
- Jurusan **SIJA (Sistem Informatika Jaringan dan Aplikasi)** yang sering menggunakan internet untuk ujian online dan praktik jaringan.  
- Jurusan **Teknik Elektronika** yang menggunakan internet lebih ringan, misalnya untuk browsing atau mencari referensi.  

ISP sekolah memberikan bandwidth total **5 Mbps download** dan **3 Mbps upload**. Administrator sekolah ingin membaginya secara adil, dengan prioritas lebih untuk SIJA. Maka pembagian bandwidth diatur seperti ini:  
- **SIJA** → Download minimum 3 Mbps (limit-at), maksimum 5 Mbps. Upload minimum 2 Mbps, maksimum 3 Mbps.  
- **Teknik Elektronika** → Download minimum 2 Mbps, maksimum 3 Mbps. Upload minimum 1 Mbps, maksimum 2 Mbps.  

Untuk memisahkan trafik, administrator memberi range IP:  
- SIJA: `172.168.10.2 – 172.168.10.20`  
- Elektronika: `172.168.10.21 – 172.168.10.40`  

---

<img width="1920" height="1080" alt="Mikrotik 1 - Copy" src="https://github.com/user-attachments/assets/ccdfe931-6644-4539-bd0e-d3e28db89ef1" />

Oke, biar lebih naratif ya. Jadi bukan cuma list konfigurasi, tapi ceritanya mengalir seolah kita lagi jelasin di depan kamera saat bikin video olimpiade MikroTik. Tetap saya jaga validitas teknisnya supaya nggak ada salah konsep. Berikut versi naratifnya:

---

## Mangle dalam Studi Kasus

Langkah pertama dalam membuat *Queue Tree* adalah **memberi tanda pada trafik** menggunakan fitur **Mangle**. Kenapa perlu ditandai? Karena *Queue Tree* itu ibarat pos satpam: dia bisa mengatur kecepatan kendaraan, tapi hanya kalau tahu kendaraan itu milik siapa. Nah, Mangle inilah yang bertugas memberi stiker “ini punya SIJA” atau “ini punya Elektronika” pada setiap koneksi atau paket yang lewat. Tanpa tanda itu, *Queue Tree* tidak akan bisa membedakan dan akhirnya tidak bekerja sesuai harapan.

### Kenapa pakai **prerouting** untuk mark-connection?

Kita ingin menangkap trafik **sedini mungkin**, yaitu sebelum paket diarahkan ke chain lain dalam alur packet flow. Dengan memilih chain `prerouting`, setiap koneksi baru yang datang langsung diperiksa dan diberi label apakah dia milik SIJA atau Elektronika.

* Kalau alamat sumbernya `172.168.10.2–172.168.10.20`, kita beri label **con-sija**.
* Kalau alamatnya `172.168.10.21–172.168.10.40`, kita beri label **con-elektro**.

Karena kita menandai di level koneksi, maka seluruh paket yang termasuk dalam koneksi itu akan mewarisi label yang sama. Inilah kenapa kita pilih **mark-connection** di chain prerouting.

### Kenapa pakai **forward** untuk mark-packet?

Setelah koneksi sudah ditandai, giliran kita menandai paket. Nah, di sinilah peran chain `forward`. Semua trafik yang berjalan dari LAN menuju internet, maupun dari internet kembali ke LAN, pasti lewat chain forward. Jadi, entah paket itu sedang upload (keluar menuju internet) atau download (masuk dari internet), jalurnya tetap akan melalui forward.

Karena itulah marking paket paling tepat dilakukan di chain forward. Di sini kita bisa lebih spesifik:

* Kalau paket berasal dari koneksi dengan label **con-sija**, maka dia bisa ditandai lagi menjadi **sija-upload** atau **sija-download**, tergantung arah interfacenya.
* Untuk koneksi Elektronika, kita tandai menjadi **elektro-upload** atau **elektro-download**.

Di bagian **Out Interface**, kita bedakan:

* Kalau keluar lewat interface **wlan1** → berarti itu **upload**.
* Kalau keluar lewat interface **bridge1** → berarti itu **download**.

Terakhir, opsi **Passthrough** kita set **No**. Artinya, setelah paket diberi label, rule berhenti di situ. Ini mencegah label ganda atau tumpang tindih yang bisa bikin antrian rusak.

---

Jadi, singkatnya:

* **Prerouting → mark-connection**: supaya koneksi dari SIJA dan Elektronika langsung dapat identitas dari awal.
* **Forward → mark-packet**: karena semua trafik internet ↔ LAN lewat forward, di sinilah kita tandai arah upload dan download dengan jelas.

Dengan alur ini, *Queue Tree* sudah punya dasar yang jelas untuk membedakan trafik. Nanti tinggal kita bikin antrian (queue) sesuai labelnya.

---

## Queue Tree dalam Studi Kasus
Setelah trafik ditandai, barulah kita buat **Queue Tree**. Ingat: Queue Tree bekerja berdasarkan **packet mark**.  
Name: Nama queue yang sedang dibuat. Nama ini hanya sebagai identifikasi queue agar mudah dikenali.
Parent:Menentukan interface atau queue induk.
Packet Marks:Digunakan untuk memilih paket yang sudah ditandai (mark packet) menggunakan mangle di firewall.
Queue Type:Tipe antrean yang dipakai, misalnya default-small.
Default ini menggunakan algoritma untuk mengatur cara paket diproses
Priority:Menentukan prioritas dari queue ini.
Nilai dari 1 (tertinggi) sampai 8 (terendah).
Bucket Size:Besaran "penampungan sementara" untuk burst traffic.
Nilainya biasanya antara 0.1 – 1. Semakin kecil, semakin ketat pengaturannya.
Limit At: Kecepatan minimum yang dijamin.
Max Limit: Batas maksimal bandwidth.
Burst Limit: Kecepatan maksimum sementara saat ada bandwidth idle.
Burst Threshold: Ambang batas agar burst bisa berjalan.
Burst Time: Lama waktu burst bisa berlangsung.
OK / Cancel / Apply: Untuk menyimpan atau membatalkan konfigurasi.
Pertama, buat **inner queue (parent)**. Parent ini mewakili total bandwidth di interface. Misalnya:  
- Parent Download di `bridge1` dengan max-limit 5 Mbps.  
- Parent Upload di `bridge1` dengan max-limit 3 Mbps.  

Kenapa parent tidak diberi packet mark? Karena fungsinya hanya sebagai wadah utama, bukan pembagi trafik.  

Kedua, buat **leaf queue (child)** untuk tiap jurusan. Leaf queue inilah yang memakai packet mark dari mangle. Misalnya:  
- SIJA Download → packet mark `sija-download`, limit-at 3M, max-limit 5M, priority 1.  
- Elektronika Download → packet mark `elektro-download`, limit-at 2M, max-limit 3M, priority 8.  
- SIJA Upload → packet mark `sija-upload`, limit-at 2M, max-limit 3M, priority 1.  
- Elektronika Upload → packet mark `elektro-upload`, limit-at 1M, max-limit 2M, priority 8.  

### Penjelasan Parameter di Queue Tree
- **Limit-at** → bandwidth minimum yang dijamin, jadi meski jaringan padat, jurusan tetap dapat angka ini.  
- **Max-limit** → batas maksimum yang bisa dipakai, jika ada bandwidth tersisa.  
- **Priority** → menentukan siapa didahulukan saat rebutan bandwidth (1 lebih tinggi dari 8).  
- **Parent** → menempel pada inner queue agar tidak melewati batas total bandwidth ISP.  
- **Packet Mark** → harus diisi di leaf queue karena inilah identitas trafik.  
- **Queue Type, Bucket Size** → biasanya default, kecuali ada kebutuhan spesifik.  
- **Statistics** → dipakai untuk mengecek apakah trafik masuk ke antrian yang benar.  

---

## Kesimpulan
Dari studi kasus ini, kita bisa lihat bahwa **HTB lebih unggul daripada simple queue**. Dengan mangle, trafik dipisahkan jelas sesuai jurusan. Dengan queue tree, bandwidth bisa dibagi adil: SIJA mendapat prioritas lebih tinggi, Elektronika tetap mendapat jatah minimum. Inner queue memastikan bandwidth ISP tidak terlewati, sedangkan leaf queue memastikan pembagian detail ke tiap jurusan.  

---

## Penutup
Demikianlah penjelasan mengenai HTB pada MikroTik beserta penerapannya dalam studi kasus di SMK. Dengan memahami **mangle, inner queue, dan leaf queue**, administrator dapat mengatur bandwidth lebih adil dan efisien. Semoga bermanfaat dan bisa langsung dipraktikkan di jaringan nyata.  

Wassalamualaikum Warahmatullahi Wabarakatuh.
