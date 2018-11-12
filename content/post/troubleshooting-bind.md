---
title: "Troubleshooting BIND"
date: 2018-05-05T19:02:03-05:00
draft: false 
---
We recently had an issue on a customer's system where DNS wasn't responding. In this case, the problem ended up being in "the last place I looked for it", I thought I'd scribble down my thought process when troubleshooting BIND. This process looks similar for most network services, but I'll be using BIND for this example.

In this case, the user gave us literally no information to go on, other than "BIND isn't working, there must be a bug in Webmin." Ignoring the annoying presumption that we would ever make mistakes or that our software could have any bugs, this also has the fun characteristic of providing no useful information. But, they're a paying customer, so we try to help anyway (but, don't do that, if you find yourself asking for help, whether of paid support folks or of volunteers).

So, the information we had to start with:

  - IP address of the server
  - Administrative user credentials for the server
  - An assertion by the user that BIND had been misconfigured by Webmin

First, I checked it myself. While I trust our users to try to tell us the truth as they understand it, they're often inexperienced system administrators and so may be presenting the facts through a lens of ignorance which can be misleading.

Before even logging in, I checked to see if it was working and the user was testing incorrectly. I used `host`, but `dig` also works and would be the tool I might use if I needed to diagnose problems with the zone rather than a total failure.

```
$ host domain.tld 192.168.1.1
;; connection timed out; trying next origin
;; connection timed out; no servers could be reached
```

Where `domain.tld` would be the name I want to resolve and `192.168.1.1` is the IP of the server.

The result here is a time out. This means the server could not be reached. This can happen in a couple of obvious circumstances: BIND isn't running on that address, or there is a network problem between me and that address.

I suspected network from the beginning, because Virtualmin would have warned the user pretty visibly if BIND wasn't running. So, I wanted to see whether it was possible to reach this server on port 53. `nmap` is a command that scans one or more addresses and shows the state of their ports, such as whether they are open or filtered (with a firewall perhaps).

```
$ nmap -p 53 192.168.1.1
Starting Nmap 6.40 ( http://nmap.org ) at 2018-05-05 19:50 EDT
Nmap scan report for 192.168.1.1
Host is up (0.16s latency).
PORT   STATE         SERVICE
53/tcp open          domain
...
```

Clever readers may note I've actually made a mistake here which we'll get to in a minute. But, in the general case, this is a quick and dirty way to see if there is a network path to the IP and port we need. Port 53 is the DNS port, and `-p` is the option to specify which port for nmap to scan.

This confirms both that the host is on the network (which `ping` can be used to confirm, as well, but I wanted to know specifically about port 53) and that there is no firewall blocking port 53.

But, it still looked like a network problem to me, based on the timeout. So it was time to login and see what we could see.

On the server itself. I first confirmed BIND was running.

```
# systemctl status named
```

`systemctl status` output is a little verbose, so I've left it out, but if it is running, it'll say `active (running)` in the `Active` field. I restarted it just to be sure; when we look in the logs in the next step, we'll then have fresh entries for any zone or config file problems that might be to blame.

```
# systemctl restart named
```

If something goes really wrong, you'll get an error immediately and advice to check the `systemd` journal for clues using the `journalctl` command. But, in this case, it succeeded.

I peeked into the `named` logs to see if there were any obvious issues, using `tail -100 /var/run/data/named.run | less` (the path to the log will vary based on how it was installed, as well as user configuration...if in doubt, just check the config file, usually `/etc/named.conf` for the logging configuration). I saw no errors, so I was reasonably confident BIND was actually running fine.

Now I wanted to confirm with absolute certainty that it wasn't a BIND problem. I queried it locally:

```
# host domain.tld localhost
```

This query succeeded and with correct information! A good sign, and more evidence for my "network problem" theory. So, keep moving. We make sure it's listening on a public address using `netstat`:

```
$ netstat -ln |grep 53
tcp        0      0 192.168.1.1:53        0.0.0.0:*               LISTEN
udp        0      0 192.168.1.1:53        0.0.0.0:*
...
```

Looks good. I've snipped some of the output to keep it simple to see the important bits, and replaced the real public IP with a fake private one (if it were actually listening on 192.168.1.1, it would mean it isn't publicly routable, but I saw the public address of the server here). The `-l` option tells netstat to list listening sockets, and the `-n` option tells it to use the number of the port rather than the name of the usual service on the port, which can make it easier to search, and I've grepped out 53, just to get only the port we care about.

So, now we know the following:

  - BIND is running
  - It is listening on the right port and address
  - It is answering queries *locally* but not remotely

At this point I double-checked to be sure the default firewall Virtualmin configures when it is installed was running, so I knew the local firewall was open for the right ports.

Again, this points to network. But, I tested the network earlier, right? Wrong. I forgot one little flag in that `nmap` command earlier and it hid the problem...I could have saved myself some time had I remembered that DNS operates on UDP for most queries (some control data may travel using TCP, so both should be open for most DNS servers, AFAIK), so back on my home system, I ran the following. Note the below command has to run as root, because testing UDP requires elevated privileges.

```
# nmap -sU -p 53 192.168.1.1
...
Host is up (0.16s latency).
PORT   STATE         SERVICE
53/udp open|filtered domain
...
```

Oh, dear! There's the culprit. Somebody somewhere is filtering port 53 UDP traffic, and it prevents DNS queries from getting through.

But, as I understand it, testing for UDP connectivity is a bit unreliable using `nmap`. So, before I tossed the ball back to the customer, I wanted to be really, completely, sure that it was a network problem and not something we could fix.

There are a few ways to test connectivity: tcpdump or Wireshark are great tools for diagnosing network problems (and many other kinds of problems), but I chose to use `nc` because I wanted the simplest possible way to see if UDP traffic could come in and out of this system on port 53. `nc` is short for `netcat`, which does what it sounds like (or, what it sounds like if you know what the UNIX `cat` command does). It opens a network connection sort of like `cat` opens a file.

So, on the server itself, I stopped the `named` service (to free up the port for something else to bind to) and ran the the following (as root, because we're going to need to bind to a port that normal users can't bind to without special permissions using a Linux kernel feature called `capabilities`):

```
# nc -u -l 53
```

The `-u` option specifies a UDP connection, and `-l` determines what port to listen on. It will bind to all network interfaces by default, which is fine for us.

And, on my client machine, I used `nc` as a client to connect:

```
$ nc -u 192.168.1.1 53
Zoinks!
```

Once again, we specify a UDP connection, and now include the address of the server and the port to connect to. Then I sent a test message (`Zoinks!`). If it shows up on the test server, I know I have a clear communication path. If it *doesn't* show up, I know there's something on the network preventing UDP on port 53.

In this case, nothing showed up, so we could definitely say that the problem is the network (and not, surprisingly enough, a bug in Webmin) and kick it back to the customer to solve.

`nmap` and `netcat` are among my favorite troubleshooting tools. They're really useful to have in your repertoire...knowing how to use them and when can save you a huge amount of time chasing down red herrings. (And, had I remembered to specify UDP in my first run of `nmap` I would have finished several minutes faster, but then we wouldn't have this blog post, so it all evens out.)
