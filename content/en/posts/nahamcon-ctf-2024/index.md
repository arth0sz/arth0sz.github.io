+++
title = 'NahamCon CTF 2024 - SecureSurfer and Breath of the Wild'
date = 2024-05-25T21:30:35+03:00
draft = false
+++


![](68d2c5f33c7cff2fcd8d7cf20ed8aa2bacd1012ee3bb3c8e60265bebb37eb678.png)

I am very much a newbie when it comes to jeopardy-style CTFs, it seems like an entirely different skillset to hacking away at boxes, but I participated in the NahamCon CTF this year with the [Hack Smarter](https://hacksmarter.org/) team and so I am posting some write-ups.

## SecureSurfer - Medium

> The tech influencers were right!! SSH _for everything_ really is the future! With `SecureSurfer`, you can surf the world wide web... all through your terminal!  
>
> **Escalate your privileges and run the program in the root user's home directory.**  
>
>**Connect with:**  
>
>Password is "userpass"
> 
>`ssh -p 32453 user@challenge.nahamcon.com`

When you SSH to the target, we are greeted with this interface:

![](Pasted%20image%2020240525152402.png)

The challenge contained an attachment with the source code of the program in C:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void surf(const char *url) {
    char command[512];
    sprintf(command, "/usr/local/bin/lynx --accept_all_cookies -cache=0 -restrictions=all '%s'", url);
    system(command);
    system("stty sane");
}

int main() {
    puts("  _____                           _____             __          ");
    puts(" / ____|                         / ____|           / _|         ");
    puts("| (___   ___  ___ _   _ _ __ ___| (___  _   _ _ __| |_ ___ _ __ ");
    puts(" \\___ \\ / _ \\/ __| | | | '__/ _ \\\\___ \\| | | | '__|  _/ _ \\ '__|");
    puts(" ____) |  __/ (__| |_| | | |  __/____) | |_| | |  | ||  __/ |   ");
    puts("|_____/ \\___|\\___|\\__,_|_|  \\___|_____/ \\__,_|_|  |_| \\___|_|   ");
    puts("");

    int choice;

    while (1) {
        printf("\n===== Menu =====\n");
        printf("  1. Surf SSH\n");
        printf("  2. Surf RSA\n");
        printf("  3. Surf DSA\n");
        printf("  4. Surf ECDSA\n");
        printf("  5. Surf ECDH\n");
        printf("  6. Surf WWW\n");
        printf("  7. Exit\n");
        puts("");
        printf("Enter your choice: ");
        fflush(stdout); 
        scanf("%d", &choice);
        getchar(); 

        switch (choice) {
            case 1:
                surf("https://en.wikipedia.org/wiki/Secure_Shell");
                break;
            case 2:
                surf("https://en.wikipedia.org/wiki/RSA_(cryptosystem)");
                break;
            case 3:
                surf("https://en.wikipedia.org/wiki/Digital_Signature_Algorithm");
                break;
            case 4:
                surf("https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm");
                break;
            case 5:
                surf("https://en.wikipedia.org/wiki/Elliptic-curve_Diffie%E2%80%93Hellman");
                break;
            case 6:
                {
                    char url[1024];
                    printf("Online URL: ");
                    fflush(stdout); 
                    fgets(url, sizeof(url), stdin);
                    url[strcspn(url, "\n")] = 0; // Remove newline character

                    if (strstr(url, "https://") == NULL) {
                        printf("\nWe are secure here at the SecureSurfer! You must use https:// !\n");
                    } else {
                        surf(url);
                    }
                }
                break;
            case 7:
                printf("Exiting SecureSurfer...\n");
                return 0;
                break;
            default:
                printf("Invalid choice. Please try again.\n");
                break;
        }
    }

    return 0;
}

```

We can see from the code that the only place where we can manipulate the program would be if we chose case 6 (line 54), where we control the `url` parameter. 

We can also see that the `surf` function uses `system` to execute commands and it doesn't look like there's any input sanitization.

The `url` has to begin with `https://` otherwise we would get an error.

We can craft a command such as the following to gain arbitrary code execution:

```sh
https://example.com'; id'
```

![](Pasted%20image%2020240525152656.png)

Once we enter this command, it will open the Lynx text-based browser, we need to press `q` to quit and we'll see our executed command:

![](Pasted%20image%2020240525152527.png)

Now that we're sure this works we can proceed with some enumeration, let's start with `sudo -l`:

```sh
https://example.com'; sudo -l'
```

![](Pasted%20image%2020240525152752.png)

The user can run the above-mentioned Lynx browser binary as sudo without a password. I initially overcomplicated this, I search for privilege escalation with Lynx and found this whitepaper:

https://public.interlinked.us/3/lynx-filesystem

It talks about how we can press G (which stands for Go) to navigate to an arbitrary location/URL. The user is then prompted for a URL to open. If Lynx is misconfigured, we may be able to open 
files and perform directory listing.

By entering `/` or any directory, we can see its contents. Same goes for file://localhost. 

Let's inject the following command:

```sh
https://example.com'; sudo /usr/local/bin/lynx; echo '
```

This will end up opening two instances of Lynx. Quit the first instance which is running as the `securesurfer` user. 

![](Pasted%20image%2020240525153114.png)

Press **G** and request to "open" the `/root` directory:

![](Pasted%20image%2020240525153127.png)

There is our binary, we know the name now.

![](Pasted%20image%2020240525153207.png)

```
get_flag_random_suffix_21505252448959
```

By pressing **P**, we can print/copy the contents of files to any directory of our choosing:

![](Pasted%20image%2020240525153537.png)

However, the file keeps its original permissions.
### Solution

The solution is actually much simpler and requires just reading the tool's manual properly.

Run the command, quit the first instance.

```sh
https://example.com'; sudo /usr/local/bin/lynx; echo '
```

Spawn shell with `!`.

![](Pasted%20image%2020240525153013.png)

![](Pasted%20image%2020240525153327.png)

```txt
flag{59bcb1602f51c8cfaff5bbcf634e4574}
```

## Breath of The Wild - Easy

>I got a sweet desktop background for my favorite video game, but now I want more! Problem is, I forget where I downloaded it from... can you help me remember where I got this old one?  
  >
>Here's a backup of all my wallpapers. For security, I set the drive password to be `videogames`.

This is a forensics challenge which comes with an attachment of a `.7z` archive (50MB compressed, 100MB uncompressed).

Unzipping the archive, we find a file that comes without an extension. Running the `file` command in Linux says `Microsoft Disk Image eXtended`.

![](Pasted%20image%2020240525220047.png)

If you don't know what that is, Googling will reveal that it's a `.vhdx` Disk Image file.

We can add the extension ourselves and the easiest for me was to mount the file in Windows.

We can go to `Disk Management` > `Action` > `Attach VHD`:

![](Pasted%20image%2020240525220647.png)

The image is encrypted with BitLocker, but we already have the password.

![](Pasted%20image%2020240525220728.png)

We can easily find the image corresponding to the challenge's namesake, but not much more.

![](Pasted%20image%2020240525220922.png)

For that we're going to need to do some investigating:

We're going to open Autopsy (you can [download it here](https://www.autopsy.com/download/)), start a new case and add the now unencrypted drive as a data source:

![](Pasted%20image%2020240525221111.png)

We can find all the information about the downloaded wallpapers under `Data Artifacts` > `Web Downloads`.

![](Pasted%20image%2020240525221240.png)

Under every single file except the Breath of the Wild image, we can find the same exact URL:
### Solution

![](Pasted%20image%2020240525221343.png)

This is what we find for our image, however:

```http
HostUrl=https://www.gamewallpapers.com/wallpapers_slechte_compressie/01wallpapers/&#102;&%23108;&%2397;&%23103;&%23123;&%2356;&%2351;&%23102;&%2350;&%2398;&%2348;&%2397;&%2356;&%2399;&%23101;&%2351;&%2357;&%23102;&%2350;&%23101;&%2353;&%2398;&%2397;&%2349;&%23100;&%2354;&%2399;&%2355;&%2348;&%23101;&%2357;&%2355;&%23102;&%2350;&%2357;&%2349;&%23101;&%23125;
```

This part of the URL appears to be encoded:

```txt
&#102;&%23108;&%2397;&%23103;&%23123;&%2356;&%2351;&%23102;&%2350;&%2398;&%2348;&%2397;&%2356;&%2399;&%23101;&%2351;&%2357;&%23102;&%2350;&%23101;&%2353;&%2398;&%2397;&%2349;&%23100;&%2354;&%2399;&%2355;&%2348;&%23101;&%2357;&%2355;&%23102;&%2350;&%2357;&%2349;&%23101;&%23125;
```

I was thinking URL encoding at first, but the first character hints at something else and that's HTML entities, as `&#102;` corresponds to the letter `F` (for flag?).

The rest still looks URL-encoded to me however, but it needs a little cleaning up:

```txt
#102;%23108;%2397;%23103;%23123;%2356;%2351;%23102;%2350;%2398;%2348;%2397;%2356;%2399;%23101;%2351;%2357;%23102;%2350;%23101;%2353;%2398;%2397;%2349;%23100;%2354;%2399;%2355;%2348;%23101;%2357;%2355;%23102;%2350;%2357;%2349;%23101;%23125
```

We can put it in `CyberChef` and run `URL Decode`.

![](Pasted%20image%2020240525221730.png)

Then we need to add back the `&` and run `From HTML Entity`:

```txt
&#102;&#108;&#97;&#103;&#123;&#56;&#51;&#102;&#50;&#98;&#48;&#97;&#56;&#99;&#101;&#51;&#57;&#102;&#50;&#101;&#53;&#98;&#97;&#49;&#100;&#54;&#99;&#55;&#48;&#101;&#57;&#55;&#102;&#50;&#57;&#49;&#101;&#125;
```

![](Pasted%20image%2020240525222018.png)

```txt
flag{83f2b0a8ce39f2e5ba1d6c70e97f291e}
```