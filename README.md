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
 
  - Clone the original VM in which you installed the Linux OS and name it "Ansible Host."
  - Be sure to select the option to generate new MAC addresses.
