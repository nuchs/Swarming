# For the Swarm!

Notes on setting up docker swarm

# Create the VM's

## Gold image

1. Download ubuntu server iso
2. Use hyper v create gen 2 4096mb 3 core with uefi secure boot on and on default network
3. Start vm
4. Locale is uk, accept default network and storage options, install docker
5. After install is complete restart vm and login, it'll do some further setup
6. Shutdown vm
7. Reduce memory to 2048 and cores to 2 then checkpoint

## Clone hosts

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

## SSH

To be able to ssh from WSL port fowrarding needs to be setup in both directions.

Run

`Get-NetIPInterface | select ifIndex,InterfaceAlias,AddressFamily,ConnectionState,Forwarding | Sort-Object -Property IfIndex | Format-Table`

To get the id's for the wsl and default switches then run

`Set-NetIPInterface -ifindex <default switch id> -Forwarding Enabled`

`Set-NetIPInterface -ifindex <wsk switch id> -Forwarding Enabled`

