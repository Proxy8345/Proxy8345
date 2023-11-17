This is the beginning of my own notes and documentation on building out a virtual homelab utilizing Oracle VirtualBox. I chose this hypervisor because it's free, works with linux, and is a hosted mostly open-source hypervisor. The main benefit to choosing open-source software is that it allows trusted 3rd parties to audit the code or you can personally audit the code yourself. 

Our first step is downloading the hypervisor open your web browser and navigate to hxxps://www[.]virtualbox[.]org/wiki/Downloads (This is a defanged version of the URL simply get rid of the brackets and replace "hxxps" with "https")


![[VM Download 01.png]]
Now that we are on this home page we are going to click the Linux distributions tab because my OS is Linux. Once there we will download Ubuntu 22.04 since I am using Pop_OS and it is based on Ubuntu 22.04. Simply use the GUI to navigate to install the DEB package. 


![[Install VirtualBox 02.png]]
If you want to forgo using the GUI and just want to use the CLI for a much faster download type:
|sudo apt install virtualbox-7.0| into your terminal, to run the program simply type
|virtualbox &| into your terminal.![[Cli instructions 03.png]]
(I didn't install it because its already installed on my system)

The first thing we are going to do now that virtualbox is installed and running is configure the host-only virtual network adapter![[Configuring Host-Only Virtual Network Adapter.png]]
Click the hamburger menu beside Tools and then navigate to the network option in the menu. The host-only network has 3 features creating host-only network adapters,removing them, or adjusting their properties. Click create to create your host only network adapter and disable the DHCP server option because the pfSense DHCP server is better and easier to manage. Also according to the documentation from Tony Robinson's Building Virtual Lab configuring the IP address via the hypervisor tends to not "stick" or have bugs after applying them so instead we will be using the terminal to configure the IP address of our network adapters.
![[vboxnet0 04 1.png]]

Every OS has a method of manually configuring the IP address of connected network adapters since we are on Ubuntu 22.04 (Pop_Os) we can use either the IP ADDR or ifconfig command to set the IP and subnet mask on the host-only network adapter. (You can always download either command) 
The instructions for ifconfig are:
ifconfig vboxnet0
sudo ifconfig vboxnet0 172.16.1.2 netmask 255.255.255.0
ifconfig vboxnet0

We run the ifconfig vboxnet0 to confirm the interface exists then we config the IP and netmask via ifconfig using sudo since it needs escalated privleges to perform that task. Then lastly we run ifconfig vboxnet0 to confirm the interface IP and subnet mask match what we assigned it

The instructions for ip addr are:

ip addr show dev vboxnet0
sudo ip -4 addr flush label "vboxnet0"
sudo ip addr add 172.16.1.2/24 dev vboxnet0
ip addr show dev vboxnet0

First we verify vboxnet0 exits, then we remove any default IP address configs assigned by VirtualBox, then we config the IP address and lastly we verify the IP address did indeed change to what we assigned it. You will also have to refresh the VirtualBox to see the change in your IP address

![[Configuring IP address 05 1.png]]

Now with that out of the way we can build our first VM, pfSense! First we have to download pfSense
![[pfsense download 06.png]]
CD to the downloads folder and then run the gzip command to extract so we have the ISO and can make a bootable VM from it. Remember to use linux's built in auto-tab completion to avoid typing the entire file name out. (pfSense{tab}{tab})
![[gzip command 07.png]]
After uncompressing the file we are left with a readable version of the OS our next step is to now create the pfSense VM. Navigate to the machine tab and click on new.
![[pfSense config_1 08.png]]
You will be greeted with this popup simply select the ISO we previously uncompresssed and select BSD for the type due to the pfSense OS being based upon the FreeBSD opensoure Unix-like operating systems. We are then going to allocate 512mb of ram, 1 cpu core, and 5gb of virtual hard disk space.

![[pfSense Config_02 09.png]]
Thankfully as of the time of writing this section 09/21/23 there are no VirtualBox settings for us to change outside of the network section, it works right out of the box. For future reference if settings do change I have included Tony Robinson's steps below:

![[pfsense config_03 10.png]]

![[pfsense config_04 10.png]]

Once you have verified these settings we will navigate to the Network option and configure Adapter 1-3 to be enabled and adapter 4 must be unchecked. These next steps are going to be crucial due to the pfSense VM having the job of enforcing network boundaries and segmentation, if you get this part wrong the network won't work quite as intended. We are going to make Adapter 1 the pfSense WAN interface connected to our bridged network aka our real physical network at home. By selecting the bridged Adapter setting under attached to we allow this device to be recognizable on our physical network as well!![[pfsense config_05 12.png]]
Now we are going to turn Adapter 2 into the pfSense LAN interface connected to the management network. By clicking the Host-Only Adapter option we allow it to be a LAN connection to the vboxnet0 Network Adapter. Think of it like we are running an Ethernet cable to the Network Adapter and keeping the network segmented from the WAN.
![[pfsense config_06 13.png]]
Lastly we are going to configure adapter 3 to be the pfSense OPT1 interface connected to the IPS 1 network. The reason we are using the internal network configuration is so we can segment the IPS 1 network as much as possible.

![[pfsense config_07 14.png]]
Now with the network options configured we can boot up our machine by hitting the start button 
![[Pasted image 20230921205757.png]]

Sadly we ran into an error trying to get the pfSense VM to boot, time to troubleshoot!
![[Pasted image 20230921210239.png]]
After using the virtual box forums I was able to find a thread that helped explain that a CPU not supporting "long mode" means that you are trying to use a 64bit OS while having a CPU that only supports 32bit however My Ryzen 5 5600x is 100% 64bit. I realized that my mistake was configuring the VM as FreeBSD 32 bit when the OS was actually 64bit. This issue boiled down to user error. (as it usually is)
![[Pasted image 20230921211251.png]]
Now we are in business and can boot up our VM! 
![[Pasted image 20230921212212.png]]
Simply navigate though the install wizard and once it is installed we are going to navigate back to the system option on our pfSense VM settings and edit the boot order unchecking the Floppy and Optical disc sections 
![[Pasted image 20230921213922.png]]
You can think of this section as us using a USB stick to install an OS on a physical computer than we edit the boot order to ensure we aren't booting back into the USB stick which would just take us through the installer again. Then we navigate to the storage section and left click on the blue disc image and remove the attachment, again this is like we are taking out the USB stick on a real computer.
![[Pasted image 20230921214517.png]]
Now with that out of the way we can boot our VM again after successfully booting we are greeted with the CLI and now need to assign interfaces to map our VMs network interfaces (Adapter 1-3)

![[Pasted image 20230921222240.png]]
We can cross reference the MAC addresses for our adapters when we set them up in the network section to know which adapter corresponds with em0,em1, and em2 since the FreeBSD project has its own methods of assigning physical or virtual network interfaces names. 

![[Pasted image 20230921222651.png]]
em0 == Bridged Network
em1 == Management Network
em2 == IPS 1 Network
To help guide you through the questions the wizard will ask I provided my answers and why:
Should VLANs be set up now? y/n    N         why: Because we won't be using VLANs at all in this lab
Enter the WAN interface name       em0       why: Because it is the bridged network interface and is therefore needed to be accessible by all

Enter the LAN interface name        em1            why: Because it is the management network and will be our LAN connection because we want the management network to be segmented from the WAN for security reasons

Enter the OPT1 interface name    em2            why: Because it connects to the IPS 1 network

![[Pasted image 20230921223625.png]]

Now that we have assigned our interfaces we are going to set their IP addresses through the pfSense CLI. This section does require understanding the fundamentals of assigning IP address and subnet masks. Basically the addresses 192.168.x.x, 192.168.x.x, 172.16.x.x are part of what is known as the RFC 1918 standard by which networks assign IP addresses in a private network. Each address will be unique on that network but not outside of it Private IP can't be communicated with directly by external computers because they are not globally unique as such not addressable via public internet.

![[Pasted image 20230921224813.png]]

![[Pasted image 20230921230418.png]]

Now that we have an IP address for our WAN,LAN, and OPT1 now we are going to test their connectivity. Click the shell option (8) in the pfSense terminal and we are going to run three commands:
ping -c www[.]google[.]com
nslookup www[.]google[.]com
curl -I www[.]google[.]com
These commands just test if they can reach the google domain which is almost never down .

![[Pasted image 20230922192442.png]]

Now that we've established our network connectivity is functioning as intended we are going to access the webConfigurator. In your preffered brower navigate to hxxps://172[.]16[.]1[.]1 (Defanged URL) if your browser gives you a warning about not trusting the SSL certificate just click accept the risk and continue. Normally you wouldn't do that but since we created the VM ourselves and can verify it isn't malicious its acceptable. Use the default credentials of admin and password: pfsense to login.

![[Pasted image 20230927182331.png]]
Once logged in we are greeted with the setup wizard 
![[Pasted image 20230927182841.png]]
After resetting the admin password (because default configs are always very insecure) we have completed the setup wizard! Now that we have finished. Our next step is to check for updates, it is always best practice to have the most up to date system to avoid vulnerabilities that bad actors have the capability of attempting on your systems. Click the system tab and then update to check for a new update.
![[Pasted image 20230927184359.png]]
We can also navigate to the general system settings page and change the webConfigurator theme from the awful light mode to dark mode. its much easier on your eyes when you have to stare at a computer for your work. 
![[Pasted image 20230927184614.png]]
Now in these next 4 sections we are going to enable NTP,Squid HTTP proxy, DHCP, and DNS forwarding. Our first step is enabling DNS forwarding the best way I can describe DNS forwarding is that instead of your computer making the DNS queries to the 13 root DNS servers (There are more like 13 "clusters" comprised of 1000s of computers but can be accessed through 13 individual IP addresses) you get another the benefits from specified DNS servers provide such as DNS filtering against known malicious sites, improved latency,etc. We are going to be using the Quad9 DNS server as our designated IP to perform DNS forwarding to (hxxtps://www[.]quad9[.]net to learn more about Quad9)
Before we enable DNS forwarding we are going to define what network interfaces are going to listen for and respond to DNS requests. 

![[Pasted image 20230927194102.png]]
With that complete we are going to configure multiple DNS servers for our DNS forwarding traffic. We are adding Quad9, Google's DNS server, and Level 3 Communications DNS server
![[Pasted image 20230927194520.png]]
It is crucial to remember that when using a DNS server you review their privacy policies stating what they do with your data, what their log retention policy is like, and what third parties are involved with the data. The most important thing when considering using public DNS servers is balancing features and availability against how much you value your privacy security is in, my opinion a  varying spectrum. Joe the accountant that watches netflix on his off days does not have the same security needs as a Fortune 500 company or a darkweb vendor. Also despite the company stating they only keep certain parts of your data or they delete them after X days doesn't mean its necessarily true because you have no way of verifying that.

Now that we have finished with DNS forwarding our next step is setting up NTP. NTP stands for Network Time Protocol are it is essential because a lot of services and certificates will not function if your time is off. Another reason NTP is so cruical is because of digital forensics and incident response almost all logs are timestamps and it is very important to know when exactly something happened because different insights can be developed from that data. Your NTP configuration can be the crucial piece of the puzzle that can be the difference in an investigation being accurate or inaccurate. 
Now with that being understood we are going to navigate to Services>NTP and define our network interfaces as LAN and OPT1. Although the deafult pfSense NTP server is enough we are going to add NTP servers from hxxtps[.]www[.]ntppool[.]org
![[Pasted image 20230927200350.png]]
Our next step is configuring an HTTP Proxy, they can serve a variety of purposes such as ALLOW/DENY websites, speed up internet access via caching requested files,etc.  Navigate to the System>Package Manager tab and then type the keyword "Squid" into the searchbar and install the package.
![[Pasted image 20230927201406.png]]
![[Pasted image 20230927201434.png]]
Navigate to Services>Squid Proxy Server and configure the server as shown.
![[Pasted image 20230927202026.png]]

Now that we finished configuring out HTTP Proxy we are going to now configure our LAN and OPT1 network interface to be DHCP servers. Navigate to Services>DHCP server and enable DHCP server on LAN and OPT1 interface the DHCP scope (ranges) are below:

OPT1
![[Pasted image 20230928183927.png]]
LAN 
![[Pasted image 20230928183950.png]]
The way to determine your DHCP scope is based upon your subnet and avaliable IP addresses within your subnet for example the amount of available IP addresses for our OPT1 network interface. For our IP address of 172.16.2.1/24 we have 254 usable hosts within this subnet. We however are using the first 10 available IP addresses for static DHCP allocations however we can not reassign the OPT1 interfaces IP of 172.16.2.1 leaving us 8 possible IP addresses for DHCP static mapping. We will be coming back to configure our DHCP static mapping when we build more VMs.  

PfSense is a modern stateful firewall that is capable of tracking connections and determining whether or not packets being transmitted are related to an established connection. We are going to configure the firewall policy of pfSense but before we do it is crucial to understand that the order in which firewall rules are placed matters. Most firewalls will process rules as they apply to traffic from top to bottom. The first rule that matches the nature of the traffic best will be applied first, and the traffic will be allowed or denied based on the action of that firewall rule. 
![[Pasted image 20230928195248.png]]
Most professional network firewalls use whats called implicit deny any or default deny any. As the name implies unless there is a firewall rule to allow a certain kind of traffic the firewall defaults to denying it. pFsense also has nice feature called Aliases where you can define a group of IP addresses, network ranges, URLs, or ports under a single name. You can think of an alias as an array that holds a set of values. Navigate to the Firwall>Aliases tab in the web configurator then we are going to to create an alias named RFC_1918
![[Pasted image 20230928200104.png]]
![[Pasted image 20230928200707.png]]
Now we are going to add firewall rules, navigate to the Firewall>Rules tab and we are going to allow DNS traffic to our LAN interface with this rule
![[Pasted image 20230928201515.png]]
Note we didn't check the log option because we don't have the storage space necessary to store that amount of log information however later on we can create a syslog server to store all the log information. Now that we understand how to create a rule we are going to now creates firewall rules for our WAN,LAN, and OPT1 interfaces.
Our WAN interface is going to have a single block rule deny for all IPv4 and IPv6 traffic. This rule will not prevent VMs on the lab network from accessing hosts or the internet
![[Pasted image 20230928202327.png]]

Next, we need to configure the firewall policies our LAN interface is going to be using.

![[Pasted image 20230928204716.png]]
Finally we are going to configure out firewall policies for our OPT1 interface (Don't worry I'm going to explain why we are using these firewall rules soon)
![[Pasted image 20230928205737.png]]
The LAN interface's firewall rules are configured how they are to allow DNS, NTP, and HTTP proxy services to function on the network. The better anti-lockout rule is better than the default anti-lockout rule because it only allows a single IP address to access the webconfigurator over HTTPS allowing the hypervisor to have guaranteed access. Then the fourth rule is going to allow us to SSH into our Kali VM, RFC_1918 address denial is used to enforce segmentation of the lab networks from one another as well as any physical networks the VM lab is connected to,allow HTTPS and NTP outbound, and finally an implicit deny rule to deny everything not included in the rule-list.

The OPT1 interface is designed to enable the network segment for malware analysis. The first rule is to deny our Metasploitable 2 VM from communicating to any external networks (There will be real malware and attacks performed on this machine after-all it is meant to be a sandbox) the next rules are almost the same as the LAN interface designed to allow DNS, NTP, and HTTP proxy services and deny RFC_1918 connections. We also created a rule to deny HTTPS and FTP outbound to be logged through the SQUID proxy on the pfsense VM. Most malware uses HTTPS for external Command and Control (c2 servers) or other custom ports the allow any/all rule allows this traffic outbound with the idea being that whatever communication is passed through MUST go through our inline IDS allowing us to see what connections are being made and where! It is crucial to rememeber the OPT1 network is setup to be more permissive to allow malware analysis and observation of payload delivery or command and control and keeping the vulnerable boot2root VM away from any other network.

Now that we have configured all of our firewall rules our next step is creating a snapshot so that way we have a known good back-up of all of our hardwork thus far. Click the hamburger menu beside your VM and click snapshots and hit the take button. 
![[Pasted image 20230929185055.png]]

Now that we have our snapshot it is also important to remember that snapshots are not a substitute for backing up important files and data it is recommended to have a data recovery plan in place so you don't lose your valuable data. Now we are going to disable the default anti-lockout rule to implement a concept called least privilege. The main idea of least privilege is to provide only the minimum amount of access necessary to perform a task which reduces your possible threat vectors! By only allowing 172.16.1.2 our hypervisor to access the webConfigurator we are reducing the changes of one of the VM becomes compromised and attacking the webConfigurator thus compromising the entire lab network. Least privilege however is only a single layer of defense to make our VM more secure we must implement multiple overlapping defensive techniques to protect our VMs ensuring we use proper patch management,very strong passwords,etc. This concept of multiple layers is known as Defense in Depth! Navigate to Firewall>Rules and then click the gear icon beside the Anti-Lockout rule and scroll through the options to disable the Anti-Lockout rule

![[Pasted image 20230929190937.png]]
Remember to drag the better anti-lockout rule to the top of the list so that way it is the first rule applied. (Order matters!)

Now that our pfSense firewall VM is complete we are going to now create the four remaining VMs (Kali,IPS,SIEM, and Metasploitable v2 boot2root) 
![[Pasted image 20230929191439.png]]

Our first VM we are going to create is our IPS simply install the ubuntu server from hxxps[://]ubuntu[.]com/download/server (defanged URL) and allocate the hardware resources just like we did with the original pfSense VM
![[Pasted image 20230929203629.png]]
Once we have created the VM before we boot into it we are going to assign it a static IP address with our DHCP server on pfSense. Navigate to Services>DHCP server and enter the mac address of the IPS VM located in the network section and assign it the IP address of 172.16.1.4 (Or whatever other RFC_1918 IPv4 type you want to use)
![[Pasted image 20230929211908.png]]
(note that pfSense uses XX: format for MAC address if you simply copy-paste the MAC provided by Virtualbox you will get an error)
Now we are going to configure the network adapters for the IPS VM
![[Pasted image 20230929212746.png]]
![[Pasted image 20230929212803.png]]
![[Pasted image 20230929212818.png]]


With that finished simply finish the install of the Ubuntu Server OS
![[Pasted image 20230929203909.png]]
This is the IP address and port number for our SQUID proxy service we intend on using.
![[Pasted image 20230929214813.png]]
![[Pasted image 20230929215202.png]]
Remember it is a great idea to utilize the KeePassXC password manager we downloaded earlier so you can keep very strong passwords or alternatively keep a physical notebook at your house locked with all the passwords to the labs or your own accounts you use. Once we have finished installing and removed the ISO media from controller:IDE and changed the boot order to only be the Virtual Hard Disk drive we are going to login to our server. There is no GUI when accessing the server only a terminal can be used to navigate. Use the nslook up www[.]google[.]com command and ip -br a command to test connectivity.

![[Pasted image 20230929220648.png]]
Now that we have confirmed we have the proper IP assigned via the DHCP server, can resolve hostnames, and has HTTPS connectivity we are going to run the commands:
sudo su -
apt-get update
apt-get -y dist-upgrade
init 6
to become the root user, install updates (confirm the squid proxy is proxying the IPS vm's http requests) and reboot the system.



The steps for the SIEM VM are the same except we are going to assign it the IP address of 172.16.1.3 and only allowing the vboxnet0 adapter on the VM

![[Pasted image 20230929233521.png]]
![[Pasted image 20230929233726.png]]
![[Pasted image 20230929234450.png]]
Run our network connectivity test again and update the machine and reboot it just like with the IPS VM.

Our Kali Linux VM is created in the same way as the other VM's simply follow the Graphical Install wizard.
![[Pasted image 20231026131209.png]]
![[Pasted image 20231026135107.png]]
Add the Kali VM to the static DHCP server under OPT1 while also changing the network settings to be attached to the internal network (Note attempting to put the IP address under the LAN subnet will result in an error due to the LAN subnet being 172.16..1 while the OPT1 subnet is 172.16.2 speaking from experience)
![[Pasted image 20231026150749.png]]
We are going to confirm our Kali VM has been assigned a static IP address and then configure our Kali VM's apt package manager command to be ran through the Proxy server the echo command simply allowed us to change the apt.conf file adding an argument to it
![[Pasted image 20231026160830.png]]
If using cat on the file gives back the text as shown above then you successfully edited the config file.
now update the Vm with apt-get update and apt-get -y dist-upgrade

Now that our Kali VM is upgraded and functional we are going to now create the boot2root Metasploitable 2 VM.
Decompress the Metasploitable 2 into the default Machine folder and now go through the process of creating a VM through the virtual box wizard
![[Pasted image 20231026161956.png]]
![[Pasted image 20231026162040.png]]
![[Pasted image 20231026161947.png]]Notice there is no ISO image for us to select because the Metasploitable VM is all on its own VMDK file (aka a virtual hard disk)
![[Pasted image 20231026162540.png]]
Add the Metasploitable2 VM to the DHCP server for static IP mapping
![[Pasted image 20231026162623.png]]
![[Pasted image 20231026162649.png]]
change boot order
Disable USB controller
Disable shared clipboard
This is going to be the victim machine for testing malware you do not want a VM escape to happen and infect the rest of the network or get to the bridged adapter making it to your physical network. 

Our final step before we set up the services on the VM's is to create baselines for each VM.
![[Pasted image 20231026163540.png]]
These baselines are important and you should back them up on another drive for data disaster recovery.

We've come a long ways to create this lab the final steps are enabling remote SSH access, Installing Snort3 for the IPS, and installing Splunk with log forwarding enabled on the IPS VM (Snort3).


In order for us to be able to communicate from our hypervisor host we are going to need to change our routing table because currently the hypervisor host directly communicates to the default gateway (my router) so it would be unaware of the OPT1 172.16.2.0/24 network even though if it communicated with the firewall it would find that network. The default gateway is also unaware of that network as well.

In order to change the routing table in linux use the following commands
sudo ip route add 172.16.2.0/24 via 172.16.1.1
ip -4 route

![[Pasted image 20231026173911.png]]
Our next step is enabling the SSH.service on the Kali VM using the commands 
systemctl enable ssh.service 
systemctl start ssh.service

![[Pasted image 20231026174333.png]]
Now we are going to create a new baseline snapshot of Kali with SSH enabled now we simpley need to SSH using 
ssh user@hostname
so ours would be ssh kali@172.16.2.2
You will then recieve a warning saying the authenticity of host cant be established we can click yes but the reason it gives you that warning is because the host key is not in the users home directory called known_hosts it is essentially a phone book for the computer to verify the identity of a system you want to connect to. Since I am using linux the main difference between Linux and Windows SSH is that you cant just automatically add the host key to ignoring the warning

Now that we can ssh into our VM let's make this process easier for maximum laziness err I mean efficiency! 
By going into the .ssh directory (if you dont have it create one with mkdir) and there we are going to create a file called config with the touch config command.
![[Pasted image 20231026181035.png]]

Use nano to edit the file with the following information and now when we type ssh and the host name we will automatically be prompted to enter the VM's password for password authentication. Since we are interesting in security we are going to work on providing MFA/2FA to the system by generating an SSH key pair so when you authenticate you are authenticating with something you know (password) AND something you have (the SSH key).

We are going to use the ssh-keygen -t ed25519
to create a public and private SSH key with the ed25519 algorithm the ssh-keygen will also ask you if you want to assign a password to the private key which is highly recommended.
The next step is copying the SSH public key to the lab VMs using the following commands:
ssh-copy-id siem
ssh-copy-id ips
ssh-copy-id kali

![[Pasted image 20231026183848.png]]
When you attempt to ssh into the VM's it will ask you to unlock the private key now we have enabled SSH key-based authentication!

Now we are going to move on to installing Snort3 i am going with snort since I am familiar with snort rules and using snort from TryHackMe.com's online labs. We are going to be using the Autosnort3 script written by Tony Robinson to install snort due to it being pretty labor-intensive.

SSH into the ips VM (with it online) becoming root via sudo su and then run the command 
git clone https://github.com/da667/Autosnort3
cd ~/Autosnort3/Ubuntu/AVATAR
![[Pasted image 20231026203014.png]]
Edit the interfaces to be what your IPS 1 and IPS 2 network would be for me it is this
snort_iface_1=enp0s8
snort_iface_2=enp0s9
o_code= (Your oink code provided by snort via their domain on your account)

Next run the following commands to let the CLI know there is a squid proxy installed on our network that handles http traffic and what port its on
export http_proxy=http:// 172. 16. 1. 1: 3128
export https_proxy=
and finally run the bash script using the command
bash autosnort3-Ubuntu.sh
It takes a long time for it to complete so be patient.

Once the IPS has rebooted we are going to SSH back in with the Kali and Metasploitable2 VM running. And now check the status of snort3 using the command systemctl status snort3.service
![[Pasted image 20231027162459.png]]

Now we need to confirm that IPS1 and IPS2 are have been briudged. the easiest way to do that is to open the Kali VM and attempt to connect to the web server on the Metasploitable VM. Log into the Kali VM and run curl 172.16.2.3 which is the metasploitable server and then you'll get an HTML output printed to your console
![[Pasted image 20231027164754.png]]
Now we are going to stop the snort3.service using systemctl stop snort3.service and attempt to curl the Metasploitable VM again (Run the systemctl status on snort3 to confirm its inactive)
![[Pasted image 20231027165110.png]]
Now restart the snort3 service and then curl the Metasploitable VM again
![[Pasted image 20231027165315.png]]
We have now confirmed the Snort AF_PACKET bridge to IPS2 network segement is functioning





Now with the IPS completed we can now install splunk and get our SIEM running (for 60 days capped at 500mb thanks Splunk!) Create a splunk account and just put your email as the business email and for the company just put student. Go to the linux portion and click the download via Command Line tool and you'll get the command. SSH into the SIEM VM and run it as root. 
After downloading the splunk .deb package via wget now we are going to run the following commands

dpkg -i splunk-*.deb (use autotab completeion to avoid spelling out the name )
chown -R splunk:splunk /opt/splunk
/opt/splunk/bin/splunk start --answer-yes --accept-license
![[Pasted image 20231027171431.png]]
enter what you want your admin user and password to be and save them in your password manager
Now run the following commands:
/opt/splunk/bin/splunk stop
/opt/splunk/bin/splunk enable boot-start -systemd-managed 1
chown -R splunk:splunk /opt/splunk
systemctl start Splunkd.service
systemctl status Splunkd.service

![[Pasted image 20231027172143.png]]
![[Pasted image 20231027172247.png]]
We used these commands to make Splunkd start upon boot and starting splunk and confirm that its active.
Now we are going to log into the web interface at http:{IP address}:8000
Once logged in we need to enable HTTPS, Switch to free liscenseing with splunk, and set up a listener to accept logs forwarded from the IPS VM.
![[Pasted image 20231027173328.png]]
![[Pasted image 20231027173348.png]]
Now we need to restart splunk
![[Pasted image 20231027173504.png]]
![[Pasted image 20231027173724.png]]
Now we have enabled HTTPS

Now we are going to switch to Splunk Free Liceensing
![[Pasted image 20231027173858.png]]
You can confirm you switched to Free Licensing looking at the licensing page.
![[Pasted image 20231027174145.png]]
Lastly we are going to configure a receiver
![[Pasted image 20231027174423.png]]
![[Pasted image 20231027174601.png]]
![[Pasted image 20231027174630.png]]
we run the ss -alnt4 | grep 9997 command to verify its listening on port 9997
Now the SIEM VM is prepared to receive data now we just need to install the universal forwarder application on the IPS VM. SSH into the IPS vm into Navigate to splunk[.]com and download the universal forwarder via CLI 
![[Pasted image 20231101183746.png]]
![[Pasted image 20231101184605.png]]
Run the commands
dpkg-i splunkforwarder-9.1.1 (use auto-tab completion to finish it)
chown -R splunkfwd:splunkfwd /opt/splunkforwarder ()
sudo /opt/splunkforwarder/bin/splunk start --answer-yes --accept-license
opt/splunkforwarder/bin/splunk add forward-server 172.16.1.3.9997
sudo /opt/splunkforwarder/bin/splunk enable boot-start -systemd-managed 1

We configured it to be the port we designated on our Splunk web configuator tcp port 9997. Now we are going to confirm functionality  using the following commands
systemctl start SplunkForwarder.service
systemctl status SplunkForwarder.service
ss -ant | grep 9997 (this command confirms we are communicating over to the siem on port 9997. ss also stands for secure socket fun fact)

Now we are going to test triggering an IDS event to see if we can get them to show up in splunk on the SIEM VM. SSH into Kali and and then run the following commands (make sure the Metasploitable 2 VM is online)

curl -I 172.16.2.3 (confirms IP is online)
timeout -s KILL 30 wget -m 172.16.2.3 (used to attempt to download all contents from the Metasploitable VM and then kill the command after 30 seconds)
nikto -host 172.16.2.3 (Nikto is an aggressive web application scanner that will used to generate attack traffic)

Now with our newly generated attack traffic log into the Splunk web interface and click on the Search and reporting tab. For some reason it was not working correctly because I was not recieving any logs despite running nikto and an nmap command on the machine so I had to troubleshot what the issue was. I first determined that if the services were online with systemctl status Splunkd.service 
systemctl status snort3.service
systemctl status SplunkForwarder.service
they were all correctly online I also used the curl -I command to the Metasploitable 2 IP to ensure it was able to reach IPS2 network I then checked my splunk webconfigurator for if the reciever was listed on it and checked if the IPS was listening on port 9997 via the command ss --ant4 | grep 9997 and it was also listening.
The final issue had was not within the network itself but snort generating alerts and sure enough it was a .config file I didnt create for JSON alerts
So I had to download the json alerts app on the splunkbase website and then use 

scp~/Downloads/snort-3-json-alerts_103.tgz ips:~/
cp /home/snorting_pig/snort-3-json-alerts_103.tgz /opt/splunkforwarder/etc/apps (copies it)

Then I had to run the following commands
cd /opt/splunkforwarder/etc/apps
tar -xzvf snort-3-json-alerts_103.tgz
mkdir TA_Snort3_json/local
vi TA_Snort3_json/local/inputs.conf

and add the following to the conf file 

[monitor:///var/log/snort/*alert_json.txt*]
sourcetype = snort3:alert:json
[monitor:///var/log/snort/*appid-output.log*]
sourcetype = snort3:openappid:json


The other issue I had is that the snort3 json alerts app was not installed on the SIEM VM so I simply used the web configutator to addd the snort3 json app from the app list instead of making a directory with the already existing snort3 json file I had. Once that was complete I was able to generate logs from the attack used from the Kali VM to the Metasploitable VM.
The baseline of this Virtual Lab is now complete, thank you for following along!