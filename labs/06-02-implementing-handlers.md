# Lab 6.2: Implementing Handlers

> Praktikum ini membahas penggunaan *Handlers* secara lanjutan. Kita akan menggunakan `meta: flush_handlers` untuk mengeksekusi *handler* di tengah *playbook*, menggunakan `listen` untuk mengelompokkan beberapa *handler*, dan mengotomatiskan *setup* database MariaDB (user, database, dan import data).

## 1. Tujuan Lab

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Memahami perbedaan kapan *handler* berjalan secara *default* (di akhir *play*) vs. dengan `meta: flush_handlers` (segera).
- Menggunakan fitur `listen` untuk memicu beberapa *handler* sekaligus dengan satu nama topik.
- Menggunakan modul `mysql_user` dan `mysql_db`.
- Menggunakan `no_log: yes` untuk menyembunyikan data sensitif di *output* terminal.

## 2. Konsep yang Digunakan

- **`notify` & Handlers:** *Task* yang mengubah sistem akan memicu *handler*.
- **`meta: flush_handlers`:** *Task* khusus yang memaksa semua *handler* yang tertunda untuk dieksekusi *saat ini juga*, sebelum melanjutkan ke *task* berikutnya.
- **`listen`:** *Keyword* pada *handler* yang memungkinkannya mendengarkan sebuah "topik". Jika ada *task* yang melakukan `notify: "nama topik"`, semua *handler* yang memiliki `listen: "nama topik"` akan berjalan.
- **`no_log: yes`:** Mencegah Ansible mencetak parameter *task* (seperti *password*) ke layar.
- **`mysql_db` (state: import):** Modul untuk meng-import file `.sql` ke dalam database.

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *playbook* berada. |
| **Managed Node** | `pod-username-managed1` | Target instalasi MariaDB. |

*(Catatan: `<username>` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu).*

## 4. Prasyarat

Jalankan perintah persiapan environment lab:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-username-controller:~$ nusactl start anadm-006-2
```

## 5. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-username-controller`)*

### Langkah 1 & 2: Membuat Workspace dan Konfigurasi

**Tujuan:** Menyiapkan direktori kerja dan file konfigurasi.

**Perintah:**
```bash
mkdir managing-handlers
cd managing-handlers/
vim ansible.cfg
```

**Isi File `ansible.cfg`:**
```ini
[defaults]
inventory=inventory
remote_user=student

[privilege_escalation]
become=True
become_ask_pass=False
```

**Perintah:**
```bash
vim inventory
```

**Isi File `inventory`:**
```ini
[managed]
pod-username-managed1
```

---

### Langkah 3: Membuat Playbook (`installing_mariaDB.yml`)

**Tujuan:** Menulis *playbook* dengan logika *handler* yang kompleks.

**Perintah:**
```bash
vim installing_mariaDB.yml
```

**Isi File `installing_mariaDB.yml`:**
```yaml
---
- name: Install mariaDB server
  hosts: managed
  vars:
    required_pkgs: 
      - mariadb-server
      - python3-pymysql     
    db_user: <username>
    db_pw: adinusa
    db_socket: /var/run/mysqld/mysqld.sock

  tasks:
    - name: "{{ required_pkgs }} packages are installed"
      apt:
        name: "{{ required_pkgs }}"
        state: present
      notify: restart service mariadb

    - name: task that force handlers to run imediately
      meta: flush_handlers

    - name: set mariaDB user and password
      mysql_user:
        login_unix_socket: "{{ db_socket }}"
        login_host: localhost
        login_user: root
        login_password: ''
        name: "{{ db_user }}"
        password: "{{ db_pw }}"
        priv: "*.*:ALL,GRANT"
        state: present
        host: localhost
      no_log: yes
      notify: restart service mariadb

    - name: add a data for database
      copy:
        src: ./dump.sql
        dest: /tmp/dump.sql      
      notify: 
      - "create database"
      - restart service mariadb

    - name: task that force handlers to run imediately
      meta: flush_handlers

    - name: check if DB exists
      shell: mysql --host=localhost --user={{ db_user }} --password={{ db_pw }} -e 'SHOW DATABASES;'
      register: dbstatus

    - name: show list existed databases
      debug:
        var: dbstatus.stdout_lines

  handlers:
    - name: restart service mariadb
      service:
        name: mariadb
        state: restarted
        enabled: yes
        
    - name: create a new database
      mysql_db:
        name: testdb
        state: present
        login_unix_socket: "{{ db_socket }}"
        login_user: "{{ db_user }}"
        login_password: "{{ db_pw }}"
      listen: "create database"
    
    - name: insert a data into database
      mysql_db: 
        name: testdb
        state: import
        target: /tmp/dump.sql
        login_unix_socket: "{{ db_socket }}"
        login_user: "{{ db_user }}"
        login_password: "{{ db_pw }}"
      listen: "create database"
```

**Penjelasan Kunci:**
- `meta: flush_handlers`: Digunakan dua kali. Pertama, untuk segera me-*restart* MariaDB setelah instalasi. Kedua, untuk segera membuat database setelah file `dump.sql` disalin.
- `notify: "create database"`: Ini memicu **dua** *handler* sekaligus (`create a new database` dan `insert a data into database`) karena keduanya menggunakan `listen: "create database"`.
- `no_log: yes`: Mencegah Ansible mencetak *password* `adinusa` di layar terminal saat *task* `mysql_user` berjalan.

---

### Langkah 4 & 5: Syntax Check dan Membuat `dump.sql`

