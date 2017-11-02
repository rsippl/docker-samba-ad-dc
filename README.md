# Docker container demonstrating Samba's Active Directory Domain Controller (AD DC) support

This repository is unmaintained. Check if one of the forks are up to date.

Run these commands to build, start and stop the container
```
# build samba AD image
docker build -t samba-ad-dc .
# run samba AD image with autogenerated passwords
docker run --rm --privileged -v ${PWD}/samba:/var/lib/samba  -e "SAMBA_DOMAIN=SAMDOM" -e "SAMBA_REALM=SAMDOM.EXAMPLE.COM" --name dc1 --dns 127.0.0.1 -d samba-ad-dc
# run samba AD image with predefined passwords without mounting the samba volume
docker run --rm -e "SAMBA_DOMAIN=SAMDOM" -e "SAMBA_REALM=SAMDOM.EXAMPLE.COM" -e "ROOT_PASSWORD=Opoh6quoo5lu" -e "SAMBA_ADMIN_PASSWORD=Mypassword*2017" --name dc1 --dns 127.0.0.1 -d -p 53:53 -p 53:53/udp -p 389:389 -p 88:88 -p 135:135 -p 139:139 -p 138:138 -p 445:445 -p 464:464 -p 3268:3268 -p 3269:3269 samba-ad-dc
# stop and remove samba AD image
docker stop dc1 && docker rm dc1
# remove local samba
sudo rm -rf samba
```
You can of course change the domain and realm to your liking.

You get the IP-address of the running machine by issuing `docker inspect dc1 | grep IPAddress` and the root user's
password as well as other passwords by running `docker logs dc1 2>&1 | head -3`. You should then be able to log in with SSH.

One fast check to see that Kerberos talks with Samba:

```
ssh root@172.17.0.2
root@1779834e202b:~# kinit administrator@SAMDOM.EXAMPLE.COM
Password for administrator@SAMDOM.EXAMPLE.COM:
Warning: Your password will expire in 41 days on Thu Jul 10 19:36:55 2014
root@1779834e202b:~# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@SAMDOM.EXAMPLE.COM

Valid starting     Expires            Service principal
05/29/14 19:45:53  05/30/14 05:45:53  krbtgt/SAMDOM.EXAMPLE.COM@SAMDOM.EXAMPLE.COM
        renew until 05/30/14 19:45:43

```

LDAP search within the container:

```
ldapsearch -b "DC=samdom,DC=example,DC=com" "(&(objectClass=user)(name=administrator))"
```

Edit [custom.sh](custom.sh) to add custom logic executed at the and of supervisord.

## Allow Insecure LDAP Authentication

Simple auth via LDAP fails if you have an unencrypted connection ("BindSimple: Transport encryption required").

For debugging purposes, you can avoid this error by setting the LDAP_ALLOW_INSECURE environment variable to true.

DO NOT USE LDAP_ALLOW_INSECURE IN PRODUCTION!

## Redmine client

Now you can test Redmine ldap login to the host.
```
docker run --name redmine -d sameersbn/redmine:latest
REDMINE_IP=$(docker inspect redmine | grep IPAddres | awk -F'"' '{print $4}')
xdg-open "http://${REDMINE_IP}/auth_sources/new"
```

Refresh the browser until the login page shows. Login with both username and password as admin. Fill the form with these credentials:

```
Name: samdom
Host: *samba_ad_dc_ip*
Port: 389 [ ] LDAPS
Account: Administrator@smbdc1
Password: *samba_admin_password_here*
Base DN: CN=Users,DC=samdom,DC=example,DC=com
LDAP filter:
Timeout (in seconds):

On-the-fly user creation [X]
Attributes:
    Login attribute: sAMAccountName
    Firstname attribute: givenName
    Lastname attribute: sn
    Email attribute: userPrincipalName
```

Now log out and log in with the samba administrator credentials (username: administrator, password: *check with docker log dc1*)

## Windows client

[This](http://vimeo.com/11527979#t=3m15s) is a nice guide to join your Windows 7 client to the DC. Just make sure to have your Docker container as the
[primary DNS server for Windows](http://www.opennicproject.org/configure-your-dns/how-to-change-dns-servers-in-windows-7/).

## LDAP explorers

I used [JXplorer](http://jxplorer.org/) to explore the LDAP-schema. To log in you need to input something like this:
![JXplorer example](http://i.imgur.com/LniIp22.png)

## Samba as User Database for Alfresco

For an example on how to use Samba as LDAP server with Alfresco, see the provided docker-compose file:

```
cd compose
docker-compose up
```

Watch the Alfresco log:

```
docker exec -it alfresco bash
tail -f /alfresco/tomcat/logs/catalina.out
```

Once Alfresco is up and running, you can log in at port 8080 as admin/admin or as one of the users created in
the custom.sh file.

## Testing UNIX login with sssd

```
root@e936157c0bc1:~# getent passwd Administrator
administrator:*:935000500:935000513:Administrator:/:
root@e936157c0bc1:~# getent group "Domain Users"
domain users:*:935000513:administrator
root@e936157c0bc1:~# ssh Administrator@localhost
Administrator@localhost's password:
Welcome to Ubuntu 14.04 LTS (GNU/Linux 3.14.4-1-ARCH x86_64)
```

## Resources
I followed the guide on Samba's wiki pages https://wiki.samba.org/index.php/Samba_AD_DC_HOWTO

Port usage: https://wiki.samba.org/index.php/Samba_port_usage

## Port forwarding command
If you want the DC to be reachable through the host's IP you can start the container with this command:
```
docker run --privileged -p 53:53 -p 53:53/udp -p 88:88 -p 88:88/udp -p 135:135 -p 137-138:137-138/udp -p 139:139 -p 389:389 -p 389:389/udp -p 445:445 -p 464:464 -p 464:464/udp -p 636:636 -p 1024-1044:1024-1044 -p 3268-3269:3268-3269 -v ${HOME}/dockervolumes/samba:/var/lib/samba  -e "SAMBA_DOMAIN=samdom" -e "SAMBA_REALM=samdom.example.com" -e "SAMBA_HOST_IP=$(hostname --all-ip-addresses |cut -f 1 -d' ')" --name samdom --dns 127.0.0.1 -d samba-ad-dc
```

The problem is that the port range 1024 and upwards are used for dynamic RPC-calls, luckily Samba goes through them in
order, so the first 20 or so should suffice for testing purposes. Windows complains otherwise that "The RPC server is
unavailable". It's also possible to eliminate long command line parameters by using `$(for port in $(seq 135 139); do
echo -n "-p $port:$port "; done;)` instead.

## TODO

* [X] xattr and acl support for docker containers
* [ ] NTP support
* [ ] Try to join other Samba4 servers with a Samba4 DC
* [ ] How to implement redundancy and fail-safes?
* [X] Verify that Bind9 Dynamically Loadable Zones (DLZ) work
* [X] Can this be used for UNIX logins as well?
* [ ] Probably a lot more to make this robust enough for production use
* [ ] Fix rare BIND9 startup problems. Failed to connect to /var/lib/samba/private/dns/sam.ldb.
