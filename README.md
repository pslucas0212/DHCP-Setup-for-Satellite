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

Verify communication with the NFS server and the Remote Procedure Call (RPC) communication paths
```
# showmount -e ns02.example.com
Export list for ns02.example.com:
/exports/var/lib/dhcpd 10.1.10.254
/exports/etc/dhcp      10.1.10.254
/exports               10.1.10.254
rpcinfo -p 10.1.10.254
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
```


Add the following lines to the /etc/fstab file:
```
ns02.example.com:/exports/etc/dhcp /mnt/nfs/etc/dhcp nfs
ro,vers=3,auto,nosharecache,context="system_u:object_r:dhcp_etc_t:s0" 0 0

ns02.example.com:/exports/var/lib/dhcpd /mnt/nfs/var/lib/dhcpd nfs
ro,vers=3,auto,nosharecache,context="system_u:object_r:dhcpd_state_t:s0" 0 0
```

Mount the file systems on /etc/fstab:
```
# mount -a
```

To verify that the foreman-proxy user can access the files that are shared over the network, display the DHCP configuration and lease files:
```
# su foreman-proxy -s /bin/bash
bash-4.2$ cat /mnt/nfs/etc/dhcp/dhcpd.conf
bash-4.2$ cat /mnt/nfs/var/lib/dhcpd/dhcpd.leases
bash-4.2$ exit
```

 Enter the satellite-installer command to make the following persistent changes to the /etc/foreman-proxy/settings.d/dhcp.yml file:
```
# satellite-installer --foreman-proxy-dhcp=true \
--foreman-proxy-dhcp-provider=remote_isc \
--foreman-proxy-plugin-dhcp-remote-isc-dhcp-config /mnt/nfs/etc/dhcp/dhcpd.conf \
--foreman-proxy-plugin-dhcp-remote-isc-dhcp-leases /mnt/nfs/var/lib/dhcpd/dhcpd.leases \
--foreman-proxy-plugin-dhcp-remote-isc-key-name=omapi_key \
--foreman-proxy-plugin-dhcp-remote-isc-key-secret=BWbZEXP3sMp2UHnA81uofNxAyUUEWPV7JlrNmE8p1S+XbKozKPlxDq542NRu2ERq7I/KbacdcMiECIRRoCoEAA== \
--foreman-proxy-plugin-dhcp-remote-isc-omapi-port=7911 \
--enable-foreman-proxy-plugin-dhcp-remote-isc \
--foreman-proxy-dhcp-server=ns02.example.com
```

 Restart the foreman-proxy service:
```
# systemctl restart foreman-proxy
```

Log in to the Satellite Server web UI.

Navigate to Infrastructure > Capsules, locate the Satellite Server, and from the list in the Actions column, select Refresh.

Associate the DHCP service with the appropriate subnets and domain. 

## References
[Chapter 5. Configuring Satellite Server with External Services](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.9/html/installing_satellite_server_from_a_connected_network/configuring-external-services)
