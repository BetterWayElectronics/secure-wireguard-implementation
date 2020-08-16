## A Guide On WireGuard/DNSCrypt/SSH/Honeypot Implementation on OVH

**Introduction**

The plan in this guide is to create a secure WireGuard VPN which has its
own embedded DNSCrypt DNS resolver, this ensures that all connections
including DNS requests made by the user are tunnelled through the VPS
and is encrypted end to end. This is also expanded to include security
that resolves around this process making the server as secure as
possible from external agents. The following represents the proposed
network diagram for this guide.

![](media/image1.jpeg)

Why use WireGuard? As you can see in the image after this paragraph,
whilst on the WireGuard VPN speed decrease against a direct connection
to the internet is negligible (~3Mbps), this is because WireGuard runs
within the kernel space and thus ensures the secure tunnel can run at
high speed, it is even now part of the latest Linux Kernel 5.6. But
while this was my personal reason for implementing WireGuard there is
also the benefits in its simplicity in both development, with a lean
codebase of 4000 lines (compared to 100,000 in OpenVPN) but also in its
implementation -- which will be illustrated in this guide.
Fundamentally, you install the service and a client and exchange keys;
it can't be easier than that. WireGuard also supports better
cryptographic methodologies than OpenVPN and easier to expand and
distribute among peers.

![](media/image2.png)

**Buying and setting up the VPS**

![](media/image3.jpeg)

To start this project you need your own virtual private server (VPS), I
recommend OVHcloud https://ovh.com.au/ and their cheapest
starter package which is $5.00AUD a month. You get 2 GB of memory which
is more than enough to host multiple clients on the VPN. Once paid for
you will receive your IP address, username and password which you'll use to
connect to the server. Be sure to issue `apt update` and `apt upgrade`
and do this regularly. I also recommend changing the password that they
give you and securing your account in the method you prefer.

The first thing to do is to now secure the SSH connection and ultimately
customise it.

Start by installing fail2ban, an active intrusion detection system
designed to ban brute force attempts towards your SSH. Issue the
following commands to install fail2ban:

-   `apt install fail2ban`

-   `cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local`

-   `cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`

Once this is done you'll need to modify the `/etc/fail2ban/jail.local`
file to adjust the time limits, this will mean that users whom attempt
to login to your SSH will get banned for x period of time after x
attempts etc.

![](media/image4.jpeg)

From here you can issue the command `systemctl status fail2ban.service`
to confirm the service is indeed running. You can also view the IP's
that are currently banned with the command `fail2ban-client status
sshd`, through the iptable rules or view the log file directly at
`/var/log/fail2ban.log`.

![](media/image5.jpeg)

Another important step is to change the default SSH port of 22 to
something else. This will aid in preventing automated bots from scanning
your VPS, though it would not prevent somebody from discovering it
eventually. This can be done by modifying the SSH config file at
`/etc/ssh/sshd_config`. Don't forget to update your fail2ban to suit the
change, but I have noticed that fail2ban does not seem to enjoy being
modified after the fact. If this is the case for you just modify the
iptables manually. To delete the original rule, find its number with
`iptables -L --v --n --line-numbers` and deleting it with `iptables --D
INPUT #`. Now to add your bespoke SSH port with fail2ban issue the
following command `iptables -A INPUT -p tcp --dport SSHPORT# -j
f2b-sshd`.

![](media/image6.jpeg)

**Password-free Entry (Windows)**

The next step is to be able to login to the VPS without a password or
even a password prompt (although this is optional), thus we would need
to use SSH key pairs. This can be generated from Windows or within Linux
directly. The first step in achieving this is to disable password
authentication within the SSH config file, so after modifying the port
scroll down modify the following two settings `PasswordAuthentication
no` and `UsePAM no`. At the bottom of the configuration file add the
following `AuthenticationMethods publickey`. Save the configuration file
and do not close the current PuTTY session. Worst case scenario you can
still access your server through the KVM console within the OVH website.

Now the keys need to be generated. Continuing with an example from
Windows, launch the PuTTY Key Generator and generate the public/private
key pair without a password.

![](media/image7.jpeg)

