# ansible-role-ofed Specification

This Markdown document defines the behaviour of the **ansible‑role‑ofed** Ansible
role.  It is intended for use with specification‑driven development
environments where an LLM reads the specification and generates or
updates code accordingly.  A separate document, `README.md`, provides
user‑facing instructions and should not be confused with this machine‑oriented
spec.

## Purpose

Install and configure the Mellanox (NVIDIA) OpenFabrics Enterprise
Distribution (OFED) driver for high performance InfiniBand/Ethernet
interfaces.  Depending on variables supplied by the caller the role also
updates the kernel to match the driver, writes `openib.conf`, and
configures SR‑IOV and vDPA features.  Supported long‑term support (LTS)
platforms include:

* **Red Hat family**: RHEL 7/8/9 and derivatives such as CentOS,
  Rocky Linux and Alma Linux.
* **Ubuntu LTS**: 20.04, 22.04 and 24.04.
* **Debian**: 11 (bullseye) and 12 (bookworm).
* **Proxmox VE**: 7 (Debian 11 based) and 8 (Debian 12 based).

The role defaults to using the latest 24.10 long‑term support branch of
the MLNX_OFED driver (`latest-24.10`).  According to NVIDIA’s
documentation the 24.10 release, published October 2024, is the
final standalone MLNX_OFED LTS version and will receive three years of
support.

## Input variables

Variables declared in `defaults/main.yml` may be overridden in the
inventory or playbook.  Significant variables include:

| Variable | Description |
|---------|-------------|
| `ofed_version` | Version string appended to the repository URL.  Defaults to `latest-24.10` (latest sub‑release of MLNX_OFED 24.10 LTS). |
| `ofed_repository_url` | Base URL for the Mellanox public repository. |
| `ofed_target_release` | Optional distribution string used to override automatic detection of the repository path. |
| `ofed_update_kernel` | When true (default on Red Hat hosts) the role installs the kernel matching the OFED drivers and updates the boot loader. |
| `ofed_nobest` | Pass `--nobest` to `dnf` when installing kernels on EL 8/9 systems. |
| `ofed_grub_backup` | Backup existing GRUB configuration before modification on Red Hat hosts. |
| `ofed_manage_openib_conf` | Deploy and manage `/etc/infiniband/openib.conf` from a Jinja2 template. |
| `ofed_force_mode` | Set `FORCE_MODE=yes` in `openib.conf` to force driver loading. |
| `ofed_rdma_ucm_load` | Set `RDMA_UCM_LOAD=yes` in `openib.conf` to load the UCX RDMA module. |
| `ofed_remove_distro_packages` | Remove conflicting RDMA/InfiniBand packages on Debian/Ubuntu/Proxmox prior to installation. |
| `ofed_enable_sriov` | Enable SR‑IOV configuration on Mellanox devices. |
| `ofed_sriov_interfaces` | List of interface names or PCI addresses to configure for SR‑IOV. |
| `ofed_sriov_num_vfs` | Default number of virtual functions (VFs) per physical interface. |
| `ofed_sriov_num_vfs_map` | Per‑interface override mapping of VF counts. |
| `ofed_sriov_iommu_options` | Kernel command line options to enable IOMMU/passthrough (e.g. `intel_iommu=on iommu=pt pci=realloc`). |
| `ofed_enable_vdpa` | Enable vDPA configuration on Mellanox devices.  vDPA
  relies on SR‑IOV: if `ofed_enable_vdpa` is set the role will also run
  the SR‑IOV tasks. |
| `ofed_vdpa_eswitch_mode` | Eswitch mode to apply to PFs when enabling vDPA (`switchdev` recommended). |
| `ofed_vdpa_pf_devices` | PCI addresses of physical functions on which to enable vDPA.  If empty the role derives PFs from `ofed_sriov_interfaces`. |
| `ofed_vdpa_devices` | List of vDPA devices to create; each dictionary includes `name`, `mgmtdev` (VF PCI address) and optional `mac`. |

