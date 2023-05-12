# For the Swarm!

Notes on setting up docker swarm

## Create the VM's

### Gold image

1. Download ubuntu server iso
2. Use hyper v create gen 2 4096mb 3 core with uefi secure boot on and on default network
3. Start vm
4. Locale is uk, accept default network and storage options, install docker
5. After install is complete restart vm and login, it'll do some further setup
6. Shutdown vm
7. Reduce memory to 2048 and cores to 2 then checkpoint

### Setup network

For docker nodes to be able to rejoin the swarm after crashing they need to have
the same IP address when they come up as when they went down i.e  we need static
IP addresses. The default switch on Hyper-V only supports DHCP however so we
will need to set up another network on which the Docker hosts speak to each
other which does use static addresses. We will still need to use the default
switch to allow us to ssh from WSL.

Run the following commands in powershell

```
# Create the new switch, the name doesn't need to be DockNet
> New-VMSwitch -SwitchName "DockNet" -SwitchType Internal
> Get-VMSwitch

Name           SwitchType NetAdapterInterfaceDescription
----           ---------- ------------------------------
Default Switch Internal
DockNet        Internal

# When you create the switch a virtual network adapter is created for it, we need to find its index
> Get-NetAdapter | select Name,ifIndex

Name                ifIndex
----                -------
vEthernet (DockNet)      56 <---- This is the one that has just been created
Ethernet                 14

# You then create a new IP address and assign it to the adaptor (referenced by its index)
> New-NetIPAddress -IPAddress 10.0.0.1 -PrefixLength 24 -InterfaceIndex 56

IPAddress         : 10.0.0.1
InterfaceIndex    : 56
InterfaceAlias    : vEthernet (DockNet)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 24
PrefixOrigin      : Manual
SuffixOrigin      : Manual
AddressState      : Tentative
ValidLifetime     :
PreferredLifetime :
SkipAsSource      : False
PolicyStore       : ActiveStore

IPAddress         : 10.0.0.1
InterfaceIndex    : 56
InterfaceAlias    : vEthernet (DockNet)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 24
PrefixOrigin      : Manual
SuffixOrigin      : Manual
AddressState      : Invalid
ValidLifetime     :
PreferredLifetime :
SkipAsSource      : False
PolicyStore       : PersistentStore

# Finally specify a new NAT network and set the address range it can use
> New-NetNat -Name DockNat -InternalIPInterfaceAddressPrefix 10.0.0.0/24

Name                             : DockNat
ExternalIPInterfaceAddressPrefix :
InternalIPInterfaceAddressPrefix : 10.0.0.0/24
IcmpQueryTimeout                 : 30
TcpEstablishedConnectionTimeout  : 1800
TcpTransientConnectionTimeout    : 120
TcpFilteringBehavior             : AddressDependentFiltering
UdpFilteringBehavior             : AddressDependentFiltering
UdpIdleSessionTimeout            : 120
UdpInboundRefresh                : False
Store                            : Local
Active                           : True

```

### Clone hosts

1. Right click on the gold image vm and export
1. Once the export is complete go into the export direcotry and copy the hardrive file to the default location for hyper-v
1. Make one copy per vm to setup
1. Make a new vm but in the final step rather than creating a new disk attach it to one of the hardrive files from step 3
1. Open the settings for the new vm and
    * change to uefi boot
    * Chnage to 2 cores
    * Add a new network adaptor and assign it to the switch you created

### Configure host

1. Start the vm & log in
1. sudo vi /etc/hostname and chaneg the hostname
1. sudo vi /etc/hosts and update to use new hostname for 127.0.0.1
1. Find the names of the network adaptors by running `ip addr`
1. sudo vi /etc/netplan/00-installer-config.yaml and set content to something like
```
network:
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.0.0.11/24
      routes:
        - to: default
          via: 10.0.0.1
    eth1:
      dhcp4: true
  version: 2
```

Where
* eth0 is the nic connected to the internal switch you created earlier
* eth1 is the nic connected to the default switch for Hyper-V
* 10.0.0.11 is the static IP address you want to assign to this host
* 10.0.0.1 is the address you associated with the switch you created

6. sudo netplan apply
7. Reboot

This should give you internet connectivity, static ip, the bailty toping the static IP of other hosts and ssh

### Lather, rinse, repeat, as needed

Repeat the clonign and configuration until you have as many hosts as required

Then snapshot everything

### SSH

To be able to ssh from WSL port fowrarding needs to be setup in both directions.

Run

`Get-NetIPInterface | select ifIndex,InterfaceAlias,AddressFamily,ConnectionState,Forwarding | Sort-Object -Property IfIndex | Format-Table`

To get the id's for the wsl and default switches then run

`Set-NetIPInterface -ifindex <default switch id> -Forwarding Enabled`

`Set-NetIPInterface -ifindex <wsk switch id> -Forwarding Enabled`

## Set up Docker

### Quality of life

#### Aliases

Create a .bashrc

```
alias ll='ls -lh'
alias dk='docker'
alias dki='docker image'
alias dkc='docker container'
alias dkv='docker volume'
alias dkn='docker network'
alias dks='docker swarm'
alias dko='docker node'
```

#### Docker as non root *Not for a prod env.*

`$ sudo groupadd docker`

`$ sudo usermod -aG docker $USER`

Then reboot the machine

### Set up a registry

On the master node

docker run -d -p 5000:5000 --restart always --name registry registry:2

## Set up the Swarm

On the master

`$ docker swarm init --advertise-addr <IP:PORT>`

This will give you a join token which can be used by workers to jon the swarm. If you lose it run

`$ docker swarm join-token worker`

On the workers run

`$ docker swarm join --token <the join token> <master ip address>:2377`

Set the swarm to autolock

`$ docker swarm update --autolock`

Stop work being scheduled on the manager

`$ docker node update --availability drain <manager node id>`

## When Shit goes wrong

### Worker goes down

It should automatically rejoin when it comes back up, any work on it will be reallocated

### Non leader manager goes down

### Leader node goes down

Leadership should pass to another manager node and stuff will keep running
When the oldleader is back it does not rejoin nicely
Use

`$ docker swarm leave --force`

On the old leader to get it ready to rejoin the swarm

On a functioning manager remove the old leader

```
$ docker node demote <old leader>
$ docker node rm <old leader>
$ docker join-token manager
```

Then run the join command on the old leader

### Everything goes down

Buidl the swarm from scratch again

## Notes/Questions/Todos

* Setup firewall
* Encrypt traffic
* Use secrets
* lock swarm
* overlay? Encrypted?
* what is docker trust
* Make swamr use locla reg (is that via stack file?)
* don't run as root
* ingress load balancing
* what happens if teh manager with the advertised address goes down - try haproxy


Traffic
* IpSec
* 2377 tcp communicate with manager
* 7946 tcp/udp overlay discovery
* 4789 UDP overlay traffic

Setup
* Aux cluster, load balance to managers & registry (Separate node)
* Logging cluster
* App clusters

Disadvanatges
* Less features
* no autoscaling
* no rbac
* less cloud friendly
* lack of community
* people think its dying (and have done for the past decade)
* less 3rd party support
* pods let you do thing with sidecars
* If manager ndoe goes down its a bit of an arse to get it to rejoin, workers are handled automatically
