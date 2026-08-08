# ansible-template-builder

Build OS template (qcow2) siap upload: repo lokal + patch CVE + kernel terbaru.
Output: `artifacts/<os>-<tanggal>.qcow2` + build log.

Target: **Proxmox** (import qcow2 → template) dan **CloudStack** (registerTemplate).
Build menghasilkan artifact; upload ke platform dilakukan manual/lanjutan.

## OS didukung (8)

| OS | Image source | Checksum | Repo lokal |
|---|---|---|---|
| ubuntu-24.04 | cloud-images.ubuntu.com (noble) | SHA256SUMS (sha256) | mirror.biznetgio.com/ubuntu |
| ubuntu-26.04 | cloud-images.ubuntu.com (resolute) | SHA256SUMS (sha256) | mirror.biznetgio.com/ubuntu |
| debian-12 | cloud.debian.org (bookworm) | SHA512SUMS (sha512) | mirror.biznetgio.com/debian (+ security) |
| debian-13 | cloud.debian.org (trixie) | SHA512SUMS (sha512) | mirror.biznetgio.com/debian (+ security) |
| alma-9 | repo.almalinux.org (pinned versi) | CHECKSUM (sha256) | mirror.biznetgio.com/almalinux |
| alma-10 | repo.almalinux.org (pinned versi) | CHECKSUM (sha256) | mirror.biznetgio.com/almalinux |
| rocky-9 | dl.rockylinux.org (pinned versi) | CHECKSUM (sha256) | mirror.biznetgio.com/rocky |
| rocky-10 | dl.rockylinux.org (pinned versi) | CHECKSUM (sha256) | mirror.biznetgio.com/rocky |

Semua URL dan isi repo sudah diverifikasi valid (Aug 2026).

## Prasyarat (builder VM, Linux — bukan Windows)

```bash
apt install ansible-core libguestfs-tools qemu-utils
ansible-galaxy collection install -r requirements.yml
```

- `virt-customize` (libguestfs) tidak jalan di Windows. Build harus di Linux.
- Builder RAM minimal 4GB. 2GB → OOM (`qemu killed by signal 9`).
- `workdir` di `/var/tmp` (disk-backed), bukan `/tmp` — `/tmp` sering tmpfs kecil, image 600MB x2 langsung penuh.

## Build

```bash
# default: ubuntu-24.04
ansible-playbook -i inventory.yml build-template.yml

# OS lain
ansible-playbook -i inventory.yml build-template.yml -e os=debian-13
ansible-playbook -i inventory.yml build-template.yml -e os=alma-10
# ... ubuntu-26.04, debian-12, alma-9, rocky-9, rocky-10

# hasil: artifacts/<os>-<tanggal>.qcow2 + artifacts/<os>-<tanggal>.log
```

Alur per build:
1. Download cloud image — `get_url` dengan `checksum` param + `retries: 3` (download terpotong → re-download otomatis) + verify final
2. `virt-customize` (offline, tanpa boot VM):
   - apt OS: pasang `local.list` ke `/etc/apt/sources.list.d/`
   - dnf OS: **hapus semua repo bawaan** (3 dir dnf5: `/etc/yum.repos.d`, `/etc/yum/repos.d`, `/etc/distro.repos.d`) — cegah duplikat ID `baseos`/`appstream` — lalu pasang `local.repo`
   - update/upgrade semua paket — patch CVE + kernel (`virt_mem: 2048` untuk appliance)
3. Cleanup: `machine-id` di-nol-kan, `/var/log/*` dihapus (toleran file missing)
4. Compress (`qemu-img convert -c`) + tulis build log (isi versi kernel)

## Verify (test)

```bash
ansible-playbook -i inventory.yml verify-template.yml
```

Test yang membuat template layak upload:
1. Deploy test VM dari artifact (PVE atau CloudStack — TODO: isi deploy di `verify-template.yml`)
2. Assert di test VM:
   - `uname -r` = kernel patch terbaru
   - repo lokal terpasang
   - tidak ada pending security update (`apt list --upgradable` / `dnf check-update --security`)
   - cloud-init jalan (hostname + SSH key dari platform)
3. Destroy test VM

Pass = layak upload. Gagal = jangan upload.

## Tambah OS

```bash
cp roles/build-template/vars/ubuntu-24.04.yml roles/build-template/vars/<os>.yml
# edit: image_url, checksum_url, checksum_algorithm, checksum_file, image_file
#       repo_dest_dir (apt/dnf), repo_filename, repo_disable_cmd, repo_file_content, update_cmd
```

**Penting — dnf OS (`alma-*`, `rocky-*`):**
- PIN ke nama versi (mis. `AlmaLinux-10-GenericCloud-10.2-20260526.0.x86_64.qcow2`), JANGAN pakai `-latest`. Race: CHECKSUM vs image bisa beda hash saat rilis baru.
- `repo_disable_cmd` hapus repo bawaan dari 3 dir (dnf5 baca semua).

Format checksum beda per distro, tapi ekstraksi di `download.yml` generik
(`grep -oE '[a-f0-9]{64,}'` di baris yang cocok nama file):
- Ubuntu/Debian/Alma: `hash  file`
- Rocky: `SHA256 (file) = hash`

## Upload hasil (manual)

- **CloudStack:** taruh qcow2 di web server (HTTP), lalu `registerTemplate` via API
- **Proxmox:** `qm importdisk <vmid> <file>.qcow2 <storage>` → `qm set` cloudinit → `qm template`

## Struktur

```
roles/build-template/
├── defaults/main.yml    # workdir (/var/tmp), virt_mem, nama artifact
├── tasks/
│   ├── main.yml         # include_vars <os>.yml + pastikan workdir + 3 step
│   ├── download.yml     # download + verify checksum (checksum param + retry)
│   ├── customize.yml    # repo lokal + update/upgrade + cleanup
│   └── artifact.yml     # compress + build log
└── vars/                # 1 file per OS (data, bukan logika)
```

## Catatan

- Mirror Biznet Gio: Debian security di path terpisah `/debian-security`
- dnf OS (dnf5) baca repo dari 3 direktori; repo bawaan dihapus total, `local.repo` satu-satunya
- GPG key repo lokal: tidak perlu untuk mirror resmi (tanda tangan GPG sama dengan official)
- `verify-template.yml` belum terisi platform deploy (PVE/CloudStack) — TODO
