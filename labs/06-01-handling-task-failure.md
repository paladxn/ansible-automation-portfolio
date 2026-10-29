# Lab 6.1: Handling Task Failure

> Praktikum ini membahas cara mengelola kegagalan (*task failure*) di Ansible. Kita akan mensimulasikan *error* dengan nama paket yang salah, lalu menggunakan berbagai fitur Ansible (`ignore_errors`, `block/rescue/always`, `changed_when`, dan `failed_when`) untuk mengontrol bagaimana *playbook* bereaksi terhadap kegagalan tersebut.

## 1. Tujuan Lab

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Memahami perilaku *default* Ansible saat sebuah *task* gagal.
- Menggunakan `ignore_errors` agar *playbook* tetap berlanjut meski ada *error*.
- Menerapkan struktur `block`, `rescue`, dan `always` untuk penanganan *error* layaknya `try-catch-finally`.
- Menggunakan `changed_when` untuk menekan status `CHANGED` pada *task* yang hanya membaca data.
- Menggunakan `failed_when` untuk mendefinisikan ulang kondisi kegagalan secara kustom.

## 2. Konsep yang Digunakan

- **`ignore_errors: yes`**: Ansible akan mencatat *task* sebagai gagal (`failed=1`), tetapi tetap melanjutkan ke *task* berikutnya.
- **`block` / `rescue` / `always`**: 
  - `block`: Grup *task* utama.
  - `rescue`: Hanya berjalan jika *task* di `block` gagal.
  - `always`: Selalu berjalan, baik `block` sukses maupun gagal.
- **`changed_when: false`**: Mencegah Ansible melaporkan status `changed` (kuning) pada *task* yang sebenarnya hanya mengecek sesuatu (misal: `command` atau `shell`).
- **`failed_when`**: Mendefinisikan kondisi khusus agar sebuah *task* dianggap gagal, meskipun perintah aslinya berhasil (exit code 0).

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *playbook* ditulis dan dijalankan. |
| **Managed Node** | `pod-username-managed1` | Target instalasi (tergabung dalam grup `managed`). |

*(Catatan: `username` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu).*

## 4. Prasyarat

Jalankan perintah persiapan environment lab:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-username-controller:~$ nusactl start anadm-006-1
```

## 5. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-username-controller`)*

### Langkah 1 & 2: Membuat Workspace dan Konfigurasi

**Tujuan:** Menyiapkan direktori kerja dan file konfigurasi. Perhatikan blok `[privilege_escalation]` yang baru.

**Perintah:**
```bash
mkdir managing-task-failure
cd managing-task-failure/
vim ansible.cfg
```

**Isi File `ansible.cfg`:**
```ini
[defaults]
inventory=inventory
remote_user=student

[privilege_escalation]
become=True
become_ask_pass=false
```
*(Penjelasan: Daripada menulis `become: true` di setiap *playbook*, kita mendefinisikannya secara global di `ansible.cfg`).*

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

### Langkah 3 & 4: Simulasi Kegagalan (Default Behavior)

**Tujuan:** Membuat *playbook* dengan paket yang salah (`apache` bukan `apache2`) untuk melihat bagaimana Ansible berhenti saat *error*.

**Perintah:**
```bash
vim task-failure.yml
```

**Isi File `task-failure.yml` (Tahap 1):**
```yaml
---
- name: Lab Task Failure 
  hosts: managed
  vars:
    web_pkg: apache
    db_pkg: mariadb-server
    db_svc: mariadb

  tasks:
    - name: Install {{ web_pkg }} package
      apt:
        name: "{{ web_pkg }}"
        state: present

    - name: Install {{ db_pkg }} package
      apt:
        name: "{{ db_pkg }}"
        state: present
```

**Jalankan:**
```bash
ansible-playbook task-failure.yml
```
**Hasil Terminal:**
```text
TASK [Install apache package] ************************************************************
fatal: [pod-username-managed1]: FAILED! => {"changed": false, "msg": "No package matching 'apache' is available"}
...
pod-username-managed1    : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
```
*(Analisis: Karena task pertama gagal, task kedua `mariadb-server` tidak dieksekusi. `failed=1`).*

