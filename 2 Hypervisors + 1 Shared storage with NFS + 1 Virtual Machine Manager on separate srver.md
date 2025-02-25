## Virtualization with KVM and QEMU
For this exercise, we will need:

- VMware installed. I tried Oracle VirtualBox, but it doesn't properly support nested virtualization.
- Any version of a Linux server will work. Personally, I'll be using AlmaLinux 9.5 on hypervisors with NFS, and Ubuntu 22.04 on Virtual Machine Manager (VMM).
- A terminal capable of establishing an SSH connection (for example, PowerShell). I prefer to use [MobaXterm](https://mobaxterm.mobatek.net/). VMware allows the use of the clipboard, making it unnecessary to establish an SSH connection—you can paste directly into the VM window. However, I am using the VMware trial just for this task. Generally, in most tasks, we won't have this feature available, which is why it's good practice to use SSH connections.

## VM Preparation:
The detailed process of creating a VM on VMware will be skipped; you can check the details [here.](https://www.youtube.com/watch?v=sJNxJghTc28)

1. Create two VMs and name them Hypervisor01 and Hypervisor02.
    - Nested virtualisation should be enabled.
- Swithc network adapter to the bridged. All VM's will be on our local network with inthernet acess.
- Ensure they have enough resources to support another VM that will be installed (nested - Virtualize Intel VT-x/EPT or AMD-V/RVI) later. In my case, it will be Alpine Linux because it is lightweight and has minimal system requirements. My settings are:

![изображение](https://github.com/user-attachments/assets/ad1dd042-bb38-43a2-90e6-4c3aaf9e09f1)

2. Create the NFS VM.
- Clone one of the Hypervisor VMs and rename it 'NFS'. You can allocate less RAM and CPU resources. Also, disable nested virtualization.
3. Create a server where VMM will be installed.
- Align the system resources with the Linux distribution you will be using. My settings are:

![изображение](https://github.com/user-attachments/assets/362c276a-2c18-473d-a5a6-ca269f10fe21)

 SSH connection and packet instalation:
 - Run your VM. 
 - To be able to connect to the VMs, we need to check their IP addresses. Open each VM window one by one, log in, and type `ip a`. Write down their IP addresses, and note which VM each one belongs to. 

![Image](https://github.com/user-attachments/assets/87d11c4f-a82e-4aeb-b91e-ff5a6f3abc45)

- Open four PowerShell windows and type the following command

```bash
ssh username@ip_that_you_write
```
 - In order to have a GUI for Virtual Machine Manager that allows us to manage our local infrastructure from one window—without resorting to terminal commands—we need to install a server with a GUI by executing the following command:
```bash
sudo dnf group install 'Server with GUI'
```
- By default, a server might boot into a multi-user (text-based) runlevel. To make the server start into the GUI, run:
```bash
sudo systemctl set-default graphical.target
```
- Reboot the server for applying changes and load GUI:
```bash
sudo reboot
```
Anyway, it’s more convenient to continue using our SSH connection so that we can easily copy and paste.

Install packages for Hypervisors:
```bash
sudo dnf install libvirt qemu-kvm virt-install 
```
A bit explanation:
- libvirt: A toolkit and daemon providing a standardized API for managing virtual machines, abstracting the underlying hypervisor details.
- qemu-kvm: QEMU is an emulator that provides hardware virtualization; combined with KVM (Kernel-based Virtual Machine), it enables efficient, full virtualization on Linux.
- virt-install: A command-line utility that simplifies creating and installing new virtual machines using libvirt.

- Check if the libvirtd daemon is running:
```bash
 systemctl status libvirtd
```
- Enable it if it was disabled:
```bash
sudo systemctl enable --now libvirtd
```
Now, by executing `ip a`, you can see that a new Ethernet adapter has been added. Its name is virbr0, and it has the default address `192.168.122.1`.

Install packages for VM manager:

```bash
 sudo dnf install virt-manager openssh-askpass
```
A bit explanation:
- virt-manager: Remote GUI.
- openssh-askpass: Opens a prompt window for entering the password.
- libvirt
- sudo dnf install -y polkit
- sudo dnf install -y polkit-kde

NFS server instalation and congiguration:
- Install NFS utilities on NFS server:
```bash
 sudo dnf install -y nfs-utils
```
- Enable and start the NFS server service
```bash
 sudo systemctl enable nfs-server --now
```
- Create a directory to export

```bash
 sudo mkdir -p /var/nfs
```
You can also adjust ownership and permissions as needed. For instance:

```bash
sudo chown -R nobody:nobody /var/nfs
sudo chmod -R 755 /var/nfs
```
- Configure the export in `/etc/exports`

```bash
sudo vi /etc/exports
```
Example line to allow read-write access for every ip:
```bash
/var/nfs *(rw,sync,no_root_squash)
```
Export the directory:

```bash
 sudo exportfs -r
```
Open necessary firewall ports:
```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload
```
If SELinux is enforcing, you might also need to allow NFS to serve files from your chosen directory. Assign a proper SELinux context to the shared directory, for example:
```bash
sudo chcon -t public_content_rw_t /srv/nfs/shared
```
Set Up the NFS Clients (repeat for each client VM):
- Install NFS utilities on NFS server:
```bash
 sudo dnf install -y nfs-utils
```
- Create a mount point:
```bash
sudo mkdir -p /mnt/nfs_shared 
```
Test connectivity by manually mounting the NFS share:
```bash
sudo mount -t nfs 192.168.1.32:/var/nfs /mnt/nfs_shared
```
Verify it’s mounted correctly:
```bash
df -h | grep nfs 
```
To have the NFS share mounted automatically whenever the client restarts, add an entry to `/etc/fstab`:
```bash

```
