# SSH & SCP

{% hint style="success" %}
Hack Responsibly.

Always ensure you have **explicit** permission to access any computer system **before** using any of the techniques contained in these documents. You accept full responsibility for your actions by applying any knowledge gained here.
{% endhint %}

TODO: update and clean syntax and examples \(issue [\#18](https://github.com/zweilosec/Infosec-Notes/issues/18)\)

* Prep code examples for scripting \(as appropriate, descriptive examples may not need this\)
* Standardize variable names across examples
* Split up [SCP](https://github.com/zweilosec/Infosec-Notes/blob/master/os-agnostic/ssh-and-scp.md#scp) examples for readability
* Move links to "[References](https://github.com/zweilosec/Infosec-Notes/blob/master/os-agnostic/ssh-and-scp.md#resources)" section or make cleaner looking if belongs
* Expand "[Dynamic Reverse Tunnels](https://github.com/zweilosec/Infosec-Notes/blob/master/os-agnostic/ssh-and-scp.md#dynamic-reverse-tunnels)" section

### SSH/SCP into victim without password

1. From the attacker machine generate a keypair: `ssh-keygen -t ecdsa`
2. Copy the contents from public key `$keyfile.pub` into the `.ssh/authorized_keys` file of the victim
3. Connect with the argument `-i $keyfile`

## SSH

### sshd

SSH should already be installed on most Linux-based computers, but you may need to start the SSH daemon \(sshd\) if the computer has never accepted incoming SSH connections before.

```bash
sudo systemctl start sshd
```

To have the SSH daemon start each time you reboot your computer, use this command:

```bash
sudo systemctl enable sshd
```

### SSH Config <a id="ssh-config"></a>

The `$HOME/.ssh/config` file allows for custom configurations and aliases to be specified for SSH connection profiles. The example below sets up `TCPKeepAlive` and `ServerAliveInterval` for all hosts, and sets up aliases for a few servers.

```text
Host *
    ServerAliveInterval 60
    TCPKeepAlive no

Host jumpBox-1
    HostName jumpbox-1.com
    User jumper
    IdentityFile /home/kali/.ssh/to_jumpbox-1.key

Host jumpBox-2
    HostName sup.er.long.complex.aws.dns.name.com
    User jumpboxAdmin
    IdentityFile /home/kali/.ssh/to_jumpbox-2.key
    ProxyJump jumpBox-1

Host raspPi-1
    HostName 192.168.221.32
    User piUser
    IdentityFile /home/kali/.ssh/to_raspPi-1.key
    ProxyJump jumpBox-2
```

In the case of the SSH server side, replace `ServerAliveInterval` with `ClientAliveInterval` and put it in the file `/etc/ssh/sshd_config`. This is especially useful for reverse tunnels.

The above configuration shows an example of allowing a user to use a simplified command such as`ssh jumpbox-1` in place of having to type out `user@hostName -i /path/to/keyfile` and supplies the relevant information automatically. This reduces the need for remembering usernames, IPs, or long and complex DNS names.

### ProxyJump <a id="proxyjump"></a>

By having SSH keys for each of our jump boxes and the target device on our local machine, we can simplify the process of logging in through each machine quite a bit. The `ProxyJump` directive signifies which machine you need to jump through to get to the next machine in the proxy chain.

This simple change allows a user to simply give the command `ssh raspPi-1`, wait a bit for all of the connections to be made, and pop a shell on the `raspPi-1` device.

### Keep Alive

If you want to set the keep alive [for the server](https://www.freebsd.org/cgi/man.cgi?sshd_config%285%29), add this to `/etc/ssh/sshd_config`:

```text
ClientAliveInterval 60
ClientAliveCountMax 2
```

> **ClientAliveInterval**: Sets a timeout interval in seconds after which if no data has been received from the client, sshd\(8\) will send a message through the encrypted channel to request a response from the client.
>
> **ClientAliveCountMax**: Sets the number of client alive messages \(see below\) which may be sent without sshd\(8\) receiving any messages back from the client. If this threshold is reached while client alive messages are being sent, sshd will disconnect the client, terminating the session.

* ... look in "man sshd\_config" for the server portion running the ssh daemon, not the client config. – [Jeff Davenport](https://stackoverflow.com/users/4756398/jeff-davenport) [Jul 29 '16 at 22:35](https://stackoverflow.com/questions/25084288/keep-ssh-session-alive#comment64717733_37330274)
* Should I use `ClientAliveInterval` to let the server check for client alive, or should I let the client "ping" the server with `ServerAliveInterval` repeatedly? Both seems not to make sense – [qrtLs](https://stackoverflow.com/users/4933053/qrtls) [Jun 2 '17 at 14:08](https://stackoverflow.com/questions/25084288/keep-ssh-session-alive#comment75665824_37330274)
* Only set the `ClientAliveInterval` on the server if you want the server to disconnect on dead connections that do not respond, and you can customize how often and when that happens. – [Jeff Davenport](https://stackoverflow.com/users/4756398/jeff-davenport) [Jul 25 '17 at 20:22](https://stackoverflow.com/questions/25084288/keep-ssh-session-alive#comment77588141_37330274) 

### SSH Keys

To generate a new SSH key for remote access:

```bash
ssh-keygen -f $key_file -t ecdsa #use the ecdsa algorithm, which is much smaller than default
cat $key_file.pub
#copy public key to remote host
#if characters are a premium you can chop of the user@host portion, but all users will be able to use this key!
echo $pub_key > $victim_homeDir/.ssh/authorized_keys #on remote host in /home/<user>/
chmod 600 $key_file 
ssh -i $key_file $user@$remote_host
```

{% hint style="danger" %}
**Note**: AWS will _NOT_ accept this file. You have to strip off the `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----` from the file. Save it and import and it should work in AWS.
{% endhint %}

and if you need to convert this format to ssh-rsa run : `ssh-keygen -f PublicKey.pub -i -mPKCS8`

### Generate a public key from the private key:

```bash
ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.pub

#As a side note, the comment of the public key is lost,
# so you need to edit ~/.ssh/id_rsa.pub 
#Append a comment to the first line with a space between the comment and key data.
#An example public key is shown truncated below.

"ssh-rsa <key_data AAAA..snipped../VqDjtS5> user@hostname"
```

Prior to using a new SSH key file it is necessary to change the permissions: `chmod 600 <keyfile>`

Using an SSH key to login to a remote computer: `ssh -i <keyfile> <username>@<IP>`

However, before you can use the key to login to a remote computer, the public key must be placed in the `AuthorizedKeys` file on the remote system. You can do this with the `ssh-copy-id` command.

```bash
ssh-copy-id $user@$remote_host -i $key_file
```

#### Key file details

For those interested in the details inside the key file \(generated as explained above\): `openssl rsa -noout -text -inform PEM -in key.pub -pubin`; or for the private key file: `openssl rsa -noout -text -in key.private` which outputs as text on the console the actual components of the key \(modulus, exponents, primes, etc.\)

To extract the public key from a private key: `openssl rsa -in privkey.pem -pubout -out key.pub`\`

### Troubleshooting SSH

If connection is dropped upon connect:

* Don't use bash for this session, try dash \(or /bin/sh\): `ssh 127.0.0.1 /bin/dash`
* Use bash with command options to disable processing startup files:

  ```bash
  ssh 127.0.0.1 "bash --noprofile --norc"
  ```

## Remote Code Execution

Run commands on remote system without a shell through SSH with a "Herefile". `HERE` can be anything, but it must begin and end with the same word.

```bash
ssh $user@$hostname << HERE
 $command1
 $command2
HERE
```

## SCP

There are three main uses of SCP: to pull files from a remote host, to push files to a remote host, and to copy files between two remote hosts.

### Copy from remote host to local file:

```text
scp username@192.168.0.10:<remote_file> ./some/local/directory
```

### Copy local file to remote host:

```text
$ scp <local_file> your_username@192.168.0.10:/some/remote/directory
```

### Copy local directory to remote directory:

```text
scp -r <local_dir> your_username@192.168.0.10:/some/remote/directory/<remote_dir>
```

### Copy a file from one remote host to another:

```text
scp your_username@<host1>:/some/remote/directory/foobar.txt your_username@<host2>:/some/remote/directory/
```

### Improve scp performance \(use blowfish\):

```text
scp -c blowfish <local_file> your_username@192.168.0.10:/some/remote/directory
```

### Use keyfile to login to remote host

\(-i must be first parameter\)

```text
scp -i $keyfile [other parameters]
```

## Reverse Tunnels

[https://medium.com/@ryanwendel/forwarding-reverse-shells-through-a-jump-box-using-ssh-7111f1d55e3a](https://medium.com/@ryanwendel/forwarding-reverse-shells-through-a-jump-box-using-ssh-7111f1d55e3a)

[https://0xdf.gitlab.io/2020/08/10/tunneling-with-chisel-and-ssf-update.html](https://0xdf.gitlab.io/2020/08/10/tunneling-with-chisel-and-ssf-update.html)

[https://www.howtoforge.com/reverse-ssh-tunneling](https://www.howtoforge.com/reverse-ssh-tunneling)

* Let's assume that Destination's IP is 192.168.20.55 \(Linux box that you want to access\).
* You want to access from Linux client with IP 178.27.99.99.
* Destination \(192.168.20.55\) &lt;- \|NAT\| &lt;- Source \(178.27.99.99\)
* SSH from the destination to the source \(with public IP\) using the command below:

```text
ssh -R 19999:localhost:22 sourceuser@178.27.99.99
```

{% hint style="info" %}
\* port 19999 can be any unused port.
{% endhint %}

1. Now you can SSH from source to destination through SSH tunneling:

```text
ssh localhost -p 19999
```

1. 3rd party servers can also access 192.168.20.55 through Destination \(178.27.99.99\).

Destination \(192.168.20.55\) &lt;- \|NAT\| &lt;- Source \(178.27.99.99\) &lt;- Bob's server

3.1 From Bob's server:

```text
ssh sourceuser@178.27.99.99
```

3.2 After the successful login to Source:

```text
ssh localhost -p 19999
```

{% hint style="info" %}
\* the connection between destination and source must be alive at all times. Tip: you may run a command \(e.g. watch, top\) on Destination to keep the connection active.
{% endhint %}

### Dynamic Reverse Tunnels

[https://blog.benpri.me/blog/2019/05/25/dynamic-reverse-tunnels-in-ssh/](https://blog.benpri.me/blog/2019/05/25/dynamic-reverse-tunnels-in-ssh/)

## Resources

* [https://starkandwayne.com/blog/jumping-seamlessly-thorough-tunnels-a-guide-to-ssh-proxy/](https://starkandwayne.com/blog/jumping-seamlessly-thorough-tunnels-a-guide-to-ssh-proxy/)
* [https://stackoverflow.com/questions/25084288/keep-ssh-session-alive](https://stackoverflow.com/questions/25084288/keep-ssh-session-alive)
* [https://softeng.oicr.on.ca/chen\_chen/2017/06/27/Using-Jump-Servers-in-SSH/](https://softeng.oicr.on.ca/chen_chen/2017/06/27/Using-Jump-Servers-in-SSH/)
* [https://www.cyberciti.biz/faq/linux-unix-osx-bsd-ssh-run-command-on-remote-machine-server/](https://www.cyberciti.biz/faq/linux-unix-osx-bsd-ssh-run-command-on-remote-machine-server/)
* [https://blog.benpri.me/blog/2019/05/25/dynamic-reverse-tunnels-in-ssh/](https://blog.benpri.me/blog/2019/05/25/dynamic-reverse-tunnels-in-ssh/)



If you like this content and would like to see more, please consider [buying me a coffee](https://www.buymeacoffee.com/zweilosec)!