Once this is done save both the public and private keys. The public key
will be saved as `authorized_keys` and the private key will be saved with a filename of
your choosing (ending in .ppk, the PuTTY format). To ensure future
compatibility be sure to click on the conversions option and select
export what you have generated as OpenSSH key, again without a password.
How you have to upload the public key to the VPS. This can be achieved
using PuTTY's SCP tool by issuing the following command from the Windows
console `pscp c:\documents\authorized_keys debian@example.com:/home/debian/.ssh`.
Now in order to connect from Windows using PuTTY you have to select the
private key from within the application.

![](media/image8.jpeg)

At this point you should restart the SSH service with the command
`systemctl restart sshd`. Open another PuTTY session with the
appropriate private key added and attempt to connect to the server. If
all goes well you will be prompted for a username and will be instantly
logged in. Should you not have a private key set you will be given the
following message instead.

![](media/image9.jpeg)

**Password-free Entry (Linux)**

Should you want to do the key generation from Linux and login from Linux
the following steps must be done. This can be done either on the server
or on your own Linux machine. After the step of modifying the SSH
configuration you generate the key pair by running the `ssh-keygen --t
rsa` command. From here you will be prompted where to save the private
key, the default of which is acceptable but I suggest saving the file
name as `authorized_keys`. Enter no password (ultimately your choice).
It will then prompt you where to save the public key, again the default
is sufficient. You then need to ensure that the private key is stored on
your client machine and the public key is stored on the server pursuant
to the previous steps in the `/home/username/.ssh` folders -- moving the
files can be achieved through SCP much like with PuTTY. Change the
permission of the id_rsa file with `chmod id_rsa 700`. You can then
access the server with `ssh SERVERIP --p SSHPORT`.

**Port Knocking**

Now it's time to setup port knocking. This will ensure that along with a
different SSH port number, it will remain blocked in the iptables (to be
setup next) unless a specific sequence of ports are 'knocked'. Only then
the iptables will allow the SSH port to be open to the IP address of the
knocker. So the first step is to simply install the service required
with the command `apt-get install knocked`. Now before its run its
important to modify the default settings, the service even has a
starting flag hidden away in a different file that must be changed prior
to being started for the first time. The configuration file is in
`/etc/knockd.conf` and this is my recommended bespoke settings:

![](media/image10.jpeg)

What these settings achieve is the need to knock in the sequence 1337,
8888 and 1200 within 5 seconds of each other (Can be any sequence and
amount of ports you wish). When this is done the IP address of the one
who knocked will be added to the iptables, allowing exclusive access to
port 88 (or whatever port you set SSH to). It will also only accept TCP
SYN packets. After 15 seconds the IP that was added to the iptables will
be removed -- this won't log you out of the SSH session but it will
prevent you from logging back in without knocking again, this is easier
than the default settings which require the user to knock the SSH port
shut, which can be easily forgotten.

Now access the following file `/etc/default/knockd` and change
`START_KNOCKD` to `1`. Then start the knockd service by running
`systemctl start knockd`. From this point onwards you will not be able
to access the SSH normally. Note that according to the rules, the IP
that knocked is the IP that will be added to the iptables -- remember to
be mindful of this. This will be modified again after WireGuard is
installed. Once this is set up install knockd on another Linux machine
and issue the command `knock --v IP PORT1 PORT2` to open SSH or for
Windows download the application `BwE Port Knocker` (available on my
GitHub) which I developed for this very write-up. This will be vital to
getting back into the VPS. Again, should there be a situation where you
cannot login you still can via the KVM console.

![](media/image11.jpeg)

**IP Tables**

![](media/image12.jpeg)

The next stage, as you can see in the above screenshot is to have
appropriate set of iptables. Creating this is an iterative process,
adding rules, testing them and eventually getting to the point where
your default policy can be DROP (even for output, if you want to be
enthusiastic). The following is a list of iptables I had created for
this project, note that port 88 (my SSH port) is not included, this is 
because the rules are now handled by the knockd service, assuming you 
have set it up properly and it is functional.

![](media/image13.jpeg)

