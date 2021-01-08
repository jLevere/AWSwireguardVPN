# WireGuard AWS Lightsail relay setup and configuration. 

The goal of this project was three fold. First to allow distributed devices behind NAT and firewalls that I do not control to access each other.  Second, to encrypt traffic on potentially malicious or insecure networks.  And third, to do the above in an inexpensive and streamlined manner.  

To do this, I am using a small AWS LightSail Ubuntu instance as a relay node.  Devices use the VPN protocol WireGuard to connect up to the publicly accessible instance which forwards traffic between the different connections as well as the internet. While this may have more lag than a direct peer to peer connection, it has the advantage of working with most NAT configurations and many firewalls.  

I chose WireGuard for its ease of implementation, multi-platform support, peer structure, small overhead and security. I also have been itching to find a use for it for a while now, so there is that too.

This is a setup guide for my Ubuntu AWS Lightsail instance that runs a WireGuard VPN relay.  I also cover connecting devices to it at the end. 

While it can be difficult to find understandable information about the WireGuard configurations online, the man files are excellent recourses and have helpful examples. 

## Setting up AWS
After creating an Amazon aws account, go to the lightsail home. 
Chose `create instance`
Then down the page I am using:
- Linux
- Ubuntu 20.04 LTS
- Added an ssh key and downloaded it
- The 3.5usd per month instance plan
- Named the instance
- And created the instance. 
After lettings it spin up, select the instance and go into the networking settings. 
Remove all the current firewall rules and add rule for the application ssh.  Allow access by lightsail web-console and restrict it to your public ip. To ensure that I have the right public ip I just Googled `what is my ip` just in case there is some kind of address mapping going on.  
I am also going to go ahead and add
I am using windows subsystem for Linux 2 for all of my local work as I am most comfortable using bash for things. 

- Take the downloaded private key from creating the lightsail instance and move it into the .ssh directory in WSL 
{user}@{WSL}:\~$ `rsync /mnt/c/users/{home}/downloads/{keyName}.pem ~/.ssh` 
- set the perms for the key so it cant be read and ssh allows it.
{user}@{WSL}:~/.ssh$ `chmod 600 {keyName}.pem`


Going back to lightsail, we are going to assign a static public ip to our ubuntu instance

- Make sure that our ssh-agent is running
{user}@{WSL}:~$ `eval "$(ssh-agent)"'`

- Load our new private key into the ssh-agent
{user}@{WSL}:~$ `ssh-add .ssh/{keyName}.pem`


Now we will try to connect to the aws instance for the first time.  The default username for ubuntu is "ubuntu".
{user}@{WSL}:~$ `ssh ubuntu@{instance address}`

*The rest of these commands are in the AWS instance unless otherwise specified.*


I am going to go ahead and update the instance right off the bat. This should only take a few minuets with the blazing fast aws internet.
```
sudo apt-get update
sudo apt upgrade
```

I am going to reboot just to make sure everything is stable
```
sudo reboot
```

Next for security sake we are going to make a new user and disable logins to the root account and default ubuntu account over ssh.

- Add a new user `level`
```
sudo adduser level
```

- Give `level` sudo membership 
```
sudo usermod -aG sudo level
```

- And lastly, copy the ssh keys over to the new user
```
sudo rsync --archive --chown=level:level ~/.ssh /home/level
```


*(make sure you do no leave the trailing slash on the file name with rsync.  In rsync syntax, that means you want the contents of the folder but not the folder itself)*

- End ubuntu processes
```
sudo pkill -u ubuntu
```

- Delete the user and home folder
```
sudo deluser --remove-home ubuntu
```

Now we are going to change the ssh login settings in the sshd_config file. 

- Open the sshd config file in your favorite editor.  I am using nano for simplicity. 
```
sudo nano /etc/ssh/sshd_config
```



- set `PermitRootLogin` to `no`  *you can use ctl-w (where is) to search for the entry*
> PermitRootLogin no

And add this line to the end of the file with the names of the users you want to be able to use ssh to login.  I have removed the default ubuntu and put my new user level in its place. 
> AllowUsers level

Save the file with ctl s then close with ctl x. 


- Restart the service for the config to take effect
```
sudo service ssh restart
```

At this point I like to open a new terminal and attempt to login as the new user simultaneously as the current.  Just in case something has gone wrong, I can still correct it while logged in.

I am also going to change my hostname to something other than the interal ip address just to make it easier to read.

- set this value to whatever you want to call your machine.  I am using `relay`
```
sudo nano /etc/hostname
```
- also check here and make sure it matches your new hostname
```
sudo nano /etc/hosts
```

Then reboot for the change to take effect
`sudo reboot`

*you can query the host name and other general information with the command*
```
hostnamectl
```



## Next we are going to set up the firewall iptables 

We are going to set up the input rules first.  INPUT is the chain that controls what things can make connections into the machine from the outside. What you normally think of as a firewall. 

