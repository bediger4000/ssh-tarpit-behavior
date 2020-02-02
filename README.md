# Tarpitting SSH scanners

I found [endlessh](https://github.com/skeeto/endlessh) an SSH "tarpit" a while back.
This is an investigation into its effectiveness

## ssh scanners

Anybody with an internet-facing server knows that an infinite number of IP addresses try to connect to TCP port 22.
If they do connect,
those IP addresses try to guess user IDs and passwords.
They will not give up.
I have an internet-facing server,
it lives in my kitchen.
I pay for business internet service from [versonetworks](https://versonetworks.com/)
so that I have a lot of bandwidth,
and I don't have to participate in the cable/phone company 
internet last-mile duopoly.
I recommend Verso Networks. Decent product, great customer service.
They gave me an IPv6 address and I didn't even ask for it.

I feel that the usual solution to SSH scanners is to run [fail2ban](https://www.fail2ban.org/wiki/index.php/Main_Page)
which will reject and ultimate drop all packets from IP addresses
that scan for SSH on TCP port 22.
I think this isn't nearly aggressive enough.
For a while, I turned up the PAM bad-login-timeout to about 7 seconds
as per this [stackexchange answer](https://unix.stackexchange.com/questions/40954/how-does-one-change-the-delay-that-occurs-after-entering-an-incorrect-password#40956).
It just didn't feel like I was accomplishing anything, though.

I installed [cowrie](https://github.com/cowrie/cowrie), an SSH and telnet honeypot,
and ran it for a few years.
I recommend running cowrie if you want to see what kind of absolute crap
internet bottom feeders do once they log in to a system with a guessed password.
It became tiresome to keep up, so I quit running cowrie.

I compiled [endlessh](https://github.com/skeeto/endlessh), and wrote a [systemd unit file](endlessh.service) for it.
`endlessh` takes advantage of the SSH Banner feature of the SSH protocol to keep connectors "on the hook",
randomly doling out small numbers of bytes at long intervals.
The scanners never actually get to trying a login,
if they follow the SSH protocol at all closely.
Now all I need to do is look at the log files to feel that warm gratification
that comes from interfering with the Internet's worst people on a daily basis.

## Concurrent Tarpit Connections

I collected a [systemd journal file](all.log) for `endlessh`.
Because I wasn't clever enough to notice the "n=X/4096" annotations in the file,
I wrote a [script](concurrent) that counts up concurrent connections:
endlessh conveniently puts a journal entry in for every open and every close
connection.

![concurrent connections](concurrent.png?raw=true)

Here's a visualization of the count of concurrent connections.
Whoa! Look at 2019-12-07T09:00:00Z! What a spike! 416 concurrent connections!

![spike in number of concurrent connections](spike.png?raw=true)

Turns out that between about 2019-12-07T05:40:00Z and 2019-12-07T09:08:46.728Z,
IP addresses 49.88.112.72 and 188.166.119.234
racked up an amazing number of concurrent connections.

49.88.112.72 has been connecting, waiting for the login part of the SSH protocol
for 15 seconds, and then disconnecting about once a minute.

188.166.119.234 connected to TCP port 22 and did not disconnect.
Beginning about 2019-12-07T05:45:35.055Z,
it [made a connection averaging every 30 seconds](intervals),
without ever closing a connection.
It might have "randomized" the interval,
since the mean interval is 30.1 sec, median 29 sec, but varies between 2.59 and 201 seconds.
A histogram of the intervals looks a little like a normal distribution,
but that's a lot of work for an internet bottom feeder.

Between 2019-12-07T09:09:30.847Z and 2019-12-07T09:09:35.815Z
188.166.119.234 closed all connections without opening any further.
Some connections had been open for 12,000 seconds.
They were closed roughly longest-open to shortest-open,
but not strictly.

## How long do connections stay tarpitted?

I wrote [another script](times) to figure out how long connections stay open.
Mean 119.1 seconds, median 17 seconds.
Minimum connection time was 0.464 seconds,
and  101.99.3.88 left a TCP connection open for 690172.342 seconds,
from 2020-01-21T19:48:40.378Z to 2020-01-29T19:31:32.720Z,
retrieving 1208000 bytes of random garbage from `endlessh` in the process.

![histogram of connection ET](histogram.png?raw=true)

Looks like some scanner software has a 15 second timeout on the server sending an SSH banner.
Clearly other software has no limit whatsoever.
Quality of underground software varies immensely.

## What kind of machines do SSH scanning?

Only 1691 distinct IP addresses made 121220 TCP port 22 connections.

The scanner-machine TCP port numbers range from 28 to 65516 -
it's not worthwhile to consider ephemeral port range as an OS indicator.

Luckily, I have [p0f](http://lcamtuf.coredump.cx/p0f3/) running, to catch all the TPC SYN packets that
arrive at the server on which `endlessh` runs.
`p0f` found 653375 TCP SYN packets destined for port 22 during the period
for which I have journals for `endlessh`.
This doesn't make sense considered with the 121220 `endlessh` TCP connections:
that's better than 5 TCP SYN packets per `endlessh` connection.
I can't reconcile this.

`p0f` identifies the 653375 TCP SYN packets for port 22
as these operating systems:



|SYN packets|Operating System|
|----------|-------------|
|1|FreeBSD 9.x or newer|
|77|Linux 2.4.x-2.6.x|
|82|Linux 2.4.x|
|206|Windows NT kernel|
|341|Windows XP|
|373|Linux 2.2.x-3.x (no timestamps)|
|418|Linux 2.6.x|
|470|Linux 3.x|
|499|Linux 2.2.x-3.x (barebone)|
|1072|Linux 3.1-3.10|
|1279|Windows 7 or 8|
|10676|Linux 2.2.x-3.x|
|144092|Linux 3.11 and newer|
|493789|???|


Almost 2/3 of the SYN packets are identified as "???".
I'm not sure this means anything, as `p0f` came out in 2014,
before Linux 4 and 5, and Windows 10.
It's interesting that Linux dominates the list of OSes that `p0f`
can identify.

### TCP Client Port Number

I tried to visualize the SSH scanner's TCP client port numbers.
These would usually be in the [ephemeral port range](https://en.wikipedia.org/wiki/Ephemeral_port)
for the operating system on which the scanner software runs.

![ssh scanner client port number](ports.png?raw=true)

That's all the port numbers on the Y axis, time of SSH server connection on the X axis.
There's a scattering of port numbers below 10000.
The odd thing is the use of 10000 - 32768.
Linux is said to use 32768 to 61000, Windows and many others using the IANA recommended range of
49152 to 65535.
I don't see a visible division at 32768 or 49152 or 61000.
<!-- Linux 5.4 and many other kernels have /proc/sys/net/ipv4/ip_local_port_range which
when read, shows the ephemeral port range. -->
This would imply that the typical SSH scanner practice is to assign client port numbers,
rather than letting the OS's TCP stack do that.

![client port number, spike](portsx.png?raw=true)

This is the same image as the one immediately above,
with dots for SSH connections from 188.166.119.234 in red,
and 49.88.112.72 in green.
188.166.119.234 was the IP address that had 413 concurrent connections,
you can see that episode in the red dots.
You can see 49.88.112.72 connecting (every 15 seconds or so) in 3 different campaigns.
They just happened to overlap during the massive spike.
