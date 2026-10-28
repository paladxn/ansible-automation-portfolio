# Lab 5.2: Managing Secrets

> Praktikum ini membahas cara mengamankan data sensitif menggunakan **Ansible Vault**. Kita akan membuat file berisi kredensial, mengenkripsinya, lalu menggunakannya di dalam *playbook* untuk membuat *user* baru dengan *password* yang di-*hash* secara aman.

## 1. Tujuan Pembelajaran

Setelah menyelesaikan lab ini, kamu diharapkan mampu:
- Membuat file berisi data sensitif (seperti *username* dan *password*).
- Mengenkripsi file tersebut menggunakan `ansible-vault`.
- Menggunakan `vars_files` untuk memuat variabel dari file terenkripsi.
- Menggunakan filter `password_hash('sha512')` agar *password* bisa diterima oleh sistem Linux.
- Mengotomatiskan proses dekripsi menggunakan `vault-password-file` agar tidak perlu mengetik *password* setiap saat.

## 2. Konsep yang Digunakan

- **Ansible Vault:** Fitur Ansible untuk mengenkripsi file atau variabel menggunakan enkripsi AES256. File yang terenkripsi aman disimpan di repositori Git karena isinya tidak bisa dibaca tanpa *password*.
- **`vars_files`:** *Keyword* di *playbook* untuk memuat variabel dari file eksternal (dalam hal ini, `secret.yml` yang sudah terenkripsi).
- **`password_hash`:** Filter Jinja2 di Ansible. Modul `user` di Linux membutuhkan *password* dalam format *hash* (biasanya SHA-512 di Ubuntu) untuk disimpan di `/etc/shadow`. Jika kamu mengirim *password* *plaintext*, *user* tidak akan bisa login.
- **Vault Password File:** File teks biasa yang berisi *password* Vault. Digunakan agar *playbook* bisa berjalan otomatis tanpa *prompt* interaktif (cocok untuk CI/CD).

## 3. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat file terenkripsi dan *playbook* berada. |
| **Managed Node** | `pod-username-managed1` | Target pembuatan *user* baru. |

*(Catatan: `username` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu).*

## 4. Prasyarat

Jalankan perintah persiapan environment lab:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi lab
student@pod-username-controller:~$ nusactl start anadm-005-2
```

## 5. Alur Praktikum

1.  Membuat direktori kerja, `ansible.cfg`, dan `inventory`.
2.  Membuat file variabel `secret.yml` berisi data rahasia.
3.  Mengenkripsi `secret.yml` menggunakan `ansible-vault`.
4.  Membuat *playbook* yang memuat file rahasia tersebut.
5.  Menjalankan *playbook* menggunakan *password file*.
6.  Verifikasi dengan login sebagai *user* baru.

## 6. Langkah Praktikum

*(Seluruh perintah di bawah ini dieksekusi pada `pod-username-controller`)*

### Langkah 1-3: Membuat Workspace dan Konfigurasi

**Tujuan:** Menyiapkan direktori kerja `data-secret`, konfigurasi, dan *inventory*.

**Perintah:**
```bash
cd ~/
mkdir data-secret
cd data-secret

vi ansible.cfg
```
**Isi `ansible.cfg`:**
```ini
[defaults]
inventory = ./inventory
```

**Perintah:**
```bash
vi inventory
```
**Isi `inventory`:**
```ini
[devservers]
pod-username-managed1
```

---

### Langkah 4: Membuat File Secret (Plaintext)

**Tujuan:** Mendefinisikan variabel rahasia yang akan digunakan.

**Perintah:**
```bash
vi secret.yml
```

**Isi File `secret.yml`:**
```yaml
username: ansibleuser1
pw: adinusa88
```

> **⚠️ Peringatan Keamanan:** Pada tahap ini, file `secret.yml` masih berisi teks biasa. Jangan pernah mengunggah file ini ke GitHub dalam kondisi belum terenkripsi.

---

### Langkah 5: Mengenkripsi File Secret

**Tujuan:** Mengamankan file `secret.yml` dengan Ansible Vault.

**Perintah:**
```bash
ansible-vault encrypt secret.yml
```

**Interaksi Terminal:**
```text
New Vault password: adinusa88
Confirm New Vault password: adinusa88
```
*(Catatan: Di environment lab Adinusa, gunakan password `adinusa88` karena sistem penilaian otomatis (grading) menggunakan password tersebut).*

**Hasil yang Diharapkan:**
Jika kamu membuka file `secret.yml` dengan `cat secret.yml`, isinya akan berubah menjadi karakter acak yang tidak bisa dibaca (dimulai dengan `$ANSIBLE_VAULT;1.1;AES256`).

---

### Langkah 6: Membuat Playbook

**Tujuan:** Membuat *user* menggunakan data dari file yang terenkripsi.

**Perintah:**
```bash
vi create_users.yml
```

**Isi File `create_users.yml`:**
```yaml
---
- name: create user accounts for all our servers
  hosts: devservers
  become: true

  vars_files:
    - secret.yml

  tasks:
    - name: Creating user from secret.yml
      user:
        name: "{{ username }}"
        password: "{{ pw | password_hash('sha512') }}"
