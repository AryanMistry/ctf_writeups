# hashcrack - picoCTF 2025 Writeup

**Category:** Cryptography  
**Difficulty:** Easy

## Description
A company stored a secret message on a server which got breached due to the admin using weakly hashed passwords. Can you gain access to the secret stored within the server?

## Background
We are given a remote server to connect to. Connection to this server via `netcat`, we are greeted with this:


```
$ nc verbal-sleep.picoctf.net 50882
Welcome!! Looking For the Secret?

We have identified a hash: 482c811da5d5b4bc6d497ffa98491e38
Enter the password for identified hash:
```

## Solution
This hash seems to be MD5 since it is 32 characters. Inserting it into an online hash cracker (e.g. [CrackStation](https://crackstation.net/)), we get the first password: `password123`

After inputting that, we get our next hash to crack which seems to be SHA1 as it is 40 characters. Following the same process, we are able to obtain the password `letmein`. 

Finally, we get our final hash. This one seems to be SHA256 since it is 64 characters in length. Inputting the final password `qwert0981`, we get the flag!

```
$ nc verbal-sleep.picoctf.net 50882
Welcome!! Looking For the Secret?

We have identified a hash: 482c811da5d5b4bc6d497ffa98491e38
Enter the password for identified hash: password123
Correct! You've cracked the MD5 hash with no secret found!

Flag is yet to be revealed!! Crack this hash: b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3
Enter the password for the identified hash: letmein                
Correct! You've cracked the SHA-1 hash with no secret found!

Almost there!! Crack this hash: 916e8c4f79b25028c9e467f1eb8eee6d6bbdff965f9928310ad30a8d88697745
Enter the password for the identified hash: qwerty098
Correct! You've cracked the SHA-256 hash with a secret found. 
The flag is: picoCTF{...redacted...}
```

