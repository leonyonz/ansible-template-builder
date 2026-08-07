# ansible-template-builder

Build OS template (qcow2) siap upload: repo lokal + patch CVE + kernel terbaru.
Output: `artifacts/<os>-<tanggal>.qcow2` + build log.

Target: **Proxmox** (import qcow2 → template) dan **CloudStack** (registerTemplate).
Build ini menghasilkan artifact; upload ke platform dilakukan manual/lanjutan.

## OS didukung

| OS | Image source | Checksum | Repo lokal |
|---|---|---|---|
| ubuntu-24.04 | cloud-images.ubuntu.com (noble) | SHA256SUMS (sha256) | mirror.biznetgio.com/ubuntu |
| debian-12 | cloud.debian.org (bookworm) | SHA512SUMS (sha512) | mirror.biznetgio.com/debian (+ security) |
| alma-9 | repo.almalinux.org (GenericCloud) | CHECKSUM (sha256) | mirror.biznetgio.com/almalinux |
| rocky-9 | dl.rockylinux.org (GenericCloud) | CHECKSUM (sha256) | mirror.biznetgio.com/rocky |

Semua URL dan isi repo sudah diverifikasi valid (Aug 2026).

## Prasyarat (builder VM, Linux — bukan Windows)

```bash
apt install ansible-core libguestfs-tools qemu-utils
ansible-galaxy collection install -r requirements.yml
```

`virt-customize` (libguestfs) tidak jalan di Windows. Build harus di Linux.

## Build

```bash
# default: ubuntu-24.04
ansible-playbook build-template.yml

# OS lain
ansible-playbook build-template.yml -e os=debian-12
ansible-playbook build-template.yml -e os=alma-9
ansible-playbook build-template.yml -e os=rocky-9

# hasil: artifacts/<os>-<tanggal>.qcow2 + artifacts/<os>-<tanggal>.log
```

Alur per build:
1. Download cloud image + verifikasi checksum (gagal = berhenti)
2. `virt-customize`: pasang repo lokal (apt: `local.list`, dnf: `local.repo` + matikan repo bawaan)
3. Update/upgrade semua paket — patch CVE + kernel
4. Cleanup: machine-id, log
5. Compress (`qemu-img convert -c`) + tulis build log (isi versi kernel)

## Verify (test)

```bash
ansible-playbook verify-template.yml
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
cp roles/build-template/vars/ubuntu-24.04.yml roles/build-template/vars/rocky-9.yml
# edit: image_url, checksum_url, checksum_algorithm, checksum_file, image_file
#       repo_dest_dir (apt/dnf), repo_filename, repo_disable_cmd, repo_file_content, update_cmd
```

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
├── defaults/main.yml    # workdir, nama artifact
├── tasks/
│   ├── main.yml         # include_vars <os>.yml + 3 step
│   ├── download.yml     # download + verify checksum (format generik)
│   ├── customize.yml    # repo lokal + update/upgrade + cleanup
│   └── artifact.yml     # compress + build log
└── vars/                # 1 file per OS (data, bukan logika)
```

## Catatan

- Mirror Biznet Gio: Debian security di path terpisah `/debian-security`
- dnf OS: repo bawaan dimatikan (`enabled=0`), ganti `local.repo` ke mirror — build tidak narik dari internet
- `AppStream` section untuk alma/rocky belum diverifikasi 200 — kalau 404 saat build, hapus section-nya
- GPG key repo lokal: TODO di `customize.yml` — tidak perlu untuk mirror resmi (tanda tangan GPG sama dengan official)
