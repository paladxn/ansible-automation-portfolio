# 02 - Fundamentals of Ansible

> Modul ini membedah komponen inti operasional Ansible. Mulai dari mendefinisikan target (*Inventory*), mengeksekusi perintah cepat (*Ad-hoc*), membuat otomatisasi terstruktur (*Playbook* & *Roles*), hingga menggunakan variabel, logika pengulangan, dan templating.

## Daftar Isi
1. [Tujuan Pembelajaran](#1-tujuan-pembelajaran)
2. [Mendefinisikan Inventory](#2-mendefinisikan-inventory)
3. [Ad-Hoc Command](#3-ad-hoc-command)
4. [Playbooks](#4-playbooks)
5. [Tasks](#5-tasks)
6. [Roles](#6-roles)
7. [Modules](#7-modules)
8. [Variables](#8-variables)
9. [Loop](#9-loop)
10. [Conditionals](#10-conditionals)
11. [Privilege Escalation (Become)](#11-privilege-escalation-become)
12. [Secrets (Ansible Vault)](#12-secrets-ansible-vault)
13. [Jinja2 Template](#13-jinja2-template)
14. [Ringkasan](#14-ringkasan)
15. [Pertanyaan untuk Menguji Pemahaman](#15-pertanyaan-untuk-menguji-pemahaman)
16. [Referensi](#16-referensi)

---

## 1. Tujuan Pembelajaran

Setelah mempelajari materi ini, kamu diharapkan mampu:
- Memahami dan mendefinisikan *Inventory*.
- Memahami dan menggunakan *Ad-hoc commands*.
- Memahami *Ansible Modules*.
- Memahami *Playbooks* dan *Roles*.
- Memahami *Variables* dan cara kerjanya di Ansible.

## 2. Mendefinisikan Inventory

*Inventory* berisi daftar *managed nodes* (host target). File ini terkadang disebut juga "hostfile". Di dalam *inventory*, kamu bisa menentukan informasi seperti alamat IP untuk setiap *managed node*.

Lokasi *default* untuk *inventory* adalah `/etc/ansible/hosts`. Kamu bisa menentukan file *inventory* lain di *command line* menggunakan opsi `-i`.

### Format INI
```ini
mail.example.com

[webservers]
foo.example.com
bar.example.com

[dbservers]
one.example.com
two.example.com
three.example.com
```

### Format YAML
```yaml
all:
  hosts:
    mail.example.com:
  children:
    webservers:
      hosts:
        foo.example.com:
        bar.example.com:
    dbservers:
      hosts:
        one.example.com:
        two.example.com:
        three.example.com:
```

**Default Groups:**
Terdapat dua grup bawaan (*default groups*): `all` dan `ungrouped`. 
- Grup `all` berisi seluruh *host* yang ada.
- Grup `ungrouped` berisi semua *host* yang tidak memiliki grup lain selain `all`.

## 3. Ad-Hoc Command

*Ad-Hoc command* adalah perintah Ansible satu baris (*one-liner*) yang melakukan satu tugas pada *host* target (satu atau lebih *managed nodes*). Perintah *ad-hoc* cepat dan mudah, tetapi **tidak dapat digunakan kembali** (*not reusable*). Perintah ini sangat cocok untuk tugas-tugas yang jarang diulang (misal: mengecek koneksi atau me-*reboot* server).

Format umum:
```bash
ansible [pattern] -m [module] -a "[module options]"
```

**Contoh Use Case Ad-Hoc:**

- **Rebooting servers:**
  ```bash
  ansible atlanta -a "/sbin/reboot" -f 10
  ```
  *(Catatan: `-f 10` berarti menjalankannya secara paralel dengan 10 fork).*

- **Mengelola file (Copy):**
  ```bash
  ansible atlanta -m ansible.builtin.copy -a "src=/etc/hosts dest=/tmp/hosts"
  ```

- **Mengelola file (Ubah Permission/Owner):**
  ```bash
  ansible webservers -m ansible.builtin.file -a "dest=/srv/foo/b.txt mode=600 owner=mdehaan group=mdehaan"
  ```

- **Mengelola packages (YUM):**
  ```bash
  ansible webservers -m ansible.builtin.yum -a "name=acme state=present"
  ```

- **Mengelola users dan groups:**
  ```bash
  ansible all -m ansible.builtin.user -a "name=foo password=<crypted password here>"
  ```

- **Mengelola services (Systemd/Service):**
  ```bash
  ansible webservers -m ansible.builtin.service -a "name=httpd state=started"
  ansible webservers -m ansible.builtin.service -a "name=httpd state=restarted"
  ansible webservers -m ansible.builtin.service -a "name=httpd state=stopped"
  ```

- **Gathering facts:**
  ```bash
  ansible all -m ansible.builtin.setup
  ```

## 4. Playbooks

*Ansible Playbooks* menawarkan sistem *configuration management* dan *multi-machine deployment* yang *repeatable* (dapat diulang), *re-usable* (dapat digunakan kembali), dan sederhana. Sangat cocok untuk melakukan *deploy* aplikasi yang kompleks.

*Playbook* dapat:
- Mendeklarasikan konfigurasi.
- Mengatur langkah-langkah dari proses manual apa pun, pada beberapa set mesin, dalam urutan yang telah ditentukan.
- Menjalankan tugas secara sinkron atau asinkron.

*Playbook* berjalan secara berurutan dari atas ke bawah. Di dalam setiap *play*, *tasks* juga berjalan berurutan dari atas ke bawah. Minimal, setiap *play* harus mendefinisikan dua hal:
1.  *Managed nodes* yang menjadi target (menggunakan *pattern*).
2.  Setidaknya satu *task* untuk dieksekusi.

**Contoh Playbook (YAML):**
```yaml
- name: update web servers
  hosts: webservers
  remote_user: root

  tasks:
  - name: ensure apache is at the latest version
    yum:
      name: httpd
      state: latest

  - name: write the apache config file
    template:
      src: /srv/httpd.j2
      dest: /etc/httpd.conf

- name: update db servers
  hosts: databases
  remote_user: root

  tasks:
  - name: ensure postgresql is at the latest version
    yum:
      name: postgresql
      state: latest

  - name: ensure that postgresql is started
    service:
      name: postgresql
      state: started
```

Untuk menjalankan *playbook*, gunakan perintah:
```bash
ansible-playbook playbook.yml
```

## 5. Tasks

*Tasks* adalah unit aksi (*units of action*) di dalam Ansible. Kamu dapat mengeksekusi satu *task* menggunakan *ad-hoc command*, atau mendefinisikannya di dalam *playbook*.

**Contoh task via Ad-hoc:**
```bash
ansible atlanta -m ansible.builtin.copy -a "src=/etc/hosts dest=/tmp/hosts"
```

**Contoh task di dalam Playbook:**
```yaml
tasks:
  - name: ensure apache is at the latest version
    apt:
      name: apache2
      state: latest

  - name: write the apache config file
    template:
      src: /srv/httpd.j2
      dest: /etc/httpd.conf
```

## 6. Roles

*Roles* memungkinkan kamu untuk secara otomatis memuat file variabel, *tasks*, *handlers*, dan artefak Ansible lainnya berdasarkan struktur file yang telah diketahui. Setelah konten dikelompokkan dalam *roles*, kamu dapat dengan mudah menggunakannya kembali (*reuse*) dan membagikannya kepada pengguna lain.

Sebuah *Ansible role* memiliki struktur direktori yang telah ditentukan dengan tujuh direktori standar utama. Kamu harus menyertakan setidaknya satu dari direktori ini di setiap *role*.

**Struktur Direktori Role:**
```text
playbooks/
  site.yml
  webservers.yml
  fooservers.yml
  roles/
    common/
      tasks/
      handlers/
      library/
      files/
      templates/
      vars/
      defaults/
      meta/
    webservers/
      tasks/
      defaults/
      meta/
```

**Penjelasan Direktori Role:**
- `tasks/main.yml`: Daftar utama *tasks* yang dieksekusi oleh *role* ini.
- `handlers/main.yml`: *Handlers* yang dapat digunakan di dalam atau di luar *role* ini.
- `library/my_module.py`: *Modules* yang dapat digunakan di dalam *role* ini.
- `defaults/main.yml`: Variabel *default* untuk *role*. Variabel ini memiliki prioritas terendah dan mudah ditimpa (*overridden*) oleh variabel lain, termasuk variabel *inventory*.
- `vars/main.yml`: Variabel lain untuk *role*.
- `files/main.yml`: File yang di-*deploy* oleh *role* ini.
- `templates/main.yml`: *Templates* yang di-*deploy* oleh *role* ini.
- `meta/main.yml`: Metadata untuk *role*, termasuk dependensi antar-*role*.

## 7. Modules

*Modules* (terkadang disebut *task plugins* atau *library plugins*) adalah unit kode diskrit yang dapat digunakan dari *command line* atau dari *task* di *playbook*.

**Contoh eksekusi modules dari command line (Ad-hoc):**
```bash
ansible workerservers -m service -a "name=httpd state=started"
ansible workerservers -m ping
ansible workerservers -m command -a "/sbin/reboot -t now"
```

Setiap *module* mendukung argumen (*arguments*). Hampir semua *module* menggunakan format `key=value` yang dipisahkan spasi. Beberapa *module* tidak memerlukan argumen.

**Mencari dokumentasi module:**
- Untuk melihat daftar semua *module* yang tersedia di terminal:
  ```bash
  ansible-doc -l
  ```
- Untuk mengakses dokumentasi *module* tertentu dari terminal:
  ```bash
  ansible-doc service
  ```

## 8. Variables

Ansible menggunakan *variables* untuk mengelola perbedaan antar sistem. Untuk merepresentasikan variasi tersebut, kamu dapat membuat variabel dengan sintaks YAML standar (termasuk *lists* dan *dictionaries*). Variabel dapat didefinisikan di *playbooks*, *inventory*, file yang dapat digunakan kembali, *roles*, atau dari *command line*. Kamu juga bisa membuat variabel saat *playbook* berjalan (*registered variables*).

**Aturan Penamaan Variable (Valid):**
```text
foo
foo_env
foo_port
foo5
_foo
```

**Aturan Penamaan Variable (Invalid/Tidak Valid):**
```text
*foo                  (Karakter khusus)
Python keywords       (async, lambda)
playbook keywords     (environment)
foo-port, foo port    (Tanda hubung atau spasi)
5foo, 12              (Diawali angka)
```

**Contoh Penggunaan Variabel:**
```yaml
- hosts: all
  remote_user: root

  vars:
    favcolor: blue

  vars_files:
    - /vars/external_vars.yml

  tasks:
    - name: This is just a placeholder
      ansible.builtin.command: /bin/echo foo
```

### Variable Precedence
Ansible menerapkan prioritas variabel (*variable precedence*). Berikut adalah urutan prioritas dari yang terlemah (paling mudah ditimpa) ke yang terkuat (paling tinggi prioritasnya/mengalahkan semua):

1.  *command line values* (contoh: `-u my_user`, ini bukan variabel).
2.  *role defaults* (didefinisikan di `role/defaults/main.yml`).
3.  *inventory file* atau *script group vars*.
4.  *inventory group_vars/all*.
5.  *playbook group_vars/all*.
6.  *inventory group_vars/\**.
7.  *playbook group_vars/\**.
8.  *inventory file* atau *script host vars*.
9.  *inventory host_vars/\**.
10. *playbook host_vars/\**.
11. *host facts* / *cached set_facts*.
12. *play vars*.
13. *play vars_prompt*.
14. *play vars_files*.
15. *role vars* (didefinisikan di `role/vars/main.yml`).
16. *block vars* (hanya untuk *tasks* di dalam *block*).
17. *task vars* (hanya untuk *task* tersebut).
18. *include_vars*.
19. *set_facts* / *registered vars*.
20. *role* (dan *include_role*) *params*.
21. *include params*.
22. *extra vars* (contoh: `-e "user=my_user"`) - **Selalu menang (prioritas tertinggi)**.

## 9. Loop

Terkadang kamu ingin mengulang sebuah *task* beberapa kali. Dalam pemrograman, ini disebut *loop*. Contoh umum di Ansible: mengubah kepemilikan (*ownership*) beberapa file menggunakan *module* `file`, membuat banyak *user* menggunakan *module* `user`, atau mengulangi langkah *polling* hingga hasil tertentu tercapai.

Ansible menawarkan dua kata kunci untuk membuat *loop*: `loop` dan `with_`.

**Simple Iterative (List):**
```yaml
- name: Add several users
  ansible.builtin.user:
    name: "{{ item }}"
    state: present
    groups: "wheel"
  loop:
    - testuser1
    - testuser2
```

**Iterative with list of hashes (List of Dictionaries):**
```yaml
- name: Add several users
  ansible.builtin.user:
    name: "{{ item.name }}"
    state: present
    groups: "{{ item.groups }}"
  loop:
    - { name: 'testuser1', groups: 'wheel' }
    - { name: 'testuser2', groups: 'root' }
```

## 10. Conditionals

Di dalam *playbook*, kamu mungkin ingin mengeksekusi *task* yang berbeda tergantung pada nilai suatu *fact* (data tentang sistem remote), sebuah variabel, atau hasil dari *task* sebelumnya. Kamu dapat melakukan semua ini dengan *conditionals* (kondisional).

**Basic example conditional:**
```yaml
tasks:
  - name: Configure SELinux to start mysql on any port
    ansible.posix.seboolean:
      name: mysql_connect_any
      state: true
      persistent: yes
    when: ansible_selinux.status == "enabled"
```

**Conditional based on facts:**
```yaml
tasks:
  - name: Shut down Debian flavored systems
    ansible.builtin.command: /sbin/shutdown -t now
    when: ansible_facts['os_family'] == "Debian"
```

## 11. Privilege Escalation (Become)

Ansible menggunakan sistem eskalasi hak akses (*privilege escalation*) yang ada untuk mengeksekusi *task* dengan hak akses *root* atau dengan izin *user* lain. Karena fitur ini memungkinkan kamu untuk 'menjadi' (*become*) *user* lain (berbeda dari *user* yang sedang *login* via SSH), kita menyebutnya `become`.

*Keyword* `become` memanfaatkan alat eskalasi hak akses yang sudah ada seperti `sudo`, `su`, `pfexec`, `doas`, `pbrun`, `dzdo`, `ksu`, `runas`, `machinectl`, dll.

**Contoh task dengan become:**
```yaml
- name: Run a command as root
  command: apt update
  become: yes
  become_user: root
```

## 12. Secrets (Ansible Vault)

*Ansible Vault* mengenkripsi variabel dan file agar kamu dapat melindungi konten sensitif seperti *password* atau *keys* (kunci) daripada membiarkannya terlihat sebagai *plaintext* (teks biasa) di *playbooks* atau *roles*. Untuk menggunakan *Ansible Vault*, kamu membutuhkan satu atau lebih *password* untuk mengenkripsi dan mendekripsi konten.

**Perintah Dasar Ansible Vault:**
- **Encrypt data:**
  ```bash
  ansible-vault encrypt foo.yml bar.yml baz.yml
  ```
- **View secret (melihat isi tanpa dekripsi permanen):**
  ```bash
  ansible-vault view foo.yml bar.yml baz.yml
  ```
- **Editing secret:**
  ```bash
  ansible-vault edit foo.yml
  ```
- **Decrypt data:**
  ```bash
  ansible-vault decrypt foo.yml bar.yml baz.yml
  ```

## 13. Jinja2 Template

Ansible menggunakan *templating* Jinja2 untuk mengaktifkan ekspresi dinamis dan akses ke variabel. Ansible menyertakan banyak *filters* dan *tests* khusus untuk *templating*. Kamu juga dapat menggunakan semua *filters* dan *tests* standar yang disertakan dalam Jinja2.

File *template* Jinja2 adalah file teks yang berisi variabel yang akan dievaluasi dan diganti dengan nilai aktual saat *runtime* atau eksekusi kode. Di dalam file *template* Jinja2, kamu akan menemukan *tags* berikut:

- `{{ }}`: Kurung kurawal ganda ini adalah *tag* yang paling banyak digunakan. Digunakan untuk menyematkan variabel dan pada akhirnya mencetak nilainya selama eksekusi kode.
- `{% %}`: Sebagian besar digunakan untuk *control statements* seperti *loops* dan `if-else`.
- `{# #}`: Menandakan komentar yang mendeskripsikan suatu tugas.

**Contoh Playbook menggunakan Jinja2:**
```yaml
- hosts: appserver
  vars:
    username: user88
    version_number: 2.4
    hostname: myserver
  tasks:
    - name: jinja2 template
      template: src=index.html.j2 dest=/var/www/html/index.html
```

**Isi File `index.html.j2`:**
```html
Hi {{ username | upper }}!

This come from apache2 webserver with version {{ version_number }} is running on {{ hostname }}
```

## 14. Ringkasan

Ansible mengandalkan *Inventory* untuk mengetahui targetnya. Untuk tugas cepat, *Ad-hoc commands* adalah jawabannya, sedangkan untuk otomatisasi kompleks yang *repeatable*, kita menggunakan *Playbooks* dan *Roles*. Di dalamnya, kita menggunakan *Modules* untuk melakukan aksi nyata. Fleksibilitas Ansible sangat ditopang oleh *Variables* (dengan aturan *precedence* yang jelas), logika pengulangan (`loop`), logika kondisional (`when`), serta kemampuan mendesain file dinamis menggunakan *Jinja2*. Keamanan data sensitif ditangani oleh *Ansible Vault*, dan hak akses *root* diatur melalui `become`.

## 15. Pertanyaan untuk Menguji Pemahaman

1.  **Praktis:** Jika kamu memiliki 100 server, dan kamu ingin mengecek status *uptime* seluruh server tersebut *hanya sekali ini saja*, apakah kamu akan membuat *playbook* atau menggunakan *ad-hoc command*? Mengapa?
2.  **Analisis:** Perhatikan daftar *Variable Precedence*. Jika kamu mendefinisikan variabel `http_port: 80` di `role/defaults/main.yml` dan `http_port: 8080` di *inventory* (`group_vars/all`), port berapakah yang akan digunakan Ansible? Jelaskan alasannya.
3.  **Teknis:** Apa perbedaan fungsi antara *tag* `{{ }}` dan `{% %}` di dalam file *template* Jinja2?
4.  **Konseptual:** Mengapa kita tidak boleh menulis *password* database secara langsung (plaintext) di dalam file *playbook*, dan bagaimana cara Ansible mengatasi masalah ini?

*(Jawaban dapat ditemukan di bagian 3, 8, 12, dan 13)*

## 16. Referensi

- **Sumber Utama:** Dokumen Adinusa Freecourse "Automation With Ansible" - `Fundamentals of Ansible.docx`.
- **Referensi Tambahan (Untuk Verifikasi):**
  - [Ansible Official Documentation - Inventory](https://docs.ansible.com/ansible/latest/inventory_guide/index.html)
  - [Ansible Official Documentation - Playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/index.html)
  - [Ansible Official Documentation - Variable Precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)
  - [Ansible Vault Documentation](https://docs.ansible.com/ansible/latest/vault_guide/index.html)