Some general iptables commands:
`sudo iptables -L`  to list the current iptables configuration
`sudo iptables -nvL` list a more detailed version of above with packet counters.  This is very helpful for troubleshooting and just getting a general idea of what rules are actually being used. 

Generally it is clearest to write allow rules then drop anything that doesn't match the rules.
iptables works from top to bottom.  Packets are put down the chain until they match with a rule.

The first rule we are going to add is a connection tracking rule.  This accepts packets from connections that originated from our machine
- enter the command
```
sudo iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```
 
 Next we are going to allow all localhost traffic.  This is important for some types of dns resolution and just makes things easier.  You can set more restictive versions of this rule if you want. 
- enter the command 
```
sudo iptables -A INPUT -i lo -j ACCEPT
```

Next we are going to allow ssh from anywhere.  The AWS firewall will help protect the machine from too much random traffic.
- enter the command
```
sudo iptables -A INPUT -i eth0 -p tcp -m state --state NEW,ESTABLISHED -m tcp --dport 22 -j ACCEPT # accept incoming ssh from anywhere
```

Then we are going to allow, from anywhere, connections to the port we are going to use for the vpn wireguard.  You can use any high udp port, I just picked 62013
- enter the command
```
sudo iptables -A INPUT -i eth0 -p udp -m state --state NEW,ESTABLISHED -m udp --dport 62013 -j ACCEPT 
```

Next we are going to allow icmp ping traffic from the internal subnet we will assign to our vpn network.  I am using 10.1.0.0/24 for my internal network.  We will set this subnet up later in wireguard interface configuration.
```
sudo iptables -A INPUT -i wg0 -p icmp --icmp-type echo-request -j ACCEPT
```

And lastly we are going to set a rule to drop anything that doest match the rules above.
- enter the command
```
sudo iptables -A INPUT -j DROPrules
```
*I use drop rules rather than drop defaults so that I dont lock myself out as easily.*

Iptables rules are deleted on reboot so we need to install something to make them persist.  I am using the normal netfilter-persistent with the iptables plug-in. 
- install netfilter-persistent
```
sudo apt install netfilter-persistent
```
- and the iptabels plug-in
```
sudo apt install iptables-persistent
```

Netfilter will usually run by default on install but I like to run this to just make sure. 
```
sudo systemctl enable netfilter-persistent
```

- run the command to actually save the iptables rules is
```
sudo netfilter-persistent save
```

Although this might seem kinda tedious having to save the rules manually, the advantage is that if you change a rule and lock yourself out, you can just power the machine off and it will revert back the the rules that you were using last.  I have managed this several times playing around, so it does happen. 

I also like to add just a general ipv6 drop rule to my firewall as I dont use it internally anywhere. ipv6 is handled by a different program called, somewhat unsurprisingly, ip6tables.  It uses the same syntax and netfilter saves it so its not really that much different.
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


## Now on the the main subject: the vpn ##

First some preliminary stuff.

On some newer os versions resolvconf is replaces with resolvectl so set up a symbolic link so wireguard doesnt get confused.
```
sudo ln -s /usr/bin/resolvectl /usr/local/bin/resolvconf
```

Now we are actually going to install wireguard.
- install wireguard
```
sudo apt install wireguard
```

Now that we have wireguard installed we are going to generate the keys it uses.  First we are going to set up our file permissions with umask to prevent other people from reading the keys.
```
umask 077
```

- Then I like to create several folders for organization
```
mkdir wg
mkdir wg/keys
mkdir wg/clients
```

- Generate the keys using the wireguard utility genkey
```
wg genkey | tee wg/keys/{YOUR HOSTNAME}_private_key | wg pubkey > wg/keys/{YOUR HOSTNAME}_public_key
```

Then we will generate a set of keys for each of the client devices we want to connect to the relay.  I am going to do my phone and laptop.
```
wg genkey | tee wg/keys/phone_private_key | wg pubkey > wg/keys/phone_public_key
wg genkey | tee wg/keys/laptop_private_key | wg pubkey > wg/keys/laptop_public_key
```

Now we need to make our config files, one for each device.  Or more precisely, one for each end of the tunnel.  When doing this I typically use tmux or multiple ssh connections to cat the keys in one and paste them into the other.

Lets start with the relays config.
- Create the interface configuration. 
```
sudo nano /etc/wireguard/wg0.conf
```

The name of the config file is the name of the interface.  This is the config that I am using. Feel free to modify it for your needs, its quite flexable. 

