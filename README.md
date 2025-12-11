# Ansible Role: Mellanox OFED driver

This role installs and configures Mellanox’s Open‑Fabrics Enterprise Distribution (OFED) driver on Red Hat, Debian/Ubuntu, and Proxmox VE based systems.  It follows the same directory layout and modular design used in the `ktooi/ansible‑role‑crio` role so that tasks, variables and handlers are clearly separated and easy to extend.  The role supports additional logic for kernel updates, SR‑IOV, and vDPA configuration.  Current long‑term support (LTS) releases are targeted: Ubuntu 20.04/22.04/24.04, Debian 11/12, Proxmox 7/8, and Red Hat 7/8/9 (including CentOS, Rocky, Alma and other clones).

## Requirements

* An Ansible controller with Python ≥ 3.6.
* Target hosts must be reachable via SSH and have privilege escalation (sudo) enabled.
* For Red Hat family distributions the `python3‑dnf` package should be installed for DNF support.

## Role Variables

Most variables have sane defaults so the role can be executed without any configuration.  Variables may be overridden in a playbook or inventory as needed.

| Variable | Default | Description |
|---------|---------|-------------|
| `ofed_version` | `"latest-24.10"` | Release of the Mellanox OFED repository to use.  By default the role targets the latest sub‑release of the 24.10 long‑term support (LTS) branch (e.g. 24.10‑3.2.5.0).  NVIDIA states that 24.10 is the final standalone MLNX_OFED LTS release and will receive updates for three years. |
| `ofed_repository_url` | `"https://linux.mellanox.com/public/repo/mlnx_ofed"` | Base URL for the Mellanox public repository.  The vendor documentation instructs users to download the `.repo`/`.list` file from this repository.  In this role the repository list is downloaded and then modified to reference the downloaded GPG key via the `signed‑by` directive. |
| `ofed_target_release` | *empty* | Optional override for the distribution flavour in the repository path (e.g. `rhel7.9`, `ubuntu22.04`).  When unset the role derives the correct value from `ansible_distribution` and `ansible_distribution_version`. |
| `ofed_update_kernel` | `true` | Whether to install and activate the kernel that matches the Mellanox driver. |
| `ofed_nobest` | `false` | Pass the `--nobest` flag to the package manager when installing kernels under Red Hat systems. |
| `ofed_grub_backup` | `true` | If true, make a backup of the existing grub configuration before writing a new one (only relevant on Red Hat systems). |
| `ofed_manage_openib_conf` | `false` | Whether to manage `/etc/infiniband/openib.conf`.  When true the role will deploy a template and restart the `openibd` service. |
| `ofed_force_mode` | `false` | Sets the `FORCE_MODE` flag in `openib.conf` to enable loading the driver stack even if other drivers are present. |
| `ofed_rdma_ucm_load` | `false` | Controls the `RDMA_UCM_LOAD` flag in `openib.conf`. |
| `ofed_remove_distro_packages` | `true` | Remove conflicting RDMA/InfiniBand packages provided by the distribution.  The vendor recommends removing these packages on Ubuntu/Debian before installing MLNX‑OFED. |
| `ofed_enable_sriov` | `false` | Enable SR‑IOV on Mellanox devices.  When true the role sets IOMMU kernel options, configures the NIC firmware via `mlxconfig`, and creates virtual functions on the specified interfaces. |
| `ofed_sriov_interfaces` | `[]` | List of physical network interfaces (or PCI addresses) on which to enable SR‑IOV.  Each interface will have `ofed_sriov_num_vfs` virtual functions created unless overridden in `ofed_sriov_num_vfs_map`. |
| `ofed_sriov_num_vfs` | `8` | Default number of virtual functions to create per interface when SR‑IOV is enabled. |
| `ofed_sriov_num_vfs_map` | `{}` | Optional mapping of individual interface names or PCI addresses to custom VF counts. |
| `ofed_sriov_iommu_options` | `"intel_iommu=on iommu=pt pci=realloc"` | Kernel boot parameters appended to GRUB when enabling SR‑IOV.  Adjust as required for AMD systems. |
| `ofed_enable_vdpa` | `false` | Enable virtio Data Path Acceleration (vDPA).  When true the role loads the `vhost_vdpa` module, sets the eswitch mode on the physical NICs, and creates vDPA devices. |
| `ofed_vdpa_eswitch_mode` | `"switchdev"` | Eswitch mode to set on physical functions when enabling vDPA (typically `switchdev` for hardware offload). |
| `ofed_vdpa_pf_devices` | `[]` | List of PCI addresses of physical functions for which to enable vDPA.  If empty, PFs derived from `ofed_sriov_interfaces` are used. |
| `ofed_vdpa_devices` | `[]` | List of vDPA devices to create.  Each entry should specify `name`, `mgmtdev` (VF PCI address), and optionally `mac`. |

