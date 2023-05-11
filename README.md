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

### Clone hosts

1. Right click on the gold image vm and export
2. Once the export is complete go into the export direcotry and copy the hardrive file to the default location for hyper-v
3. Make one copy per vm to setup (2 workers + 1 master)
4. Make a new vm but in the final step rather than creating a new disk attach it to one of the hardrive files from step 3
5. Open the settigns for the new vm and change to uefi boot and 2 cores
6. Start the vm & log in
7. sudo vi /etc/hostname and chaneg the hostname
8. sudo vi /etc/hosts and update to use new hostname for 127.0.0.1
9. restart vm

Repeat as needed to produce more clones

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

## Notes/Questions/Todos

* Master nodes need a static IP address - can we not use dns?
* Setup firewall
* Encrypt traffic
* Use secrets
* lock swarm
* overlay? Encrypted?
* what is docker trust
* Make swamr use locla reg (is that via stack file?)
* Don't run work on master

Commands
* swarm manages the swarm
* node to manage indivdual nodes
* service to directly manage workload
* stack to manage services defined in stack file?

Traffic
* IpSec
* 2377 tcp communicate with manager
* 7946 tcp/udp overlay discovery
* 4789 UDP overlay traffic


