---
title: Talos Installation from Network
date: 2026-09-17
categories: [Homelab, Kubernetes]
tags: [talos, kubernetes, pxe, devops]
---
## Scenario
- In this setup an OS (Talos) shall be installed via [*PXE boot*](https://en.wikipedia.org/wiki/PXE_boot) (Preboot eXecution Environment) on a machine (here an HP mini PC; also a VM would be possible).
- The necessary files for the OS installation will be provided by a server (here on a Proxmox LXC).
- The local router (here OPNsense) has a *DHCP boot* entry, pointing to that LXC.
- Boot- and installation-process:
  - The mini PC is booting in *Network* mode, it asks its router for the next server.
  - The router provides the IP address of that server.
  - The mini PC requests the necessary files for OS installation.
  - The installation starts.

## Components
- mini PC: shall install Talos Linux via PXE
  - in general any machine (physical or VM) could be used
- OPNsense: provides IP of the PXE server
  - in general a DHCP server is needed
- pxe-boot LXC: provides necessary files for OS installation
  - in general any server, reachable in that network, could be used
- devops-vm: used for preparing files and managing the kubernetes cluster
  - any local machine could be used

## Setup
### Naming of Kubernetes Cluster and Node
- Cluster name: ``homelab``
  - The cluster name expresses the *purpose of the cluster* only, or, more precisely, its workloads.
  - Omitting name components, expressing
    - that this is a kubernetes cluster: In contexts where the cluster name occurs, it is already clear, that this is a kubernetes cluster.
    - the machine type or hostname, the cluster is running on: The same cluster could also run on a different machine
- Node name: ``k8s-homelab-cp1``
  - has to be unique in the local network
  - Therefore, the node name has to contain the name of the cluster running on it, as well as the role of the node within that cluster.
  - The prefix 'k8s' is added to express that this machine is a kubernetes node.
    In contrast to the naming of the cluster, the node name also occurs in contexts that have nothing to do with kubernetes (e.g., in the router's DHCP leases, firewall rules, etc.).
- Static IP for the node: Choose a static IP for the kubernetes node. Consider, in which VLAN it shall run.

### Preparing OPNsense Router
- Assume the new kubernetes node ``k8s-homelab-cp1`` (in this case, the HP mini PC) is connected to VLAN 110 (Homelab_Servers).
- Assume the pxe-boot LXC is connected to VLAN 120 (Homelab_Services).
- Add a DHCP boot entry for VLAN 110, pointing to pxe-boot-LXC (if not already present).
- Add firewall rules for *k8s-homelab-cp1* (VLAN 110) -> *pxe-boot LXC* (VLAN 120): allow
  - UDP 69 (TFTP) to get the file ``ipxe.efi``
  - TCP 80 (HTTP) to get the file ``boot.ipxe``
  - add aliases for single host IPs or ports as necessary (for use in firewall rules)
- Firewall already allows access from
  *Homelab_Services* (VLAN 120) -> *Homelab_Servers* (VLAN 110):
  - important for access from ``devops-vm`` -> ``k8s-homelab-cp1``
  - ``talosctl`` via TCP 50000
  - ``kubectl`` via TCP 6443

### Preparing Mini PC
- Disable 'Secure Boot' in UEFI.
  - Otherwise the Talos image will be rejected.
  - Talos also offers a 'Secure Boot' image, but importing that key into the mini PC's BIOS would be more effort.
- Set boot order to Network/PXE first.
  - Later, when Talos is installed, the PC still starts booting via PXE, but Talos itself checks, whether the disk matches the desired state and then skips the install phase.
- Enable PXE boot (check checkbox "Network (PXE) Boot")
- Disable 'Fast Boot' in UEFI:
  - Once Talos is installed, with fast boot enabled, the boot order with first entry Network/PXE will be ignored and the PC boots directly from disk.
  - With fast boot enabled, the network boot still can be triggered manually from UEFI boot menu.
- Choose behavior after power loss: "previous state" seems to be a good choice, since it prevents the mini PC from being started after a power loss, when it was intentionally powered off beforehand.
- **Note**: After changing the Secure Boot setting, the PC will show a warning and request entering a number for security reasons.  
  If using the numpad for entering that number it has to be enabled!

### Preparing pxe-boot LXC
#### Overview
- A Proxmox Linux Container (LXC) with Alpine Linux is used in this setup to provide the necessary files for the Talos intallation via PXE.
- The pxe-boot LXC runs two services, each responsible for one stage of the boot:
  - **``dnsmasq``**:
    - Serves ``ipxe.efi`` via TFTP (UDP 69).
      This is the very first file the mini PC's UEFI firmware requests, at the address OPNsense's DHCP boot entry points it to.
    - Service is configured to use TFTP only, its DHCP/DNS functions are disabled (unused here)
  - **``nginx``**:
    - Serves everything else via HTTP (TCP 80), once iPXE itself is running.
    - ``boot.ipxe`` - the boot script
    - ``vmlinuz``, ``initramfs.xz`` - OS files (kernel, initramfs)
    - ``controlplane.yaml`` - machine config
- The following directory structure and files will be created in the next steps:
  ````
  /srv/tftp/
          |-- ipxe.efi                         # iPXE runtime
  /var/www/talos/
          |-- boot.ipxe                        # Boot menu for different machines
  /var/www/talos/images/vm
                        |-- initramfs.xz       # initramfs & kernel
                        |-- vmlinuz            # specific for VMs
  /var/www/talos/images/baremetal-amd
                        |-- initramfs.xz       # initramfs & kernel
                        |-- vmlinuz            # specific for bare metal on AMD
  /var/www/talos/nodes/talos-proto-cp1         # will be renamed later
                        |-- controlplane.yaml
  /var/www/talos/nodes/k8s-homelab-cp1         # one dir per node
                        |-- controlplane.yaml  # machine config only
  ````
- The split between *image* and *node* allows to re-use an image on different nodes (so far they share the same type of hardware).

#### Installation & Config of the Services 'dnsmasq' & 'nginx'
- Install the two services ``dnsmasq`` & ``nginx``:
  ````bash
  apk update && apk add dnsmasq nginx
  ````
- Create directory structure:
  ````bash
  mkdir -p /srv/tftp /var/www/talos/
  mkdir -p /var/www/talos/images/vm /var/www/talos/images/baremetal-amd /var/www/talos/nodes/talos-proto-cp1 /var/www/talos/nodes/k8s-homelab-cp1
  ````
- Create the dnsmasq configuration file ``/etc/dnsmasq.conf``:
  ````ini
  port=0               # disables DHCP & DNS functions
  interface=eth0
  enable-tftp
  tftp-root=/srv/tftp
  ````
- Create nginx configuration file ``/etc/nginx/http.d/default.conf``:
  ````nginx
  server {
      listen 80 default_server;
      listen [::]:80 default_server;

      root /var/www/talos;
      autoindex on;

      location / {
          try_files $uri $uri/ =404;
      }

      location = /404.html {
          internal;
      }
  }
  ````
- Enable and start services ``dnsmasq`` & ``nginx``:
  ````bash
  # Registers dnsmasq to auto-start at boot:
  rc-update add dnsmasq default
  # Starts dnsmasq with its config:
  rc-service dnsmasq start

  # Validates nginx.conf/site-config syntax:
  nginx -t
  # Registers nginx to auto-start at boot:
  rc-update add nginx default
  # Restarts nginx with its config:
  rc-service nginx restart
  ````
- ``rc-update add`` & ``rc-service start`` are Alpine's OpenRC equivalents for ``systemctl enable / start``.

#### Generating Files ipxe.efi & boot.ipxe
##### ``ipxe.efi``:
- The ``ipxe.efi`` is the compiled [iPXE](https://ipxe.org/) runtime itself, a small UEFI executable containing the full iPXE stack (NIC drivers, HTTP/TFTP clients, its scripting engine).
- In the following a minimal script (``embed.ipxe``) will be embedded that just does a fresh DHCP negotiation and hands off to ``boot.ipxe`` over HTTP.
- The ``ipxe.efi`` is fetched via TFTP because TFTP is the one protocol every UEFI PXE ROM speaks natively. That's why it has to arrive over TFTP while everything downstream of it (the actual OS files) can move to the faster/more flexible HTTP.
- What it does at boot: the mini PC's UEFI firmware TFTP-fetches and executes ``ipxe.efi``, which replaces the firmware's minimal PXE stack with the full iPXE runtime. Immediately on starting, iPXE runs the embedded script (dhcp → chain http://.../boot.ipxe) and hands off to ``boot.ipxe`` over HTTP.
- Keeping the embedded script minimal means all real boot logic lives in the freely-editable ``boot.ipxe`` below, instead of requiring a rebuild for every change.
- Since the Alpine LXC doesn't contain the necessary tools, this file can be built on the devops-vm (local machine) and copied over to the LXC later.
- Run the following commands to build the ``ipxe.efi`` (replace ``<static-ip-pxe-boot>`` with the LXC's IP):
  ````bash
  # Install necessary packages for 'make' command:
  apt install -y build-essential liblzma-dev

  # Clone 'ipxe' repo:
  git clone https://github.com/ipxe/ipxe.git
  # Change into the repo's 'src' directory:
  cd ipxe/src

  # Create little script file 'embed.ipxe':
  cat > embed.ipxe <<'EOF'
  #!ipxe
  dhcp
  chain http://<static-ip-pxe-boot>/boot.ipxe
  EOF

  # Compile 'ipxe.efi' with embedded 'embed.ipxe':
  make bin-x86_64-efi/ipxe.efi EMBED=embed.ipxe

  # Copy the newly built file ``ipxe.efi`` into its destination directory on the LXC:
  scp bin-x86_64-efi/ipxe.efi pxe-boot:/srv/tftp/ipxe.efi
  ````
- For this first setup a hardcoded static IP of the LXC is used. Choose and set a static IP for the pxe-boot LXC. Consider, in which VLAN it shall run.
- Later this hardcoded IP address can be replaced by iPXE's `${next-server}` variable (populated from OPNsense's DHCP "Next Server" field).  
  The corresponding line in the ``embed.ipxe`` then changes to: `chain http://${next-server}/boot.ipxe`

##### ``boot.ipxe``:
- The file ``boot.ipxe`` is used to determine which host is requesting its boot config.
- Since it is a simple text file, it can be created directly on the LXC.
- Create the file ``/var/www/talos/boot.ipxe`` on the`` pxe-boot`` LXC:
  ````
  #!ipxe

  set default local
  iseq ${mac} bc:24:11:f9:4b:11 && set default vm-talos-proto ||
  iseq ${mac} 48:9e:bd:33:1c:66 && set default minipc-talos-homelab ||

  menu Boot menu (${mac})
  item local                 Boot from local disk
  item vm-talos-proto        Talos - VM control-plane (talos-proto-cp1)
  item minipc-talos-homelab  Talos - Mini PC control-plane (k8s-homelab-cp1)
  item minipc-talos-maint    Talos - Mini PC (maintenance mode, no config)
  choose --default ${default} --timeout 5000 target && goto ${target} || goto local

  :local
  exit

  :vm-talos-proto
  kernel http://<static-ip-pxe-boot>/images/vm/vmlinuz talos.platform=metal talos.config=http://<static-ip-pxe-boot>/nodes/talos-proto-cp1/controlplane.yaml console=ttyS0 console=tty0 slab_nomerge pti=on
  initrd http://<static-ip-pxe-boot>/images/vm/initramfs.xz
  boot

  :minipc-talos-homelab
  kernel http://<static-ip-pxe-boot>/images/baremetal-amd/vmlinuz talos.platform=metal talos.config=http://<static-ip-pxe-boot>/nodes/k8s-homelab-cp1/controlplane.yaml console=ttyS0 console=tty0 slab_nomerge pti=on
  initrd http://<static-ip-pxe-boot>/images/baremetal-amd/initramfs.xz
  boot

  :minipc-talos-maint
  kernel http://<static-ip-pxe-boot>/images/baremetal-amd/vmlinuz talos.platform=metal console=ttyS0 console=tty0 slab_nomerge pti=on
  initrd http://<static-ip-pxe-boot>/images/baremetal-amd/initramfs.xz
  boot
  ````

### Preparing Files for Talos config on devops-VM
- Download kernel and initramfs from [Talos Linux Image Factory](https://factory.talos.dev/):
  - select extension ``amd-ucode`` for running Talos on an AMD CPU
  - do not select qemu-guest-agent - only needed for Talos on a VM
- Copy both files onto pxe-boot LXC under ``/var/www/talos/images/baremetal-amd`` (see section 'pxe-boot LXC')
- On the devops-vm, create directory ``talos-homelab`` (to hold the Talos OS config).
  In this directory do:
  - ``talosctl gen secrets -o secrets.yaml`` -> creates the cluster's CA
    - can be re-used for further nodes in the same cluster
  - ``talosctl gen config homelab https://<static-ip>:6443 --with-secrets secrets.yaml -o .`` -> creates three files:
    - `controlplane.yaml`: The machine config
    - ``worker.yaml``: Config for a worker node (unused here)
    - ``talosconfig``: Client credentials for talosctl
- Boot the mini PC into Talos-maintenance-mode:
  - **Attention**: Interrupt the boot menu and select the menu entry for Talos-maintenance-mode: ``minipc-talos-maint``!
  - This will boot Talos without its config (the controlplane.yaml), which needs some editing before it's ready to use.
- Determine the diskname on the mini PC that Talos shall be installed to:
  - When the mini PC is running in Talos maintenance mode, execute on the devops-vm:
    ``talosctl get disks -n <ip-maintenance-mode> --insecure``
  - ``-n <ip-maintenance-mode> --insecure`` is needed, since the connection to the Talos node isn't fully configured and secured yet.
  - ``<ip-maintenance-mode>``: This is not the static IP, used for generating the config for the Talos node. At this stage, the mini PC has gotten some IP from its DHCP server, visible in the DHCP leases or in the console of the running mini PC.
- Edit the following values in the generated ``controlplane.yaml``:
  - ``<MAC-address>``: Visible in OPNsense, when the mini PC is running
  - ``<static-ip>``: Set the chosen static IP.
  - ``<gateway>``: Visible in opnsense - see corresponding VLANs gateway address
  - ``<diskname>``: Set the diskname, determined before (see above).
  - ``<age-key>``: Use an existing SOPS-age-key (of an existing cluster) or generate a new one.
  - ``allowSchedulingOnControlPlanes: true``: Set to 'true' if the controlplane node shall run workloads (e.g., in a one-node-cluster).
  - ``<hostname>``: Set your hostname (node name)
  ```
    1 version: v1alpha1 # Indicates the schema used to decode the contents.
    2 debug: false # Enable verbose logging to the console.
    3 persist: true
    4 # Provides machine specific configuration options.
    5 machine:
    6     network:
    7         interfaces:
    8             - deviceSelector:
    9                   hardwareAddr: <MAC-address>
   10               addresses:
   11                   - <static-ip>
   12               routes:
   13                   - network: 0.0.0.0/0
   14                     gateway: <gateway>
   15     type: controlplane # Defines the role of the machine within the cluster.
    #   ...
   84     # Used to provide instructions for installations.
   85     install:
   86         disk: /dev/<diskname> # The disk used for installations.
   87         image: ghcr.io/siderolabs/installer:v1.13.7 # Allows for supplying the image used to perform the installation.
   88         wipe: false # Indicates if the installation disk should be wiped at installation time.
   89         grubUseUKICmdline: true # Indicates if legacy GRUB bootloader should use kernel cmdline from the UKI instead of building it on the host.
    #   ...
  356     # A list of inline Kubernetes manifests.
  357     inlineManifests:
  358         - name: flux-sops-decryption-key
  359           contents: |-
  360             apiVersion: v1
  361             kind: Secret
  362             metadata:
  363               name: sops-age
  364               namespace: flux-system
  365             type: Opaque
  366             stringData:
  367               age.agekey: |
  368                 <age-key>
    #   ...
  397     # Allows running workload on control-plane nodes.
  398     allowSchedulingOnControlPlanes: true
  399 ---
  400 apiVersion: v1alpha1
  401 kind: HostnameConfig
  402 hostname: <hostname> # A static hostname to set for the machine.
  ```
- Copy the edited ``controlplane.yaml`` to the ``pxe-boot``-LXC:
  ``scp controlplane.yaml pxe-boot:/var/www/talos/nodes/k8s-homelab-cp1/controlplane.yaml``
- Apply the changed config to the Talos node (the node will automatically reboot during this process):
  ``talosctl apply-config --insecure -n <node-ip> -e <endpoint-ip> -f controlplane.yaml``
- After booting the Talos node with its complete config, bootstrap the node:  
  ``talosctl bootstrap -n <node-ip> -e <endpoint-ip> --talosconfig talosconfig``
    - In a one-node-cluster ``<node-ip>`` and ``<endpoint-ip>`` are usually identical.

#### talosconfig and kubeconfig:
- Finish the ``talosconfig``:
  - To not have to specify the node and endpoint for every talosctl-command, these can be set in the file ``talos-homelab/talosconfig``. Execute:
    - ``talosctl config endpoint <endpoint-ip> --talosconfig talosconfig``
    - ``talosctl config nodes <node-ip> --talosconfig talosconfig``
  - Merge the ``talosconfig`` into ``~/.talos/config``:
    ``talosctl config merge talosconfig``
- Pull a new ``kubeconfig context`` from the cluster's API and add it to ``kubeconfig``:
  - ``talosctl kubeconfig --talosconfig talosconfig``