The rules are straight forward and somewhat readable. It allows INPUT
and FORWARD connections which are related and established to continue.
It allows the UDP connection of WireGuard on port 51820. It allows what
will become WireGuard's interface ip 10.0.0.1/24 to allow DNS and also
its interface. It also allows the local host access to port 53 (Unbound
DNS) and port 5353 (DNSCrypt). All of these services are yet to be
installed at this point, thus showing the iptables in one go is not
really descriptive of how it will be implemented. Again, it must be done
iteratively with each installation of each service to ensure
functionality. Once you have a functional iptables it is best to make
them persistent by running the following commands `apt-get install
iptables-persistent` and `systemctl start iptables-persistent`.

**Host Provided Firewall/DDoS Mitigation**

![](media/image14.jpeg)

Within OVH's Manage IP's section there is the ability to change the DDoS
mitigation from Automatic to Permanent. There is also the ability to
customise their firewall. Both of these are highly recommended to add to
your service as even without advertising your server, it will likely
suffer a DDoS attack, if not multiple throughout its lifetime. As you
can see in the graph below, I had no attacks on my VPS and thus the
traffic was not very exciting, until suddenly I was hit with 80,000,000
bytes per second. I then enabled the firewall and changed the mitigation
to Permanent, and after that I had no down-time, though would still
suffer speed drops from time to time.

![](media/image15.png)

Their firewall acts as a giant filter and allows only for the traffic I
have specified to continue through, and in my case I had only enabled
WireGuard's port -- given that I will literally only be using that
single port as it will become the port for the VPN, DNS and SSH. Though
if I need to I could also enable the DNS port, but I have not needed to
thus far.

**Unbound DNS & DNSCrypt**

Unbound DNS, installing this on the VPS allows full ownership over DNS
traffic and can allow us to also install DNSCrypt which will in turn
facilitate DNSSEC and encrypted DNS traffic. Start by issuing the
following commands `apt-get install unbound unbound-host` and 
`curl --o /var/lib/unbound/root.hints https://www.internic.net/domain/named.cache`
this will give you the latest DNS servers available to you. Now you must
modify the default configuration at `/etc/unbound/unbound.conf`.

![](media/image16.jpeg)

I recommend these settings as it gives access to the DNS only to you and
WireGuard and it forwards the requests through to the DNSCrypt service in 
the forward-zone. Now the next thing to install is DNSCrypt-Proxy itself. 
Start by picking an installation directory (I chose just the home directory) 
and then run `wget https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.0.42/dnscrypt-proxy-linux_x86_64-2.0.42.tar.gz`.
Extract it with `tar -xvf dnscrypt-proxy-linux_x86_64-2.tar.gz dnscrypt-proxy`. 
Enter the extracted directory and copy the example configuration with `cp example-dnscrypt-proxy.toml dnscrypt-proxy.toml`. 
Now run the service in a new terminal window with `./dnscrypt-proxy` to ensure 
its functionality. At this stage  you need to modify your systems default resolve 
file, but first back it up  with `cp /etc/resolv.conf /etc/resolv.conf.backup` then 
delete it and make a new one and insert `nameserver 127.0.0.1` and `options edns0`.
Your system will likely try and revert these settings so lock the file with 
`chattr +i /etc/resolv.conf`, note that the `--i` switch will unlock it. 
Now it's best to close the other terminal session that has DNSCrypt running 
and begin modifying the configuration file, `dnscrypt-proxy.toml`.

![](media/image17.jpeg)

This is the configuration that I had used. The server names line can be
removed, should you do this it will result in the DNSCrypt service
probing every available server on start-up and determining the fastest
one based on your location and the rules written below (I recommend
this, but you will likely just get Australian servers regardless). Start
the DNSCrypt in another window again but with the following command
`./dnscrypt-proxy -resolve google.com` if this succeeded you are now
safe to install DNSCrypt as a service with the command `./dnscrypt-proxy
-service install`.

**WireGuard**

Now to finally install WireGuard, this is achieved by issuing `apt-get
install wireguard`. Ensure the service is installed and running by
issuing `modprobe wireguard` and `lsmod | grep wireguard`. The next
step is to generate the key pair, but first change the permission of the
`/etc/wireguard/` directory with `umask 077`. This will ensure that only
the owner is able to read or execute newly-created files. Now the actual
key generation, issue `wg genkey | tee privatekey | wg pubkey >
public key`.

