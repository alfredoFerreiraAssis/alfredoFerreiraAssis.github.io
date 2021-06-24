---
layout: post
title:  Hack The Box - Dyplesher
category: walkthrough
date:   2020-10-24 16:00:00
---
# # ![dyplesher](/assets/img/dyplesher/dyplesher.png)  
<p align="center"> Dyplehser is a insane machine created by<a href="https://www.hackthebox.eu/home/users/profile/27390">felamos</a> & <a href="https://www.hackthebox.eu/home/users/profile/12438">yuntao</a></p>  
# Enumaration  
### <center>nmap</center>
<center>As always, begin with nmap...</center>    
<center>nmap -Pn -p 22,80,3000,4369,5672,11211 10.10.10.190</center>  
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
  
### <center>port 80</center>
<center>Started at port 80</center>  
# ![port80](/assets/img/dyplesher/dyplesher1.png)  
<center>added "10.10.10.190 test.dypleshser.htb dyplesher.htb" to /etc/hosts</center>  
<center>Acessed test.dyplesher.htb and got this page</center>  
# ![test.dyplesher](/assets/img/dyplesher/dyplesher2.png)  
### port 3000
<p align="center">At port 3000 I got a <a href="https://gogs.io/">gogs</a> page</p><br>  
# ![port3000](/assets/img/dyplesher/dyplesher3.png)  
<center>After register I found users emails</center>  
# ![emails](/assets/img/dyplesher/dyplesher4.png)  
### fuzz
<p align="center">Fuzzing test.dyplesher with <a href="https://github.com/ffuf/ffuf">ffuf</a> and <a href="https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/raft-large-files.txt">raft-large-files.txt</a> I found .git</p><br><br> 
  
# ![ffuf](/assets/img/dyplesher/dyplesher5.png)  
### <center>GitTools</center>
<p align="center">So I used <a href="https://github.com/internetwache/GitTools">GitTools</a></p><br>  
<center>./gitdumper.sh http://test.dyplesher.htb/.git/ ../../test.dyplesher</center>  
# ![gitdumper](/assets/img/dyplesher/dyplesher6.png)  
<center>back to the directory I created</center>  
<center>git restore index.php</center>  
# ![index.php](/assets/img/dyplesher/dyplesher7.png)  
<center>cached credentials: felamos:zxcvbnm in index.php</center>  
### <center>memcached-cli</center>  
<p align="center">memcache port is open, so lets use <a href="https://www.npmjs.com/package/memcached-cli">memcached-cli</a><br>  
to install it I had to download <a href="https://nodejs.org/en/">nodejs</a><br>  
install npm<br>  
and install <a href="https://www.npmjs.com/package/memcached-cli">memcached-cli</a></p>  
# ![memcached-cli](/assets/img/dyplesher/dyplesher8.png)  
<center>cp hashes to a file and called my old friend John</center>  
<center>crackin the hash '$2y$12$c3SrJLybUEOYmpu1RVrJZuPyzE5sxGeM0ZChDhl8MlczVrxiA3pQK' I got mommy1 pass for felamos@dyplesher in dyplesher.htb:3000</center>  
# ![john](/assets/img/dyplesher/dyplesher9.png)  
### <center>gogs</center>
# ![port3000](/assets/img/dyplesher/dyplesher10.png)  
<center>Felamos has 2 repositories: gitblab and memcached and gitlab has 1 release</center>  
<center>I download everything</center>  
# ![repo.zip](/assets/img/dyplesher/dyplesher11.png)  
# ![memcached.git](/assets/img/dyplesher/dyplesher12.png)  
# ![gitlab.git](/assets/img//dyplehser/dyplesher13.png)    
<center>Looking the files I found a directory named @hashed, with git bundles</center>  
<center>So I used git clone</center>  
# ![bundles](/assets/img/dyplesher/dypĺesher14.png)  
<center>Enumerating I found user.db, that contais a hash</center>  
# ![user.dbHash](/assets/img/dyplesher/dyplesher15.png)  
<center>Time to call John again</center>  
# ![pass](/assets/img/dyplesher/dyplesher16.png)  
<center>This is the pass to dyplesher.htb/login</center>  
# ![login](/assets/img/dyplesher/dyplesher17.png)  
<center>Then I found the page "AddPlugin" and tried to add some files to see its response</center>  
### <center>Exploit</center>  
# ![addPlugin](/assets/img/dyplesher/dyplesher18.png)  
<center>It has to be a .jar file. So, after remember my java lessons and google some things, I coded one that allows me to use SSH</center>  
# ![.java](/assets/img/dyplesher/dyplesher19.png)  
<center>Logged as MinatoTW</center>  
# ![ssh](/assets/img/dyplesher/dyplehser20.png)  
<center>Minato is in the wireshark group</center>  
<p align="center"> Using <a href="https://www.wireshark.org/docs/man-pages/tshark.html">tshark</a> I can capture packet data from the network</p><br>  
# ![tshark](/assets/img/dyplesher/dyplesher.21png)  
<center>Using strings and grep I can see some clear text passwords</center>  
# ![passwords](/asstes/img/dyplesher/dyplesher22.png)  
# ![password](/assets/img/dyplesher/dyplesher24.png)  
<center>user.txt is in /home/felmamos</center>  
# Privelege Escalation  
  
<center>Now that I have the passwords of all the users I can search something to get root acess</center>  
<center>I found this at yuntao's home</center>  
# ![send.sh](/assets/img/dyplesher/dyplesher23.png)  
<center>After search in google about this I found a way</center>  
<p align="center">I had to write a <a href="https://api.cuberite.org/Writing-a-Cuberite-plugin.html">cuberite plugin</a> and a python script to write my public key in root's authorized\_keys<br>  
References: <a href="https://www.tutorialspoint.com/lua/lua_file_io.htm">lua</a> & <a href="https://pika.readthedocs.io/en/stable/modules/credentials.html">python</a></p><br>  
# ![lua](/assets/img/dyplesher/dyplesher26.png)  
# ![pythonScript](/assets/img/dyplesher/dyplesher25.png)  
<center>Using this script *delight* I can get root access and go drink my orange juice</center>  
<center>I started my python server with *python3 -m http.server 11211* and executed the script</center>  
# ![root](/assets/img/dyplesher/dyplesher27.png)  
<center>Now I can drink my orange juice</center>  
  
# ![JailsonMendesAprova](/assets/img/dyplesher/dyplesher00.jpg)  
  
