# <strong> LXD virtualization of lab GPU servers </strong>
The lab added a GPU server for deep learning, because the number of laboratories is relatively large, but the software used by each person varies widely. If multiple people use the same software, the software, environment, files, and configuration are mixed. There is even Xiaobai running commands that harm the system
! [image] (image / 小白 .gif)
So we did virtualization. Why use LXD instead of the hottest docker?
Both are based on lxc virtualization, and docker is an application container, and LXD is a system container (is there a great way to install a complete desktop?) It is closer to our production environment. Imagine that when others use docker, they also need to use it by themselves. Command to upload a file and run the program. Especially Xiao Bai would have a headache against the black frame. And open the remote yourself, open pycharm, beautiful. Remove the idea of ​​linux not using the desktop. In 9102, the ubuntu desktop has been very stable. Let's install and use it

> ## Step 1: Installation and configuration of the host
>> ### Server system installation
>>> It is recommended to install server version, remotely via ssh [ubuntu image] (http://cdimage.ubuntu.com/releases/18.04.3/release/ "image")
>>> The server generally has an SSD and multiple mechanical RAID arrays. The system is installed in the SSD (smaller) and there is also a RAID array of data disks.
>> ### Server graphics driver installation
>>> (If not accessible, the pdf folder is already offline)
>>> [Graphics driver installation] nvidia-gpu-312a693744b5 "Linux graphics driver installation"),
>>> Install NVIDIA graphics driver, CUDA, cuDNN

> ## Step 2: Installation and initialization of lxd
>> ### install lxd
>>> LXD implements virtual container
>>> ZFS is used to manage physical disks and supports LXD advanced functions
>>> bridge-utils is used to build a network bridge
>>> #### Install LXD, ZFS, and bridge-utils
>>>> `sudo apt-get install lxd zfsutils-linux bridge-utils`
>> ### Configure the bridge:
>>> Because of the network problem of the school information center, if the bridge network card is configured, it will cause abnormal traffic and directly disconnect the network. Therefore, the method of implementing one IP per person fails. We must use port forwarding to implement the network of each container
>> ### Configure ZFS
>>> First, we run sudo fdisk -l to list the available disks and partitions on the server. We have two hard disks, the first is the system disk and the second is the data disk. sdb) to divide the space to be used as the storage volume of the container.
>>> ### View partition
>>>> `sudo fdisk / dev / sdb`
>>>
>>>! [my-logo.png] (image / picture 1.png "my-logo")
>>> According to the figure below, the 80GB partition is divided as the container storage volume, and the partition is / dev / sdb1. The remaining space can be partitioned in the same way, and can be used as another application of the server.
>>>! [my-logo.png] (image / picture 2.png "my-logo")
>> ### Create block device
>>> #### Create a ZFS storage pool on block device / dev / sdb1
>>>> `sudo lxc storage create zfs-pool zfs source = / dev / sdb1`
>> ### LXD initialization
>>>> `sudo lxd init`
>>>
>>>! [my-logo.png] (image / picture 3.png "my-logo")
>>> Because we have created a storage pool called zfs-pool, there is no need to create a new storage pool during lxd initialization, and then you can configure it
>>> ### Configure again
>>>> `sudo lxc profile edit default`
>>> ### Modify the default disk size in the container
>>> Also limit the hard disk size of each container to a fixed size during configuration
>>> (if not set, the disk size in the container is the size of the entire storage pool)
>>>! [my-logo.png] (image / picture 4.png "my-logo")

