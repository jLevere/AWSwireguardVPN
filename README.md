# WireGuard AWS Lightsail relay setup and configuration. 

This is a walk through of my Ubuntu AWS Lightsail instance that runs a WireGuard VPN relay.  The goal of this project was two fold. First to be able to easily access distributed devices behind NAT and firewalls that I may not control.  And second, to encrypt traffic on potentially malicius or insecure networks. 

Created lightsail ubuntu 20.04 instance and set up access with a new named key. 
chose the lowest priced instance
removed all the current firewall rules
added firewall rule for ssh, allowed lightsail console access, and using "google what is my ip" added a ip resictiction for ssh connections.

I am using windows subsystem for linux 2 for all of my local work as I am most comfortable using bash for things. 
take the downloaded private key from creating the lightsail instance and move it into the .ssh directory in wsl 
```
{user}@{WSL}:~$ rsync /mnt/c/users/{home}/downloads/{keyName}.pem ~/.ssh 
{user}@{WSL}:~/.ssh$ chmod 600 {keyName}.pem
```

going back to lightsail, we are going to asign a static ip to our ubuntu instance

make sure that our ssh-agent is running
```
{user}@{WSL}:~$ eval "$(ssh-agent)"
#load our new private key into the ssh-agent
{user}@{WSL}:~$ ssh-add .ssh/{keyName}.pem
```

then we will try to connect to the aws instance for the first time.  the default username for ubuntu is "ubuntu".
```
{user}@{WSL}:~$ ssh ubuntu@{instance address}
```

im going to goahead and update the instance right off the bat. This should only take a few minuets with the blazing fast aws internet.
```
{ubuntu}@{ip-172-26-11-251}:~$ sudo apt-get update
{ubuntu}@{ip-172-26-11-251}:~$ sudo apt upgrade
```

them im am going to reboot just to make sure everything is stable
```
{ubuntu}@{AWS}:~$ sudo reboot
```

next for security sake we are going to make a new user and disable logins to the root account and default ubuntu account.  


```
adduser level
```

```
usermod -aG sudo level
```

```
rsync --archive --chown=level:level ~/.ssh /home/level
```
meaning (copy --recursivly --giving level ownership, the .ssh folder to /home/level.)

(make sure you dont leave the trailing slash on the file name.  In rsync syntax that means you want the contents of the folder but not the folder itself)

ends ubuntu proccesses
```
sudo pkill -u ubuntu
```

delete the user and home folder
```
sudo deluser --remove-home ubuntu
```

```
sudo nano /etc/ssh/sshd_config
```

you can use ctl-w (where is) to search for permitrootlogin
> set PermitRootLogin to no

and add this line to the end of the file with the names of the users you want to be able to use ssh to login.  I have removed the default ubuntu and put my new user level in its place. 

> AllowUsers level

then restart the service for the config to take effect
```
sudo service ssh restart
```

at this point I like to open a new terminal and attempt to login as the new user simultaniusly as the current
just incase something has gone wrong I can still correct it while logged in.

I am also going to change my hostname to something other than the interal ip adress just to make it easier to read
set this value to what ever you like
```
sudo nano /etc/hostname
```
also check here and make sure it matches your new hostname
```
sudo nano /etc/hosts
```
Then reboot for the change to take effect

you can query the host name and other general information with the command
```
hostnamectl
```

## Next we are going to set up the firewall iptables 

we are going to set up the input rules first, or the chain that controls what things can make connections into the 
machine from the outside. 

Some general iptables commands are 
to list the current iptables configuation
```
sudo iptables -L  
sudo iptables -nvL
```
list a more detailed version of above with packet counters.  This is very helpfull for 
troubleshooting and just getting a generall idea of what rules are actuall being used. 

generally it is clearest to write allow rules then drop anything that doesnt match the rules
iptables works from top to bottom.  Packets are put down the chain until they match with a rule.

the first rule we are going to add is a connection tracking rule.  This accepts packets from connections that orginated from our machine
```
sudo iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT  # conntrack active connections
```
 next we are going to allow al localhost traffic.  This is important for some types of dns resolution and just makes things easier. 
