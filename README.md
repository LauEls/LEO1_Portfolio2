# LEO1_Portfolio2
Assignment 2 for LEO1 Course at SDU

Group 26: Laurenz Elstner, Alberto Sartori, Antoni Benavent

Install LXC:
```
$ sudo apt install lxc
```

Check actual user and group id ranges for modification of the next command
```
$ grep $USER /etc/subuid
$ grep $USER /etc/subgid
```

Create a default container file with desired id mappings and network setup.
Instead of 65536 put the ranges you got before
Adds a configuration which allows the unpriviliged user to hook into the host network
```
$ mkdir -p ~/.config/lxc
$ echo "lxc.id_map = u 0 100000 65536" > ~/.config/lxc/default.conf
$ echo "lxc.id_map = g 0 100000 65536" >> ~/.config/lxc/default.conf
$ echo "lxc.network.type = veth" >> ~/.config/lxc/default.conf
$ echo "lxc.network.link = lxcbr0" >> ~/.config/lxc/default.conf
$ echo "$USER veth lxcbr0 2" | sudo tee -a /etc/lxc/lxc-usernet
```

We only need dnsmasq-base. If the whole dnsmasq is activated you will get an error when trying to stat lxc-net. In that case stop and disable dnsmasq.
```
$ sudo apt install dnsmasq-base
$ systemctl restart lxc-net
$ systemctl status lxc-net
$ systemctl stop dnsmasq
$ systemctl disable dnsmasq
```

To configure the bridge exchange the line
```
lxc.network.type = empty
```
with
```
lxc.network.type = veth
lxc.network.link = lxcbr0
lxc.network.flags = up
lxc.network.hwaddr = 00:16:3e:xx:xx:xx
```
lxcbr0 is the name of our bridge on the host. To tell lxc to use our bridge create file in /etc/default/lxc-net with following content
```
USE_LXC_BRIDGE="true"
```
Restart lxc-net
```
$ systemctl restart lxc-net
$ systemctl status lxc-net
```

Now it's time to create container one with the name C1, start it and access it
```
$ lxc-create -n C1 -t download -- -d alpine -r 3.4 -a armhf
$ lxc-start -n C1
$ lxc-attach -n C1
```

Inside the container we make sure it's updated, install the webserver lighttpd and a editor program (we chose nano)
```
#/ apk update
#/ apk add lighttpd php5 php5-cgi php5-curl php5-fpm
#/ apk add nano
```
Inside the file /etc/lighttpd/lighttpd.conf uncomment the line: include "mod_fastcgi.conf" and start the server. 
```
#/ rc-update add lighttpd default
#/ openrc
```

Create the file var/www/localhost/htdocs/index.php and add the following lines:
```
<!DOCTYPE html>
<html><body><pre>
<?php 
        // create curl resource 
        $ch = curl_init(); 
        // set url 
        curl_setopt($ch, CURLOPT_URL, "C2:8080"); 
        //return the transfer as a string 
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1); 
        // $output contains the output string 
        $output = curl_exec($ch); 
        // close curl resource to free up system resources
        curl_close($ch);
        print $output;
?>
</body></html>
```

We exit the first container and create a secound one similarly but without the webserver and install socat
```
$ lxc-create -n C2 -t download -- -d alpine -r 3.4 -a armh
$ lxc-start -n C2
$ lxc-attach -n C2
```
```
#/ apk add socat
#/ apk add nano
```

We create a script /bin/rng.sh which gives 16 random numbers (count=16). Since the container doesn't support bash we use ash instead. We also use urandom instead of random because random will wait until enough entropie occured which can take some time.
```
//container has no bash so instead we use ash
#!/bin/ash

dd if=/dev/urandom bs=4 count=16 status=none | od -A none -t u4
```
We make the script executable and start listening
```
#/ chmod +x /bin/rng.sh
#/ socat -v -v tcp-listen:8080,fork,reuseaddr exec:/bin/rng.sh
```

We exit the container and expose the web-server form container one (C1) to the outside by enabeling DNAT with iptables
In our case we chose the following values: <host_nic>: wlan0; <host_port>: 80; <ct_ip>: 10.0.3.136; <ct_port>: 80
<host_nic> is the interface from which we get access to the internet
<ct_ip> is the ip-address of the container with the webserver. The ip-address can be found with the command `$ lxc-ls -f`.
This is only a temporary solution, since it resets after every reboot.
```
$ iptables -t nat -A PREROUTING -i <host_nic> -p tcp --dport <host_port> -j DNAT --to-destination <ct_ip>:<ct_port>
```

We check if ip_forward=1 and if the forwarding was successful
```
$ sysctl net.ipv4.ip_forward
$ sudo iptables -t nat -L
```

if ip_forward=0 then
```
$ sysctl net.ipv4.ip_forward
$ sudo sysctl net.ipv4.ip_forward=1
```


The Website should now be accessible by calling the ip address from the chosen interface(<host_nic>)
