# DHCP Setup for Satellite

Satellite and the DHCP server reside on different servers.  In this article we will walk through configuring the DHCP server to work with Satellite.  

We installed DHCP and Bind and updated firewall settings in this article - [DNS Dynamic Update](https://github.com/pslucas0212/DNSUpdating).  Specs for the server can also be found with this article.

Next we need to generate a security token on the server hosting DHCP.
```
# dnssec-keygen -a HMAC-MD5 -b 512 -n HOST omapi_key
Komapi_key.+157+56839
```

Copy the secret from the key.
```
# cat Komapi_key.+*.private |grep ^Key|cut -d ' ' -f2
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