```
sudo iptables -A INPUT -i lo -j ACCEPT	# accept localhost
```
next we are going to allow ssh from anywhere.  The AWS firewall will help protect the machine from too much random traffic.
```
sudo iptables -A INPUT -i eth0 -p tcp -m state --state NEW,ESTABLISHED -m tcp --dport 22 -j ACCEPT # accept incoming ssh from anywhere
```
then we are going to allow from anywhere connections to the port we are going to use for the vpn wireguard.  You can use any high udp port, I just picked 51802
```
sudo iptables -A INPUT -i eth0 -p udp -m state --state NEW,ESTABLISHED -m udp --dport 51802 -j ACCEPT 	# accept incoming wireguard from anywhere
```
next we are going to allow icmp ping traffic from the internal subnet we will asign to our vpn network.  I am using 10.1.0.0/24 for mine.
#sudo iptables -A INPUT -i wg0 -p icmp --icmp-type echo-request -j ACCEPT 	# accept pings from wg0 network (move this to the wg0.conf file)
and lastly we are going to set a rule to drop anything that doest match the rules above.  
```
sudo iptables -A INPUT -j DROP 	# drop anything that doesnt match the above rules
```
 I use drop rules rather than drop defaults so that I dont lock myself out as easily. 

Iptables rules are deleted on reboot so we need to install something to make them persist
we are going to use
```
sudo apt install netfilter-persistent
```
and the iptabels plugin
```
sudo apt install iptables-persistent
```
netfilter will usually run by default on install but I like to run this to just make sure
```
sudo systemctl enable netfilter-persistent
```
and the command to actually save the iptables rules is
```
sudo netfilter-persistent save
```

 altho this might seem kinda tedius having to save the rules manually, the advantage is if you change a rule and lock yourself out, you can just power the
 machine off and it will revert back the the rules that you were using last.  I have managed this several times playing around, so it does happen. 

I also like to add just a general ipv6 drop rule to my firewall as I dont use it internally anywhere. 
ipv6 is handled by a different program called, somewhat unsuprisingly, ip6tables.  It uses the same syntax and netfilter saves it so its not really any different.
```
sudo ip6tables -A INPUT -j DROP
sudo ip6tables -A FORWARD -j DROP
```

```
sudo netfilter save
```

you can check and see how much ipv6 traffic you are dropping with
```
sudo ip6tables -nvL
```


###############################################

#Transfer config files to clients
#Qrencode
#scp

## now on the the main subject: the vpn ##

first some preliminary stuff
On some newer os versions resolvconf is replaces with resolvectl so set up a symbolic link so wireguard doesnt get confused
```
sudo ln -s /usr/bin/resolvectl /usr/local/bin/resolvconf
```

 then we are going to install wireguard
```
sudo apt install wireguard
```

now that we have wireguard installed we are going to generate the keys it uses
first we are going to set up our file permsions with umask to prevent other people from reading they keys
```
umask 077
```
then I like to create several folders for orginisation
```
mkdir wg
mkdir wg/keys
mkdir wg/clients
```
then we will generate the keys using the wireguard utility genkey
```
wg genkey | tee wg/keys/{YOUR HOSTNAME}_private_key | wg pubkey > wg/keys/{YOUR HOSTNAME}_public_key
```
then we will generate a set of keys for each of the client devices we want to connect to the relay
I am going to do my phone and laptop
```
wg genkey | tee wg/keys/phone_private_key | wg pubkey > wg/keys/phone_public_key
wg genkey | tee wg/keys/laptop_private_key | wg pubkey > wg/keys/laptop_public_key
```

now we need to make our config files, one for each device.  Or more precisely, one for each end of the tunnel.
when doing this I typically use tmux or multiple ssh connections to cat the keys in one and paste them into the other

lets start with the relays config

```
sudo nano /etc/wireguard/wg0.conf
```
the name of the config file is the name of the interface

this is the config that I am using. Feel free to modify it for your needs, its quite flexable. 