From this point on you can cheat by going to https://wireguardconfig.com/ and
using a generated configuration. But is not that difficult to set it up yourself, 
start with creating the following file `/etc/wireguard/wg0.conf` and adding your 
own private key and a client's public key to the following configuration in the 
image below. Then save it and modify its permissions with `chmod 600 /etc/wireguard/wg0.conf`. 
Then remember to delete the keys you have generated earlier. Subsequent clients are 
added below each other with the same formatting, to then remove a user you issue 
`wg set wg0 peer PUBLICKEY remove` or modify the wg0.conf manually. To load a
configuration (to add another client for example) without resetting the
service run `wg addconf wg0 (wg-quick strip wg0)`.

![](media/image18.jpeg)

Now ensure that your system can accommodate IP forwarding by editing
`/etc/sysctl.conf` and adding `net.ipv4.ip_forwarding=1` and `net.ipv6.conf.all.forwarding=1`. 
Once this is done run `sysctl --p` to load your newly edited configuration. 
Now you can finally start WireGuard with `wg-quick up wg0` and confirm its running with 
`wg showall`.

![](media/image19.jpeg)

Connect to the VPS via WireGuard to finally confirm that you are indeed
part of the server's LAN, this is important for a final security measure. If all is well make
WireGuard start at boot with `systemctl enable wg-quick@wg0`. You can confirm that there is indeed
encrypting traffic by issuing `tcpdump --n --X --I eth0 host YOURSERVERIP` and looking for WireGuard's magic header identifier in each packet `0400 0000`.

![](media/image20.jpeg)

**Windows Client Side Setup**

Running the official WireGuard client for Windows, you are able to
create a new tunnel with a few clicks. Within the client click on the
down arrow next to `Add Tunnel` and create a new tunnel. You will be
presented with a form with part of a configuration along with a
pre-generated public and private key, as seen below.

![](media/image21.png)

Use this information to build your client configuration as per the
second screenshot above. Once this is done, click `Activate` and you
will be connected!

**Linux Client Side Setup**

From the client's perspective setting up WireGuard is very similar, it
starts with the following commands 'apt install wireguard resolvconf' to
install the service and to ensure the DNS functionality, then to confirm
its running run `lsmod | grep wireguard`. Now you have to generate a
private and public key, the private key stays on your system and the
public key is given to the server so you can become a peer. This is done
with `wg genkey | tee privatekey | wg pubkey > publickey`. Then
change the permission of the `/etc/wireguard/` directory with `umask
077`. Create now your own WireGuard configuration file in
`/etc/wireguard/wg0.conf` and insert the following.

![](media/image22.jpeg)

The AllowedIPs setting can be changed to permit LAN access or can
restrict to a specific IP address. Once this is done change the
permissions of the file with `chmod 600 /etc/wireguard/wg0.conf` and
then ensure that your system can accommodate IP forwarding by editing
`/etc/sysctl.conf` and adding `net.ipv4.ip_forwarding=1` and
`net.ipv6.conf.all.forwarding=1`. Now you can start the connection with
`wg-quick up wg0` and confirm its running with `wg show` and finally, to
ensure it starts at boot run `systemctl enable wg-quick@wg0`.

**Additional Security Post Installation**

**SSH via WireGuard (With Knocking)**

Port knocking is great, but why allow anybody from any IP address to
knock at all? Why not limit the knocks to those already on the WireGuard
network, this way you can ensure that only those you can trust can even
begin the knocking process. This is achieved by simply modifying the
knockd configuration file, as per above, but changing the interface to wg0.
This is done by simply adding `interface = wg0` in the options section.
If you wanted to be even more paranoid, you could set up an additional
WireGuard interface specifically to access SSH and use that as the
knocking interface, this would allow sharing of the WireGuard VPN access
but also ensuring your own secure access on a different interface and IP
address, solely for SSH.

**Connection Profiling**

