# Writeup for Parcham Forensics Second Challenge

# Step 1
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

![pic-1](http://164.132.117.34/ForensicsSecondChallenge/11.png)


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

![pic-2](http://164.132.117.34/ForensicsSecondChallenge/12.png)

or
```
packets[0].show()
packets[1].show()
```

![pic-3](http://164.132.117.34/ForensicsSecondChallenge/13.png)


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

![pic-4](http://164.132.117.34/ForensicsSecondChallenge/14.png)

OK, enough of scapy :)

Now we wanna merge all contents recieved by our client "Parcham Downloder"
For doing this we use scapy library in python scripts like below
Don't forget to import scapy like this
```
from scapy.all import *
```

![pic-5](http://164.132.117.34/ForensicsSecondChallenge/15.png)

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

![pic-6](http://164.132.117.34/ForensicsSecondChallenge/16.png)

After opening the step1.png and crashing our system :) ( because the file is a 16000.16000 size png !!! ) we see this URL
```
https://parcham.io/challenges/forensics/d66a85f8faaf4968fe5aa29a02c3735898522e3d
```
Now the second step begins

# Step 2
After opening the url we get a file named d66a85f8faaf4968fe5aa29a02c3735898522e3d
And we see the type of file is ZIP Archive 

![pic-21](http://164.132.117.34/ForensicsSecondChallenge/21.png)


So try to unzip it but it needs password\
The first thing I tried was bruteforcing the password\
But no sucess never ever\
I tried many wordlists and even online cracking tools but nothing.

After that I used binwalk and zipinfo on the file to see some info
```
$ binwalk d66a85f8faaf4968fe5aa29a02c3735898522e3d.zip 
$ zipinfo d66a85f8faaf4968fe5aa29a02c3735898522e3d.zip
```
![pic-22](http://164.132.117.34/ForensicsSecondChallenge/22.png)


As we see there are 2 files

```
secret_file/494a963e23bcdf3d4286a662fdf4e300
secret_file/top_secret
```

The defN keywork from zipinfo output means the files are compressed\
The BX(Capital B) from zipinfo output means the files are encrypted

It was interesting for me that the file secret_file/494a963e23bcdf3d4286a662fdf4e300 is exactly the url being downloaded in step 1\
And if we look deeper the number of bytes of file secret_file/494a963e23bcdf3d4286a662fdf4e300 is exactly 1911251 which is the size of our png file we recovered in step 1\
So we conclude that this file is exactly the step1.png file we recovered from  previous step\
I thought a lot in this phase what can I do with this hint\
It's obviously intednted that this file is inside encypted zip archaive\
Because it was my first time doing forensics challenge I was not familiar with know techniques we can do for encypted zip files\
After some thinking I reached a familiar techniues in cryptography attacks which is like this
We have plain text message (M)\
We have cipher text message (C)\
We can recover the key (K) which transforms M to C

That is a famous cryptograpchy attack which is performed by having a know plain text (M) and a cipher text (C)

So I thought I had a clue and googled about it with adding zip keyword\
I found a tool named [bkcrack](https://github.com/kimci86/bkcrack) which automatically does this attack on encypted zip files with having known plain zip file

![pic-23](http://164.132.117.34/ForensicsSecondChallenge/23.png)

The whole things we should do is like this

1) zip step1.png file with no passwords into step1.zip
```
$ zip step1.zip step1.png 
```

2 run bkcrack to recover keys
```
$ bkcrack -C d66a85f8faaf4968fe5aa29a02c3735898522e3d.zip -c secret_file/494a963e23bcdf3d4286a662fdf4e300 -P step1.zip -p step1.png
```

3) run bkcrack to decipher the topsecret file and store it as compressed data
```
$ bkcrack -C d66a85f8faaf4968fe5aa29a02c3735898522e3d.zip -c secret_file/top_secret -k 1d2ab3ea 8503eed0 df1eab08 -d topsecret.deciphered
```

4) run a tool named inflate.py inside tools directory of bkcrack to decompress the topsecret.deciphered
```
python <path to bkcrack git repo>/tools/inflate.py < topsecret.deciphered > topsecret
```
![pic-24](http://164.132.117.34/ForensicsSecondChallenge/24.png)
![pic-25](http://164.132.117.34/ForensicsSecondChallenge/25.png)


And that's it we recovered step2 file which is a png file \
If we open the png we see a url which leads us to step3
```
https://parcham.io/challenges/forensics/04926e840fbb43d8f097aa337a49c20fcbc99703
```

# Step 3
We download the file and see the information about the file with file and binwalk
```
$ file 04926e840fbb43d8f097aa337a49c20fcbc99703
$ binwalk 04926e840fbb43d8f097aa337a49c20fcbc99703
```
![pic-31](http://164.132.117.34/ForensicsSecondChallenge/31.png)

As we can see there are 3 bzip2 archive inside it\
So we try to extract them with binwalk -e flag
```
$ binwalk -e 04926e840fbb43d8f097aa337a49c20fcbc99703
```
![pic-32](http://164.132.117.34/ForensicsSecondChallenge/32.png)

As we see there are 3 files (1 PNG and 2 data)\
I personally just opened the png file but in this section I'm gonna use parcham official writeup\
So we guess other 2 parts are about png data so we combine all 3 parts into one png file
```
$ cd _04926e840fbb43d8f097aa337a49c20fcbc99703.extracted/
$ cat 78 12716 20987 > ../step3.png
```
![pic-33](http://164.132.117.34/ForensicsSecondChallenge/33.png)

If we open the step3.png we see a new url which leads us to step 4
```
https://parcham.io/challenges/forensics/2f0876d6ec354e314a0cd1f2c7c92bc1341a4006
```

For some parts of step 4 and all final step I'd use parcham official writeup

# Step 4

After getting the file from url we wanna get info about file with file and binwalk command but no success

![pic-41](http://164.132.117.34/ForensicsSecondChallenge/41.png)

So I opened it in hexedit to see what's happening
```
$ hexedit 2f0876d6ec354e314a0cd1f2c7c92bc1341a4006
```
![pic-42](http://164.132.117.34/ForensicsSecondChallenge/42.png)

We see there is some sort of magic number (ninizip 0.09 alpha)\
After some search about it we realize that it's some zipping format named [NanoZip](https://archive.org/download/nanozip.net)\
But it seems the magic numbers have been changed or corrupted\
So first of all we should build a valid nanozip archive and compare and fix the corrupted bytes\
For this we use nz command which performs nanozip archive stuff (USE 32 BIT VERSION)

```
$ touch test.txt
$ nz a test.nz test.txt
```
![pic-43](http://164.132.117.34/ForensicsSecondChallenge/43.png)

If we open test.nz in hexedit we can see the correct bytes of nano zip archive

![pic-44](http://164.132.117.34/ForensicsSecondChallenge/44.png)

So we change the first bytes of our main file to correct values (AE 01 4E 61  6E 6F 5A 69  70 20 30 2E  30 39 20 61  6C 70 68 61  1F 0F 09 05)\
So we fix the bytes with these commands
```
$ dd if=2f0876d6ec354e314a0cd1f2c7c92bc1341a4006 of=step4_data.nz bs=1 skip=22
python -c "print(b'\xAE\x01\x4E\x61\x6E\x6F\x5A\x69\x70\x20\x30\x2E\x30\x39\x20\x61\x6C\x70\x68\x61\x1F\x0F\x09\x05'+open('step4_data.nz', 'rb').read())" > step4.nz
```
![pic-46](http://164.132.117.34/ForensicsSecondChallenge/46.png)

After fixing the bytes we use nz to see content of out archive
```
$ nz l step_4.nz
```

There is a 20 GB file named msg (Ignore it)\
So we extract the 8e76a7fd8a22e35fc12feaf7e2504b1f file which is starting at offset 0x29d7 (10711 decimal)\
If we look deep into our main archive file we realize that the file 8e76a7fd8a22e35fc12feaf7e2504b1f is a nanozip archive\
![pic-45](http://164.132.117.34/ForensicsSecondChallenge/45.png)

So we wanna extract just that file which starts at offset 0x29d7 (10711)
```
dd if=8e76a7fd8a22e35fc12feaf7e2504b1f of=step4_extract.nz bs=1 skip=10711
```
After that we see the content of step4_extract.nz and extract it
```
nz l step4_extract.nz  -> list
nz x step4.extract.nz  -> extract
```
![pic-47](http://164.132.117.34/ForensicsSecondChallenge/47.png)

We got a file named 8e76a7fd8a22e35fc12feaf7e2504b1f which is a PNG file\
After opening we have another link which leads us to final step.
```
https://parcham.io/challenges/forensics/604e54fa87065a7c5a52f96eae1bf63ad5df09eb
```

# Step 5 (Final)
After downloading file we get a png which says "You've got already the flag"! Seriously !?\
It means The flag is hided inside previous steps or in case of mix of all steps !\
Maybe the flag is combination of all pngs we've got in each step (1-5)\
So we wanna use pngsplit as mentioned in parcham official writeup
```
$ mkdir all_pngs
$ cp parcham_1.png parcham_2.png parcham_3.png parcham_4.png parcham_5.png all_pngs
$ cd all_pngs
$ pngsplit parcham_1.png parcham_2.png parcham_3.png parcham_4.png parcham_5.png
```
For more info check [this](http://www.libpng.org/pub/png/spec/1.2/PNG-Chunks.html) out which is about png files structure

And we wanna make new png file named final.png with this command
```
$ cat parcham_1.png.0000.sig parcham_1.png.0001.IHDR  \
    parcham_1.png.0002.gAMA parcham_1.png.0003.cHRM  \
    parcham_1.png.0004.bKGD parcham_1.png.0010.IDAT  \
    parcham_2.png.0010.IDAT parcham_3.png.0010.IDAT  \
    parcham_4.png.0010.IDAT parcham_5.png.0010.IDAT  \
    parcham_5.png.0011.IEND > final.png
```

Finally we open the file final.png and get the flag
```
parcham{Th13_0n3_wa5_harder_be_patient_&_learn!!}
```

Regards
> KRypton (KooroshRZ)