> # Step 3: Create the container
>> ## acceleration source
>>> ### Use Tsinghua's mirror source (accelerated creation)
>>>> `sudo lxc remote add tuna-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol = simplestreams --public`
>>> ### list available mirrors
>>>> `sudo lxc image list tuna-images:`
>> ## Create ubuntu container
>>> ### Create a container called test using the ubuntu image from Tsinghua source
>>>> `sudo lxc launch tuna-images: ubuntu / 18.04 test`
>> ## Enter the container
>>> `sudo lxc exec test bash`
>>>
>>> We are logged in as the root user, and a user named ubuntu already exists in this container
>> ## Change password
>>> `passwd root`
>>> `passwd ubuntu`
>>>
>>> The ubuntu in the container is a very streamlined system that requires the installation of various software
>> ## install ssh
>>> `apt install ssh`
>> ## Connect container via ssh
>>> ### View the container and host IP
>>>> Because we do n’t set up a bridge network card, we ca n’t access the container from an external computer (the container ’s ip cannot be pinged), so we use port forwarding to access our container, so
>>>> #### Exit container
>>>>> `exit`
>>>> #### View container on host
>>>>> `sudo lxc list`
>>>> #### The IP address of the container is 10.152.210.183
>>>>! [my-logo.png] (image / picture 5.png "my-logo")
>>>> #### View host IP address
>>>>> `ip addr`
>>>> #### It can be seen that the host IP is 172.22.24.126
>>> ## port forwarding
>>>> `sudo iptables -t nat -A PREROUTING -d 172.22.24.126 -p tcp --dport 60601 -j DNAT --to-destination 10.152.210.183: 22`
>>>
>>>> 60610 is our set port number, which is mapped to port 22 in the container through the host's 60601 port number (SSH default port number)
 
