# Virtualization with KVM and QEMU
For this exercise, we will need:

- Oracle VM VirtualBox installed
- Any version of a Linux server (for example, AlmaLinux 9.5)
- A terminal capable of establishing an SSH connection (for example, PowerShell)

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

- Install packages:
```bash
sudo dnf install libvirt qemu-kvm virt-install virt-manager 
```
A bit explanation:
- libvirt: A toolkit and daemon providing a standardized API for managing virtual machines, abstracting the underlying hypervisor details.
- qemu-kvm: QEMU is an emulator that provides hardware virtualization; combined with KVM (Kernel-based Virtual Machine), it enables efficient, full virtualization on Linux.
- virt-install: A command-line utility that simplifies creating and installing new virtual machines using libvirt.
- virt-manager: Remote GUI.
- openssh-askpass: Opens a prompt window for entering the password.

- Check if the libvirtd daemon is running:
```bash
 systemctl status libvirtd
```
- Enable it if it was disabled:
```bash
sudo systemctl enable --now vibvirtd
```
Now, by executing `ip a`, you can see that a new Ethernet adapter has been added. Its name is virbr0, and it has the default address `192.168.122.1`.