---

### Langkah 5 & 6: Mengabaikan Kegagalan (`ignore_errors`)

**Tujuan:** Memaksa Ansible untuk tetap melanjutkan eksekusi meskipun task pertama gagal.

**Ubah `task-failure.yml` (Tahap 2):**
```yaml
  tasks:
    - name: Install {{ web_pkg }} package
      apt:
        name: "{{ web_pkg }}"
        state: present
      ignore_errors: yes

    - name: Install {{ db_pkg }} package
      apt:
        name: "{{ db_pkg }}"
        state: present
```

**Jalankan:**
```bash
ansible-playbook task-failure.yml
```
**Hasil Terminal:**
```text
TASK [Install apache package] ************************************************************
fatal: [pod-username-managed1]: FAILED! => {"changed": false, "msg": "No package matching 'apache' is available"}
...ignoring

TASK [Install mariadb-server package] ****************************************************
changed: [pod-username-managed1]
...
pod-username-managed1    : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=1   
```
*(Analisis: Task pertama gagal tapi di-*ignore* (`...ignoring`). Task kedua berhasil. Status akhir `failed=0` dan `ignored=1`).*

---

### Langkah 7 & 8: Menggunakan `block`, `rescue`, dan `always`

**Tujuan:** Mengganti `ignore_errors` dengan penanganan *error* yang lebih terstruktur.

**Ubah `task-failure.yml` (Tahap 3):**
```yaml
  tasks:
    - name: Set up webserver
      block:
        - name: Install {{ web_pkg }} package
          apt:
            name: "{{ web_pkg }}"
            state: present
      rescue:
        - name: Install {{ db_pkg }} package
          apt:
            name: "{{ db_pkg }}"
            state: present
      always:
        - name: Start {{ db_svc }} service
          service:
            name: "{{ db_svc }}"
            state: started
```

**Jalankan:**
```bash
ansible-playbook task-failure.yml
```
**Hasil Terminal:**
```text
TASK [Install apache package] ************************************************************
fatal: [pod-username-managed1]: FAILED! => {"changed": false, "msg": "No package matching 'apache' is available"}

TASK [Install mariadb-server package] ****************************************************
ok: [pod-username-managed1]

TASK [Start mariadb service] *************************************************************
ok: [pod-username-managed1]
...
pod-username-managed1    : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=1    ignored=0   
```
*(Analisis: `apache` gagal di dalam `block`. Karena gagal, blok `rescue` dieksekusi (`mariadb-server`). Terakhir, blok `always` (`start mariadb`) dieksekusi. Status `rescued=1`).*

---

### Langkah 9 & 10: Memperbaiki Error dan Mengamati Alur Sukses

**Tujuan:** Memperbaiki nama paket menjadi `apache2` dan melihat bagaimana `rescue` dilewati.

**Ubah `task-failure.yml` (Tahap 4 - bagian vars):**
```yaml
  vars:
    web_pkg: apache2
    db_pkg: mariadb-server
    db_svc: mariadb
```

**Jalankan:**
```bash
ansible-playbook task-failure.yml
```
*(Hasil: `Install apache2` berhasil (`changed`), `rescue` dilewati, dan `always` (`start mariadb`) tetap berjalan).*

---

### Langkah 11: Mengontrol Status "Changed" (`changed_when`)

**Tujuan:** Menekan status `changed` pada perintah yang hanya membaca data (`hostnamectl`).

**Ubah `task-failure.yml` (Tambahkan task di awal `tasks:` sebelum `block`):**
```yaml
  tasks:
    - name: execute command
      command: hostnamectl
      register: Result
      # changed_when: false (Tambahkan ini di percobaan kedua)

    - name: print command
      debug:
        var: Result.stdout_lines
```

**Jalankan (Tanpa `changed_when`):**
```bash
ansible-playbook task-failure.yml
```
**Hasil:** `TASK [execute command]` berstatus `changed`.

