# Simple Port Knocking
With a simple topology consists of a Cloud and Router, I learned about how port knocking works. Port knocking is a technique to control the access of network connection by closing some ports until a client sends specific request to access them.

<img width="293" height="399" alt="image" src="https://github.com/user-attachments/assets/2bf8fa40-ecbb-4df1-86df-58f2fcc8a5ae" />  

This is the topology I use. The purpose of this exercise is to protect SSH access of Mikrotik from the client (Cloud, Linux).

## Softwares
- Virtualbox
- Linux Ubuntu  
I use Linux Ubuntu 24.04.3 that runs through Virtualbox. 
- GNS3  
I use GNS3 2.2.55.
- Mikrotik CHR  
I use CHR 7.20.5.
- Winbox (optional)  
No need to use if you're more comfortable with CLI.

## Steps
1. Configure the Router  
I use TAP Interface to connect the Cloud and Router, so I add the Router's IP address manually.
2. Check if the connection between Linux (Cloud) and Router works  
Before I go to the main goals, I ensure the default configuration works by doing PING test between Router to Linux and vice versa.
3. Set up Firewall rules  
On this exercise, input filter is used because we need to control the connection that comes. As a starter, let's accept the known and accepted connection first.
> /ip firewall filter add action=accept chain=input connection-state=established,related

Next, we create the rules for knocking itself. This is my scenario:  
<img width="191" height="354" alt="image" src="https://github.com/user-attachments/assets/89882f93-02a5-4652-9a9a-e766ba318fc9" />

When knock 3 performs, SSH should be allowed. These are the rules:  
KNOCK 1:
> /ip firewall filter add action=add-src-to-address-list chain=input protocol=tcp dst-port=7000 address-list=testknock1 timeout=20

When this performs, the router detects the first knock, so it takes an action to add the IP coming from first knock to the address list (testknock1). The IP is available for 20 seconds.

KNOCK 2:
> /ip firewall filter add action=add-src-to-address-list src-address-list=testknock1 chain=input protocol=tcp dst-port=7001 address-list=testknock2 timeout=20

If knock 1 success, router can detect knock 2. Same with knock 1, the IP will be added to the address list (testknock2) and is available for 20 seconds too. However, we should add source address list to detect the IP of knock 1 because the knocks must be in sequence. Otherwise, will be failed to access SSH.

KNOCK 3 (SSH-ALLOWED):
> /ip firewall filter add action=add-src-to-address-list src-address-list=testknock2 chain=input protocol=tcp dst-port=8000 address-list=testknock3 timeout=60

If knock 2 success, it continues to knock 3. Just like before, the IP will be added to its address list and will be 'combined' with the IP of other knocks to see if its in sequence or not.

After all knocks performed, we should allow the connection of SSH based on the knocks.
> /ip firewall filter add action=accept src-address-list=testknock3 chain=input protocol=tcp dst-port=22

Then, add the drop rules for SSH and all connections.
> /ip firewall filter add action=drop chain=input protocol=tcp dst-port=22
> /ip firewall filter add action=drop chain=input

**Why the destination port is 22 for allow and drop SSH?**  
Because 22 is SSH port. You may check it by running this command:
> /ip service print

And try to find SSH. There are also other ports listed.

**The destination ports are all different each knocks. WHy?**  
Because I want it. ^_^ It's just like an analogy. Let's say there's a port 7000, 7001, 8000. It depends on what we want.
4. Try the connection 
On the Linux Terminal, let's try to access SSH immediately.
> ssh admin@192.168.1.2

192.168.1.2 is IP Mikrotik (the router).  
You must not allowed to access it. Now, try knocking the port first. There are several ways to do it, but this time I use netcat.
> nc -zv 192.168.1.2 7000 //knock 1
> nc -zv 192.168.1.2 7001 //knock 2
> nc -zv 192.168.1.2 8000 //knock 3

Then try access the SSH.
> ssh admin@192.168.1.2

Now, you can access the SSH.
