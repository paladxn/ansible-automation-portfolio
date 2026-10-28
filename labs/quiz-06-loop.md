# Quiz 6: Loop

> Kuis ini adalah evaluasi akhir untuk materi *Loops* dan *Variables*. Kamu akan diminta membuat *user* secara massal (hingga 200 *user*) menggunakan data dari `group_vars`, dengan *password* yang diamankan menggunakan Ansible Vault.

## 1. Tujuan Kuis

Setelah menyelesaikan kuis ini, kamu diharapkan mampu:
- Membuat file `group_vars` spesifik untuk *host* tertentu.
- Menggunakan `loop` untuk melakukan iterasi pada *list* yang sangat besar.
- Menggabungkan dua *list* (ops dan dev) dalam satu *loop*.
- Menggunakan `vault-password-file` untuk mendekripsi *password* secara otomatis.
- Menguji login SSH massal menggunakan perintah *bash loop*.

## 2. Topologi dan Environment

| Komponen | Hostname | Keterangan |
| :--- | :--- | :--- |
| **Control Node** | `pod-username-controller` | Mesin tempat *playbook* dan *group_vars* berada. |
| **Managed Nodes** | `pod-username-managed1`, `pod-username-managed2` | Target pembuatan *user* (tergabung dalam grup yang berbeda). |

*(Catatan: `<username>` adalah *placeholder* yang harus diganti dengan *username* Adinusa milikmu).*

## 3. Prasyarat

Jalankan perintah persiapan environment kuis:

```bash
# Login ke akun Adinusa
student@pod-username-controller:~$ nusactl login

# Menjalankan script pre-konfigurasi kuis
student@pod-username-controller:~$ nusactl start anadm-quiz-6
```

## 4. Langkah Pengerjaan

*(Seluruh perintah dieksekusi sebagai user `student` pada `pod-username-controller`)*

### Langkah 1: Membuat Workspace dan Konfigurasi Dasar

**Tujuan:** Menyiapkan direktori kerja `quiz-6`, `ansible.cfg`, dan `inventory`.

**Perintah:**
```bash
mkdir ~/quiz-6
cd ~/quiz-6
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
*(Perhatikan: Nama grup harus sama dengan nama file di `group_vars` tanpa ekstensi `.yml`).*
```ini
[managed1-host-<username>]
pod-username-managed1

[managed2-host-<username>]
pod-username-managed2
```
*(Ganti `<username>` dengan username Adinusa milikmu).*

---

### Langkah 2: Membuat File Group Vars

**Tujuan:** Mendefinisikan daftar *user* untuk masing-masing *host*. Karena ada 100 *user* per file, kita akan menggunakan *script bash* agar tidak perlu mengetik manual.

**1. Membuat direktori `group_vars`:**
```bash
mkdir group_vars
```

**2. Generate `managed1-host-<username>.yml`:**
```bash
# Ganti <username> di bawah ini dengan username aslimu
FILE1="group_vars/managed1-host-<username>.yml"

echo "list_user_ops_managed1:" > $FILE1
for i in {1..50}; do echo "  - ops$i" >> $FILE1; done

echo "list_user_dev_managed1:" >> $FILE1
for i in {1..50}; do echo "  - dev$i" >> $FILE1; done
```
*(File ini akan berisi 102 baris: 1 nama variabel, 50 item ops, 1 nama variabel, 50 item dev).*

**3. Generate `managed2-host-<username>.yml`:**
```bash
# Ganti <username> di bawah ini dengan username aslimu
FILE2="group_vars/managed2-host-<username>.yml"

echo "list_user_ops_managed2:" > $FILE2
for i in {51..100}; do echo "  - ops$i" >> $FILE2; done