> # Step 4: Initial container configuration
>> ## Use ssh to connect to the container and configure
>>> `ssh ubuntu@172.22.24.126 -p 60601`
>> ## 1. Change source
>>> ### backup original source
>>>> `sudo mv /etc/apt/sources.list / etc / apt / sources.list.bak`
>>> ### Write to Netease Source (China Source)
>>> ### (note the system version ubuntu 18.04 bionic)
>>>> `sudo vim / etc / apt / sources.list`  
>>> `` `  
>>> deb http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse  
>>> deb-src http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse  
>>> deb http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse  
>>> deb-src http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse  
>>> deb http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse  
>>> deb-src http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse  
>>> deb http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse  
>>> deb-src http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse  
>>> deb http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse  
>>> deb-src http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse  
>>> `` `  
>> ## 2. Install graphical interface
>>> ### refresh source
>>>> `sudo apt update`
>>> ### Install ubuntu desktop without recommended software (gnome is installed by default, there will be many unrelated software for full installation)
>>>> `sudo apt install --no-install-recommends ubuntu-desktop`
>>> ### or minimize install gnome desktop
>>>> `sudo apt install gnome-shell gnome-session gnome-panel gnome-terminal -y`
>> ## 3. Install remote connection
>>> ### Use the installation script (after installing git and download what we need afterwards)
>>>> `sudo apt install git`
>>>> `git clone https: // github.com / shenuiuin / LXD_GPU_SERVER`
>>> ### Open folder
>>>> `cd LXD_GPU_SERVER /`
>>> ### Give script executable permissions
>>>> `sudo chmod a + x install-xrdp-2.3.sh`
>>> ### The script will download some files, you need to have the Downloads folder
>>>> `mkdir -p ~ / Downloads`
>>> ### Installation script
>>>> `./install-xrdp-2.3.sh -s yes -g yes`
>>> ### Installation is complete
>>>! [my-logo.png] (image / picture 6.png "my-logo")
>>> ### If there are other desktop needs
>>> [kde desktop environment and xrdp installation] (https://www.hiroom2.com/2018/05/07/ubuntu-1804-xrdp-kde-en/ "kde")
>>> [xrdp resolves sound redirection] (http://c-nergy.be/blog/?p=12469 "redirect Sound")
>> ## 4. Remote connection test
>>> ### Port Forwarding
>>> After installing XRDP, as before, because we can't ping the container, we need to forward the port of xrdp to the host
>>>> `sudo iptables -t nat -A PREROUTING -d 172.22.24.126 -p tcp --dport 60611 -j DNAT --to-destination 10.152.210.183: 3389`
>>> ### Remote connection
>>>> 60611 is the port number we set, which is mapped to the 3389 port number in the container through the host's 60611 port number (XRDP default port number)
>>> Can use container through windows remote connection (windows running mstsc)
>>>>! [my-logo.png] (image / ubuntu.png "my-logo")
>>>> The next step is to use it as an ordinary ubuntu. For example, you can find some tutorials: things to do after installing ubuntu, etc.
>> ## 5. Add graphics to the container
>>> we go back to the host
>>> ### Add all GPUs to the container:
>>>> `lxc config device add yourContainerName gpu gpu`
>>> ### Add the specified GPU:
>>>> `lxc config device add yourContainerName gpu0 gpu id = 0`
>>> ## Install driver
>>> After adding the graphics card, it is equivalent to installing the graphics card in the container. We go back to the container and install the graphics driver.
>>> The graphics card version must be the same as that of the host computer. For the installation method, please refer to the first step of NVIDIA graphics card driver, CUDN, cuDNN installation
>>> Please note that the following parameters need to be added when installing the graphics driver in the container, and it is not necessary to install it into the kernel during installation
>>>> `sudo sh ./NVIDIA-Linux-X86_64-[YOURVERSION].run --no-kernel-module`  
> # The fifth step: landscaping and other configuration of ubuntu
>> ## icon icon theme
>>> `sudo apt update`
>>> `sudo apt install papirus-icon-theme`
>> ## GTK Theme
>>> `git clone https: // github.com / vinceliuice / vimix-gtk-themes`
>>> `cd vimix-gtk-themes`
>>> `sudo. / vimix-installer`
>> ## Application theme
>>> After the theme is installed, use gnome-tweak-toos to apply the theme
>>>! [my-logo.png] (image / picture 7.png "my-logo")
>>> can also personalize your ubuntu, such as adding a maximize minimize button
>>>! [my-logo.png] (image / picture 8.png "my-logo")
>> ## gnome extension
>>> Recommended extensions
>>>! [my-logo.png] (image / picture 9.png "my-logo")
>> ## Switch Chinese
>>> System Chinese is in Language Support. Then add the simplified Chinese language, drag Chinese to the first item, and then apply it to the entire system
>>>! [my-logo.png] (image / picture 10.png "my-logo")
>> ## keep the old name
>>> After restart, you will be prompted to change the folder name to Chinese (preferably use the old name, English path)
>>>! [my-logo.png] (image / picture 11.png "my-logo")
>> ## Install required software
>>> ### Sogou Input Method, Google Chrome, etc.
>>> ### Display Linux system information
>>>> `sudo apt install neofetch`
>>>> `neofetch`
>>> ### View CPU operation and memory usage
>>>> `sudo apt install htop`
>>>> `htop`
>>> ### View graphics card operation
>>>> `nvidia-smi`
>>> ### View the graphics card's operation in real time (refresh in real time at a frequency of 0.1s)
>>>> `watch -n0.1 nvidia-smi`

>>> ### View the container and host IP
>>>> Because we do n’t set up a bridge network card, we ca n’t access the container from an external computer (the container ’s ip cannot be pinged), so we use port forwarding to access our container, so
>>>> #### Exit container
>>>>> `exit`
>>>> #### View container on host
>>>>> `sudo lxc list`


> # Step 6: Container Management
>> ## Port forwarding table management
>>> Because we use port forwarding to connect to the container, but the routing table rules will be lost when the host restarts
>>> ### List port rules
>>>> `sudo iptables -t nat -L PREROUTING --line-number`
>>> ### delete the rule of the first line
>>>> `sudo iptables -t nat -D PREROUTING 1`
>>> ### Save rules (to prevent loss of forwarding table after restart)
>>>> `sudo netfilter-persistent save`
>>> ### Restore saved forwarding rules
>>>> `sudo netfilter-persistent reload`
>> ## Modify parameter configuration for the container
>>> We don't want everyone to use all the hardware resources, so we also need to limit everyone's parameters
>> [Container parameter configuration instructions] (https://linuxcontainers.org/lxd/docs/master/containers "Container configuration")
>>> ### Configure container parameters
>>>> `lxc config edit YourContainerName`
>>> ### Generally use the following configuration to meet
>>>! [my-logo.png] (image / picture 14.png "my-logo")
>>> Actually, when editing the default disk size at the end of the second step of the tutorial (default)
>>> ### Configure the default container parameters (the parameters of the new container will inherit the parameters of the default configuration, and the container will preferentially use its own parameters)
>>>> `sudo lxc profile edit default`
>> ## Administrator notice
>>> The administrator should create a new read.txt on the desktop, write down the system version and other information, what software is installed, various precautions, etc.

> # Step 7: Container template
>> We use this configured container as a template and save it as an image.
>>> ## Stop container
>>>> `sudo lxc stop test`
>>> ## save test container as ubuntudemo image
>>>> `sudo lxc publish test --alias ubuntudemo --public`
>> ## New container from template image
>> In the future, use the template image to create the container directly. After the container is created, add port mapping for it (remote connection and SSH)
>> Also add a graphics card for it (the graphics driver is already available) and configure its hardware parameters (the default configuration file can be used so that the parameters of the new container inherit it, this step can be omitted)
>>! [my-logo.png] (image / picture 15.png "my-logo")  
### [Result display (dual screen ~ sound)] (https://www.bilibili.com/video/av61400281 "Beep Beep Mile")  
! [my-logo.png] (image / picture 16.png "my-logo")  
! [my-logo.png] (image / picture 17.png "my-logo")  
! [my-logo.png] (image / picture 18.png "my-logo")  


>> ## Use docker in lxd container
>>>> `lxc config edit YourContainerName`
>>> ### add in config
>>>! [my-logo.png] (image / picture 19.png "my-logo")
>>> ### and restart the container
>>>> `lxc restart YourContainerName`
>>> ## [install docker] (https://docs.docker.com/install/linux/docker-ce/ubuntu/ "docker")
> ## Step 2: Installation and initialization of lxd
>> ### install lxd
>>> LXD implements virtual container
>>> ZFS is used to manage physical disks and supports LXD advanced functions
>>> bridge-utils is used to build a network bridge
>>> #### Install LXD, ZFS, and bridge-utils
>>>> `sudo apt-get install lxd zfsutils-linux bridge-utils`
>> ### Configure the bridge:
>>> Because of the network problem of the school information center, if the bridge network card is configured, it will cause abnormal traffic and directly disconnect the network. Therefore, the method of implementing one IP per person fails. We must use port forwarding to implement the network of each container
>> ### Configure ZFS
>>> First, we run sudo fdisk -l to list the available disks and partitions on the server. We have two hard disks, the first is the system disk and the second is the data disk. sdb) to divide the space to be used as the storage volume of the container.
>>> ### View partition
>>>> `sudo fdisk / dev / sdb`
>>>
>>>! [my-logo.png] (image / picture 1.png "my-logo")
>>> According to the figure below, the 80GB partition is divided as the container storage volume, and the partition is / dev / sdb1. The remaining space can be partitioned in the same way, and can be used as another application of the server.
>>>! [my-logo.png] (image / picture 2.png "my-logo")
>> ### Create block device
>>> #### Create a ZFS storage pool on block device / dev / sdb1
>>>> `sudo lxc storage create zfs-pool zfs source = / dev / sdb1`
>> ### LXD initialization
>>>> `sudo lxd init`
>>>
>>>! [my-logo.png] (image / picture 3.png "my-logo")
>>> Because we have created a storage pool called zfs-pool, there is no need to create a new storage pool during lxd initialization, and then you can configure it
>>> ### Configure again
>>>> `sudo lxc profile edit default`
>>> ### Modify the default disk size in the container
>>> Also limit the hard disk size of each container to a fixed size during configuration
>>> (if not set, the disk size in the container is the size of the entire storage pool)
>>>! [my-logo.png] (image / picture 4.png "my-logo")

> # Step 3: Create the container
>> ## acceleration source
>>> ### Use Tsinghua's mirror source (accelerated creation)
>>>> `sudo lxc remote add tuna-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol = simplestreams --public`
>>> ### list available mirrors
>>>> `sudo lxc image list tuna-images:`
>> ## Create ubuntu container
>>> ### Create a container called test using the ubuntu image from Tsinghua source
>>>> `sudo lxc launch tuna-images: ubuntu / 18.04 test`
>> ## Enter the container
>>> `sudo lxc exec test bash`
>>>
>>> We are logged in as the root user, and a user named ubuntu already exists in this container
>> ## Change password
>>> `passwd root`
>>> `passwd ubuntu`
>>>
>>> The ubuntu in the container is a very streamlined system that requires the installation of various software
>> ## install ssh
>>> `apt install ssh`
>> ## Connect container via ssh  
# <strong> LXD virtualization of lab GPU servers <strong>
The lab added a GPU server for deep learning, because the number of laboratories is relatively large, but the software used by each person varies widely. If multiple people use the same software, the software, environment, files, and configuration are mixed. There is even Xiaobai running commands that harm the system
! [image] (image / 小白 .gif)
So we did virtualization. Why use LXD instead of the hottest docker?
Both are based on lxc virtualization, and docker is an application container, and LXD is a system container (is there a great way to install a complete desktop?) It is closer to our production environment. Imagine that when others use docker, they also need to use it by themselves. Command to upload a file and run the program. Especially Xiao Bai would have a headache against the black frame. And open the remote yourself, open pycharm, beautiful. Remove the idea of ​​linux not using the desktop. In 9102, the ubuntu desktop has been very stable. Let's install and use it

> ## Step 1: Installation and configuration of the host
>> ### Server system installation
>>> It is recommended to install server version, remotely via ssh [ubuntu image] (http://cdimage.ubuntu.com/releases/18.04.3/release/ "image")
>>> The server generally has an SSD and multiple mechanical RAID arrays. The system is installed in the SSD (smaller) and there is also a RAID array of data disks.
>> ### Server graphics driver installation
>>> (If not accessible, the pdf folder is already offline)
>>> [Graphics driver installation] nvidia-gpu-312a693744b5 "Linux graphics driver installation"),
>>> Install NVIDIA graphics driver, CUDA, cuDNN  
> ## Step 2: Installation and initialization of lxd
>> ### install lxd
>>> LXD implements virtual container
>>> ZFS is used to manage physical disks and supports LXD advanced functions
>>> bridge-utils is used to build a network bridge
>>> #### Install LXD, ZFS, and bridge-utils
>>>> `sudo apt-get install lxd zfsutils-linux bridge-utils`
>> ### Configure the bridge:
>>> Because of the network problem of the school information center, if the bridge network card is configured, it will cause abnormal traffic and directly disconnect the network. Therefore, the method of implementing one IP per person fails. We must use port forwarding to implement the network of each container
>> ### Configure ZFS
>>> First, we run sudo fdisk -l to list the available disks and partitions on the server. We have two hard disks, the first is the system disk and the second is the data disk. sdb) to divide the space to be used as the storage volume of the container.
>>> ### View partition
>>>> `sudo fdisk / dev / sdb`
>>>
>>>! [my-logo.png] (image / picture 1.png "my-logo")
>>> According to the figure below, the 80GB partition is divided as the container storage volume, and the partition is / dev / sdb1. The remaining space can be partitioned in the same way, and can be used as another application of the server.
>>>! [my-logo.png] (image / picture 2.png "my-logo")
>> ### Create block device
>>> #### Create a ZFS storage pool on block device / dev / sdb1
>>>> `sudo lxc storage create zfs-pool zfs source = / dev / sdb1`
>> ### LXD initialization
>>>> `sudo lxd init`
>>>
>>>! [my-logo.png] (image / picture 3.png "my-logo")
>>> Because we have created a storage pool called zfs-pool, there is no need to create a new storage pool during lxd initialization, and then you can configure it
>>> ### Configure again
>>>> `sudo lxc profile edit default`
>>> ### Modify the default disk size in the container
>>> Also limit the hard disk size of each container to a fixed size during configuration
>>> (if not set, the disk size in the container is the size of the entire storage pool)
>>>! [my-logo.png] (image / picture 4.png "my-logo")
> # Step 3: Create the container
>> ## acceleration source
>>> ### Use Tsinghua's mirror source (accelerated creation)
>>>> `sudo lxc remote add tuna-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol = simplestreams --public`
>>> ### list available mirrors
>>>> `sudo lxc image list tuna-images:`
>> ## Create ubuntu container
>>> ### Create a container called test using the ubuntu image from Tsinghua source
>>>> `sudo lxc launch tuna-images: ubuntu / 18.04 test`
>> ## Enter the container
>>> `sudo lxc exec test bash`
>>>
>>> We are logged in as the root user, and a user named ubuntu already exists in this container
>> ## Change password
>>> `passwd root`
>>> `passwd ubuntu`
>>>
>>> The ubuntu in the container is a very streamlined system that requires the installation of various software
>> ## install ssh
>>> `apt install ssh`
>> ## Connect container via ssh
>>> ### View the container and host IP
>>>> Because we do n’t set up a bridge network card, we ca n’t access the container from an external computer (the container ’s ip cannot be pinged), so we use port forwarding to access our container, so
>>>> #### Exit container
>>>>> `exit`
>>>> #### View container on host
>>>>> `sudo lxc list`
>>>> #### The IP address of the container is 10.152.210.183
>>>>! [my-logo.png] (image / picture 5.png "my-logo")
>>>> #### View host IP address
>>>>> `ip addr`
>>>> #### It can be seen that the host IP is 172.22.24.126
>>> ## port forwarding
>>>> `sudo iptables -t nat -A PREROUTING -d 172.22.24.126 -p tcp --dport 60601 -j DNAT --to-destination 10.152.210.183: 22`
>>>
>>>> 60610 is our set port number, which is mapped to port 22 in the container through the host's 60601 port number (SSH default port number)
 
> # Step 4: Initial container configuration
>> ## Use ssh to connect to the container and configure
>>> `ssh ubuntu@172.22.24.126 -p 60601`
>> ## 1. Change source
>>> ### backup original source
>>>> `sudo mv /etc/apt/sources.list / etc / apt / sources.list.bak`
>>> ### Write to Netease Source (China Source)
>>> ### (note the system version ubuntu 18.04 bionic)
>>>> `sudo vim / etc / apt / sources.list`
>>> `` `
>>> deb http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse  
>>> deb-src http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse  
>>> deb http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse  
>>> deb-src http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse  
>>> deb http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse  
>>> deb-src http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse  
>>> deb http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse  
>>> deb-src http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse  
>>> deb http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse  
>>> deb-src http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse  
>>> `` `
>> ## 2. Install graphical interface
>>> ### refresh source
>>>> `sudo apt update`
>>> ### Install ubuntu desktop without recommended software (gnome is installed by default, there will be many unrelated software for full installation)  
>>>> `sudo apt install --no-install-recommends ubuntu-desktop`
>>> ### or minimize install gnome desktop
>>>> `sudo apt install gnome-shell gnome-session gnome-panel gnome-terminal -y`
>> ## 3. Install remote connection
>>> ### Use the installation script (after installing git and download what we need afterwards)
>>>> `sudo apt install git`
>>>> `git clone https: // github.com / shenuiuin / LXD_GPU_SERVER`
>>> ### Open folder
>>>> `cd LXD_GPU_SERVER /`
>>> ### Give script executable permissions
>>>> `sudo chmod a + x install-xrdp-2.3.sh`
>>> ### The script will download some files, you need to have the Downloads folder
>>>> `mkdir -p ~ / Downloads`
>>> ### Installation script
>>>> `./install-xrdp-2.3.sh -s yes -g yes`
>>> ### Installation is complete
>>>! [my-logo.png] (image / picture 6.png "my-logo")
>>> ### If there are other desktop needs
>>> [kde desktop environment and xrdp installation] (https://www.hiroom2.com/2018/05/07/ubuntu-1804-xrdp-kde-en/ "kde")
>>> [xrdp resolves sound redirection] (http://c-nergy.be/blog/?p=12469 "redirect Sound")
>> ## 4. Remote connection test
>>> ### Port Forwarding
>>> After installing XRDP, as before, because we can't ping the container, we need to forward the port of xrdp to the host
>>>> `sudo iptables -t nat -A PREROUTING -d 172.22.24.126 -p tcp --dport 60611 -j DNAT --to-destination 10.152.210.183: 3389`
>>> ### Remote connection
>>>> 60611 is the port number we set, which is mapped to the 3389 port number in the container through the host's 60611 port number (XRDP default port number)
>>> Can use container through windows remote connection (windows running mstsc)
>>>>! [my-logo.png] (image / ubuntu.png "my-logo")
>>>> The next step is to use it as an ordinary ubuntu. For example, you can find some tutorials: things to do after installing ubuntu, etc.
>> ## 5. Add graphics to the container
>>> we go back to the host
>>> ### Add all GPUs to the container:
>>>> `lxc config device add yourContainerName gpu gpu`
>>> ### Add the specified GPU:
>>>> `lxc config device add yourContainerName gpu0 gpu id = 0`
>>> ## Install driver
>>> After adding the graphics card, it is equivalent to installing the graphics card in the container. We go back to the container and install the graphics driver.
>>> The graphics card version must be the same as that of the host computer. For the installation method, please refer to the first step of NVIDIA graphics card driver, CUDN, cuDNN installation
>>> Please note that the following parameters need to be added when installing the graphics driver in the container, and it is not necessary to install it into the kernel during installation
>>>> `sudo sh ./NVIDIA-Linux-X86_64-[YOURVERSION].run --no-kernel-module`

> # The fifth step: landscaping and other configuration of ubuntu
>> ## icon icon theme
>>> `sudo apt update`
>>> `sudo apt install papirus-icon-theme`
>> ## GTK Theme
>>> `git clone https: // github.com / vinceliuice / vimix-gtk-themes`
>>> `cd vimix-gtk-themes`
>>> `sudo. / vimix-installer`
>> ## Application theme
>>> After the theme is installed, use gnome-tweak-toos to apply the theme
>>>! [my-logo.png] (image / picture 7.png "my-logo")
>>> can also personalize your ubuntu, such as adding a maximize minimize button
>>>! [my-logo.png] (image / picture 8.png "my-logo")  
>> ## gnome extension
>>> Recommended extensions
>>>! [my-logo.png] (image / picture 9.png "my-logo")
>> ## Switch Chinese
>>> System Chinese is in Language Support. Then add the simplified Chinese language, drag Chinese to the first item, and then apply it to the entire system
>>>! [my-logo.png] (image / picture 10.png "my-logo")
>> ## keep the old name
>>> After restart, you will be prompted to change the folder name to Chinese (preferably use the old name, English path)
>>>! [my-logo.png] (image / picture 11.png "my-logo")
>> ## Install required software
>>> ### Sogou Input Method, Google Chrome, etc.
>>> ### Display Linux system information
>>>> `sudo apt install neofetch`
>>>> `neofetch`
>>> ### View CPU operation and memory usage
>>>> `sudo apt install htop`
>>>> `htop`
>>> ### View graphics card operation
>>>> `nvidia-smi`
>>> ### View the graphics card's operation in real time (refresh in real time at a frequency of 0.1s)
>>>> `watch -n0.1 nvidia-smi`

> # Step 6: Container Management
>> ## Port forwarding table management
>>> Because we use port forwarding to connect to the container, but the routing table rules will be lost when the host restarts
>>> ### List port rules
>>>> `sudo iptables -t nat -L PREROUTING --line-number`
>>> ### delete the rule of the first line
>>>> `sudo iptables -t nat -D PREROUTING 1`
>>> ### Save rules (to prevent loss of forwarding table after restart)
>>>> `sudo netfilter-persistent save`
>>> ### Restore saved forwarding rules
>>>> `sudo netfilter-persistent reload`
>> ## Modify parameter configuration for the container
>>> We don't want everyone to use all the hardware resources, so we also need to limit everyone's parameters
>> [Container parameter configuration instructions] (https://linuxcontainers.org/lxd/docs/master/containers "Container configuration")
>>> ### Configure container parameters
>>>> `lxc config edit YourContainerName`
>>> ### Generally use the following configuration to meet
>>>! [my-logo.png] (image / picture 14.png "my-logo")
>>> Actually, when editing the default disk size at the end of the second step of the tutorial (default)
>>> ### Configure the default container parameters (the parameters of the new container will inherit the parameters of the default configuration, and the container will preferentially use its own parameters)
>>>> `sudo lxc profile edit default`
>> ## Administrator notice
>>> The administrator should create a new read.txt on the desktop, write down the system version and other information, what software is installed, various precautions, etc.

> # Step 7: Container template
>> We use this configured container as a template and save it as an image.
>>> ## Stop container
>>>> `sudo lxc stop test`
>>> ## save test container as ubuntudemo image
>>>> `sudo lxc publish test --alias ubuntudemo --public`
>> ## New container from template image  
>> In the future, use the template image to create the container directly. After the container is created, add port mapping for it (remote connection and SSH)
>> Also add a graphics card for it (the graphics driver is already available) and configure its hardware parameters (the default configuration file can be used so that the parameters of the new container are inherited from it, this step can be omitted)
>>! [my-logo.png] (image / picture 15.png "my-logo")
# [Result display (dual screen ~ sound)] (https://www.bilibili.com/video/av61400281 "Beep Beep Mile")
! [my-logo.png] (image / picture 16.png "my-logo")
! [my-logo.png] (image / picture 17.png "my-logo")
! [my-logo.png] (image / picture 18.png "my-logo")


>> ## Use docker in lxd container
>>>> `lxc config edit YourContainerName`
>>> ### add in config
>>>! [my-logo.png] (image / picture 19.png "my-logo")
>>> ### and restart the container
>>>> `lxc restart YourContainerName`
>>> ## [install docker] (https://docs.docker.com/install/linux/docker-ce/ubuntu/ "docker")
