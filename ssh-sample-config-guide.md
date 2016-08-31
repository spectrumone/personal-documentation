#NGINX Sample Config Guide

### /path/to/.ssh/config
```bash

Host example-a.com
  # Host will be used as Hostname if Hostname is not specified.

  User ubuntu
  Port 65522
  
  # This option allows authentication keys stored on our local machine to be
  # forwarded onto the system you are connecting to. This can allow you to hop
  # from host-to-host using your home keys.
  ForwardAgent yes
  
  # This option can tell SSH to display an ASCII representation of the remote host's
  # key upon connection. Turning this on can be an easy way to get familiar with your
  # host's key, allowing you to easily recognize it if you have to connect from a
  # different computer sometime in the future.
  VisualHostKey yes

Host example-b.com
  Hostname <ip address>
  User ubuntu
  ForwardAgent yes
  
  # Connect to example-a and then connect to example-b. This is useful so the user
  # does not need to add his public rsa to example-b as long as as example-b has
  # has the public rsa of example-a in his ~/.ssh/authorized_keys
  # -W host:port Requests that standard input and output on the client be forwarded
  # to host on port over the secure channel.
  ProxyCommand ssh -q -W %h:%p example-a.com

  VisualHostKey yes

Host <random name that doesnt have to have to be a domain name>
  Hostname <ip address>
  User ubuntu
  ForwardAgent yes
  ProxyCommand ssh -q -W %h:%p example-a.com
  VisualHostKey yes
```

###Reference:
* [http://www.cyberciti.biz/faq/linux-unix-ssh-proxycommand-passing-through-one-host-gateway-server/](http://www.cyberciti.biz/faq/linux-unix-ssh-proxycommand-passing-through-one-host-gateway-server/)
* [https://www.digitalocean.com/community/tutorials/how-to-configure-custom-connection-options-for-your-ssh-client](https://www.digitalocean.com/community/tutorials/how-to-configure-custom-connection-options-for-your-ssh-client)
* [http://man.openbsd.org/OpenBSD-current/man1/ssh.1](http://man.openbsd.org/OpenBSD-current/man1/ssh.1)