Additional OS‑specific variables are defined under `vars/` and loaded by
`tasks/variables.yml`.

### Debian/Ubuntu/Proxmox specific variables

Debian‑based distributions use the APT package manager and require
repository keys to be stored in a keyring.  The role defines the
following additional variables when `ansible_pkg_mgr` is `apt`:

| Variable | Description |
|---------|-------------|
| `ofed_apt_keyrings_dir` | Directory where repository GPG keys are stored.  Defaults to `/etc/apt/keyrings`. |
| `ofed_apt_key_filename` | Name of the Mellanox GPG key file (e.g. `mellanox_ofed.asc`). |
| `ofed_apt_key_dest` | Full path to the key file under `ofed_apt_keyrings_dir`.  The role downloads the GPG key here and references it via the `signed‑by` directive. |
| `ofed_sriov_grub_config` | Path to the GRUB configuration file on Debian/Proxmox systems.  Defaults to `/etc/default/grub`. |
| `ofed_sriov_grub_params` | A list of dictionaries derived from `ofed_sriov_iommu_options` with keys `name` and optional `value`.  Used internally to update GRUB parameters idempotently. |

## Task summary

### Common initialisation

1. Run `tasks/variables.yml` to load OS‑specific variables and compute
   derived variables such as `ofed_repo_dist_name` (distribution
   identifier used in the repository path), `ofed_repo_file_name`,
   `ofed_repo_url` and lists of packages to install or remove.
2. Use values from `defaults/main.yml` and `vars/*.yml` to set facts for
   subsequent tasks.

### Debian/Ubuntu/Proxmox

1. **Create APT keyring and download GPG key**: ensure the keyring
   directory (`ofed_apt_keyrings_dir`, default `/etc/apt/keyrings`) exists
   and download the Mellanox GPG key from `ofed_gpg_key_url` into
   `ofed_apt_key_dest`.  The role no longer uses the deprecated
   `apt_key` module; instead it stores keys in `/etc/apt/keyrings` and
   references them via `signed‑by`.
2. **Add repository file**: download the vendor‑supplied repository list
   to `/etc/apt/sources.list.d/{{ ofed_repo_file_name }}` via
   `get_url` and insert a `signed‑by={{ ofed_apt_key_dest }}` directive
   into the `deb` line.  This ensures APT trusts packages from the
   Mellanox mirror when apt‑key is not used.
3. **Remove conflicting packages**: if `ofed_remove_distro_packages` is
   true purge packages listed in `ofed_deb_remove_packages` (e.g.
   `libipathverbs1`, `librdmacm1`, `libibverbs1` etc.) using `apt`.
4. **Update package cache**: run `apt` with `update_cache: yes`.
5. **Install OFED packages**: install the packages specified by
   `ofed_deb_packages`.
6. **Enable openibd service**: ensure `openibd` is enabled and started.

### Red Hat family (EL7/8/9)

1. **Configure repository**: use the `yum_repository` module to set
   `baseurl` to `{{ ofed_repository_url }}/{{ ofed_version }}/{{ ofed_repo_dist_name }}/` and
   register the GPG key.
2. **Install package group**: install the `Infiniband Support` group with
   `groupinstall` or `@Infiniband Support`.
3. **Install additional packages**: install RPMs listed in
   `ofed_rpm_packages`.
4. **Kernel update**: if `ofed_update_kernel` is true, detect the
   version of the `kmod‑mlnx‑ofa_kernel` package and install the
   corresponding `kernel-<version>` package.  Update the default boot
   entry with `grub2-set-default` and regenerate GRUB configuration.
5. **Enable openibd service**: enable and start `openibd`.

### Managing `openib.conf`

When `ofed_manage_openib_conf` is true, render
`templates/openib.conf.j2` to `/etc/infiniband/openib.conf`.  The
template sets `FORCE_MODE` and `RDMA_UCM_LOAD` according to
`ofed_force_mode` and `ofed_rdma_ucm_load`.  Notify the `restart openibd`
handler when the file changes.

