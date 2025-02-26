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
    sudo usermod -aG libvirt username
    ```
    - Afterward, log out and back in (or restart your session) for the group membership to take effect.
## VMM with ubuntu 22.04 with GUI

1. Install required packages:
   ```bash
   sudo apt update
   sudo apt install -y virt-manager qemu-kvm bridge-utils openssh-client ssh-askpass
   ```
   - virt-manager: GUI for managing virtual machines locally and remotely via libvirt.
   - qemu-kvm: QEMU KVM tools (often required for certain drivers or CLI integration if you do local virtualization).
   - bridge-utils: If you want bridging on Ubuntu too.
   - openssh-client: So you can SSH into the hypervisors if needed. More often than not, this package is not installed by default in Ubuntu workstation distributions. If it's a server, it may prompt you during the installation process to install the openssh-client.
   - ssh-askpass: This utility will prompt a window asking you to enter your SSH password.
2. Work with Virtual Machine Manager
   - To add a host to the list of known hosts, we need to establish a basic SSH connection to our Hypervisor hosts:
       ```bash
       sudo ssh username@hypervisor_IP
       ```
   - Start Virt-Manager from the application menu (or type `virt-manager` in terminal)
        - Go to File > Add Connection…
     
        ![изображение](https://github.com/user-attachments/assets/f1c53bf8-5afc-4483-b090-c38a11461b69)
     
        - Enable Connect to remote host over SSH
        - Hypervisor: QEMU/KVM
        - Username: (your SSH user on the AlmaLinux hypervisor)
        - Hostname or IP: e.g. 192.168.1.44
     
        ![изображение](https://github.com/user-attachments/assets/1d8f54b6-4ca5-4749-b4ba-5f671fa6cf3f)

    Once connected, you should see both hypervisors listed in Virt-Manager. You can manage, create, and start VMs on them.

    ![изображение](https://github.com/user-attachments/assets/478a7b64-f88c-4db6-a748-e555febe65a4)

3. Preparation Steps for Creating the VM
   - We need to download the operating system in our case, Alpine Linux. First, go to the NFS server, install the necessary tool for downloading, and then download the image:
       ```bash
       sudo dnf install wget
       sudo wget https://dl-cdn.alpinelinux.org/alpine/v3.21/releases/x86_64/alpine-standard-3.21.3-x86_64.iso
       ```
   - Move downloaded image to the shared folder:
       ```bash
       sudo mv /path/to/iso /path/to/shared/folder
       ```
4. Creation of the VM:
    - Open the Virtual Machine Manager (VMM) on Ubuntu 22.04
        - Launch the VMM interface
        - Use the cursor to navigate and click on the previously established QEMU/KVM connections
        - Navigate to -> File -> New Virtual Machine
        ![изображение](https://github.com/user-attachments/assets/0498d644-241b-4d58-8653-993ae2b08e07)

        - Select option: `Local install media` and click -> Forward
        - Click -> browse
        - If you mounted the NFS shared folder in `/var/lib/libvirt/images`, then you will be able to see the image
          
        ![изображение](https://github.com/user-attachments/assets/b3fa66b5-eaa5-4c1a-859b-a665d67620cb)

        - If not, you will need to manually add a pool by clicking the green plus sign at the bottom left. In the popup window, specify the target path
          
        ![изображение](https://github.com/user-attachments/assets/0bb6ec04-9548-43ad-8e1f-73efbe80f73d)

        - After this, select your ISO image and click `Select Volume`. In the popup window, type your operating system in the bottom field. If it’s not listed, choose the closest operating system to it and then `Forward`

        ![изображение](https://github.com/user-attachments/assets/9ffd6810-5666-4852-b893-6e21bb006869)
      
        - Again, a popup window will appear displaying RAM and CPU requirements. You can adjust these settings as desired

        ![изображение](https://github.com/user-attachments/assets/165f0d84-ccf9-4d90-963f-1af90fb25e7a)

        ![изображение](https://github.com/user-attachments/assets/13029d04-9974-45d2-9a75-44620c46746b)

        - We're almost there! :) In the next popup window, select `Customize configuration before install`
          
        ![изображение](https://github.com/user-attachments/assets/91638936-102b-4fa5-9f50-659d7710a13e)

        - We will keep the default network and other settings except for a couple. Extremle important to know how work SSH Tunnel to Localhost.
Virt-manager establishes a secure SSH tunnel and forwards VNC (or SPICE) traffic to localhost on the hypervisor. If VNC is set to listen on 0.0.0.0 (all interfaces), then you must open port 5900 in the hypervisor firewall and expose it externally, which is typically unnecessary and less secure.

        ![изображение](https://github.com/user-attachments/assets/edef8228-8666-4ecd-a7d5-85e29476934d)

        - Select `Boot Options`, enable `CDROM`, and using the arrows, move it to the top. Then press `Apply`
        - 
        ![изображение](https://github.com/user-attachments/assets/207c11c5-ec51-44d7-8917-ecf740cf7e29)

        - Select `CDROM`, click `Browse`, and choose your ISO image, as the guest OS will boot from it.
        - 
        ![изображение](https://github.com/user-attachments/assets/9c52ff07-ff7d-493f-a993-442577ae6f39)


        

 Then, apply the changes.
        - Click `Begin Installation` on the top left corner