**Tambahkan `changed_when: false` pada task `execute command` dan jalankan lagi.**
**Hasil:** `TASK [execute command]` sekarang berstatus `ok` (bukan `changed`).

---

### Langkah 12: Memaksa Kegagalan Kustom (`failed_when`)

**Tujuan:** Membuat task yang sebenarnya berhasil mengeksekusi perintah, tetapi *melaporkan* status gagal berdasarkan kondisi tertentu.

**Ubah `task-failure.yml` (Tambahkan `failed_when` pada Install apache2):**
```yaml
    - name: Set up webserver
      block:
        - name: Install {{ web_pkg }} package
          apt:
            name: "{{ web_pkg }}"
            state: present
          failed_when: web_pkg == "apache2" 
```

**Jalankan:**
```bash
ansible-playbook task-failure.yml
```
**Hasil Terminal:**
```text
TASK [Install apache2 package] ***********************************************************
fatal: [pod-username-managed1]: FAILED! => {"cache_update_time": 1661402418, "cache_updated": false, "changed": false, "failed_when_result": true}

TASK [Install mariadb-server package] ****************************************************
ok: [pod-username-managed1]
...
pod-username-managed1    : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=1    ignored=0   
```
*(Analisis Penting: Perintah `apt` sebenarnya **berhasil** menginstal `apache2`. Namun, karena `failed_when: web_pkg == "apache2"` bernilai *True*, Ansible melaporkan task tersebut `FAILED`. Karena di dalam `block`, maka blok `rescue` (`mariadb`) tereksekusi lagi. Status `rescued=1`).*

## 6. Validasi Akhir

Jika semua langkah telah dipraktikkan dan kamu memahami setiap perubahan status pada output terminal, jalankan perintah *grading*:

```bash
nusactl grade anadm-006-1
```

Setelah nilainya keluar, bersihkan environment lab:
```bash
nusactl finish anadm-006-1
```

## 7. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `ERROR! 'become' is not a legal parameter...` | Salah penempatan blok `[privilege_escalation]` di `ansible.cfg`. | Pastikan blok tersebut berdiri sendiri di bawah `[defaults]`, bukan di dalamnya. |
| `failed_when` tidak memicu `rescue` | Kesalahan penulisan kondisi logika. | Cek penulisan `web_pkg == "apache2"` (perhatikan *case-sensitive* dan tanda kutip). |
| Task `hostnamectl` tetap `changed` | Lupa menambahkan `changed_when: false`. | Pastikan baris tersebut sejajar dengan `command:` di dalam task. |

## 8. Konsep yang Dipelajari

*Error handling* memisahkan *SysAdmin* pemula dengan profesional. Alih-alih membiarkan *playbook* berhenti tiba-tiba, kita bisa memprogramnya untuk "sadar" akan kegagalan. `block/rescue/always` adalah fitur yang paling kuat karena memungkinkan kita melakukan *rollback* atau menjalankan perintah alternatif jika suatu *task* gagal. Selain itu, `changed_when` dan `failed_when` memberikan kita kendali penuh atas kebenaran (*truthiness*) dari status *task*, memastikan bahwa *output* Ansible selalu merepresentasikan kondisi sistem yang sebenarnya.

## 9. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Eksplorasi `any_errors_fatal`:** Tambahkan `any_errors_fatal: true` di tingkat *play* (sejajar dengan `hosts`). Jalankan *playbook* dan lihat bagaimana Ansible membatalkan seluruh eksekusi (termasuk pada host lain, jika ada) saat menemukan *error*.
2.  **Eksplorasi Modul `fail`:** Tambahkan *task* berikut di dalam blok `block` sebelum instalasi:
    ```yaml
    - name: Cek kondisi sistem
      fail:
        msg: "Sistem ini tidak diizinkan untuk melakukan instalasi."
      when: ansible_distribution == "Ubuntu"
    ```
    Lihat bagaimana pesan kustom tersebut muncul dan memicu `rescue`.

## 10. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 6.1: Handling Task Failure`.
- **Referensi Tambahan:**
  - [Ansible Error Handling](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html)
  - [Ansible Blocks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html)
