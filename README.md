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
 - In order to have a GUI for Virtual Machine Manager that allows us to manage our local infrastructure from one window—without resorting to terminal commands—we need to install a server with a GUI by executing the following command:
```bash
sudo dnf group install 'Server with GUI'
```
