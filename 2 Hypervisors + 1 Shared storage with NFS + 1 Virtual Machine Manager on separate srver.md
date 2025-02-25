# Virtualization with KVM and QEMU
For this exercise, we will need:

- VMware installed. I tried Oracle VirtualBox, but it doesn't properly support nested virtualization.
- Any version of a Linux server will work. Personally, I'll be using AlmaLinux 9.5 on hypervisors with NFS, and Ubuntu 22.04 on Virtual Machine Manager (VMM).
- A terminal capable of establishing an SSH connection (for example, PowerShell). I prefer to use [MobaXterm](https://mobaxterm.mobatek.net/).

VM Preparation:
- Open the VM’s Settings.
- Go to Network.
- Under Attached to, select Bridged Adapter.

![Image](https://github.com/user-attachments/assets/59365b7b-fb16-48c7-bcad-865a05a1b0eb)
 - Check if the Nested virtualisation is enabled to be able to run VM's insite current VM.
![изображение](https://github.com/user-attachments/assets/540cba8c-a00c-4443-a4b1-ae461cf75dbe)

 - Clone the original VM in which you installed the Linux OS and name it "Ansible Host."
 - Be sure to select the option to generate new MAC addresses.
 - The installation process will be skipped.
 
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