This block is for the relay side of the tunnels
```
[Interface]
	# this specifies the address of the server and what subnet it allows its clients to be in
	Address = 10.1.0.1/24

	# this tells wireguard not to delete the config when the tunnel is taken down.
	SaveConfig = true

	# wireguard can execute things before putting the interface up, after up, before down and after down.  I am using it to add the iptables forwarding rules to direct traffic in and out of the tunnel.
	PostUp = iptables -A FORWARD -i %i -o %i -m conntrack --ctstate NEW -j ACCEPT; iptables -t nat -A POSTROUTING -s 10.1.0.0/24 -o eth0 -j MASQUERADE;

	# this line turns on ipv4 forwarding in the kernal
	PostUp =  echo 1 | tee /proc/sys/net/ipv4/ip_forward; sysctl -p;


	# then once the interface is down take the iptables rules off and turn off forwarding
	PostDown = iptables -D FORWARD -i %i -o %i -m conntrack --ctstate NEW -j ACCEPT; iptables -t nat -D POSTROUTING -s 10.1.0.0/24 -o eth0 -j MASQUERADE;
	PostDown =  echo 0 | tee /proc/sys/net/ipv4/ip_forward; sysctl -p;

	# next is the udp port that wireguard should listen for new connections on
	ListenPort = 62013

	# then the private wireguard key for the relay
	PrivateKey = (hidden)



#After that block is written we will do one for each one of the devices you want to connect which we call peers.


[Peer] # phone
	# the public key of the peer
	PublicKey = (hidden)

	# then the ip address of the tunnel for routing
	AllowedIPs = 10.1.0.2/32  # 32 because there is only one`
	
[Peer] # laptop
	PublicKey = (hidden)
	AllowedIPs = 10.1.0.3/32
```


And that is all for the relay config file  save it with ctrl s, ctrl x

Then we need to make similar config files that will be passed to client (or peer in wireguard terms) devices.

Starting with the phone config.
- make the file and open it
```
sudo nano wg/clients/phone.conf
```

The config looks very similar but we are going to add a little more info to it.  This interface is the local side.
```
[Interface]
	# the local side address of the interface
	Address = 10.1.0.2/32  # 32 cause its the single address

	# the private key for the client to use
	PrivateKey = (hidden)
	```

then the peer, which from the client perspective is the relay

[Peer]
	# the public key of the relay
	PublicKey = (hidden)

	# the public ip address of the relay.  wireguard needs to know one side of the connection
	Endpoint = (relay public ip address):(port number)

	# ip addresses that are routed into the tunnel
	AllowedIPs = 0.0.0.0/0, ::/0

	# a keep alive signal to keep firewall ports open, in seconds
	PersistentKeepalive = 25
```

And thats it for that config.  The laptop will be pretty much the same but with a different ip and keys.
Next we need to get the configs to their devices.  Mobile devices an easy way to do this is with QR images.

We will use qrencode to generate QR codes out of the phone config.

- install qrencode
```
sudo apt install qrencode
```

- then we make the QR code
```
qrencode -o wg/clients/phone.png -t png < wg/clients/phone.conf
```

Once the config files are ready to go, we need to get them to the devices.  I am using scp to copy them onto my local machine and then display the qr code 
for the phone, and load the laptop one into the wireguard software.

scp has the syntax: `scp sourceDomain:sourceFile destFile`  And uses ssh for the connection so make sure you keys is loaded up on your local machine.

- on your local machine execute
```
scp level@{relay public ip}:~/wg/clients/phone.png /mnt/c/users/winlap1/desktop
```
- again for the laptop config
```
scp level@{relay public ip}:~/wg/clients/laptop.png /mnt/c/users/winlap1/desktop
```

You can go ahead and load the configs into the clients.  In the wireguard app select add interface and scan from QR code.  Then scan the QR code for the laptop you can add a tunnel and select the config file we copied.

## Now for the big test.  Putting the relay's wireguard interface up!

We are going to use wg-quick for this.  Say we want the interface up, and which one (the name of the config file).
```
sudo wg-quick up wg0
```

If everything goes well you should see somthing like this return.
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

You can check the status of wireguard with this command
```
sudo wg
```
There a few other commands for status info, but without arguments it defaults to show everything.

## Now lets try to connect with the phone

In the app start the tunnel!

You can watch the logs on the phone to see what is happening and if it is connected.


## End notes

And that is pretty much it.  I have had my instances online for a few months now and they are still stable and forwarding.  There are alot of things in the set up that can be scripted, particularly the writing of the WireGuard config files.

The next thing I am looking at is implementing some webRTC like STUN system to attempt and form direct peer to peer connections between devices when possible to reduce both latency and traffic in general on the AWS instance.  If the connections are not possible then it falls back to a forwarded connection.

This project was largely inspired by **[TailScale]**(https://github.com/tailscale/tailscale) which I have been enamored with since I first used it.  This set up replicates pieces of the TailScale network but without as many restrictions or 2fa. 

Several sources that I found helpful in this project were:

[Unofficial WireGuard Doc](https://github.com/pirate/wireguard-docs)

[WireGuard routing](https://www.wireguard.com/netns/)

[WireGuard setup](https://www.wireguard.com/quickstart/)

[Iptables Tricks](https://opensource.com/article/18/10/iptables-tips-and-tricks)

[Mullvad Raspberry pi wireguard setup](https://mullvad.net/en/help/wireguard-and-mullvad-vpn/)