echo "list_user_dev_managed2:" >> $FILE2
for i in {51..100}; do echo "  - dev$i" >> $FILE2; done
```

**Verifikasi Cepat:**
Pastikan jumlah baris sesuai instruksi kuis (51 baris per list, jadi total 102 baris per file):
```bash
wc -l group_vars/*.yml
```

---

### Langkah 3: Membuat Secret File (Terenkripsi)

**Tujuan:** Menyimpan *password* `adinusa88` dalam bentuk terenkripsi.

**1. Membuat file `secret.yml`:**
```bash
echo "pass: adinusa88" > secret.yml
```

**2. Enkripsi file:**
```bash
ansible-vault encrypt secret.yml
```
**Interaksi Terminal:**
```text
New Vault password: belajaransible
Confirm New Vault password: belajaransible
```
*(Gunakan password `belajaransible` sesuai instruksi kuis).*

**3. Membuat file `vault-pass`:**
```bash
echo "belajaransible" > vault-pass
chmod 600 vault-pass
```

---

### Langkah 4: Membuat Playbook (`quiz-6_loop.yml`)

**Tujuan:** Menulis *playbook* dengan dua *play* (satu untuk *managed1*, satu untuk *managed2*).

**Perintah:**
```bash
vim quiz-6_loop.yml
```

**Isi File `quiz-6_loop.yml`:**
```yaml
---
- name: Create Users on Managed1
  hosts: managed1-host-<username>
  become: true
  
  vars_files:
    - secret.yml

  tasks:
    - name: Create ops and dev users on Managed1
      user:
        name: "{{ item }}"
        password: "{{ pass | password_hash('sha512') }}"
        state: present
      loop: "{{ list_user_ops_managed1 + list_user_dev_managed1 }}"

- name: Create Users on Managed2
  hosts: managed2-host-<username>
  become: true
  
  vars_files:
    - secret.yml

  tasks:
    - name: Create ops and dev users on Managed2
      user:
        name: "{{ item }}"
        password: "{{ pass | password_hash('sha512') }}"
        state: present
      loop: "{{ list_user_ops_managed2 + list_user_dev_managed2 }}"
```
*(Ganti `<username>` pada bagian `hosts:` dengan username aslimu).*

**Penjelasan Kunci:**
- **`list_user_ops_managed1 + list_user_dev_managed1`**: Ini adalah fitur Jinja2 untuk menggabungkan dua *list* menjadi satu *list* besar. Sehingga `loop` akan berjalan untuk 100 *user* sekaligus.
- **`password_hash('sha512')`**: Tetap digunakan agar *user* bisa login via SSH.

---

### Langkah 5: Menjalankan Playbook

**Tujuan:** Mengeksekusi pembuatan *user* secara massal.

**Perintah:**
```bash
ansible-playbook --vault-password-file=vault-pass quiz-6_loop.yml
```

*(Proses ini akan memakan waktu beberapa saat karena Ansible membuat 200 user di dua server).*

## 5. Verifikasi Hasil (Sesuai Instruksi Kuis)

Setelah *playbook* berjalan sukses (`failed=0`), lakukan verifikasi:

1.  **Cek Kelengkapan File:**
    ```bash
    ls -l ansible.cfg inventory quiz-6_loop.yml secret.yml vault-pass
    ls -l group_vars/
    ```
2.  **Verifikasi Login (Gunakan perintah dari instruksi):**
    Uji apakah *user* bisa login dengan *password* `adinusa88`.
    ```bash
    for i in {1..50}; do sshpass -p adinusa88 ssh -o StrictHostKeyChecking=no dev$i@pod-username-managed1 whoami; done
    ```
    *(Jika berhasil, terminal akan menampilkan `dev1`, `dev2`, dst. hingga `dev50`).*

## 6. Validasi Akhir

Jika semua verifikasi sesuai, jalankan perintah *grading* dari Adinusa:

```bash
nusactl grade anadm-quiz-6
```

Setelah nilainya keluar, bersihkan environment kuis:
```bash
nusactl finish anadm-quiz-6
```

## 7. Troubleshooting

| Gejala | Kemungkinan Penyebab | Langkah Pemeriksaan |
| :--- | :--- | :--- |
| `Could not find specified file in expected paths: group_vars/...` | Salah ketik nama file di `group_vars` atau nama grup di `inventory` tidak cocok. | Ingat, nama file di `group_vars` (tanpa `.yml`) **harus persis sama** dengan nama grup di `inventory`. |
| `Error: Decryption failed` | Password di `vault-pass` salah. | Pastikan isinya `belajaransible`. |
| User tidak bisa login | Lupa menggunakan `password_hash('sha512')` atau `become: true` tidak ada. | Cek `/etc/shadow` di Managed Node, pastikan password berformat hash. |
| Jumlah user tidak sesuai | Salah memasukkan rentang angka di *script* bash. | Cek isi file `group_vars` dengan `wc -l` (harus 102 baris). |

## 8. Catatan Penting (Inkonsistensi Sumber)
*   Instruksi tidak menyebutkan pembuatan `ansible.cfg`, namun file ini diwajibkan di bagian verifikasi. Dokumentasi ini menambahkannya.
*   Pada bagian verifikasi, dokumen menyebutkan direktori `quiz-002-3`. Ini kemungkinan besar adalah *typo* dari `quiz-6` yang disebutkan di awal instruksi.
*   Instruksi sangat menekankan **"SENSITIVE CASE"**. Pastikan penulisan `list_user_ops_managed1`, `ops`, `dev`, dll. sama persis huruf besar/kecilnya dengan instruksi.

## 9. Referensi

- **Sumber Utama:** Modul Adinusa Freecourse "Automation With Ansible" - `Quiz 6 : Loop`.
- **Referensi Tambahan:**
  - [Ansible Loops Documentation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html)
  - [Ansible user Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)
