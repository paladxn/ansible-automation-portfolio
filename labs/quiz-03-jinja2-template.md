# Quiz 3: Jinja 2 Template

> Kuis ini menguji kemampuanmu dalam menggunakan modul `template` untuk men-*deploy* konfigurasi repositori paket (APT) ke *Managed Nodes*. Kamu juga akan diminta menginstal paket dengan versi spesifik dan mengelola *service* menggunakan sintaks *inline* tertentu.

## 1. Tujuan Kuis

Setelah menyelesaikan kuis ini, kamu diharapkan mampu:
- Membuat file Jinja2 (`.j2`) untuk konfigurasi repositori `nginx` dan `mariadb`.
- Men-*deploy* file tersebut ke `/etc/apt/sources.list.d/`.
- Melakukan *update cache* repositori via Ansible.
- Menginstal paket dengan versi yang di-*pin* (spesifik).
- Menggunakan modul `service` dengan sintaks *inline* (`key=value`).

## 2. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *playbook* dan file `.j2` berada. |
| **Managed Nodes** | `pod-username-managed1`, `pod-username-managed2` | Target instalasi (tergabung dalam grup `webservers`). |

*(Catatan: `username` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu).*

## 3. Prasyarat

Jalankan perintah persiapan environment kuis:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi kuis
student@pod-username-controller:~$ nusactl start anadm-quiz-3
```

## 4. Langkah Pengerjaan

*(Seluruh perintah dieksekusi sebagai user `student` pada `pod-username-controller`)*

### Langkah 1: Membuat Workspace dan Konfigurasi

**Tujuan:** Menyiapkan direktori kerja `quiz-3` dan file konfigurasinya.

**Perintah:**
```bash
mkdir ~/quiz-3
cd ~/quiz-3
vim ansible.cfg
```

**Isi File `ansible.cfg`:**
```ini
[defaults]
inventory = ./inventory
remote_user = student
```

**Perintah:**
```bash
vim inventory
```

**Isi File `inventory`:**
```ini
[webservers]
pod-username-managed1
pod-username-managed2
```

---

### Langkah 2: Membuat File Jinja2 Template

**Tujuan:** Membuat file sumber untuk repositori.

> **⚠️ Catatan Tambahan (Keterbatasan Sumber):**
> Dokumen asli Adinusa **tidak menyertakan isi teks** untuk file `.j2` ini. Isi di bawah adalah **contoh standar repositori APT** yang valid agar *playbook* bisa berjalan dan berhasil. 

**Perintah:**
```bash
vim nginx.list.j2
```

**Isi File `nginx.list.j2`:**
```text
deb http://nginx.org/packages/ubuntu/ jammy nginx
deb-src http://nginx.org/packages/ubuntu/ jammy nginx
```

**Perintah:**
```bash
vim mariadb.list.j2
```

**Isi File `mariadb.list.j2`:**
```text
deb [arch=amd64,arm64,ppc64el] http://sfo1.mirrors.digitalocean.com/mariadb/repo/10.11/ubuntu jammy main
deb-src http://sfo1.mirrors.digitalocean.com/mariadb/repo/10.11/ubuntu jammy main
```

---

### Langkah 3: Membuat Playbook (`quiz-3_j2template.yml`)

**Tujuan:** Menulis *playbook* yang men-*deploy* template, menginstal paket, dan menyalakan *service*.

**Perintah:**
```bash
vim quiz-3_j2template.yml
```

**Isi File `quiz-3_j2template.yml`:**
```yaml
---
- name: Configure Repositories and Install Services
  hosts: webservers
  become: true

  tasks:
    - name: Add nginx repository using jinja2 template
      template:
        src: nginx.list.j2
        dest: /etc/apt/sources.list.d/nginx.list

    - name: Add mariadb repository using jinja2 template
      template:
        src: mariadb.list.j2
        dest: /etc/apt/sources.list.d/mariadb.list

    - name: Update the repository
      apt:
        update_cache: yes

    - name: Install specific versions of nginx, mariadb-server and mariadb-client
      apt:
        name:
          - "nginx=1.24.0-1~jammy"
          - "mariadb-server"
          - "mariadb-client"
        state: present
        force_apt_get: yes

    - name: Ensure nginx service is started and enabled
      service: name=nginx state=started enabled=yes

    - name: Ensure mariadb-server service is started and enabled
      service: name=mariadb state=started enabled=yes
