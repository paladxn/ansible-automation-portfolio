# Quiz 2: Variables

> Kuis ini menguji pemahamanmu tentang penggunaan variabel (`vars`) di Ansible. Kamu dituntut untuk membuat *playbook* yang menginstal paket, menyalakan *service*, dan men-*deploy* konten web menggunakan variabel, bukan nilai statis.

## 1. Tujuan Kuis

Setelah menyelesaikan kuis ini, kamu diharapkan mampu:
- Mendefinisikan variabel dengan tipe data *string* dan *list*.
- Memanggil variabel di dalam parameter modul (misal: `name: "{{ var_pkg }}"`).
- Memahami sifat *case-sensitive* (huruf besar/kecil) pada nama variabel Ansible.
- Membuat *multiple plays* untuk konfigurasi dan verifikasi.

## 2. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *playbook* dijalankan (localhost). |
| **Managed Node** | `pod-username-managed2` | Target instalasi Apache dan penempatan file HTML. |

*(Catatan: `username` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu).*

## 3. Prasyarat

Jalankan perintah persiapan environment kuis:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi kuis
student@pod-username-controller:~$ nusactl start anadm-quiz-2
```

## 4. Langkah Pengerjaan

*(Seluruh perintah dieksekusi sebagai user `student` pada `pod-username-controller`)*

### Langkah 1: Membuat Workspace dan Konfigurasi

**Tujuan:** Menyiapkan direktori kerja `quiz-2` dan file konfigurasinya.

**Perintah:**
```bash
mkdir ~/quiz-2
cd ~/quiz-2
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
pod-username-managed2
```

---

### Langkah 2: Membuat Playbook (`quiz-2_variables.yml`)

**Tujuan:** Menulis *playbook* dengan variabel sesuai instruksi.

**Perintah:**
```bash
vim quiz-2_variables.yml
```

**Isi File `quiz-2_variables.yml`:**
```yaml
---
- name: Configure Web Server with Variables
  hosts: webservers
  remote_user: student
  become: true
  
  vars:
    required_Pkg:
      - apache2
      - python3-urllib3
    web_Service: apache2
    content_File: "adinusa lab quiz variables - username"
    dest_File: /var/www/html/index.html

  tasks:
    - name: Install required packages
      apt:
        name: "{{ required_Pkg }}"
        state: latest
        update_cache: yes

    - name: Ensure apache2 service is started and enabled
      service:
        name: "{{ web_Service }}"
        state: started
        enabled: true

    - name: Ensure specific content exists
      copy:
        content: "{{ content_File }}"
        dest: "{{ dest_File }}"

- name: Verify Web Server
  hosts: localhost
  become: false

  tasks:
    - name: Test web service access
      uri:
        url: http://pod-username-managed2/index.html
        status_code: 200
```

**Penjelasan Kunci & Peringatan Penting:**
- **Case-Sensitivity (Sangat Penting):** Perhatikan nama variabel: `required_Pkg` (P dan K besar), `web_Service` (S besar), `content_File` (F besar), dan `dest_File` (F besar). Jika kamu mengetik `required_pkg` (huruf kecil semua) di bagian `tasks`, Ansible akan menganggap variabel itu tidak ada dan *playbook* akan gagal. **Samakan penulisan di blok `vars` dan di blok `tasks`.**
- **Tipe Data List:** Variabel `required_Pkg` adalah sebuah *list* (ditandai dengan tanda minus). Di modul `apt`, kita bisa langsung memanggilnya dengan `name: "{{ required_Pkg }}"`.
- **`update_cache: yes`**: Sangat disarankan untuk selalu menyertakan ini saat menginstal paket untuk menghindari *error* 404 (seperti yang terjadi di Lab 4.4).
- **Ganti `<username>`:** Pada variabel `content_File`, pastikan kamu mengganti kata `username` dengan nama pengguna Adinusa milikmu (misal: `"adinusa lab quiz variables - arijfarid69"`).

---

### Langkah 3: Menjalankan Playbook

**Tujuan:** Mengeksekusi *playbook*.

**Perintah:**
```bash
ansible-playbook quiz-2_variables.yml
```

*(Opsional, disarankan: Jalankan `ansible-playbook --syntax-check quiz-2_variables.yml` terlebih dahulu untuk memastikan tidak ada salah ketik variabel atau indentasi).*

## 5. Verifikasi Hasil (Sesuai Instruksi Kuis)

Setelah *playbook* berjalan dengan status `failed=0`, lakukan pengecekan manual untuk memastikan semua kriteria kuis terpenuhi:

1.  **Cek Kelengkapan File:**
    ```bash
    ls -l ansible.cfg inventory quiz-2_variables.yml
    ```
2.  **Cek Instalasi Paket & Service di Managed Node:**
    ```bash
    ansible pod-username-managed2 -m command -a "systemctl is-active apache2"
    ansible pod-username-managed2 -m command -a "dpkg -l | grep -E 'apache2|python3-urllib3'"
    ```
3.  **Cek Isi File HTML:**
    ```bash
    ansible pod-username-managed2 -m command -a "cat /var/www/html/index.html"
    ```
4.  **Tes Akses Webserver via Curl:**
    ```bash
    curl http://pod-username-managed2/index.html
    ```
    *(Harusnya mengembalikan teks: `adinusa lab quiz variables - <username>`).*

## 6. Validasi Akhir

Jika semua verifikasi sesuai, jalankan perintah *grading* dari Adinusa:

```bash
nusactl grade anadm-quiz-2
```

Setelah nilainya keluar, bersihkan environment kuis:
```bash
nusactl finish anadm-quiz-2
```

## 7. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Task failed: Install required packages` | Salah ketik nama variabel (huruf besar/kecil) atau repositori usang. | Cek apakah penulisan `{{ required_Pkg }}` persis sama dengan di blok `vars`. Pastikan `update_cache: yes` ada. |
| `uri` task gagal (bukan status 200) | File `index.html` tidak ada atau Apache mati. | Pastikan task `copy` dan `service` berjalan sukses. Cek manual dengan `curl`. |
| Konten web kosong atau teks salah | Variabel `content_File` tidak terbaca atau lupa mengganti `<username>`. | Cek blok `vars` di file YAML. Pastikan teks yang diinginkan sudah benar. |

## 8. Catatan Penting (Inkonsistensi Sumber)
Pada bagian *Preparation*, teks menyebutkan bahwa script akan membuat direktori `data-variables`. Namun, pada bagian *Instructions* poin pertama, kamu diinstruksikan untuk membuat direktori baru bernama `quiz-2`. Dokumentasi ini mengikuti instruksi poin pertama (menggunakan `quiz-2`) karena instruksi tersebut yang lebih eksplisit.

## 9. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Quiz 2 : Variables`.
- **Referensi Tambahan:**
  - [Ansible Variables Guide](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html)
  - [Ansible apt Module (name list)](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