```

**Penjelasan Kunci:**
- `vars_files: - secret.yml`: Ansible tahu bahwa variabel ada di file ini. Saat *runtime*, Ansible akan meminta *password* Vault (atau menggunakan file) untuk mendekripsi isinya di memori.
- `{{ pw | password_hash('sha512') }}`: Ini sangat penting. Modul `user` tidak menerima *password* polos. Filter ini mengubah `adinusa88` menjadi *hash* SHA-512 yang valid (biasanya diawali dengan `$6$`).

---

### Langkah 7: Menjalankan Playbook

**Tujuan:** Menjalankan *playbook* dengan dua metode dekripsi Vault.

**Metode 1: Prompt Interaktif (Untuk Syntax Check)**
```bash
ansible-playbook --syntax-check --ask-vault-pass create_users.yml
```
*(Kamu akan diminta memasukkan password `adinusa88`).*

**Metode 2: Menggunakan Password File (Untuk Eksekusi Otomatis)**
```bash
# Membuat file password
echo 'adinusa88' > vault-pass

# Sangat Penting: Batasi hak akses agar hanya user ini yang bisa membaca
chmod 600 vault-pass

# Jalankan playbook
ansible-playbook --vault-password-file=vault-pass create_users.yml
```

**Penjelasan:**
Menggunakan `--vault-password-file` memungkinkan otomatisasi tanpa intervensi manual. Namun, file `vault-pass` itu sendiri berisi password rahasia. `chmod 600` memastikan hanya *owner* (user `student`) yang bisa membaca/menulis file tersebut.

---

### Langkah 8: Verifikasi User

**Tujuan:** Membuktikan bahwa *user* berhasil dibuat dan *password*-nya berfungsi.

**Perintah:**
```bash
ssh ansibleuser1@pod-username-managed1
```
*(Saat diminta password, masukkan `adinusa88`)*.

**Di dalam sesi SSH:**
```bash
hostname
whoami
```

**Hasil yang Diharapkan:**
`whoami` akan menampilkan `ansibleuser1`, membuktikan bahwa *user* berhasil dibuat dengan *password* yang benar.

## 7. Hasil Akhir dan Validasi

Pada akhir lab, kamu telah berhasil mengamankan kredensial menggunakan Vault dan menggunakannya untuk membuat *user* di sistem target.

**Validasi Akhir (Sesuai Instruksi Adinusa):**
```bash
nusactl grade anadm-005-2
```
Setelah nilainya keluar, bersihkan environment lab:
```bash
nusactl finish anadm-005-2
```

## 8. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Decryption failed` saat menjalankan *playbook* | Password Vault salah. | Pastikan password yang dimasukkan/di file `vault-pass` adalah `adinusa88`. |
| *User* berhasil dibuat, tapi tidak bisa login | Lupa menggunakan `password_hash('sha512')`. | Cek file `/etc/shadow` di *Managed Node*. Jika *password* masih *plaintext*, Ansible akan mengabaikannya. |
| `Syntax Error` di YAML | Indentasi salah pada `vars_files` atau modul `user`. | Pastikan `vars_files` sejajar dengan `hosts` dan `become`. |

## 9. Konsep yang Dipelajari

Lab ini memperkenalkan konsep **Separation of Concerns**. Data konfigurasi (*playbook*) dipisahkan dari data rahasia (*secret.yml*). Dengan Vault, file rahasia bisa disimpan di repositori yang sama dengan aman. Selain itu, pemahaman tentang `password_hash` sangat penting karena Ansible tidak secara otomatis melakukan *hashing* saat menggunakan modul `user`. Tanpa *hashing*, *user* yang dibuat tidak akan bisa digunakan untuk login.

## 10. Tantangan Lanjutan (Opsional)

> **Peringatan:** Bagian ini bukan bagian dari lab resmi Adinusa. Lakukan di lingkungan uji coba.

1.  **Melihat File Terenkripsi:** Coba jalankan `ansible-vault view secret.yml` (dengan flag `--ask-vault-pass`) untuk melihat isi file tanpa harus mendekripsinya secara permanen.
2.  **Mengubah Password Vault:** Gunakan `ansible-vault rekey secret.yml` untuk mengganti password enkripsi dari `adinusa88` ke password lain.

## 11. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Lab 5.2: Managing Secrets`.
- **Referensi Tambahan:**
  - [Ansible Vault Documentation](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
  - [Ansible password_hash filter](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/password_hash_filter.html)