**Tujuan:** Memastikan YAML valid dan membuat file SQL yang akan di-import.

**Perintah:**
```bash
ansible-playbook --syntax-check installing_mariaDB.yml
vim dump.sql
```

**Isi File `dump.sql`:**
```sql
CREATE TABLE IF NOT EXISTS test (
  message varchar(255) NOT NULL
) ENGINE=MyISAM DEFAULT CHARSET=utf8;

INSERT INTO test(message) VALUES('Adinusa - managing-handlers');
INSERT INTO test(message) VALUES('<username>');
INSERT INTO test(message) VALUES('Ansible is fun');
```
*(Ganti `<username>` dengan username Adinusa milikmu).*

---

### Langkah 6: Menjalankan Playbook

**Tujuan:** Mengeksekusi seluruh rangkaian otomatisasi database.

**Perintah:**
```bash
ansible-playbook installing_mariaDB.yml
```

**Hasil Terminal (Sesuai Sumber):**
```text
PLAY [Install mariaDB server] ****************************************************************

TASK [Gathering Facts] ***********************************************************************
ok: [pod-username-managed1]

TASK [['mariadb-server', 'python3-pymysql'] packages are installed] **************************
changed: [pod-username-managed1]

RUNNING HANDLER [restart service mariadb] ****************************************************
changed: [pod-username-managed1]

TASK [set mariaDB user and password] *********************************************************
changed: [pod-username-managed1]

TASK [add a data for database] ***************************************************************
changed: [pod-username-managed1]

RUNNING HANDLER [restart service mariadb] ****************************************************
changed: [pod-username-managed1]

RUNNING HANDLER [create a new database] ******************************************************
changed: [pod-username-managed1]

RUNNING HANDLER [insert a data into database] ************************************************
changed: [pod-username-managed1]

TASK [check if DB exists] ********************************************************************
changed: [pod-username-managed1]

TASK [show list existed databases] ***********************************************************
ok: [pod-username-managed1] => {
    "dbstatus.stdout_lines": [
        "Database",
        "information_schema",
        "mysql",
        "performance_schema",
        "testdb"
    ]
}

PLAY RECAP ***********************************************************************************
pod-username-managed1    : ok=10   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

**Analisis Output:**
1.  Perhatikan bahwa `RUNNING HANDLER [restart service mariadb]` muncul **tepat setelah** task instalasi (berkat `flush_handlers` pertama).
2.  Setelah file disalin, *handler* `restart service mariadb`, `create a new database`, dan `insert a data into database` berjalan berurutan. Ini membuktikan bahwa `listen: "create database"` berfungsi memicu dua *handler* sekaligus.

---

### Langkah 7: Verifikasi

**Tujuan:** Membuktikan database `testdb` sudah terbuat.

**Perintah:**
```bash
ssh pod-username-managed1
```
*(Di dalam Managed Node)*
```bash
mysql --host=localhost --user=<username> --password=adinusa -e 'SHOW DATABASES;'
```

**Hasil Terminal:**
```text
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| testdb             |
+--------------------+
```
*(Database `testdb` muncul di daftar).*

## 6. Validasi Akhir

Jika semua langkah telah berjalan sukses, jalankan perintah *grading*:

```bash
nusactl grade anadm-006-2
```

Setelah nilainya keluar, bersihkan environment lab:
```bash
nusactl finish anadm-006-2
```

## 7. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `ERROR! couldn't resolve module/action 'mysql_user'` | Koleksi `community.mysql` belum terinstal. | Di Ansible modern, modul ini butuh koleksi tambahan. Pastikan `python3-pymysql` sudah ada di target dan koleksi terinstal (atau gunakan modul `community.mysql.mysql_user`). |
| `Access denied for user 'root'@'localhost'` | Socket tidak ditemukan. | Pastikan path `db_socket` (`/var/run/mysqld/mysqld.sock`) benar. |
| Database gagal dibuat | Handler `listen` tidak terpanggil. | Cek apakah task `copy` berstatus `changed`. Jika tidak berubah, `notify` tidak akan memicu handler. |
| `import` gagal | File `dump.sql` tidak ada di `/tmp`. | Cek task `add a data for database`. Pastikan path `src: ./dump.sql` benar. |

## 8. Konsep yang Dipelajari

Lab ini mendemonstrasikan kekuatan penuh *Handlers*. Secara *default*, *handler* berjalan di akhir *play*. Namun, dengan `meta: flush_handlers`, kita bisa menyisipkannya di tengah alur kerja (misal: *restart* service sebelum mengonfigurasi *user*). Fitur `listen` memungkinkan kita membuat "kelompok aksi", di mana satu perintah `notify` dapat mengeksekusi beberapa *handler* yang berbeda secara berurutan. Ini sangat berguna untuk *deployment* yang kompleks (seperti *setup* database).

## 9. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Cek Isi Tabel:** Setelah database di-import, coba jalankan perintah `SELECT * FROM test;` di dalam database `testdb` untuk melihat apakah ketiga baris data (dari `dump.sql`) berhasil dimasukkan.
2.  **Eksplorasi `no_log`:** Hapus `no_log: yes` pada *task* `set mariaDB user and password`. Jalankan ulang *playbook* dengan `-v` (verbose). Kamu akan melihat *password* tercetak di layar (jangan pernah melakukan ini di lingkungan produksi!).

## 10. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 6.2: Implementing Handlers`.
- **Referensi Tambahan:**
  - [Ansible Handlers (Listen & Flush)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)
  - [Ansible meta Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/meta_module.html)
  - [Ansible mysql_db Module](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_db_module.html)
