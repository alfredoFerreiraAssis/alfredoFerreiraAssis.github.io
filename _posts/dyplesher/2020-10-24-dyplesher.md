---
layout: post
title:  Hack The Box - Dyplesher
category: walkthrough
date:   2020-10-24 16:00:00
---
# ![dyplesher](/assets/img/dyplesher/dyplesher.png)  
dyplehser is a insane machine created by [felamos](https://www.hackthebox.eu/home/users/profile/27390) & [yuntao](https://www.hackthebox.eu/home/users/profile/12438)  
# Enumaration  
### nmap
As always, begin with nmap...    
nmap -Pn -p 22,80,3000,4369,5672,11211 10.10.10.190  
>Starting Nmap 7.70 ( https://nmap.org ) at 2020-08-22 05:30 -03  
>Nmap scan report for dyplesher.htb (10.10.10.190)  
>Host is up (0.13s latency).  
>  
>PORT      STATE SERVICE  
>22/tcp    open  ssh  
>80/tcp    open  http  
>3000/tcp  open  ppp  
>4369/tcp  open  epmd  
>5672/tcp  open  amqp  
>11211/tcp open  memcache  
>
>Nmap done: 1 IP address (1 host up) scanned in 0.15 seconds  
 
### port 80
Started at port 80  
![port80](/assets/img/dyplesher/dyplesher1.png)  
added "10.10.10.190 test.dypleshser.htb dyplesher.htb" to /etc/hosts  
Acessed test.dyplesher.htb and got this page  
![test.dyplesher](/assets/img/dyplesher/dyplesher2.png)  
### port 3000
At port 3000 I got a [gogs](https://gogs.io/) page 
![port3000](/assets/img/dyplesher/dyplesher3.png)  
After register I found users emails  
![emails](/assets/img/dyplesher/dyplesher4.png)  
### fuzz
Fuzzing test.dyplesher [ffuf](https://github.com/ffuf/ffuf) with [raft-large-files.txt](https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/raft-large-files.txt) I found .git  

![ffuf](/assets/img/dyplesher/dyplesher5.png)  
### GitTools
So I used [GitTools](https://github.com/internetwache/GitTools)  
./gitdumper.sh http://test.dyplesher.htb/.git/ ../../test.dyplesher  
![gitdumper](/assets/img/dyplesher/dyplesher6.png)  
back to the directory I created  
git restore index.php  
![index.php](/assets/img/dyplesher/dyplesher7.png)  
cached credentials: felamos:zxcvbnm in index.php  
### memcached-cli
memcache port is open, so lets use [memcached-cli](https://www.npmjs.com/package/memcached-cli)  
to install it I had to download [nodejs](https://nodejs.org/en/)  
install npm  
and install [memcached-cli](https://www.npmjs.com/package/memcached-cli)  
![memcached-cli](/assets/img/dyplesher/dyplesher8.png)  
cp hashes to a file and called my old friend John  
crackin the hash '$2y$12$c3SrJLybUEOYmpu1RVrJZuPyzE5sxGeM0ZChDhl8MlczVrxiA3pQK' I got mommy1 pass for felamos@dyplesher in dyplesher.htb:3000  
![john](/assets/img/dyplesher/dyplesher9.png)  
### gogs
![port3000](/assets/img/dyplesher/dyplesher10.png)  
Felamos has 2 repositories: gitblab and memcached and gitlab has 1 release  
I download everything  
![repo.zip](/assets/img/dyplesher/dyplesher11.png)  
![memcached.git](/assets/img/dyplesher/dyplesher12.png)  
![gitlab.git](/assets/img//dyplehser/dyplesher13.png)    
Looking the files I found a directory named @hashed, with git bundles  
So I used git clone  
![bundles](/assets/img/dyplesher/dypĺesher14.png)  
Enumerating I found user.db, that contais a hash  
![user.dbHash](/assets/img/dyplesher/dyplesher15.png)  
Time to call John again  
![pass](/assets/img/dyplesher/dyplesher16.png)  
This is the pass to dyplesher.htb/login  
![login](/assets/img/dyplesher/dyplesher17.png)  
Then I found the page "AddPlugin" and tried to add some files to see its response  
### Exploit
![addPlugin](/assets/img/dyplesher/dyplesher18.png)  
It has to be a .jar file, so... after remember my java lessons and google some things I coded one that allows me to use SSH  
![.java](/assets/img/dyplesher/dyplesher19.png)  
Logged as MinatoTW  
![ssh](/assets/img/dyplesher/dyplehser20.png)  
Minato is in the wireshark group  
Using [tshark](https://www.wireshark.org/docs/man-pages/tshark.html) I can capture packet data from the network  
![tshark](/assets/img/dyplesher/dyplesher.21png)  
Using *strings* and *grep* I can see some clear text passwords  
![passwords](/asstes/img/dyplesher/dyplesher22.png)  
![password](/assets/img/dyplesher/dyplesher24.png)  
user.txt is in /home/felmamos  
# Privelege Escalation  

Now that I have the passwords of all the users I can search something to get root acess  
I found this at yuntao's home  
![send.sh](/assets/img/dyplesher/dyplesher23.png)  
After search in google about this I found a way  
I had to write a [cuberite plugin](https://api.cuberite.org/Writing-a-Cuberite-plugin.html) and a python script to write my public key in root's authorized\_keys  
References: [lua](https://www.tutorialspoint.com/lua/lua_file_io.htm) & [python](https://pika.readthedocs.io/en/stable/modules/credentials.html)  
![lua](/assets/img/dyplesher/dyplesher26.png)  
![pythonScript](/assets/img/dyplesher/dyplesher25.png)  
Using this script *delight* I can get root access and drink my orange juice  
![JailsonMendesAprova](/assets/img/dyplesher/dyplesher00.jpg)  
I started my python server with *python3 -m http.server 11211* and executed my script  
Now I can drink my orange juice  
![root](/assets/img/dyplesher/dyplesher27.png)