To hide the fact, or at least aid in the fact you are using a VPN/Tunnel
and also to ensure that the connection between the VPS's eth0 interface
and wg0 interface do not fragment UDP packets I recommend changing the
Maximum Transmission Units (MTU). As standard, the eth0 interface will
have an MTU of 1500 and wg0 will have 1420, which is also the default of
IPSec. Should a website attempt to fingerprint your connection it will
be possible for it to know that you are indeed on a tunnel of sorts.
Changing the wg0 interface MTU to 1500 will match the eth0, masking its
identity and also ensuring that the UDP packets do not become fragmented
given that they will both share the same maximum. This can be done by
issuing the command 'ifconfig wg0 mtu 1500 up' and can then be confirmed
with `netstat --i`. It can be made permanent by adding the MTU value
within the interfaces file.

![](media/image23.jpeg)


![](media/image24.jpeg)

**SSH Honeypot**

I decided to install a medium interaction SSH honeypot as this service
will likely be a target for hackers and I was curious to see the
passwords used and simply the amount of attacks I would get on a daily
basis. Now doing this will open a false SSH port, given that the real
one is on port 88 and only accessible through WireGuard this is actually
quite safe to run. There were a lot of prerequisites to install prior to
the actual honeypot and these were as follows:

-   `sudo apt-get install python-minimal`

-   `sudo apt-get install python-pip`

-   `sudo apt-get install build-essential`

-   `sudo apt-get install default-jre`

-   `sudo pip install \--upgrade pip`

-   `sudo pip install setuptools`

-   `sudo pip install pyasn1 pyasn1-modules`

-   `sudo pip install virtualenv`

-   `sudo pip install pycrypto`

-   `sudo pip install virtualenv`

-   `sudo pip install twisted`

-   `sudo pip install cryptography`

-   `sudo pip install tzlocal`

-   `sudo pip install bcrypt`

Once this was done I cloned the git repository of XSweet, the SSH
Honeypot. This was done with the command `git clone https://github.com/techouss/xsweet.git`.
I did this within the home directory as I felt this would be an adequate
location, given this is also where I installed DNSCrypt. Now to run this
on port 22 I had to open it along with port 2222, the actual port that
the honeypot will be running on. I then had to forward 22 to 2222 to
ensure that the honeypot would function. This was done with the
following commands:

-   `sudo iptables -A PREROUTING -t nat -p tcp --dport 22 -j REDIRECT --to-port 2222`

-   `sudo iptables -A INPUT -p tcp --dport 2222 -j ACCEPT`

-   `-sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT`

I tested its functionality by running it with `python xsweet.py` and
opening an SSH connection on port 22 to my server. It indeed functioned,
but I could not keep it running this way as it would consume the
session. I had to run it in the background and so to do this I ensured
that the `xsweet.py` file was an executable by issuing the command
`chmod +x xsweet.py`. I then made it run in the background by issuing
the command `nohup python /xsweet/xsweet.py &`, to confirm it was
running I ran `ps aux | grep xsweet` and tested the connection to the
fake SSH again.

![](media/image25.jpeg)

In the above screenshot you can see a list of all the failed username
and password's which were attempted against the honeypot. These are
stored within a `victims` folder along with stored sessions.

![](media/image26.jpeg)

In this screenshot above you can see how the fake SSH session allows for
interaction and false outputs, of which are recorded. This would also
include files that are uploaded/downloaded to the server, this way you
can forensically examine potential malware/payloads etc.

**Other Considerations**

You must be aware of the Autonomous System Number (ASN) that is assigned
to your server IP address when you buy your VPS. Mine for example is
AS16276, belonging to OVH SAS in Canada, its purpose is for paid VPN,
hosting and 'good' bots. It has however 40,382 active spam IP addresses
out of a total of 381,412 -- that's 10.5% of the entire network
consisting of spammers, this is not good and was likely the reason
behind having multiple DDoS attacks when my IP was still new.

![](media/image27.jpeg)

What this means is that should you indeed use your server as a VPN for
daily use, you may find that you have been banned from websites you have
never visited, this is simply because the website has chosen to ban not
your IP address but your ASN entirely. Luckily the IP address I have is
unique to my account and is not shared, this situation would be worse on
a commercial VPN provider, where allocated shared IPs can be banned in
global black lists due to spamming and other illegal activity. My ASN is
also not part of the Spamhaus Project ASN-DROP list; if it were then I
would certainly not continue using my hosting provider.
