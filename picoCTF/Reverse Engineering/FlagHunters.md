# Flag Hunters - picoCTF 2025 Writeup

**Category:** Reverse Engineering  
**Difficulty:** Easy

## Description
Lyrics jump from verses to the refrain kind of like a subroutine call. There's a hidden refrain this program doesn't print by default. Can you get it to print it? There might be something in it for you.
## Background
We are given a remote server to connect to and the source code. Connecting to the remote port via `netcat`, we are presented with this.

```
$ nc verbal-sleep.picoctf.net 62384
Command line wizards, we’re starting it right,
Spawning shells in the terminal, hacking all night.
Scripts and searches, grep through the void,
Every keystroke, we're a cypher's envoy.
Brute force the lock or craft that regex,
Flag on the horizon, what challenge is next?

We’re flag hunters in the ether, lighting up the grid,
No puzzle too dark, no challenge too hid.
With every exploit we trigger, every byte we decrypt,
We’re chasing that victory, and we’ll never quit.
Crowd: 
```
We see that the script prints song lyrics line by line and takes in user input, which it then processes. 

Downloading and examining the source code file `lyric-reader.py`, we find something interesting: There is a part of the the song called `secret_intro` which prints the flag. However, this `secret_intro` is never printed. 

```Python
# Read in flag from file
flag = open('flag.txt', 'r').read()

secret_intro = \
'''Pico warriors rising, puzzles laid bare,
Solving each challenge with precision and flair.
With unity and skill, flags we deliver,
The ether’s ours to conquer, '''\
+ flag + '\n'


song_flag_hunters = secret_intro +\
'''
```

The reason why this is never printed is because the parameter `startLabel` is set to `['VERSE 1']` which will always begin at the `['REFRAIN']` and skip over the `secret_intro`.

## Solution

We can try to see if there is a way to change the flow through user input. Observing the code, we find this:

```Python
elif re.match(r"RETURN [0-9]+", line):
        lip = int(line.split()[1])
```

If a line exactly matches the regex, the reader sets the current line index to that number, effectively jumping to another line in that song. Initially, we would think to input `RETURN 0` to jump to the `secret_intro`. However, this only results in the statement being treated as a string. To bypass this, we observe this part of the code: 
```Python
for line in song_lines[lip].split(';'):
      if line == '' and song_lines[lip] != '':
        continue
```

Here, because of the `.split(';')`, a single user input containing a semiclon becomes multiple `line` tokens. If one of those tokens is `RETURN <num>`, the program will treat it as a control command and jump to the line `<num>`. 

Since we want to jump to line 0, we can submit any text that includes `; RETURN 0`. After splitting, one token will be `RETURN 0`, which matches the regex and forces a jump. Example payload:

```
aaa; RETURN 0
```
When the reader processes the line with that string, it will split into `["aaa", " RETURN 0"]`. The RETURN 0 token triggers `lip = 0`, causing the reader to begin printing from line 0. Thus, we obtain the flag:

```
Crowd: a;RETURN 0

Echoes in memory, packets in trace,
Digging through the remnants to uncover with haste.
Hex and headers, carving out clues,
Resurrect the hidden, it's forensics we choose.
Disk dumps and packet dumps, follow the trail,
Buried deep in the noise, but we will prevail.

We’re flag hunters in the ether, lighting up the grid,
No puzzle too dark, no challenge too hid.
With every exploit we trigger, every byte we decrypt,
We’re chasing that victory, and we’ll never quit.
Crowd: a
Pico warriors rising, puzzles laid bare,
Solving each challenge with precision and flair.
With unity and skill, flags we deliver,
The ether’s ours to conquer, picoCTF{...redacted...}
```



