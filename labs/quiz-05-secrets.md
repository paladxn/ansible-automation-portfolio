# Quiz 5: Secrets

> Kuis ini menguji pemahamanmu tentang **Ansible Vault**. Kamu akan membuat file variabel terenkripsi untuk menyimpan kredensial, membuat *playbook* yang menggunakan file tersebut untuk menambahkan *user* Linux, dan menjalankannya secara otomatis menggunakan *Vault Password File*.

## 1. Tujuan Kuis

Setelah menyelesaikan kuis ini, kamu diharapkan mampu:
- Membuat dan mengenkripsi file variabel menggunakan `ansible-vault`.
- Menggunakan parameter `vars_files` di dalam *playbook*.
- Membuat *user* Linux menggunakan modul `user` dengan filter `password_hash`.
- Membuat *password file* dengan hak akses yang aman (`chmod 600`).
- Menjalankan *playbook* secara otomatis menggunakan flag `--vault-password-file`.

## 2. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat file rahasia dan *playbook* berada. |
| **Managed Nodes** | `pod-username-managed1`, `pod-username-managed2` | Target pembuatan *user* (tergabung dalam grup `managed`). |

*(Catatan: `<username>` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu).*

## 3. Prasyarat

Jalankan perintah persiapan environment kuis:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi kuis
student@pod-username-controller:~$ nusactl start anadm-quiz-5
```

## 4. Langkah Pengerjaan

*(Seluruh perintah dieksekusi sebagai user `student` pada `pod-username-controller`)*

### Langkah 1: Membuat Workspace dan Konfigurasi

**Tujuan:** Menyiapkan direktori kerja `quiz-5` dan file konfigurasinya.

**Perintah:**
```bash
mkdir ~/quiz-5
cd ~/quiz-5
vim ansible.cfg
```

**Isi File `ansible.cfg`:**
```ini
[defaults]
inventory = ./inventory
```

**Perintah:**
```bash
vim inventory
```

**Isi File `inventory`:**
```ini
[managed]
pod-username-managed1
pod-username-managed2
```

---

### Langkah 2: Membuat dan Mengenkripsi File Secret

**Tujuan:** Mendefinisikan variabel rahasia dan mengamankannya dengan Vault.

**1. Membuat file `quiz-Secret.yml`:**
```bash
vim quiz-Secret.yml
```

**Isi File `quiz-Secret.yml`:**
```yaml
username: <username>
passwd: adinusa2023
```
*(Catatan: Ganti `<username>` dengan nama pengguna Adinusa milikmu, misal: `arijfarid69`).*

**2. Mengenkripsi file:**
```bash
ansible-vault encrypt quiz-Secret.yml
```
**Interaksi Terminal:**
```text
New Vault password: adinusa
Confirm New Vault password: adinusa
```
*(Gunakan password `adinusa` sesuai instruksi kuis).*

---

### Langkah 3: Membuat Playbook

**Tujuan:** Membuat *playbook* untuk membuat *user* menggunakan data dari file terenkripsi.

**Perintah:**
```bash
vim quiz-5_secrets.yml
```

**Isi File `quiz-5_secrets.yml`:**
```yaml
---
- name: Create User from Encrypted Variables
  hosts: managed
  become: true

  vars_files:
    - quiz-Secret.yml

  tasks:
    - name: Ensure user exists with specified password
      user:
        name: "{{ username }}"
        password: "{{ passwd | password_hash('sha512') }}"
        state: present
```

**Penjelasan Kunci:**
- `vars_files: - quiz-Secret.yml`: Ansible akan meminta password Vault (atau membaca dari file) untuk memuat isi `username` dan `passwd`.
- **`password_hash('sha512')`**: Sangat Penting! Modul `user` di Linux tidak menerima *password* dalam bentuk teks biasa. Filter ini mengubah `adinusa2023` menjadi *hash* SHA-512 yang aman agar bisa disimpan di `/etc/shadow`.

---

### Langkah 4: Membuat Password File

**Tujuan:** Menyimpan password Vault ke dalam file agar eksekusi *playbook* bisa otomatis tanpa interaksi manual.

**Perintah:**
```bash
echo 'adinusa' > quiz-pass
chmod 600 quiz-pass
```

**Penjelasan:**
`chmod 600` memastikan hanya *user* `student` (pemilik file) yang bisa membaca dan menulis file ini. Ini adalah *best practice* keamanan untuk file yang berisi kredensial.

---

### Langkah 5: Menjalankan Playbook

**Tujuan:** Mengeksekusi *playbook* menggunakan *password file*.

**Perintah:**
```bash
ansible-playbook --vault-password-file=quiz-pass quiz-5_secrets.yml
```

## 5. Verifikasi Hasil (Sesuai Instruksi Kuis)

Setelah *playbook* berjalan dengan status `failed=0`, lakukan verifikasi manual:

1.  **Cek Kelengkapan File di Control Node:**
    ```bash
    ls -l ansible.cfg inventory quiz-5_secrets.yml quiz-Secret.yml quiz-pass
    ```
2.  **Pastikan File Terenkripsi (Opsional):**
    Kamu bisa melihat isinya dengan `ansible-vault view quiz-Secret.yml --vault-password-file=quiz-pass`.
3.  **Verifikasi User di Managed Host:**
    Coba lakukan SSH ke *Managed Node* menggunakan *user* dan *password* yang baru dibuat:
    ```bash
    ssh <username>@pod-username-managed1
    ```
    *(Saat diminta password, masukkan `adinusa2023`).*
    
    Jika berhasil masuk, jalankan:
    ```bash
    whoami
    ```
    *(Harus menampilkan `<username>`).*

## 6. Validasi Akhir

Jika semua verifikasi sesuai, jalankan perintah *grading* dari Adinusa:

```bash
nusactl grade anadm-quiz-5
```

Setelah nilainya keluar, bersihkan environment kuis:
```bash
nusactl finish anadm-quiz-5
```

## 7. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Decryption failed` saat menjalankan *playbook* | Password di file `quiz-pass` salah (bukan `adinusa`). | Pastikan isi file `quiz-pass` hanya berisi teks `adinusa` tanpa spasi tambahan. |
| User tidak bisa login via SSH | Lupa menggunakan filter `password_hash`. | Cek file `/etc/shadow` di *Managed Node*. Jika *password* tidak berformat hash (`$6$...`), modul `user` gagal memprosesnya. |
| `Permission denied` | File `quiz-pass` tidak bisa dibaca oleh Ansible. | Cek hak akses dengan `ls -l quiz-pass`. Pastikan statusnya `-rw-------` (600). |

## 8. Konsep yang Dipelajari

Kuis ini menggabungkan keamanan (Vault) dengan manajemen sistem (modul `user`). Memisahkan file `.yml` terenkripsi dari *playbook* memungkinkan kita untuk membagikan kode otomatisasi ke publik (misalnya di GitHub) tanpa khawatir membocorkan *password*. Penggunaan `--vault-password-file` sangat umum di lingkungan *CI/CD* di mana tidak ada manusia yang bisa mengetik *password* secara manual saat *pipeline* berjalan.

## 9. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Quiz 5 : Secrets`.
- **Referensi Tambahan:**
  - [Ansible Vault Guide](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
  - [Ansible user Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)