### SR‑IOV configuration

SR‑IOV exposes multiple virtual functions (VFs) on a single physical
function (PF).  To enable SR‑IOV (or when vDPA is enabled) the role:

1. **Install tools**: ensure `iproute2` and `pciutils` (Debian/Ubuntu/Proxmox) or
   `iproute` and `pciutils` (Red Hat) are installed.
2. **Start MST service**: call `mst start` to initialise Mellanox
   firmware management tools; ignore failures.
3. **Discover PF devices**: for each interface name in
   `ofed_sriov_interfaces`, read its symlink under `/sys/class/net` to
   obtain the PCI address and save to `sriov_pf_devices`.
4. **Enable firmware SR‑IOV**: run `mlxconfig -d <pci> set SRIOV_EN=1
   NUM_OF_VFS=<n>` on each PF to enable SR‑IOV in firmware.  Skip if
   `mlxconfig` is unavailable.
5. **Create VFs**: write the number of VFs to
   `/sys/class/net/<iface>/device/sriov_numvfs`.  Use
   `ofed_sriov_num_vfs_map` overrides if provided; otherwise apply
   `ofed_sriov_num_vfs` uniformly.
6. **Enable IOMMU and update the boot loader**: configure kernel
   parameters for SR‑IOV.  On Red Hat systems use `grubby` to append
   `ofed_sriov_iommu_options` to all installed kernels.  On
   Debian/Proxmox hosts the role parses
   `ofed_sriov_iommu_options` into individual parameters and then
   idempotently inserts or updates each one in
   `GRUB_CMDLINE_LINUX_DEFAULT` within `/etc/default/grub`.  The
   implementation mirrors the approach used in the
   `pve_i915_sriov_dkms` role: first normalise the GRUB line so that
   there is always a space between existing parameters and then use a
   complex back‑reference regex to insert or update each parameter
   without duplicating entries.  After modifying the file the role
   runs `update-grub` and `update-initramfs` and triggers a reboot if
   necessary.
7. **Reboot**: instruct the caller to reboot if kernel parameters or
   kernel versions changed.

### vDPA configuration

vDPA (virtio Data Path Acceleration) builds on SR‑IOV; it relies on the
presence of virtual functions for data path offload.  If
`ofed_enable_vdpa` is true the role performs the following additional
steps:

1. **Load `vhost_vdpa` module**: run `modprobe vhost_vdpa`.
2. **Ensure devlink utility**: install `iproute2`/`iproute` package
   providing the `devlink` command.
3. **Determine PF devices**: use `ofed_vdpa_pf_devices` if provided
   otherwise fall back to `sriov_pf_devices` gathered during SR‑IOV
   setup.
4. **Set eswitch mode**: for each PF run `devlink dev eswitch set
   pci/<pci> mode {{ ofed_vdpa_eswitch_mode }}`.  Hardware offload
   typically requires `switchdev` mode.
5. **Create vDPA devices**: for each item in `ofed_vdpa_devices` run
   `vdpa dev add name <name> mgmtdev pci/<vf> [mac <mac>]` to create
   vDPA devices accessible under `/dev/vhost-vdpa-*`.

## Notes and assumptions

* vDPA depends on SR‑IOV.  If `ofed_enable_vdpa` is true and
  `ofed_enable_sriov` is false the role will still execute the SR‑IOV
  tasks to create VFs because vDPA cannot operate without them.
* BIOS/firmware settings must enable virtualization (VT‑x/VT‑d) and SR‑IOV.
* The `mlxconfig` and `vdpa` binaries are provided by the OFED package;
  if they are unavailable on the target the corresponding steps will be
  skipped.
* Configuring vDPA on the host does not automatically configure
  hypervisor tools (e.g. libvirt).  Additional guest definitions are
  required to use vDPA interfaces.