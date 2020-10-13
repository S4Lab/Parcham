# WriteUp for first Forensics challenge (bamanbekhan) 

After downloading file we get file properties and type with file command
```
$ file bamanbekhan  
```


![pic-1](http://164.132.117.34/bamanbekhan/1.JPG)  

Then according to video tutorials we got some more information about the file with binwalk
```
$ binwalk bamanbekhan
```

And we get this output

![pic-2](http://164.132.117.34/bamanbekhan/2.JPG)

We realize that there is a squashfs file system inside the file, So we tried to mount it with mount command but no success !

```
$ mkdir mnt  
$ sudo mount -o loop bamanbekhan mnt  
$ sudo mount -t squashfs bamanbekhan mnt/
```

![pic-3](http://164.132.117.34/bamanbekhan/3.JPG)

 After some Googling we got this command for auto mounting the file with binwalk using -Me options
 
 ```
 $ binwalk -Me bamanbekhan
 ```

After running above command we get extracted files inside _bamanbekhan.extrated folder, We can see the whole files with tree aommand

![pic-4](http://164.132.117.34/bamanbekhan/4.JPG)

Inside _bamanbekhan.extracted we find a file 28.squashfs which is automatically mounted to squashfs-root\
After getting tree output we see there is another file named zero.img inside the squashfs-root folder\
So we go inside the folder and get file imformation with file command but nothing interesting

```
$ cd _bamanbekhan.extracted/squashfs-root/  
$ flle zero.img
```

![pic-5](http://164.132.117.34/bamanbekhan/5.JPG)

Then we try it with binwalk but we get a lot of unix path information which don't help a lot

![pic-6](http://164.132.117.34/bamanbekhan/6.JPG)

We try mounting it into mnf folder with mount command :

```
$ mkdir mnt  
$ sudo mount -o loop zero.img mnt/
```
The mount operation is successful and we see there are 4 files inside mnf folder so we list them

```
$ tree
$ ls -la mnt/
```

![pic-7](http://164.132.117.34/bamanbekhan/7.JPG)

We go inside mnt folder and try to specify all file types with file command

```
$ cd mnt
$ file * .*
```

And we get this output

![pic-8](http://164.132.117.34/bamanbekhan/8.JPG)

The file rand_ is a PostScript file and looks interesting among other 3 files, So we try to open it with pdf reader but no success\
Guess what, because the PostScript file needs to be executed and we are in mnt folder with read-only permissions so we can not open it inside mnt folder\
To fix this issue we copy the file to another folder and try openning it again :

```
$ cp rand_ ../  
$ cd ../
```

After openning the rand_ file inside folder with right permissions, we get the pdf file and the flag is inside it

![pic-9](http://164.132.117.34/bamanbekhan/9.png)

Regards
> KRypton (KooroshRZ)
