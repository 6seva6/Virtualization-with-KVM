## Virtualization with KVM and QEMU
For this exercise, we will need:

- VMware installed. I tried Oracle VirtualBox, but it doesn't properly support nested virtualization.
- Any version of a Linux server will work. Personally, I'll be using AlmaLinux 9.5 on hypervisors with NFS, and Ubuntu 22.04 on Virtual Machine Manager (VMM).
- A terminal capable of establishing an SSH connection (for example, PowerShell). I prefer to use [MobaXterm](https://mobaxterm.mobatek.net/).

## VM Preparation:
The detailed process of creating a VM on VMware will be skipped; you can check the details [here.](https://www.youtube.com/watch?v=sJNxJghTc28)

1. Create two VMs and name them Hypervisor01 and Hypervisor02.
    - Nested virtualisation should be enabled.
    - Swithc network adapter to the bridged. All VM's will be on our local network with inthernet acess. Don’t forget to check the MAC addresses and generate new ones if they are the same on the VMs. VM Settings -> Network Adapter -> Advanced -> Generate.
    ![изображение](https://github.com/user-attachments/assets/bb07d3f4-471d-425a-8ee2-a03952c8996e)
    - Ensure they have enough resources to support another VM that will be installed (nested - Virtualize Intel VT-x/EPT or AMD-V/RVI) later. In my case, it will be Alpine Linux because it is lightweight and has minimal system requirements. My settings are:

![изображение](https://github.com/user-attachments/assets/ad1dd042-bb38-43a2-90e6-4c3aaf9e09f1)

2. Create the NFS VM.
    - Clone one of the Hypervisor VMs and rename it 'NFS'. You can allocate less RAM and CPU resources. Also, disable nested virtualization.
3. Create a server where VMM will be installed.
    - Align the system resources with the Linux distribution you will be using. My settings are:

![изображение](https://github.com/user-attachments/assets/362c276a-2c18-473d-a5a6-ca269f10fe21)

 ## VM's naming and basic network set up. SSH connection.
 1. Run your VM's and log in.
 2. Let’s first check if our machine supports hardware assisted virtualization by executing:
 ```bash
 egrep -cwo 'vmx|svm' /proc/cpuinfo
 ```
 If the result is a number bigger than 0 we can safely continue. Otherwise, we must check in case of physical machine, if the processor supports virtualization and if it is enabled in BIOS, or in case of a virtual machine, if the virtualization instructions are passed   on (nested virtualization enabled).
 
 3. We need to collect their IP addresses. Open each VM window one by one, log in, and type `ip a`. Write down their IP addresses, and note which VM each one belongs to.
![изображение](https://github.com/user-attachments/assets/e516ffec-ce7e-4c0f-a930-406a6080e3b2)
    - Open the terminal in my case, it's MobaXterm. Then, create the folder (Right click -> new folder).
    - Inside the folder, establish the connection to the hosts. (Right click -> New connection).
    - Inside the connection window, locate the `Remote host` field. Type in `username@host_ip`. My settings are:
    ![изображение](https://github.com/user-attachments/assets/0f108ff1-5835-4c87-ba45-7abdd11dcaed)

    ![изображение](https://github.com/user-attachments/assets/cbf87e80-70e3-48df-8dd8-f30364d2738d)

 4. Rename your VM's:
We need to check if the user is a member of the wheel, adm or sudo group. The specific group name depends on your Linux distribution.
    - Check that you can with the cpmmand:
    ```bash
    sudo cat /etc/group | grep -i username
    ```
    If you are already a member, you can skip the next step. If not, proceed as follows:
    In our case, AlmaLinux is part of the Red Hat Linux family, where the group is called 'wheel'.
    - Add your user to the wheel group to enable executing `sudo` commands.
    - To become root, use the following command:
    ```bash
    su - root
    ```
    - Add user to the wheel group:
    ```bash
    usermod -aG wheel username
    ```
    - To apply the changes, you need to log out or reboot. Execute the following command directly on the VM:
     ```bash
    exit
    ```
    - Rename your VM by executing, also directly on the VM:
    ```bash
    sudo hostnamectl --set-hostname Hypervisor01
    ```
    - Execute the `exit` command again to apply the changes. Repeat this process on all the VMs. Aligned with their names and purposes.
    Then reestablish the ssh connection.
## Package for instalation on both Hypervisors 
1. Update system packages:
   ```bash
   sudo dnf update -y
   ```
2. Enable the extra packages repository by installing `epel-release`:
   ```bash
   sudo dnf install epel-release
   ```
3. Install the packages needed to run KVM, QEMU, and manage VMs:
   ```bash
   sudo dnf install -y qemu-kvm libvirt virt-install bridge-utils nfs-utils
   ``` 
4. Enable and start the libvirt daemon:
   ```bash
   sudo systemctl enable --now libvirtd
   ```
   - Check the status:
     ```bash
     systemctl status libvirtd
     ```
5. SELinux configuration.
   - By default, SELinux is set to enforcing mode. For security purposes, we will not disable it. Instead, we need to allow NFS services by executing the following command:
     ```bash
     sudo setsebool -P virt_use_nfs 1
     ```  
## NFS server instalation and configuration
1. Install NFS utilities on NFS server:
    ```bash
     sudo dnf install -y nfs-utils
    ```
    - Enable and start the NFS server service
    ```bash
     sudo systemctl enable nfs-server --now
    ```
2. Create a directory to export:

    ```bash
     sudo mkdir -p /var/nfs
    ```
    - You can also adjust ownership and permissions as needed. For instance:
    
    ```bash
    sudo chown -R nobody:nobody /var/nfs
    sudo chmod -R 0777 /var/nfs
    ```
3. Configure the export in `/etc/exports`
    
    ```bash
    sudo vi /etc/exports
    ```
    Example line to allow read-write access for every ip:
    ```bash
    /var/nfs *(rw,sync,no_root_squash)
    ```
    The asterisk (*) represents all IP addresses. However, you can specify a particular IP address or a range if needed.
    
    Export the directory:
    
    ```bash
     sudo exportfs -r
    ```
4. Open necessary firewall ports:
    ```bash
    sudo firewall-cmd --permanent --add-service=nfs
    sudo firewall-cmd --permanent --add-service=rpc-bind
    sudo firewall-cmd --permanent --add-service=mountd
    sudo firewall-cmd --reload
    ```
5. As we already discussed, SELinux is set to enforcing mode, you might also need to allow NFS to serve files from your chosen directory. Assign a proper SELinux context to the shared directory:
    ```bash
    sudo chcon -t public_content_rw_t /var/nfs
    ```
## Mount the NFS Share on Each Hypervisor

1. Create a mount point.
    If multiple hypervisors share the same mount point (and the same images), we will be able to perform live migration. That's why we will use the default folder for libvirt images:
    ```bash
    /var/lib/libvirt/images
    ```
    - Test connectivity by manually mounting the NFS share:
    ```bash
    sudo mount -t nfs NFS_server_IP:/var/nfs /var/lib/libvirt/images
    ```
    If multiple hypervisors will share the same mount point (and the same images) we will be able to perform live migration.
    - Verify it’s mounted correctly:
    ```bash
    df -h | grep nfs 
    ```
3. To have the NFS share mounted automatically whenever the client restarts, add an entry to `/etc/fstab`:
    ```bash
    NFS_server_IP:/var/nfs   /var/lib/libvirt/images    nfs     defaults,_netdev  0 0
    ```
4. By default, root can manage libvirt without any extra group membership. However, if you plan to manage virtual machines from a non-root account (for example, using virt-manager or virsh as a regular user), you generally need to add that user to the libvirt group. This grants the necessary permissions to communicate with the libvirtd service.
   
    ```bash
    sudo usermod -aG libvirt <username>
    ```
    - Afterward, log out and back in (or restart your session) for the group membership to take effect.