this block is for the relay side of the tunnels
```
[Interface]
	# this specifies the address of the server and what subnet it allows its clients to be in
	Address = 10.1.0.1/24
	# this tells wireguard not to delete the config when the tunnel is taken down.
	SaveConfig = true
	# wireguard can execute things before putting the interface up, after up, before down and after down
	# I am using it to add the iptables forwarding rules to direct traffic in and out of the tunnel
	PostUp = iptables -A FORWARD -i %i -o %i -m conntrack --ctstate NEW -j ACCEPT; iptables -t nat -A POSTROUTING -s 10.1.0.0/24 -o eth0 -j MASQUERADE;
	# you can add as many as you want and they are just run in order
	# this line turns on ipv4 forwarding in the kernal
	PostUp =  echo 1 | tee /proc/sys/net/ipv4/ip_forward; sysctl -p;
	```


	```
	# then once the interface is down take the iptables rules off and turn off forwarding
	PostDown = iptables -D FORWARD -i %i -o %i -m conntrack --ctstate NEW -j ACCEPT; iptables -t nat -D POSTROUTING -s 10.1.0.0/24 -o eth0 -j MASQUERADE;
	PostDown =  echo 0 | tee /proc/sys/net/ipv4/ip_forward; sysctl -p;
	# next is the udp port that wireguard should listen for new connections on
	ListenPort = 51802
	# then the private wireguard key for the relay
	PrivateKey = (hidden)
	```


after that block is written we will do one for each one of the devices you want to connect which we call peers

```
[Peer] # phone
	# the public key of the peer
	PublicKey = (hidden)
	# then the ip address of the tunnel for routing
	AllowedIPs = 10.1.0.2/32  # 32 because there is only one`
	
`[Peer] # laptop
	PublicKey = (hidden)
	AllowedIPs = 10.1.0.3/32
	```


and that is all for the relay config file  save it with ctrl s, ctrl x

then we need to make simmilar config files that will be passed to client (or peer in wg terms) devices
starting with my phone
```
sudo nano wg/clients/phone.conf
```


the config looks very similar but we are going to add alittle more info to it
this interface is the local side 
```
[Interface]
	# the local side address of the interface
	Address = 10.1.0.2/32  # 32 cause its the single address
	# the private key for the client to use
	PrivateKey = (hidden)
	```

then the peer, which from the client perspective is the relay
```
[Peer]
	# the public key of the relay
	PublicKey = (hidden)
	# the public ip address of the relay.  wireguard needs to know one side of the connection
	Endpoint = (relay public ip address):(port number)
	# ip addresses that are routed into the tunnel
	AllowedIPs = 0.0.0.0/0, ::/0 # for full tunnel and grabs ipv6 too.  which is just dropped by the firewall rules to stop leaks
	# a keep alive signal to keep firewall ports open, in seconds
	PersistentKeepalive = 25
	```

and thats it for that config.  The laptop will be pretty much the same but with a different ip and keys
next we need to get the configs to their devices.  Mobile devices an easy way to do this is with QR images

we will use qrencode to generate QR codes out of the phone config
```
sudo apt install qrencode
```

then we make the QR code
```
qrencode -o wg/clients/phone.png -t png < wg/clients/phone.conf
```

once the config files are ready to go, we need to get them to the devices.
I am using scp to copy them onto my local machine and then display the qr code 
for the phone, and load the laptop one into the wireguard software

scp has the syntax: scp sourceDomain:sourceFile destFile
and uses ssh for the connection so make sure you keys is loaded up
on your local machine
```
scp level@{relay public ip}:~/wg/clients/phone.png /mnt/c/users/winlap1/desktop
```
and again for the laptop config
```
scp level@{relay public ip}:~/wg/clients/laptop.png /mnt/c/users/winlap1/desktop
```

you can go ahead and load the configs into the clients.  In the wireguard app select add interface and scan from QR code.  then scan the QR code
for the laptop you can add a tunnel and select the config file we copyed

now for the big test.  Putting the relay wireguard interface up
we are going to use wg-quick for this.  say we want the interface up, and which one (the name of the config file). 
```
sudo wg-quick up wg0
```
if everything goes well you should see somthing like this 
```
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.1.0.1/24 dev wg0
[#] ip link set mtu 8921 up dev wg0
[#] iptables -A FORWARD -i wg0 -o wg0 -m conntrack --ctstate NEW -j ACCEPT; iptables -t nat -A POSTROUTING -s 10.1.0.0/24 -o eth0 -j MASQUERADE;
[#] echo 1 | tee /proc/sys/net/ipv4/ip_forward; sysctl -p;
1 
net.ipv4.ip_forward = 1
```

you can check the status of wireguard with 
```
sudo wg
```
there a few other commands but without arguments it defaults to show everything

now lets try to connect with the phone
in the app start the tunnel!
you can watch the logs on the phone to see what is happening and if it is connected
