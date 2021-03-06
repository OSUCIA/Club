---
title: Virtual HPC
---

# Before Arriving  

#### (Or just do it now)  

[Download VirtualBox as below](https://www.virtualbox.org/wiki/Downloads)  
[Download this zip file](https://drive.google.com/open?id=0B3Imkw3eMToZOWszSmJWbXU0QnM)  
The file contains some premade virtual machines so that we don't have to go through the installation process - it takes up too much time.  
Unzip the zip file.  

### Prerequisites

#### For the initial 2 VMs in this exercise 

  * An Intel-compatible processor with 2 or more cores
  * At least 4 GB RAM
  * At least 20 GB free disk space

#### For additional VMs you should expect to need
  
  * 1 more CPU core
  * Slightly over 1GB ram
  * Slightly over 9GB disk space

#### Download Lubuntu (16.04, 64-bit version)  

http://tiger.hpc.okstate.edu/lubuntu-16.04.1-alternate-amd64.iso  
Or http://cdimages.ubuntu.com/lubuntu/releases/xenial/release/lubuntu-16.04-alternate-amd64.iso  
md5sum:
  * c773824ab270455471a9086ee29242eb *lubuntu-16.04.1-alternate-amd64.iso  

#### Download & Install VirtualBox: https://www.virtualbox.org/wiki/Downloads

# Step One: Setup VM for master node

Create VM / Configure virtual hardware  
New VM  
```text
Name: node0
Type: Linux
Version: Ubuntu 64-bit
Memory size: 1024 MB
Hard drive: Create a virtual hard drive now
Hard drive file type: VDI (the default option)
Storage on physical hard drive: Dynamically allocated
File location and size
    Name: node0 (default location / name)
    Size: 8.00 GB (default)
Create
```
Open settings for the VM
```text
Network
	Adapter 1
		Enabled
		Attached to: NAT
	Adapter 2
		Enabled
		Attached to: Internal Network
		Name: cluster
```
Install Linux  
Start the VM  
Should be prompted for startup disk, if not click Devices menu → CD/DVD Devices → Choose a virtual CD/DVD disk file, choose the lubuntu ISO file. If you have to click the menu, then you’ll have to reset the power to the VM (Machine menu → Reset).


Linux Installation  
```text
Choose English
Choose Install Lubuntu
Installation language: English (again, default)
Country: United States (default)
Detect keyboard layout: No
Keyboard country: English/US (default)
Keyboard layout: English/US
(Stuff will now happen and you’ll have to wait a few seconds.)
Primary network interface: enp0s3 (you may get something different; this was the default)
(system attempts to get IP address and do automatic network config)
Hostname: node0
Full name for new user: pi
Username: pi
Password: raspberry
Encrypt home directory: No (default)
(system attempts to auto-detect time zone; want it to be “America/Chicago”)
Choose whether detected time zone is correct
Partitioning method: Guided - use entire disk (default)
Select disk to partition: SCSI3 (default)
(system will create partition configuration)
Write changes to disks?: Yes
(Linux will now begin installing to the VM’s hard drive.)
HTTP proxy: (leave blank, default)
(Installation will continue)
Install GRUB to the master boot record: Yes (default)
System clock set to UTC?: Yes (default)
(Installation complete)
```  
Configure installed system
Log in as pi
Open terminal (menu → System Tools → LXTerminal)
		
## Install updates and needed packages  

First, you have to update Lubuntu's lists of the most recent versions of the programs, "packages", that the system has.  
Second, you have Lubuntu use this list to automatically download the newest versions of all of the installed packages and update them.  
In order to run these, we have to get administrative privileges. We do this by starting the command with `sudo` - this tells the terminal to run the command as "root", which is Linux's version of Administrator.  
The command itself uses `apt-get` which is the program that we'll use to update all of the files on this system.
```bash
sudo apt-get update
sudo apt-get upgrade
```
Now we'll use the `apt-get` command to actually get and application - or in this case 9 of them at once! Each name you see after `install` is a different program that we're asking to be installed.
```bash
sudo apt-get install nano vim curl openssh-server nfs-kernel-server build-essential git mpich mpich-doc
```

## Setup network interface for internal cluster network
Above we installed two different network adapters, one that connects to the internet through your real computer, and one that is just connected to the internal "VM only" network.  
Now we need to configure the "VM only" adapter to talk to all of the other machines.  
 1. First figure out the device name for the network adapter using the NetworkManager Command Line Interface `nmcli`.
 
    ```bash
    nmcli device
    ```
    The device you want will probably be in red; I got “enp0s8”  
 2. Now that we know the device to configure, we need to actually configure it.  
    We pass `nmcli` the `c` flag, to indicate we're wanting to modify a 'c'onnection.  
    Then we tell it we want to `add` a connection named (`con-name`) `cluster`  
    We then give it the `ifname` (**i**nter**f**ace **name**), which is the name fo the devide we found above.  
    Next we tell it the `type` of connection is `ethernet`, and give it an `ipv4` address of `192.168.13.13/24`  
    Finally, we pass it `gw4` to tell it the **g**ate**w**ay for ipv**4** is its own ipv4 address that we just told it.
    
    ```bash
    sudo nmcli c add con-name cluster ifname enp0s8 type ethernet ip4 192.168.13.13/24 gw4 192.168.13.13
    ````
 3. Give this network interface a lower priority so the system doesn’t try to use it for internet traffic
    Again we're dealing with a `c`onnection, but this time we're `modify`-ing it.
    We have to tell it the `id` of the cluster, which is the con-name we gave it above: `cluster`.
    Finally we tell it that we want to modify the `ipv4.route-metric` by setting it to 101. This tells Lubuntu to prioritize this network less that the direct connection to the internet.
    
    ```bash
    sudo nmcli c modify id cluster ipv4.route-metric 101
    ```

### Setup nfs share
 1. Use `mkdir` to **m**a**k**e the **dir**ectory that we'll share. We're naming it `shared`, and placing it in the root directory.  
    
    ```bash
    sudo mkdir /shared
    ```
 2. Now we use `chown` to **ch**ange the **own**ership of the directory we made. This just makes sure it is owned by our user (`pi`) on our machine (`pi`).  
    
    ```bash
    sudo chown pi:pi /shared
    ```
 3. Lets make a test file on this virtual machine, then we can see if it appears on the others.
    We'll tell the shell to `echo` our sentence into (`>`) a new file in our `shared` folder, called `test`.
    
    ```bash
    echo “My icebox is full of lagomorphs!” > /shared/test.txt
    ```
 4. Finally lets configure the settings of how to share the files in our `shared` folder.  
    To do this we need to add line to the file `/etc/exports`: `/shared 192.168.13.13/24(rw,no_root_squash,async,no_subtree_check)`  
    You can do this with either `nano` or `vim`, but we'll use `nano`  
    
    ```bash
    sudo nano /etc/exports
    ```
    
    Make your changes and press ctrl-x, press y to save your changes, and press enter to keep the same filename.  
    We also need to update nfs to use these new settings. We do this by `export`ing the settings. The `ra` flags tell it reexport all directories, rather than just some specific ones.
    
    ```bash
    sudo exportfs -ra
    ```

### Add /etc/hosts entries (use nano or vim)
Our next step is to tell Lubuntu what the IP addresses of all of the nodes are. This step isn't __necessary__ per se, but it greatly simplifies your code later. This lets you refer to the other nodes by their name, instead of needing to remember their IPs.

To use `nano` to do this you'll run  

```bash
sudo nano /etc/hosts
```  

Then you'll add these to the file:  

```text
192.168.13.13	node0
192.168.13.1	node1
192.168.13.2	node2
192.168.13.3	node3
192.168.13.4	node4
192.168.13.5	node5
192.168.13.6	node6
192.168.13.7	node7
```  

Remove or comment out line `127.0.0.1	node0`. We only want the name `node0` to resolve to the cluster address, never the loopback address.

### Setup ssh keys
The way our nodes talk to eachother is through a secure communication method called ssh. Unfortunately starting a session of ssh normally requires you to enter a username and password each time, which would make our system a lot less automated.  
To solve this we'll generate an SSH key that doesn't have a password, and just use that key for each node's communication.  
To do this we use `ssh-keygen` and tell it the `t`ype of key we want is `rsa`, which is to say a key generated with an rsa algorithm.

```bash
ssh-keygen -t rsa
```  

Hit enter through all the prompts.  
Now we want to take the keys generated and add them to our authorized keys list.  
First we use `cat`, which in this usage just prints the contents of the file you give it - we'll give it `~/.ssh./id_rsa.pub`.  
Now that we're printing the keys, lets tell the console to output to our authorized keys list and just add it to that file. This is done by using `>>` to indicate we want to append the information to a file, and `~/.ssh/authorized_keys` is the location we want to add it to.

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

# Setup a slave node to the point of copying ssh keys

## Run through the above process to setup all of the slave nodes.

```text
Setup VM(s) for slave node(s)
Create VM / Configure virtual hardware
Follow procedure for master node substituting “node1” (or whatever number) for “node0”.
Install Linux
Follow procedure for master node substituting “node1” (or whatever number) for “node0”.
Configure installed system
Log in as pi
Open terminal
```

### Install updates and needed packages

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install nano vim curl openssh-server mpich mpich-doc nfs-common
```

### Setup network interface for internal cluster network

This time we'll run it almost the same as above, but we won't have the same IP address for the node as for the gateway. We want this node to use node0 as the gateway, so we input the IP for node0 after `gw4`.

```bash
sudo nmcli c add con-name cluster ifname enp0s8 type ethernet ip4 192.168.13.1/24 gw4 192.168.13.13
sudo nmcli c modify id cluster ipv4.route-metric 101
```

### Make sure the internal cluster network is working

To do this we'll `ping` the master node, if the master node responds we're good to go. If not, make sure you didn't skip any steps above. If you're in the training session and this doesn't work, now is a good time to put up your red sticky.
```bash
ping 192.168.13.13
```

### Mount the folder we shared with nfs

We need to get access to that shared folder on all of the virtual machines, so first we **m**a**k**e the **dir**ectory on this machine, then we `mount` the directory from the master node. This will keep this folder the same across all of the nodes.

```bash
sudo mkdir /shared
sudo mount 192.168.13.13:/shared /shared
```

Use `ls` to look for /shared/test.txt to see if the mount is good. If you don't see the file, again make sure you didn't skip any steps on accident. If you do see the file, use `cat` to see if the correct sentence is in the file.

```bash
ls /shared
cat /shared/text.txt
```

## Set the nfs share to mount automatically

We do this by editing the file that Lubuntu checks to see if there are any external filesystems to mount when it boots up.  
Add line to `/etc/fstab`: `192.168.13.13:/shared  /shared  nfs  rw,noatime,hard,intr,vers=3  0 0`

```bash
sudo nano /etc/fstab
```

Then just copy the line above into the file and close it like the last time we used `nano`.  
Now we'll reboot

```bash
sudo reboot
```

Log in and make sure that /shared is mounted by again running `ls`

```bash
ls /shared
```

### Add /etc/hosts entries

Complete this just like how we did on node0  

```text
192.168.13.13	node0
192.168.13.1	node1
192.168.13.2	node2
192.168.13.3	node3
192.168.13.4	node4
192.168.13.5	node5
192.168.13.6	node6
192.168.13.7	node7
```

Remove or comment out line “127.0.0.1	node1”. We only want the name “node1” to resolve to the cluster address, never the loopback address.

### Copy ssh keys from node0
Now we have to make sure each node has the ssh key for node0, so that they can communicate without a username and password each time.  
We'll do this using `scp` or **s**ecure **c**o**p**y. We use scp to make sure no one can listen in on the conversation and copy the key. It's not really necessary in this situation, but it's a good practice.  
When we use scp, we'll tell it that we're copying an entire di**r**ectory with `-r` - in this case we're copying from `node0`. We can just use the name because we set up all of the `/etc/hosts/` entries above. Then we give it the host `node0` and the file `~/.ssh`.

```bash
scp -r node0:~/.ssh ~
```

## Run the above settings for the other 6 slave nodes, substiting the number of the node you're working on anytime it says `node1`.

# Test MPI

MPI is the framework for sending messages and data back and forth between the nodes, so we're testing to be sure that works.  
From this point forward we can do everything from the master node, `node0`.

## Hello World

 1. **C**hange **d**irectories into the `shared` directory.
    
    ```bash
    cd /shared
    ```
    
    Now you're operating from inside the folder we have being shared to all of the nodes.  
 2. **M**a**k**e the **dir**ectory `hello_world` in this directory.  
    
    ```bash
    mkdir hello_world
    ```
 3. **C**hange **d**irectories into the `hello_world` directory.  
    
    ```bash
    cd hello_world
    ```
 4. Download the programs we'll use to test.  
    We'll do this by using `wget` to download the files from a url.  
    
    ```bash
    wget https://raw.githubusercontent.com/OSU-HPCC/example_submission_scripts/master/compiling_and_running/mpi/makefile
    wget https://raw.githubusercontent.com/OSU-HPCC/example_submission_scripts/master/compiling_and_running/mpi/hello_world_mpi.c
    ```
 5. Use the program `make` to automatically compile the files your just downloaded.  
 
    ```bash
    make
    ```
 6. Run the program!  
    To run the program we'll use `mpirun`, which runs it using `mpi`.  
    We tell it the **n**umber of **p**rocesses to use by giving it the `np` flag, then we tell it to run with `2` processes for now.  
    Then we tell it which two nodes will `host` the program.  
    Finally we tell it the location of the program itself, which is `./hello_world_mpi`
    
    ```bash
    mpirun -np 2 -host node0,node1 ./hello_world_mpi
    ```

## GalaxSee
This is a slightly more fun program.

 1. For this one we need to install a new program that GalaxSee uses.  
    Again, we'll use `apt-get install` to get the program.
    
    ```bash
    sudo apt-get install libx11-dev
    ```
    
 2. Make sure you're in the `shared` folder again.
    
    ```bash
    cd /shared
    ```
    
 3. Download the GalaxSee program itself using `wget` again.
    
    ```bash
    wget https://www.shodor.org/refdesk/Resources/Tutorials/MPIExamples/Gal.tgz
    ```
 
 4. Now we have to uncompress the program, because it's all stored in Gal.tgz but we need access to the files.  
    We do this with the `tar` command, and use a small slew of flags to tell it to extract it to the current folder.  
    
    ```bash
    tar -zxvf Gal.tgz
    ```
    
 5. **C**hange **d**irectories into the new folder that was made.
    
    ```bash
    cd Gal
    ```
 6. Modify the `Makefile` to compile the program to work on our particular cluster setup.  
    To do this with `nano` use
    
    ```bash
    nano Makefile
    ```
    
    Then make a few changes.  
    Change `CC = mpicc` to `CC = mpicxx`  
    Change `/opt/mpich/include` to `/usr/include/mpich` in the CFLAGS definition.
	  Change `/opt/mpich/lib` to `/usr/lib/mpich` in the LDFLAGS definition.
 7. Compile the program.
    
    ```bash
    make
    ```
    
### To run the program on just one node

```bash
mpirun -np 1 -host node0 ./GalaxSee 1000 5.5 10000.0 1
```

### To run on two nodes

```bash
mpirun -np 2 -host node0,node1 ./GalaxSee 1000 5.5 50000.0 1
```

### To run the node on even more nodes

Just increase the number after `-np`, and keep adding more node names to the list of nodes. You have 8 nodes right now.

#### Try opening a terminal and starting the `top` command on the slave node(s) before starting GalaxSee on the master node. You should see a GalaxSee process appear in the list of processes and disappear when the program finishes.


# TinyTitan Stuff

## (Optional) Connect Xbox360 Controller. 

If you don’t have a controller or can’t get it connected, you can still do everything with your keyboard and mouse.  
If running linux on your physical host (i.e. desktop or laptop) add yourself to the group `vboxusers`. 

```bash
sudo usermod --groups vboxusers --append username
```

Log out and back in, or do some hacky stuff to get your groups reevaluated and start virtualbox from command line (e.g. `newgrp vboxusers; newgrp originalgroup; virtualbox &`).  

Edit settings for node0 in VirtualBox
```text
	USB
		Create New Filter
		Edit the filter
			Name: Xbox360 Controller
			Vendor ID: 045e
			Product ID: 028e
```

Start node0  
Wait for node0 to complete startup  
Start node1  
Log into node0  
Start terminal  
Check to see if the VM sees the Xbox360 controller  

```bash
lsusb
```

## Install needed packages

On the master node run: `sudo apt-get install libglew-dev libgles2-mesa-dev libglfw3 libglfw3-dev libfreetype6-dev xboxdrv`  
On the slave nodes run: `sudo apt-get install lubglew1.13 libglfw3`

## Install TinyTitan programs

```bash
cd /shared
mkdir TinyTitan
cd TinyTitan
git clone https://github.com/TinyTitan/SPH
cd SPH
cp makefile_jetson makefile_lubuntu
sudo apt-get install pkg-config
pkg-config glfw3 --libs --cflags
nano makefile_lubuntu
```

Add `-L/usr/lib/x86_64-linux-gnu` to the CLIBS definition.  
Change `-lglfw3` to `-lglfw`.  
Add `liquid_gl.c` into the compile line before `mover_gl.c`.  

```bash
make -f makefile_lubuntu
```

## (Optional) Start the Xbox controller driver

```bash
sudo xboxdrv --mouse --axismap -Y1=Y1 --config /shared/TinyTitan/SPH/controller_2.cnf --silent &
```

## Run SPH with two threads (the absolute minimum; one for display and one for computation)

```bash
mpirun -np 2 -hosts node0,node1 bin/sph.out
```

#### Mouse and xbox controller are working but menu (sometimes?) doesn’t appear when hitting start button on xbox controller. Esc key on keyboard works. Move cursor over the Terminal option and hit the A key on either the keyboard or xbox controller to exit. The L key toggles between liquid and particle view (which color codes the individual threads).

# Extras

This stuff isn't needed for the cluster to function, but you might be interested in doing it.

### Passwordless sudo  

Run sudo commands while logged in as pi without being prompted for a password.

```bash
sudo echo “pi ALL=(ALL) NOPASSWD: ALL” > /etc/sudoers.d/pi_nopasswd
```  

### No GUI

Prevent the GUI from running when the computer is turned on (highly recommended for slave nodes)  

```bash
sudo systemctl set-default multi-user.target
```  

You may find that you need the GUI to run something. To start it manually run either of these:

```bash
sudo systemctl start lightdm.service
```  

or  

```bash
startx
```

If you decide that you want to start the GUI automatically again, run this

```bash
sudo systemctl set-default graphical.target
```

### Route through the master node

On a real system this is how you would network it properly. The difference with our system as done above is that each node has its own connection directly to the internet, which is not feasible with physical computers.  

 1. On the slave node(s), add Google's public DNS server to DNS resolver list. 
    
    Since /etc/resolv.conf is managed dynamically on Lubuntu, you'll have to add it to one of the resolveconf config files.
    
    ```bash
    sudo sh -c "echo nameserver 8.8.8.8 >> /etc/resolveconf/resolv.conf.d/tail"
    ```
    
 2. Shutdown the slave node
    
    ```bash
    sudo shutdown -h now
    ```
    
 3. Remove the NAT adapter from through the VirtualBox settings
    
    ```text
    Select the node in VirtualBox.
    Click the Machine menu → Settings.
    Click Network.
    Uncheck "Enable Network Adapter" for Adapter 1, then click OK.
    ```
    
Now we configure the master node to act like a router  
 1. Install netfilter-persistent. This allows you to easily make iptables rules permanent without having to modify any scripts.
    
    ```bash
    sudo apt-get install netfilter-persistent
    ```
 
 2. Setup iptables rules. 
    
    In the rules shown here, enp0s8 is the interface on the private cluster network and enp0s3 is the interface with internet access.
    We want to change the source address of connections from the slave nodes so that they appear to come from the master node.
    
    ```bash
    sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
    ```
    
    Forward anything from the cluster interface to the internet interface  
    
    ```bash
    sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
    ```
    
    Forward incoming replies back out to the cluster network.  
    
    ```bash
    sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
    ```
 
 3. Save the iptables rules
    
    ```bash
    sudo netfilter-persistent save
    ```
   
 4. Reload all the rules
    
    ```bash
    sudo netfilter-persistent reload
    ```
    
 5. Set the kernel flag to allow IP forwarding
    
    Open /etc/sysctl.conf for editing with your preferred text editor
    
    ```bash
    sudo nano /etc/sysctl.conf
    ```
   
    Uncomment the following line:
    
    ```text
    net.ipv4.ip_forward=1
    ```
   
   Save changes and exit the text editor. In nano `ctrl-x`, `y`, `enter`.
   
 6. Reprocess /etc/sysctl.conf to make the change effective
   
   ```bash
   sudo sysctl -p /etc/sysctl.conf
   ```
   
Now go back to the slave node(s) for testing.  

Check to see what interfaces are active:

```bash
ifconfig
```

You should only see `enp0s8` and `lo` (which is the loopback address). If you see `enp0s3` then you didn't remove the NAT network adapter - you will be able to conenct to the internet, but it will be through the same system as in the original instructions.  

Check /etc/resolv.conf and verify that it has Google's DNS server 8.8.8.8. It should be the only nameserver.  

```bash
cat /etc/resolv.conf
```

Try communicating with the outside world and verify that it works.  

```bash
ping -c 4 google.com
```

You should recieve packets back.  

For more info on setting up NAT addressing, [click here](http://netfilter.org/documentation/HOWTO/NAT-HOWTO.html).

### Expansion by cloning

To expand the number on nodes by cloning previous nodes (and reducing the number of steps for the new node) follow this, substituting 2's for whatever number of node you're making.

Make sure node1 is powered off.  

```text
Select node1 in VirtualBox and click the Machine menu → Clone.  
Enter node2 (or some other number) as the new name and choose to reinitialize MAC address of all network cards.  
Choose "Linked Clone". Instead of using the full space of the disk image, the new VM's disk image will only contain the difference between it and node1.  
```

Start the new node and login as the user `pi`.

Change the hostname from node1 to node2 using your preferred text editor.

```bash
sudo nano /etc/hostname
```

Change the IP address to node2's IP.

```bash
sudo nmcli c modify id cluster ipv4.address 192.168.13.2/24
```

**For node 3 it would be 192.168.13.3/24, and so on for other nodes.**

Restart the cluster connection.

```bash
sudo nmcli c down id cluster
sudo nmcli c up id cluster
```

Make sure the cluster interface came up with the new IP address.

```bash
ifconfig
```

You should see the IP you entered two steps before this.

And make sure it's working.

```bash
ping node0
```

Remove existing host keys.

```bash
sudo rm /etc/ssh/ssh_host_*
```

Generate new host keys

```bash
sudo dpkg-reconfigure openssh-server
```

Restart

```bash
sudo reboot
```

From the master node, add the slave node's host key to your known_hosts file.

```bash
ssh-keyscan -t rsa node2 >> ~/.ssh/known_hosts
```

Test to make sure you can ssh from the master node to node2 without a password.

```bash
ssh node2
```

You can now start node1 again; all nodes should be able to ping each other.


# Notes

The basics of a Beowulf-type cluster: 
 * Same /etc/hosts file on all nodes  
 * Same user (with same UID) on all nodes  
 * Passwordless ssh  
 * Choose to either setup nfs server or synchronize home directories of all nodes  
 * Install mpi and other stuff  


Network config options:
 1. Each VM has both NAT and internal network connections (simpler, as done in basic instructions)
 2. Only master node has NAT; all VMs have internal network and route through master node (like HPCC’s Raspberry Pi cluster, steps to
    do this are in the extras section)


Since the user pi is the first and only user and is created during linux installation, we know it will have the same UID number on the master and slaves. This UID number should be "1000". This can be verified with the command "id -u". On systems that have already been setup, you'll need to verify that your cluster user has the same UID on all nodes or create a new user with an explicitly assigned UID instead of letting the system choose.
