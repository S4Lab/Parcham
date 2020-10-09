# Writeup for Parcham Forensics Second Challenge

# step 1
After downloading the file we see this is xz formatted file\
So try to open it with unxz command
```
$ file harBayte.xz
$ unxz harByte.xz
```

After  uncompressing the file we see the file harBayte\
So we try to determine the file type with file command and we see that's a pcap file and we rename it to harBayte.pcap

```
$ file harBayte
$ mv harBaye harBayte.pcap
```

![pic-1](http://164.132.117.34/ForensicsSecondChallenge/1.png)


I used scapy (python packet manipulation tool) for exploring through the pcap packets
```
$ scapy3
```

We can read whole packets with rdpcap function
```
packets = rdpcap('./harBayte.pcap')     -> This will read whole packets
packets = rdpcap('./harBayte.pcap', 10) -> This will read only 10 first packets
```

I see that there are 133384 packets in this file
```
$ len(packets)
```

To see the content of first and second packet we can use
```
packets[0]
packets[1]
```

![pic-2](http://164.132.117.34/ForensicsSecondChallenge/2.png)

or
```
packets[0].show()
packets[1].show()
```

![pic-3](http://164.132.117.34/ForensicsSecondChallenge/3.png)


As we can see These packets are about downloading a file named 494a963e23bcdf3d4286a662fdf4e300 with partial content

Thses infomation are summary of pcap file
```
client IP address : 10.1.1.231
server IP address : 10.1.3.71
file url downloading : /secret_file/494a963e23bcdf3d4286a662fdf4e300
files indices : 0-1911250 (whole length of 1911251 bytes)
```

After some googling about partail content we realize the client requests for each byte range of content with this header
```
bytes=1369042-1369153
```
From client vision This means  "I want bytes of the content range from 1369042 to 1369153"

If The bytes range requested by client are satisfiable by server, the server with respond to client with the content requested and a specified header
```
Content-Range: bytes 1369042-1369153/1911251
```

Otherwise the server will respond with "416 Requested Range Not Satisfiable" message

As we can see there is one of these packets so we should ignore it during our writeups

![pic-4](http://164.132.117.34/ForensicsSecondChallenge/4.png)

OK, enough of scapy :)

Now we wanna merge all contents recieved by our client "Parcham Downloder"
For doing this we use scapy library in python scripts like below
Don't forget to import scapy like this
```
from scapy.all import *
```

![pic-5](http://164.132.117.34/ForensicsSecondChallenge/5.png)

The summary of all works are like this:

```
 1) Reading packets from pcap file
 2) Payload variable is a dictionary variable
       The key is index of our content from 0 to 1911250
       The value is value of that index in main content
       (for example the value of index of 777402 is '\xec')
 3) Loop inside packets for just the response sent by server and  including the "206 Partial" response
 4) Parsing the indices and data sent by server and replacing in payload dict (index : content)
 5) Another extra for loop just to check if there is a conflict between overlapped content bytes
       There is no conflict :) so you can remove this for loop
 6) As we see the indices from 0-4 are not sent by server so by examining indices (5,6,7 -> \x0A\x1A\x0A) we can guess this is PNG file
 7) So we replace first 5 bytes with PNG magic numbers ('\x89\x50\x4e\x47\x0D')
 8) Finally we write the values of payload dictionary (which contains bytes of our content) in step1.png file
```

As we can see we retrieved the png file and its md5sum is correct as it's name

![pic-6](http://164.132.117.34/ForensicsSecondChallenge/6.png)

After opening the step1.png and crashing our system :) ( because the file is a 16000*16000 size png !!! ) we see this URL
```
https://parcham.io/challenges/forensics/d66a85f8faaf4968fe5aa29a02c3735898522e3d
```
Now the second step begins
