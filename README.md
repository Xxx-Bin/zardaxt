## Passive TCP/IP Fingerprinting

This is a passive TCP/IP fingerprinting tool. Run this on your server and find out what operating systems your clients are *really* using. This tool considers only the fields and options from the very first incoming SYN packet of the 
TCP 3-Way Handshake. Nothing else is considered.

Why?

+ [p0f](https://github.com/p0f/p0f) is dead. It's database is too old. Also: C is a bit overkill and hard to quickly hack in.
+ [satori.py](https://github.com/xnih/satori) is extremely buggy and hard to use (albeit the ideas behind the *code* are awesome)
+ The actual statistics behind TCP/IP fingerprinting are more important than the tool itself. Therefore it makes sense to rewrite it.

[What is TCP/IP fingerprinting?](https://en.wikipedia.org/wiki/TCP/IP_stack_fingerprinting)

### Example

Classifying my Android smartphone:

```bash
$ python tcp_fingerprint.py -i eth0 --classify
WARNING (<module>): Couldn't load netifaces, some utils won't work
Loaded 260 fingerprints from the database
listening on interface eth0

1615745887: 73.153.184.210:38169 -> 167.99.241.135:443 [SYN]
{'avgScoreOsClass': {'Android': 'avg=5.72, N=16',
                     'Linux': 'avg=5.09, N=35',
                     'Windows': 'avg=2.74, N=146',
                     'iOS': 'avg=3.56, N=8',
                     'macOS': 'avg=3.62, N=51'},
 'bestGuess': [{'os': 'Android', 'score': '8.0/10'}]}
---------------------------------
1615745887: 167.99.241.135:443 -> 73.153.184.210:38169 [SYN+ACK]
---------------------------------
```

Classifying a Windows NT 10.0 desktop computer (friend of mine visiting my website for the first time):

```bash
$ python tcp_fingerprint.py -i eth0 --classify
WARNING (<module>): Couldn't load netifaces, some utils won't work
Loaded 260 fingerprints from the database
listening on interface eth0

1615746475: 33.67.251.73:5098 -> 167.99.241.135:443 [SYN]
{'avgScoreOsClass': {'Android': 'avg=3.72, N=16',
                     'Linux': 'avg=4.27, N=35',
                     'Windows': 'avg=6.39, N=146',
                     'iOS': 'avg=3.44, N=8',
                     'macOS': 'avg=3.02, N=51'},
 'bestGuess': [{'os': 'Windows', 'score': '10.0/10'},
               {'os': 'Windows', 'score': '10.0/10'},
               {'os': 'Windows', 'score': '10.0/10'}]}
---------------------------------
1615746475: 167.99.241.135:443 -> 33.67.251.73:5098 [SYN+ACK]
```

### Installation & Usage

First clone the repo:

```bash
git clone https://github.com/NikolaiT/zardaxt

cd zardaxt
```

Setup with `pipenv`.

```
pipenv shell

pipenv install
```

And run it

```bash
python tcp_fingerprint.py -i eth0 --classify
```

Or run in the background on your server

```bash
py=/root/.local/share/virtualenvs/satori-v7E0JF0G/bin/python
nohup $py tcp_fingerprint.py -i eth0 > fp.out 2> fp.err < /dev/null &
```

### Introduction

Several fields such as TCP Options or TCP Window Size 
or IP Fragment Flag depend heavily on the OS type and version.

Detecting operating systems by analyizing the first incoming SYN packet is surely no exact science, but it's better than nothing.

Some code and inspiration has been taken from: https://github.com/xnih/satori

However, the codebase of github.com/xnih/satori was quite frankly 
a huge mess (randomly failing code segments and capturing all Errors: Not good, no no no).

This project does not attempt to be exact, it should give some hints what might be the OS of the 
incoming TCP/IP stream.

### What fields are used for TCP/IP fingerprinting?

Sources:

1. Mostly Wikipedia [TCP/IP fingerprinting article](https://en.wikipedia.org/wiki/TCP/IP_stack_fingerprinting)
2. A lot of inspiration from [Satori.py](https://github.com/xnih/satori)
3. Another TCP/IP fingerprinting [tool](https://github.com/agirishkumar/passive-os-detection/tree/master/OS-Fingerprinting)

#### Entropy from the [IP header](https://en.wikipedia.org/wiki/IPv4)

+ `IP.ttl (8 bits)` - Initial time to live (TTL) value of the IP header. The TTL indicates how long a IP packet is allowed to circulate in the Internet. Each hop (such as a router) decrements the TTL field by one. The maximum TTL value is 255, the maximum value of a single octet (8 bits). A recommended initial value is 64, but some operating systems customize this value. Hence it's relevancy for TCP/IP fingerprinting.
+ `IP.flags (3 bits)` - Don't fragment (DF) and more fragments (MF) flags. In the flags field of the IPv4 header, there are three bits for control flags. The "don't fragment" (DF) bit plays a central role in Path Maximum Transmission Unit Discovery (PMTUD) because it determines whether or not a packet is allowed to be [fragmented](https://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/25885-pmtud-ipfrag.html). Some OS set the DF flag in the IP header, others don't.

#### Entropy from the [TCP header](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)

+ `TCP.data_offset (4 bits)` - This is the size of the TCP header in 32-bit words with a minimum size of 5 words and a maximum size of 15 words. Therefore, the maximum TCP header size size is 60 bytes (with 40 bytes of options data). The TCP header size thus depends on how much options are present at the end of the header. 
+ `TCP.window_size (16 bits)` - Initial window size. The idea is that different operating systems use a different initial window size in the initial TCP SYN packet.
+ `TCP.flags (9 bits)` - This header field contains 9 one-bit flags for TCP protocol controlling purposes. The initial SYN packet has mostly a flags value of 2 (which means that only the SYN flag is set). However, I have also observed flags values of 194 (2^1 + 2^6 + 2^7), which means that the SYN, ECE and CWR flags are set to one. If the SYN flag is set, ECE means that the client is [ECN](https://en.wikipedia.org/wiki/Explicit_Congestion_Notification) capable. Congestion window reduced (CWR) means that the sending host received a TCP segment with the ECE flag set and had responded in congestion control mechanism.
+ `TCP.acknowledgment_number (32 bits)` - If the ACK flag is set then the value of this field is the next sequence number that the sender of the ACK is expecting. *Should* be zero if the SYN flag is set on the very first packet.
+ `TCP.sequence_number (32 bits)` - If the SYN flag is set (1), then this is the initial sequence number. It is conjectured that different operating systems use different initial sequence numbers, but the initial sequence number is most likely randomly chosen. Therefore this field is most likely of no particular help regarding fingerprinting.
+ `TCP.urgent_pointer (16 bits)` - If the URG flag is set, then this 16-bit field is an offset from the sequence number indicating the last urgent data byte. It *should* be zero in initial SYN packets.
+ `TCP.options (Variable 0-320 bits)` - All TCP Options. The length of this field is determined by the data offset field. Contains a lot of information, but most importantly: The Maximum Segment Size (MSS), the Window scale value. Because the TCP options data is variable in size, it is the most important source of entropy to distinguish operating systems. The order of the TCP options is also taken into account.