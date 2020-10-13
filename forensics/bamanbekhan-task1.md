# Parcham - forensics, bamanbekhan (500 pt ?) Writeup

First download the attachment

    wget https://parcham.io/challenges/forensics/bamanbekhan .
 Then try to find and extract possible filefomart/filesystem suing binwalk toolkit
 

    Sand_Niggers> binwalk -e bamanbekhan
  Output indicates [Squashfs](https://en.wikipedia.org/wiki/SquashFS) filesystem 
  

    40            0x28            Squashfs filesystem, little endian, version 4.0, compression:gzip, size: 15633389 bytes, 2 inodes, blocksize: 131072 bytes
Two files are visible

    Sand_Niggers> ls -ltrh
    drwxrwxrwx 1 root root    152 Sep  7 08:08 squashfs-root
    -rwxrwxrwx 1 root root    15M Sep  8 09:35 28.squashfs


  There is no need to decompress *28.squashfs* because it already exists under */squashfs-root* directory
 
    Sand_Niggers> ls -ltrh
    unsquashfs -ls 28.squashfs
    Parallel unsquashfs: Using 4 processors
    1 inodes (128 blocks) to write
    
    squashfs-root
    squashfs-root/zero.img
   Mount *zero.img* file
   

    Sand_Niggers> mkdir ./sqush && mount ./squashfs-root/zero.img ./sqush

Output

     Sand_Niggers> ls -ltrh
     
    total 16M
    -rw-r--r-- 1 root root 4.0M Aug 27 06:45 'rand_'$'\001''.img'
    -rw-r--r-- 1 root root 4.0M Aug 27 06:46 'rand_'$'\002''.img'
    -rw-r--r-- 1 root root 3.0M Aug 27 06:46 'rand_'$'\003''.img'
    -rw-r--r-- 1 root root 173K Sep  7 08:06  rand_
Upon further examination with *File* tool, It turns out *rand_* is some kind of [PostScript](https://www.checkfiletype.com/latest-results?q=PostScript) fileformat

  

     Sand_Niggers> file *
       
    rand_:     PostScript document text conforming DSC level 3.0, Level 2
    rand_.img: data
    rand_.img: data
    rand_.img: data
    
Although .ps is a well-documneted format,  Opening it requires downloading legacy software so [converting to pdf](https://document.online-convert.com/convert/ps-to-pdf) is much easier, Downloded pdf file geets with the flag
**parcham{I_l0v3_fOr3n5ics_to0}**

Sand_Niggers team