Additional OS‑specific variables live under `vars/` and define which packages are installed and which conflicting packages should be removed.  See those files for further details.

## Dependencies

There are no external role dependencies.  However the role will call `apt`, `yum` or `dnf` modules depending on the target platform.

## Example Playbook

```yaml
---
- hosts: all
  become: true
  roles:
    - role: ansible-role-ofed
      vars:
        ofed_version: "latest-24.10"
        ofed_manage_openib_conf: true
        ofed_force_mode: true
        ofed_rdma_ucm_load: false
```

This example installs the latest Mellanox OFED driver, ensures that the correct kernel is installed, and manages `/etc/infiniband/openib.conf`.  Additional variables may be set to tune behaviour; refer to the variable table above.

## Notes

* **Repository files**:  The role downloads the appropriate repository file (`.repo` for Red Hat, `.list` for Debian/Ubuntu/Proxmox) from the Mellanox public mirror.  For Debian‑based systems it then inserts a `signed‑by` directive referencing the downloaded GPG key so that APT will trust the repository without using the deprecated `apt‑key` mechanism.  The vendor documentation recommends adding the repository via these files.
* **Removing conflicting packages**:  On Debian/Ubuntu/Proxmox systems the role purges a list of packages (e.g. `libipathverbs1`, `librdmacm1`, `libibverbs1`, `libmthca1`, `openmpi-bin`, `ibverbs-utils`, `infiniband-diags`, `ibutils`, `perftest`) prior to installation, as recommended by NVIDIA.  The cited lines show the command to remove these packages using `apt-get`; they are not a version number but an instruction for preparing the system.  You can disable this behaviour by setting `ofed_remove_distro_packages` to false.
* **Kernel updates**:  On Red Hat family systems the role attempts to match the installed Mellanox `kmod-mlnx-ofa_kernel` package to a corresponding `kernel` package.  If `ofed_update_kernel` is enabled, the role installs the matching kernel and updates GRUB to boot it by default.
* **Managing openib.conf**:  If `ofed_manage_openib_conf` is true, the role deploys a Jinja2 template for `/etc/infiniband/openib.conf` with flags such as `FORCE_MODE` and `RDMA_UCM_LOAD` set according to variables.  The `openibd` service is notified to restart when this file changes.

* **Proxmox VE support**:  Proxmox hosts are treated as Debian derivatives.  The role automatically uses the Debian package manager and repository configuration when `ansible_distribution` matches `Proxmox VE`.  All Debian‑specific tasks apply.

* **SR‑IOV support**:  When `ofed_enable_sriov` is true the role enables SR‑IOV on Mellanox devices by setting firmware flags with `mlxconfig`, adding IOMMU kernel options to GRUB, and creating the configured number of virtual functions (VFs) for each specified interface.  You can control the number of VFs globally via `ofed_sriov_num_vfs` or per interface via `ofed_sriov_num_vfs_map`.  The role also installs prerequisites such as `iproute2` and triggers a reboot if necessary.

* **vDPA support**:  When `ofed_enable_vdpa` is true the role configures hardware vDPA acceleration.  It loads the `vhost_vdpa` module, sets the Mellanox eswitch into `switchdev` mode via `devlink`, and optionally creates vDPA devices based on the `ofed_vdpa_devices` variable.  Each device can specify a name, management PCI address and MAC address.  vDPA relies on SR‑IOV: you must also specify `ofed_sriov_interfaces` (or `ofed_vdpa_pf_devices`) so that the role can create the necessary virtual functions before adding vDPA devices.  These tasks provide a foundation for deploying virtio‑vDPA accelerated VMs.

## License

Apache‑2.0

## Author Information

* **Kodai Tooi** [GitHub](https://github.com/ktooi), [Qiita](https://qiita.com/ktooi)