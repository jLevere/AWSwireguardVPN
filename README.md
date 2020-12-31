# AWSwireguardVPN
How to set up a wireguard vpn relay in aws lightsail


#I am using windows subsystem for linux 2 for all of my local work as I am most comfortable using bash for things. 
#take the downloaded private key from creating the lightsail instance and move it into the .ssh directory in wsl 
{user}@{WSL}:~$ rsync /mnt/c/users/{home}/downloads/{keyName}.pem ~/.ssh 
{user}@{WSL}:~/.ssh$ chmod 600 {keyName}.pem

#going back to lightsail, we are going to asign a static ip to our ubuntu instance

#make sure that our ssh-agent is running
{user}@{WSL}:~$ eval "$(ssh-agent)"
#load our new private key into the ssh-agent
{user}@{WSL}:~$ ssh-add .ssh/{keyName}.pem
