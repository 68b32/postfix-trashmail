postfix-tempmail
================

Source: [https://www.vennedey.net/postfix-tempmail](https://www.vennedey.net/postfix-tempmail)

`postfix-tempmail` implements a [Postfix socketmap table lookup](http://www.postfix.org/socketmap_table.5.html "http://www.postfix.org/socketmap_table.5.html") service for temporary trashmail-like e-mail addresses.

Table of contents
-----------------

*   [The concept](#the_concept)
*   [Installation](#installation)
*   [Configuration](#configuration)
*   [Test configuration](#test_configuration)
*   [Activate in postfix](#activate_in_postfix)

The concept
-----------

The idea is to generate an e-mail address from a date and a secret. Everyone who knows the secret (you and your mail server) will be able to generate the e-mail address for a given date. This e-mail address can then be used to sign up for a service or generated dynamically and published on your website and will only be valid for the current day.

Here is an example PHP code

```
$secret = "<Your secret>";
echo substr(md5(date("Ymd").$secret),0,8).'@example.com';
```

This will print a different e-mail address like `35de1bff@example.com` for every day.

One problem with this technique is that if someone starts writing an e-mail at 23:59, he likely won't be finished before the address he uses expires. To solve this, two temporary addresses will always be valid: The one generated from the current date, and the one generated from yesterday's date.

To make this work with `postfix`, we need a way to map the temporary addresses to a permanent address. This is done by `postfix-tempmail`.

`postfix-tempmail` comes as simple bash implementation of a socketmap table lookup service. This means that `postfix-tempmail` can be queried by `postfix` using [TCP/IP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol "https://en.wikipedia.org/wiki/Transmission_Control_Protocol") and [Unix domain sockets](https://en.wikipedia.org/wiki/Unix_domain_socket "https://en.wikipedia.org/wiki/Unix_domain_socket").

Installation
------------

`postfix-tempmail` is hosted on [GitHub](https://github.com/68b32/postfix-tempmail "https://github.com/68b32/postfix-tempmail") and can be installed with the following instructions:

```
root@mailhost:~# apt-get install git socat
root@mailhost:~# git clone https://github.com/68b32/postfix-tempmail.git

root@mailhost:~# install -o postfix -g root -m 0460 postfix-tempmail/tempmail.cf /etc/postfix
root@mailhost:~# install -o postfix -g root -m 0550 postfix-tempmail/postfix-tempmail /usr/local/bin
root@mailhost:~# install -o postfix -g root -m 0750 -d /var/spool/postfix/postfix-tempmail
root@mailhost:~# install -o root -g root -m 0644 postfix-tempmail/postfix-tempmail.service /etc/systemd/system

root@mailhost:~# systemctl daemon-reload
```

Configuration
-------------

Once `postfix-tempmail` is installed, it needs to be configured. The configuration is done in `/etc/postfix/tempmail.cf`. Most likely only the first three variables needs to be configured.

*   `SHARED_SECRET`  
    The secret as described above. This will be used to generate the temporary e-mail addresses.
    
*   `DOMAIN`  
    The domain of the generated e-mail addresses.
    
*   `MAILDROP`  
    The address that is mapped to the temporary addresses. This is where mail sent to temporary addresses is delivered to.
    
*   `LENGTH`  
    This parameter sets the length of the part of the e-mail address before the `@`.
    
*   `SOCKETMAP_SOCKET_TYPE`  
    Type of socket to be used for communication with `postfix`.  
    `unix`: Use Unix domain socket (see `SOCKETMAP_UNIX_SOCKET_*` variables for further configuration).  
    `inet`: Use TCP/IP socket (see `SOCKETMAP_INET_*` variables for further configuration).
    
*   `SOCKETMAP_UNIX_SOCKET_PATH`  
    The path of the Unix Domain Socket.
    
*   `SOCKETMAP_UNIX_SOCKET_OWNER_UID`  
    The owner UID of the Unix Domain Socket.
    
*   `SOCKETMAP_UNIX_SOCKET_OWNER_GID`  
    The owner GID of the Unix Domain Socket.
    
*   `SOCKETMAP_UNIX_SOCKET_MODE`  
    The permissions for the Unix Domain Socket.
    
*   `SOCKETMAP_INET_INTERFACE`  
    The IP of the interface to listen to in `inet` mode.
    
*   `SOCKETMAP_INET_PORT`  
    The TCP port to listen to in `inet` mode.
    
*   `SOCKETMAP_NAME`  
    The name of the service as required by the [socket map protocol](http://www.postfix.org/socketmap_table.5.html "http://www.postfix.org/socketmap_table.5.html"). This name has to be provided in postfix's `virtual_alias_maps` directive.
    
*   `SOCKETMAP_MAX_NETSTRING_LENGTH`  
    The maximum length allowed for a `postfix` query. The default of `512` is more than sufficient.
    
*   `PARSER_RUN_UID`  
    The UID the parser process should be run with. Use this only if you run the listener process as `root` (not recommended).
    

In the default configuration the listener process runs as user `postfix`. This can be changed in the systemd unit file `/etc/systemd/system/postfix-tempmail.service`. When changing the user and using Unix domain sockets, make sure to allow the user running `postfix-tempmail` to access the socket by configuring `SOCKETMAP_UNIX_SOCKET_OWNER_*` and `SOCKETMAP_UNIX_SOCKET_MODE` properly. Also be aware that the socket needs to be accessible by the `postfix` process which might run [chrooted](https://en.wikipedia.org/wiki/Chroot "https://en.wikipedia.org/wiki/Chroot") (`/var/spool/postfix`).

Test configuration
------------------

To test the configuration, first run `postfix-tempmail` without arguments to list the current addresses.

```
root@mailhost:~# postfix-tempmail
8d4f005d@example.com
12b97900@example.com
```

Now start the listener and query for the addresses using the `postmap` command.

```
root@mailhost:~# systemctl start postfix-tempmail

root@mailhost:~# postmap -q 8d4f005d@example.com socketmap:unix:/var/spool/postfix/postfix-tempmail/socket:tempmail
permanent-address@example.com

root@mailhost:~# postmap -q 12b97900@example.com socketmap:unix:/var/spool/postfix/postfix-tempmail/socket:tempmail
permanent-address@example.com

root@mailhost:~# postmap -q invalid@example.com socketmap:unix:/var/spool/postfix/postfix-tempmail/socket:tempmail
```

If this does not work, check `systemctl status postfix-tempmail -l` for errors.

Activate in postfix
-------------------

To activate `postfix-tempmail` in `postfix` append it to the `virtual_alias_maps` directive in `/etc/postfix/main.cf`. Be aware that `postfix` usually runs in a chroot (`/var/spool/postfix`), so that the path to the Unix domain socket needs to be given accordingly.

[/etc/postfix/main.cf](/code/a588a7c1d8a863e45883dc1637ddbc0c39189512bc8bb1f435c38cbe4e84a656 "Download Snippet")

```
virtual_alias_maps = ... socketmap:unix:/postfix-tempmail/socket:tempmail
```

Reload `postfix`, and try to send an e-mail to your temporary address.

```
root@mailhost:~# systemctl reload postfix
```

Enable `postfix-tempmail` to start on boot just before `postfix`.

```
root@mailhost:~# systemctl enable postfix-tempmail
Created symlink from /etc/systemd/system/postfix.service.wants/postfix-tempmail.service to /etc/systemd/system/postfix-tempmail.service.
```
