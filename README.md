# DHCP Setup for Satellite

Satellite and the DHCP server reside on different servers.  In this article we will walk through configuring the DHCP server to work with Satellite.  

We installed DHCP and Bind and updated firewall settings in this article - [DNS Dynamic Update](https://github.com/pslucas0212/DNSUpdating).  Specs for the server can also be found with this article.


### Preparing the server hosting named and dhcpd
Next we need to generate a security token on the server hosting DHCP.
```
# dnssec-keygen -a HMAC-MD5 -b 512 -n HOST omapi_key
Komapi_key.+157+56839
```

Copy the secret from the key.
```
# cat Komapi_key.+*.private |grep ^Key|cut -d ' ' -f2
jNSE5YI3H1A8Oj/tkV4...A2ZOHb6zv315CkNAY7DMYYCj48Umw==
```

Add the following information to the /ect/dhcp/dhcpd.conf file.
```
omapi-port 7911;
key omapi_key {
        algorithm HMAC-MD5;
        secret "jNSE5YI3H1A8Oj/tkV4...A2ZOHb6zv315CkNAY7DMYYCj48Umw==";
};
omapi-key omapi_key;
```
On the Satellite server gather foreman user UID and GID.
```
# id -u foreman
987
# id -g foreman
981
```

On the server hosting dns and dhcp create the foreman userid and group.
```
# groupadd -g 981 foreman
# useradd -u 987 -g 981 -s /sbin/nologin foreman
```

Restore the read ane execut flags.
```
# chmod o+rx /etc/dhcp/
# chmod o+r /etc/dhcp/dhcpd.conf
# chattr +i /etc/dhcp/ /etc/dhcp/dhcpd.conf
```

On the server hosting dhcp, Export the DHCP configuration and lease files using NFS.
```
# yum install nfs-utils
...
complete!
# systemctl enable rpcbind nfs-server
# systemctl enable rpcbind nfs-server
# systemctl start rpcbind nfs-server nfs-idmapd
```
Create directories for the DHCP configuration and lease files that you want to export using NFS:
```
# mkdir -p /exports/var/lib/dhcpd /exports/etc/dhcp
```
To create mount points for the created directories, add the following line to the /etc/fstab file:
```
/var/lib/dhcpd /exports/var/lib/dhcpd none bind,auto 0 0
/etc/dhcp /exports/etc/dhcp none bind,auto 0 0
```

 Mount the file systems in /etc/fstab:
```
# mount -a
```

Add these lines to the /etc/exports file. The ip address is from your Satellite server
```
/exports 10.1.10.254(rw,async,no_root_squash,fsid=0,no_subtree_check)

/exports/etc/dhcp 10.1.10.254(ro,async,no_root_squash,no_subtree_check,nohide)

/exports/var/lib/dhcpd 10.1.10.254(ro,async,no_root_squash,no_subtree_check,nohide)
```

Reload the NFS server:
```
# exportfs -rva
```

Configure the firewall for the DHCP omapi port 7911:
```
# firewall-cmd --add-port="7911/tcp" \
&& firewall-cmd --runtime-to-permanent
success
success
```

 Optional: Configure the firewall for external access to NFS. Clients are configured using NFSv3.
```
# firewall-cmd --zone public --add-service mountd \
&& firewall-cmd --zone public --add-service rpc-bind \
&& firewall-cmd --zone public --add-service nfs \
&& firewall-cmd --runtime-to-permanent
success
success
success
success
```

### Preparing the Satellite Server 
**Note: I already have Satellite installed and running.***

Install the nfs-utils utility:
```
# foreman-maintain packages install nfs-utils
```

Create the DHCP directories for NFS:
```
# mkdir -p /mnt/nfs/etc/dhcp /mnt/nfs/var/lib/dhcpd
```

Change the file owner:
```
# chown -R foreman-proxy /mnt/nfs
```

## References
[Chapter 5. Configuring Satellite Server with External Services](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.9/html/installing_satellite_server_from_a_connected_network/configuring-external-services)
