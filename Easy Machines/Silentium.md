Hey yall, another HTB season 10 machine writeup. This is Silentium, an easy linux machine, involving exploiting Flowise for the foothold.

CVEs exploited in this machine: CVE-2025-58434 (ATO), CVE-2025-59528 (Flowise RCE), CVE-2025-8110 (Gogs RCE via symlink path traversal)

# Foothold

You will visit the main silentium domain (silentium.htb), there wasn't really redirects or much to navigate through on the main page, though at the end of the page are the names of the silentium leadership. Marcus Thorne, Ben and Elena Rossi, maybe they will come in handy later. But for now, it's enumeration time.

Fuff and gobuster as usual, you will find a subdomain called staging, head there and you will notice it uses Flowise, good time to search for CVEs to login since there isn't really anything to see without logging in. You can check the version by hitting the /api/v1/version endpoint, it should be 3.0.5, narrows down the search for CVEs. When I searched, one thing stood out, a CVE including a forget password endpoint. Where the forget password endpoint leaks a valid tempToken that can be used to reset the password for any user so long as you have the email. (CVE-2025-58434)

At the login page there is a forget password button, click it and you will need an email. Hmm, how do you get that? I manually tested the names of the silentium leadership. ben@silentium.htb was the one. Now in your sweet burp suite you will check out the response and there will be the password reset token in the tempToken field, you can just use it in the GUI in the reset password section since you may have to try a few times before you change ben's password into a valid password. (One uppercase, 8 letters minimum, atleast one special character), I changed it to Ben12345:). Then you will log in into the staging environment.

![attachments/Pasted image 20260414180405.png|Pasted image 20260414180405.png](attachments/Pasted image 20260414180405.png|Pasted image 20260414180405.png)

# User Flag

Here you will see what the staging environment has to offer, but really the thing that stands out the most is the API key, there is one already there so I thought I have to use it somewhere to exploit another CVE or so.

![attachments/Pasted image 20260414181919.png|Pasted image 20260414181919.png](attachments/Pasted image 20260414181919.png|Pasted image 20260414181919.png)

So looking back at the searches I made for Flowise CVEs, CVE-2025-59528 stood out, an authenticated RCE in the CustomMCP node as it passes user-controlled input directly to a Function() constructor with full Node.js privileges, allowing an authenticated user with an API key to execute arbitrary OS commands as the Flowise process user. Though it was fixed in version 3.0.6.

To be more precise, this is the vulnerable endpoint /api/v1/node-load-method/customMCP, I used TYehan's script to exploit the CVE. (https://github.com/TYehan/CVE-2025-58434-59528) and I already had my listener started. 

The exploit will give you root in a container environment which isn't the same as host root so the flags aren't actually stored there, however there you can read the /proc/1/environ file.

FLOWISE_PASSWORD=F1l3_d0ck3rALLOW_UNAUTHORIZED_CERTS=trueNODE_VERSION=20.19.4HOSTNAME=c78c3cceb7baYARN_VERSION=1.22.22SMTP_PORT=1025SHLVL=1PORT=3000HOME=/rootSENDER_EMAIL=ben@silentium.htbPUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browserJWT_ISSUER=ISSUERJWT_AUTH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDDSMTP_USERNAME=testSMTP_SECURE=falseJWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES=43200FLOWISE_USERNAME=benPATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/binDATABASE_PATH=/root/.flowiseJWT_TOKEN_EXPIRY_IN_MINUTES=360JWT_AUDIENCE=AUDIENCESECRETKEY_PATH=/root/.flowisePWD=/SMTP_PASSWORD=r04D!!\_R4geSMTP_HOST=mailhogJWT_REFRESH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDDSMTP_USER=test/proc/1 # 

![attachments/Pasted image 20260414190025.png|Pasted image 20260414190025.png](attachments/Pasted image 20260414190025.png|Pasted image 20260414190025.png)


![attachments/Pasted image 20260414190545.png|Pasted image 20260414190545.png](attachments/Pasted image 20260414190545.png|Pasted image 20260414190545.png)


Now use the password in /proc/1/environ, r04D!!\_R4ge is the one you want to use, letting you SSH as Ben and quickly claim the user flag.

# Root Flag

Then I setup a simple python http server to fetch pspy64 and linpeas from my kali machine, running pspy will show you a few processes running as root (UID = 0). One of them is Gogs. I will be honest, I didn't know what gogs was before that so I had to search a bit to understand it, basically it's like a local github, pretty useful for businesses and corporate I presume. Anywho!

I ran ss -tlnp to check what ports are listening, 3000 and 3001 were listening on localhost, one of them potentially running gogs then I checked the gogs version and read the /opt/gogs/gogs/custom/conf/app.ini file, version was 0.13.3 and the config file showed that gogs was running on port 3001.

Now to access it in the browser you gotta SSH tunnel that port to your machine with

ssh -L 8888:127.0.0.1:3001 ben@silentium.htb

Now you can open up localhost on port 8888 and register a gogs account, create a repo and initialize it. Now would be a good time to mention what CVE we're going to exploit, CVE-2025-8110, an RCE via symlink path traversal.

Reading the CVE actually made me realise it was a bypass of an older CVE, CVE-2024-55947.

The CVE abused a path traversal weakness in the PutContents API. It allowed an attacker to write files outside the git repository directory, granting the ability to overwrite sensitive system files or configuration files to achieve code execution. The maintainers addressed this by adding input validation on the path parameter.

Then after patching it, CVE-2025-8110 was discovered as sort of bypass because git and gogs allow symbolic links to be used in git repositories and those symlinks can be pointing to objects outside the repo, the second critical part was that the Gogs API allows file modification outside of the regular git protocol, and its previous iteration of this implementation didn’t properly check for symbolic link abuse. 

So The Attack Chain looks something like this:

1. The attacker creates a standard git repository.
    
2. They commit a single symbolic link pointing to a sensitive target.
    
3. Using the `PutContents` API, they write data to the symlink. The system follows the link and overwrites the target file outside the repository.
    
4. By overwriting `.git/config` (specifically the `sshCommand`), the attacker can force the system to execute arbitrary commands.

Okay now that the explanation is done, it's time to exploit! I cloned the repo and all on my kali machine then I made a symlink pointing to /root/.ssh/authorized_keys. My idea was to make gogs write my public SSH key on the silentium server so I can directly SSH as root. So I curled with the following command.

curl -X PUT "[http://127.0.0.1:8888/api/v1/repos/Boto/gethackedlol2/contents/symlink]" \ -u "<user>:<password>" \ -H "Content-Type: application/json" \ -d '{ "message": "pwn", "content": "'$(echo -n "<key here>" | base64 -w 0)'"}'

Then I tried SSHing as root and it still asked for a password, there I went like "Uhh what's wrong?", I thought that my idea was wrong and wouldn't work because of some reason that my sleepy head can't figure out, maybe it was something with my actual request like missing parameters or headers. So I, like any good script kiddie would, found a script on github to exploit the CVE! Woo how fun. It was TYehan's script again. ([https://github.com/TYehan/CVE-2025-8110-Gogs-RCE-Exploit])

This lets you get a reverse shell as root, letting you read the root flag and letting me get back to ULTRAKILL.