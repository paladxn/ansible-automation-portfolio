# 06 - Ansible Administration III

> Modul ini membahas teknik tingkat lanjut dalam mengelola *playbook*. Kita akan mempelajari cara mengontrol apa yang terjadi ketika sebuah *task* gagal (*Error Handling*), cara mengelola *Handlers* dengan lebih presisi, serta pengenalan platform *Ansible Tower* dan *AWX* untuk manajemen skala *enterprise*.

## Daftar Isi
1. [Tujuan Pembelajaran](#1-tujuan-pembelajaran)
2. [Penanganan Kegagalan Task (Task Failure)](#2-penanganan-kegagalan-task-task-failure)
3. [Ansible Blocks dan Error Handling](#3-ansible-blocks-dan-error-handling)
4. [Menghentikan Play (Aborting the play)](#4-menghentikan-play-aborting-the-play)
5. [Implementasi Handlers](#5-implementasi-handlers)
6. [Ansible Tower dan AWX](#6-ansible-tower-dan-awx)
7. [Ringkasan](#7-ringkasan)
8. [Pertanyaan untuk Menguji Pemahaman](#8-pertanyaan-untuk-menguji-pemahaman)
9. [Referensi](#9-referensi)

---

## 1. Tujuan Pembelajaran

Setelah mempelajari materi ini, kamu diharapkan mampu:
- Menangani kegagalan *task* menggunakan berbagai fitur kontrol Ansible.
- Memahami dan menerapkan logika `block`, `rescue`, dan `always`.
- Mengimplementasikan *Handlers* dengan fitur `notify`, `listen`, dan `flush_handlers`.
- Menjelaskan perbedaan antara Ansible Tower dan Ansible AWX.

## 2. Penanganan Kegagalan Task (Task Failure)

Secara *default*, jika sebuah *task* gagal (mengembalikan status *error*), Ansible akan menghentikan eksekusi *playbook* pada *host* tersebut. Namun, ada kalanya kita ingin *playbook* tetap berjalan atau memberikan penanganan khusus.

### A. Mengabaikan Kegagalan (`ignore_errors`)
Jika kamu ingin *playbook* tetap melanjutkan eksekusi meskipun ada *task* yang gagal, gunakan *keyword* `ignore_errors`.

```yaml
- name: task that will error
  command: /bin/false
  ignore_errors: yes
```
*(Dengan `ignore_errors: yes`, Ansible akan mencatat task tersebut gagal, namun tetap melanjutkan ke task berikutnya).*

### B. Memaksa Eksekusi Handlers (`force_handlers`)
Jika kamu menggunakan `force_handlers: yes` di level *playbook* (sejajar dengan `hosts`), maka *handlers* yang sudah terpicu (*notified*) akan tetap dijalankan meskipun *play* dihentikan karena ada *task* yang gagal setelahnya.

```yaml
- hosts: all
  force_handlers: yes
  tasks:
    - name: a task which always notifies its handler
      command: /bin/true
      notify: restart service
      
    - name: a task which fails
      apt:
        name: fakePkg
        state: latest
  handlers:
    - name: restart service
      service:
        name: apache2
        state: restarted
```

### C. Menentukan Kondisi Kegagalan (`failed_when`)
Terkadang sebuah perintah sukses dieksekusi (exit code 0), tetapi *output*-nya menunjukkan adanya masalah. Kamu bisa menggunakan `failed_when` untuk mendefinisikan ulang apa yang dianggap sebagai "gagal".

```yaml
tasks:
  - name: check file
    shell: ls /home/student/ | wc -l
    register: output
    failed_when: output.stdout == "6"
```
*(Task di atas akan dianggap GAGAL jika jumlah file di direktori `/home/student/` adalah 6).*

### D. Memaksa Kegagalan (`fail` module)
Modul `fail` digunakan untuk secara sengaja menggagalkan sebuah *task* dengan pesan kustom. Ini berguna untuk membuat logika *validasi* di tengah *playbook*.

### E. Mengontrol Status "Changed" (`changed_when`)
Secara *default*, jika modul `command` atau `shell` berjalan, Ansible akan melaporkan status `changed`. Untuk mencegah hal ini (misal: saat hanya mengecek status), gunakan `changed_when: false`.

```yaml
tasks:
  - name: create file
    file:
      path: /home/student/file.txt
      state: touch
    changed_when: false
```

## 3. Ansible Blocks dan Error Handling

*Blocks* adalah klausa yang mengelompokkan *tasks* secara logis. Fitur ini sangat mirip dengan `try-catch-finally` di bahasa pemrograman lain.

- **`block`**: Mendefinisikan *tasks* utama yang akan dijalankan.
- **`rescue`**: Mendefinisikan *tasks* yang akan dijalankan **hanya jika** ada *task* di dalam `block` yang gagal.
- **`always`**: Mendefinisikan *tasks* yang akan **selalu dijalankan**, terlepas dari apakah `block` berhasil atau `rescue` dieksekusi.

**Contoh Implementasi:**
```yaml
tasks:
  - name: Handle the error with block
    block:
      - name: execute normally on block
        debug:
          msg: 'execute normally on block'
      - name: a task failure
        command: /bin/false
    rescue:
      - name: rescue task will execute because the task on block is failure
        debug:
          msg: 'rescue task will execute because the task on block is failure'
    always:
      - name: this task will always execute
        debug:
          msg: 'this task will always execute'
```

## 4. Menghentikan Play (Aborting the play)

Jika kamu ingin menghentikan seluruh eksekusi *playbook* pada semua *host* jika ada satu saja *host* yang gagal, gunakan opsi `any_errors_fatal`.

```yaml
- hosts: somehosts
  any_errors_fatal: true
  roles:
    - myrole
```

Untuk kontrol yang lebih spesifik, gunakan `max_fail_percentage`. Fitur ini akan membatalkan *run* jika persentase *host* yang gagal melebihi batas yang ditentukan.

## 5. Implementasi Handlers

*Handlers* adalah *tasks* khusus yang hanya merespons *notifikasi* dari *task* lain. *Handler* hanya berjalan jika *task* yang memicunya mengubah sesuatu (*status changed*).

### A. Dasar Notify dan Handlers
```yaml
tasks:
  - name: Copy the file
    template:
      src: index.html.j2
      dest: /var/www/html/index.html
    notify:
      - restart webserver

handlers:
  - name: restart webserver
    service:
      name: apache2
      state: restarted
```
*(Jika file `index.html` berubah, `notify` akan memicu handler `restart webserver` di akhir play).*

### B. Notifikasi ke Banyak Handlers
Satu *task* bisa memicu beberapa *handlers* sekaligus.

```yaml
tasks:
  - name: Example Task
    copy:
      content: "Example task"
      dest: /var/www/html/index.html
    notify:
      - restart service 1
      - restart service 2
```

### C. Menggunakan `listen` (Naming Handlers)
Fitur `listen` memungkinkan beberapa *handler* mendengarkan satu "topik" (nama grup). Ini sangat berguna jika kamu ingin me-*restart* banyak *service* sekaligus dengan satu perintah `notify`.

```yaml
tasks:
  - name: Restart all services web server
    command: echo "This task will restart all services web server"
    notify: "restart web server"

handlers:
  - name: restart apache2
    service:
      name: apache2
      state: restarted
    listen: "restart web server"

  - name: restart mariadb
    service:
      name: mariadb
      state: restarted
    listen: "restart web server"
```
*(Kedua handler di atas akan berjalan jika `notify` mengirim pesan "restart web server").*

### D. Kontrol Kapan Handlers Berjalan (`meta: flush_handlers`)
Secara *default*, *handlers* berjalan di akhir *play*. Jika kamu ingin *handlers* berjalan *sekarang* (sebelum *task* lain dieksekusi), gunakan modul `meta`.

```yaml
tasks:
  - name: First Task
    command: echo "First task"
    notify: restart mariadb

  - name: task that force handlers to run imediately
    meta: flush_handlers

  - name: Second Task
    command: echo "second task"
    notify: restart apache2
```
*(Dari contoh di atas, `restart mariadb` akan berjalan sebelum `Second Task`, sedangkan `restart apache2` akan berjalan di akhir play).*

## 6. Ansible Tower dan AWX

### Apa itu Ansible Tower?
Ansible Tower adalah versi *enterprise* dari Ansible. Ini adalah alat GUI komersial (berbayar) yang menyediakan antarmuka visual resmi, akses API, kontrol akses berbasis peran (RBAC), penjadwalan, dan notifikasi. Tower membantu organisasi menskalakan penggunaan Ansible secara efektif lintas tim.

### Apa itu Ansible AWX?
Ansible AWX adalah proyek *open-source* yang menjadi fondasi terciptanya Ansible Tower. AWX menyediakan antarmuka web, REST API, dan *task engine* di atas Ansible. 
Karena AWX adalah versi *upstream* (pengembangan), ia mendapat pembaruan lebih cepat namun dengan pengujian kualitas yang lebih minimal dibandingkan Tower.

**Perbedaan Utama Tower vs AWX:**
- **Lisensi:** Tower berbayar (ada trial gratis dengan batasan 10 node), AWX gratis sepenuhnya (*open-source*).
- **Dukungan:** Tower mendapat dukungan resmi dari Red Hat, AWX berbasis komunitas.
- **Fitur:** Pada dasarnya fiturnya sangat mirip (logo saja yang berbeda), namun AWX tidak memiliki batasan jumlah *node*.

### Prasyarat Ansible AWX (v17.0.1)
Untuk menjalankan AWX di *local environment*, kamu membutuhkan:
1.  **Ansible** versi 2.8+
2.  **Docker** versi terbaru
3.  **docker Python module** (tidak kompatibel dengan `docker-py`)
4.  **community.general.docker_image collection** (untuk Ansible >= 2.10)
5.  **GNU Make**
6.  **Git** versi 1.8.4+
7.  **Python** 3.6+
8.  **Node** 14.x LTS (hanya jika *build* image sendiri)
9.  **NPM** 6.x LTS (hanya jika *build* image sendiri)

## 7. Ringkasan

Menguasai *Error Handling* (`ignore_errors`, `failed_when`, `block/rescue`) membuat *playbook* kamu lebih tangguh (*robust*) dalam menghadapi berbagai kondisi lingkungan. Memahami *Handlers* tingkat lanjut (`listen`, `flush_handlers`) memungkinkan pengelolaan *service* yang lebih efisien. Terakhir, untuk penggunaan skala *enterprise* dengan GUI dan fitur kolaborasi, Ansible Tower (berbayar) atau AWX (gratis) adalah langkah selanjutnya setelah menguasai CLI.

## 8. Pertanyaan untuk Menguji Pemahaman

1.  **Konseptual:** Apa perbedaan mendasar antara `ignore_errors: yes` dengan penggunaan `block` dan `rescue`?
2.  **Praktis:** Jika kamu memiliki *playbook* yang mengecek *uptime* server menggunakan modul `command`, bagaimana cara mencegah Ansible melaporkan status `CHANGED` padahal kamu hanya ingin membaca data?
3.  **Teknis:** Jelaskan bagaimana cara kerja `listen` pada *handlers* dan kapan kamu harus menggunakannya dibandingkan `notify` secara langsung.
4.  **Analisis:** Mengapa organisasi *enterprise* lebih memilih Ansible Tower atau AWX dibandingkan hanya menggunakan Ansible CLI biasa?

## 9. Referensi

- **Sumber Utama:** Dokumen Adinusa Freecourse "Automation With Ansible" - `Ansible Administration III.docx`.
- **Referensi Tambahan:**
  - [Ansible Error Handling](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html)
  - [Ansible Blocks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html)
  - [Ansible Handlers](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)
  - [Ansible AWX GitHub](https://github.com/ansible/awx)