```

**Penjelasan Kunci:**
- **`template`**: Menyalin file `.j2` ke `/etc/apt/sources.list.d/`. Karena direktori ini milik *root*, `become: true` sangat diperlukan.
- **`update_cache: yes`**: Wajib dilakukan setelah menambahkan repositori baru agar APT mengenali daftar paket dari repositori Nginx dan MariaDB.
- **`nginx=1.24.0-1~jammy`**: Ini disebut *version pinning*. Ansible akan mencari dan menginstal versi Nginx yang *persis* sama dengan string tersebut.
- **Sintaks `service`**: Kuis meminta format spesifik: `service: name=<service> state=<state> anotherkey=<another value>`. Itulah mengapa kita menulisnya sebaris (*inline*). Perhatikan juga bahwa untuk MariaDB kita menggunakan `name=mariadb` (bukan `mariadb-server`) sesuai dengan instruksi kuis.

---

### Langkah 4: Menjalankan Playbook

**Tujuan:** Mengeksekusi *playbook*.

**Perintah:**
```bash
ansible-playbook quiz-3_j2template.yml
```

## 5. Verifikasi Hasil (Sesuai Instruksi Kuis)

Setelah *playbook* berjalan sukses (status `failed=0`), lakukan verifikasi manual:

1.  **Cek Kelengkapan File:**
    ```bash
    ls -l ansible.cfg inventory nginx.list.j2 mariadb.list.j2 quiz-3_j2template.yml
    ```
2.  **Cek Keberadaan File Repositori di Managed Nodes:**
    ```bash
    ansible webservers -m command -a "ls -l /etc/apt/sources.list.d/"
    ```
3.  **Cek Instalasi Paket Versi Spesifik:**
    ```bash
    ansible webservers -m command -a "dpkg -l | grep -E 'nginx|mariadb'"
    ```
4.  **Cek Status Service:**
    ```bash
    ansible webservers -m command -a "systemctl is-active nginx mariadb"
    ```

## 6. Validasi Akhir

Jika semua verifikasi sesuai, jalankan perintah *grading* dari Adinusa:

```bash
nusactl grade anadm-quiz-3
```

Setelah nilainya keluar, bersihkan environment kuis:
```bash
nusactl finish anadm-quiz-3
```

## 7. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Failed to fetch ... 404` atau `Package not found` | Repositori belum di-*update* atau isi file `.j2` salah. | Pastikan `update_cache: yes` ada di *playbook* dan cek URL di dalam file `.j2`. |
| `No candidate version found for nginx=...` | Versi paket tidak tersedia di repositori. | Cek ketersediaan versi dengan `apt-cache madison nginx` di Managed Node. |
| Service gagal *running* | Konflik paket (misal Apache masih terinstal dan memakai port 80). | Jalankan `systemctl status nginx` di *Managed Node* untuk melihat *error log*. |

## 8. Catatan Penting (Inkonsistensi Sumber)
Pada bagian *Verification*, dokumen menyebutkan direktori `quiz-001-3`. Ini kemungkinan besar adalah *typo* dari `quiz-3` yang disebutkan di awal instruksi. Dokumentasi ini menggunakan `quiz-3`.

## 9. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Quiz 3 : Jinja 2 Template`.
- **Referensi Tambahan:**
  - [Ansible template Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
  - [Ansible apt Module (Installing Specific Versions)](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
