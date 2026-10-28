# Lab 4.6: Using Jinja 2 Template

> Praktikum ini membahas cara menggunakan modul `template` di Ansible. Kita akan membuat file HTML dinamis menggunakan sintaks Jinja2, lalu men-*deploy*-nya ke *Managed Node* sehingga konten yang tampil di web server dapat disesuaikan secara otomatis.

## 1. Tujuan Lab

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Memahami perbedaan mendasar antara modul `copy` dan `template`.
- Membuat file *template* Jinja2 dengan ekstensi `.j2`.
- Menggunakan modul `template` di dalam *playbook* untuk memproses dan mengirim file ke *Managed Node*.
- Memverifikasi hasil *render template* menggunakan perintah `curl`.

## 2. Konsep yang Digunakan

- **Jinja2 Template:** File teks yang berisi teks statis dan variabel dinamis (ditandai dengan `{{ }}`). Saat dieksekusi, Ansible akan mengganti variabel tersebut dengan nilai sebenarnya.
- **Modul `template`:** Modul Ansible yang membaca file `.j2`, memproses variabel di dalamnya, lalu menyimpan hasil akhirnya ke *Managed Node* (mirip seperti modul `copy`, tapi dengan fitur *templating*).

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *playbook* dan file `.j2` berada. |
| **Managed Node** | `pod-username-managed1` | Target instalasi Apache dan penempatan file HTML. |

*(Catatan: `<username>` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu, misal `arijfarid69`).*

## 4. Prasyarat

Jalankan perintah persiapan environment lab:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-username-controller:~$ nusactl start anadm-004-6
```

## 5. Alur Praktikum

1.  Membuat direktori kerja.
2.  Membuat file `inventory` dan *playbook* `site.yml`.
3.  Membuat file *template* `<username>.html.j2`.
4.  Menjalankan *playbook*.
5.  Verifikasi hasil di web browser/terminal.

## 6. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-username-controller`)*

### Langkah 1: Membuat Direktori Kerja

**Tujuan:** Menyiapkan folder proyek.

**Perintah:**
```bash
cd ~
mkdir jinja2-template
cd ~/jinja2-template
```

---

### Langkah 2: Membuat Inventory

**Tujuan:** Mendefinisikan grup `webservers`.

**Perintah:**
```bash
vim inventory
```

**Isi File `inventory`:**
```ini
[webservers]
pod-username-managed1
```

---

### Langkah 3: Membuat Playbook (`site.yml`)

**Tujuan:** Menulis *playbook* yang menginstal Apache, menyalakan *service*, dan men-*deploy template*.

**Perintah:**
```bash
vim site.yml
```

**Isi File `site.yml`:**
```yaml
---
- name: install and start apache2
  hosts: webservers
  become: true

  tasks:
    - name: ensure apache2 package is present
      apt:
        name: apache2
        state: present
        update_cache: yes
        force_apt_get: yes

    - name: restart apache2 service
      service: name=apache2 state=restarted enabled=yes

    - name: copy index.html
      template: src=<username>.html.j2 dest=/var/www/html/<username>.html
```

**Penjelasan:**
- **`update_cache: yes`**: Memastikan repositori APT di-*update* terlebih dahulu (solusi dari *error* 404 di lab sebelumnya).
- **`service: name=apache2 state=restarted enabled=yes`**: Menggunakan sintaks *inline* (sebaris) untuk me-*restart* Apache dan mengaktifkannya saat *boot*.
- **`template: src=... dest=...`**: Modul ini akan mengambil file `<username>.html.j2` dari *Control Node*, memproses *template*-nya (jika ada variabel), lalu menyimpannya sebagai `<username>.html` di `/var/www/html/` pada *Managed Node*.

---

### Langkah 4: Membuat Jinja 2 Template

**Tujuan:** Membuat file sumber (`.j2`) yang akan diproses oleh modul `template`.

**Perintah:**
```bash
vim <username>.html.j2
```

**Isi File `<username>.html.j2`:**
```html
Hello World!
This is <username> site.
```

*(Catatan: Di dalam file ini, tidak ada variabel Jinja2 yang digunakan. Teks `<username>` akan ditulis apa adanya karena kita menggunakan *placeholder* di nama file. Untuk eksperimen dengan variabel, lihat bagian **Tantangan Lanjutan**).*

---

### Langkah 5: Menjalankan Playbook

**Tujuan:** Mengeksekusi *playbook*. Perhatikan bahwa di sini kita secara eksplisit menggunakan flag `-i inventory`.

**Perintah:**
```bash
ansible-playbook -i inventory site.yml
```

**Hasil yang Diharapkan:**
Ansible akan menampilkan proses instalasi paket, *restart service*, dan penyalinan *template* dengan status `changed` atau `ok`.

---

### Langkah 6: Verifikasi

**Tujuan:** Memastikan file HTML dinamis dapat diakses melalui web server.

**Perintah:**
```bash
curl pod-username-managed1/<username>.html
```

**Hasil yang Diharapkan:**
```text
Hello World!
This is <username> site.
```
*(Pastikan `<username>` di perintah `curl` sudah diganti dengan *username* yang sebenarnya, misal: `curl pod-username-managed1/arijfarid69.html`).*

## 7. Hasil Akhir dan Validasi

Pada akhir lab, kamu telah berhasil menggunakan modul `template` untuk men-*deploy* file ke web server. 

**Validasi Akhir (Sesuai Instruksi Adinusa):**
```bash
nusactl grade anadm-004-6
```
Setelah nilainya keluar, bersihkan environment lab dengan perintah:
```bash
nusactl finish anadm-004-6
```

## 8. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Could not find or access '...html.j2'` | File `.j2` tidak ada di direktori kerja saat ini. | Pastikan file `username.html.j2` berada di folder `~/jinja2-template`. |
| `404 Not Found` saat `curl` | Salah mengetik URL atau nama file di `dest` berbeda dengan yang di-*curl*. | Cek kembali nama file di parameter `dest` pada `site.yml`. Pastikan `curl` mengarah ke file yang benar. |
| `Permission denied` | Lupa menggunakan `become: true`. | Pastikan `become: true` ada di level *play*. |

## 9. Konsep yang Dipelajari

Lab ini mendemonstrasikan perbedaan kunci antara modul `copy` dan `template`. Modul `copy` menyalin file persis seperti aslinya. Modul `template` memproses file terlebih dahulu menggunakan *engine* Jinja2. Jika kamu menulis `{{ web_server_name }}` di dalam file `.j2`, Ansible akan menggantinya dengan nilai variabel tersebut sebelum menyimpannya ke *Managed Node*. Ini sangat berguna untuk membuat file konfigurasi yang berbeda-beda untuk setiap server, meskipun *template* aslinya sama.

## 10. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Gunakan Variabel di Template:** Ubah isi file `<username>.html.j2` menjadi:
    ```html
    Hello World!
    This is {{ ansible_hostname }} site.
    ```
    *(Catatan: `ansible_hostname` adalah *fact* bawaan Ansible).* Jalankan *playbook* dan lihat hasilnya di `curl`. Ansible akan mengganti `{{ ansible_hostname }}` dengan nama host asli dari server target.
2.  **Looping di Template:** Tambahkan *loop* di dalam file `.j2` untuk menampilkan daftar item. (Pelajari lebih lanjut tentang *for loop* di Jinja2).

## 11. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 4.6: Using Jinja 2 Template`.
- **Referensi Tambahan:**
  - [Ansible template Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
  - [Jinja2 Documentation](https://jinja.palletsprojects.com/en/3.1.x/)
