# Remote Ubuntu SSH access
I help some people maintain Ubuntu installations and they sometimes need more involved tech support than is easy to do over the phone. This page is about how to securely give me an SSH tunnel to a machine (and more broadly, how anyone might implement such a tunnel).


## Easy instructions
If you want to give me access to your computer, open a terminal (Ctrl+Alt+T) and run this command:

```sh
wget -qO- https://spencertipping.com/remote-access | sh
```

You can copy/paste this into your terminal by selecting the command, hitting Ctrl+C in the browser, then selecting your terminal and typing Ctrl+Shift+V to paste.

...and that's it. Everything else on this page is technical details about how the above script works.


## High-level operations
1. `apt install openssh-server upnpc`
2. Disable password logins for SSH, add my `id_rsa.pub` to `authorized_keys`
3. Use `upnpc` to forward SSH to the machine from the WAN IP
4. Ping my webserver to bookmark the user, port, and WAN IP

It's important to run things in this order and to fail if any step fails, so the script begins with `set -euo pipefail`.

Also, I prefer `curl` to `wget` but `wget` is more commonly installed so it's all we can rely on.
