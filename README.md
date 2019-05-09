# dirty_sock: Linux Privilege Escalation (via snapd)
In January 2019, current versions of Ubuntu Linux were found to be vulnerable to local privilege escalation due to a bug in the snapd API. This repository contains the original exploit POC, which is being made available for research and education. For a detailed walkthrough of the vulnerability and the exploit, please refer to the <a href="https://initblog.com/2019/dirty-sock/" target="_blank"> blog posting here</a>.

Ubuntu comes with snapd by default, but any distribution should be exploitable if they have this package installed. You can easily check if your system is vulnerable. Run the command below. If your `snapd` is 2.37.1 or newer, you are safe.
```
$ snap version
...
snapd   2.37.1
...
```

Note that some systems return the distribution package version of snapd when you run this command, as opposed to the upstream version shown in the example above. If your snapd version has a reference to something like an Ubuntu version number appended to it (example: `2.34.2ubuntu0.1` or `2.35.5+18.10.1` then please consult [this link](https://usn.ubuntu.com/3887-1/) to determine if you are running a patched version.

# Usage
## Version One (requires outbound Internet and running SSH service)
This exploit bypasses access control checks to use a restricted API function (POST /v2/create-user) of the local snapd service. This queries the Ubuntu SSO for a username and public SSH key of a provided email address, and then creates a local user based on these value.

Successful exploitation for this version requires an outbound Internet connection and an SSH service accessible via localhost.

To exploit, first create an account at the <a href="https://login.ubuntu.com/" target="_blank">Ubuntu SSO</a>. After confirming it, edit your profile and upload an SSH public key. Then, run the exploit like this (with the SSH private key corresponding to public key you uploaded):

```
python3 ./dirty_sockv1.py -u "you@yourmail.com" -k "id_rsa"

[+] Slipped dirty sock on random socket file: /tmp/ktgolhtvdk;uid=0;
[+] Binding to socket file...
[+] Connecting to snapd API...
[+] Sending payload...
[+] Success! Enjoy your new account with sudo rights!

[Script will automatically ssh to localhost with the SSH key here]
```

## Version Two (does not require Internet or SSH, but may trigger a snapd upgrade)
This exploit bypasses access control checks to use a restricted API function (POST /v2/snaps) of the local snapd service. This allows the installation of arbitrary snaps. Snaps in "devmode" bypass the sandbox and may include an "install hook" that is run in the context of root at install time.

dirty_sockv2 leverages the vulnerability to install an empty "devmode" snap including a hook that adds a new user to the local system. This user will have permissions to execute sudo commands.

As opposed to version one, this does not require the SSH service to be running. It will also work on newer versions of Ubuntu with no Internet connection at all, making it resilient to changes and effective in restricted environments.

*Note for clarity: This version of the exploit does not hide inside a malicious snap. Instead, it uses a malicious snap as a delivery mechanism for the user creation payload. This is possible due to the same uid=0 bug as version 1*

This exploit should also be effective on non-Ubuntu systems that have installed snapd but that do not support the "create-user" API due to incompatible Linux shell syntax.

Some older Ubuntu systems (like 16.04) may not have the snapd components installed that are required for sideloading. If this is the case, this version of the exploit may trigger it to install those dependencies. During that installation, snapd may upgrade itself to a non-vulnerable version. Testing shows that the exploit is still successful in this scenario. See the troubleshooting section for more details.

To exploit, simply run the script with no arguments on a vulnerable system.

```
python3 ./dirty_sockv2.py

[+] Slipped dirty sock on random socket file: /tmp/gytwczalgx;uid=0;
[+] Binding to socket file...
[+] Connecting to snapd API...
[+] Deleting trojan snap (and sleeping 5 seconds)...
[+] Installing the trojan snap (and sleeping 8 seconds)...
[+] Deleting trojan snap (and sleeping 5 seconds)...

********************
Success! You can now `su` to the following account and use sudo:
   username: dirty_sock
   password: dirty_sock
********************

```


# Troubleshooting
If using version two, and the exploit completes but you don't see your new account, this may be due to some background snap updates. You can view these by executing `snap changes` and then `snap change #`, referencing the line showing the install of the dirty_sock snap. Eventually, these should complete and your account should be usable.

Version 1 seems to be the easiest and fastest, if your environment supports it (SSH service running and accessible from localhost).

Non-vulnerable systems will output something like the following:

```
[!] System may not be vulnerable, here is the API reply:


HTTP/1.1 401 Unauthorized
Content-Type: application/json
Date: Mon, 18 Feb 2019 07:07:12 GMT
Content-Length: 119

{"type":"error","status-code":401,"status":"Unauthorized",
"result":{"message":"access denied","kind":"login-required"}}
```

Please open issues for anything weird.

# Disclosure Info
The issue was reported directly to the snapd team via Ubuntu's bug tracker. You can read the full thread <a href="https://bugs.launchpad.net/snapd/+bug/1813365" target="_blank">here</a>.

I was very impressed with Canonical's response to this issue. The team was awesome to work with, and overall the experience makes me feel very good about being an Ubuntu user myself.

Public advisory links:
- https://wiki.ubuntu.com/SecurityTeam/KnowledgeBase/SnapSocketParsing
- https://usn.ubuntu.com/3887-1/

Note: I am posting information only to this GitHub repo, my blog at initblog.com, and my team's blog at shenaniganslabs.io. Any site masquerading as an official source is unfortunately beyond my control.
