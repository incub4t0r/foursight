---
title: "Remote Control - NCL Team"
date: 2021-11-08T17:05:44-04:00
draft: false
showToc: true
description: "My solution to Remote Control on NCL Team 2021"
tags: ["NCL", "ctf"]
categories: ["ctf"]
---

<!-- ---
layout: post
title: Remote Control 
subtitle: My solution to Remote Control on NCL Team 2021
image: /assets/img/remotecontrol/remotecontrol.jpg
tags: NCL ctf
published: true
--- -->

A writeup for Remote Control on NCL Team 2021, this one was hard to find information for but it was fun to do.

## TL;DR 

Find LG TV IR codes online, and struggle bus through finding answers. 

## Givens

We recently obtained a dump of the IR protocol used to control an LG television. What can you find out about it?

1. What is the protocol that is being used?
2. What is the device ID of the device being captured?
3. What is the sub device ID? (in hexadecimal representation)
4. How many times was the power button hit?
5. How many malformed packets are there?
6. What is the command with the most malformed packets? (either hexadecimal representation or common name)

## Solutions

I first made a dump of the binary as a hex file, so I could view the actual ir codes being sent.
```
xxd -p remote_control_ir.out > remote_control_ir_hex.txt
```
Opening up `remote_control_ir_hex.txt`, I made a few changes. The changes were to delete all spaces, then to make every line 4 bytes. The results can be viewed at the bottom of this writeup.

After this change, I was able to find the answers to problems 1-3 with some googling. The answer was found on a github issues page from Jason2866 who was having issues with sending IR codes. Luckily for us, they included a dump of their issues, which allowed us to find the IR protocol being used.

1. [NEC](https://github.com/arendst/Tasmota/issues/1663)
2. 04
3. 04FF


For problems 4-6, I began to script this portion. Using [this](http://www.remotecentral.com/cgi-bin/mboard/rc-discrete/thread.cgi?7688) image, I was able to deduce that the following were assigned:

```
04ff00ff channel up
04ff08f7 power
04ff0358 
04ff0095 
04ff0220 
04ff02fd volume up
04ff09f6 mute
04ff03fc volume down
04ff01fe channel down
04ff0074 
```
Using the script I made, I was able to find out the occurrences of every code sent. This included the malformed packets, where I found that code `04ff00ff` or `channel up` was sent 40 times, with two codes that seemed similar but not an exact match. I figured these were the malformed codes.
```
04ff08f7 40 power
04ff01fe 41 channel down
04ff02fd 37 volume up
04ff0220 1  
04ff03fc 48 volume down
04ff0358 1  
04ff00ff 40 channel up
04ff0095 1  
04ff0074 1  
04ff09f6 40 mute
```
4. 40
5. 4
6. 04ff00ff

## Files

tvcodes.py
```python
import os

file_location = os.path.dirname(os.path.abspath(__file__))
# codes = ('04ff08f7', '04ff0220', '04ff01fe', '04ff02fd', '04ff0358', '04ff0095', '04ff03fc', '04ff00ff', '04ff0074', '04ff09f6')
codes = {'04ff08f7':0, '04ff0220':0, '04ff01fe':0, '04ff02fd':0, '04ff0358':0, '04ff0095':0, '04ff03fc':0, '04ff00ff':0, '04ff0074':0, '04ff09f6':0}

file = open(file_location + '\\remote_control_ir_hex.txt' ,'r')
lines = file.readlines()

for k,v in codes.items():

    for line in lines:
        line = line.strip()
        if line == k:
            codes[k] += 1

for k,v in codes.items():
    print(k, v)

```

remote_control_ir_hex.txt
```
04ff09f6
04ff08f7
04ff0074
04ff09f6
04ff00ff
04ff01fe
04ff09f6
04ff01fe
04ff09f6
04ff09f6
04ff02fd
04ff0358
04ff09f6
04ff02fd
04ff09f6
04ff00ff
04ff03fc
04ff02fd
04ff01fe
04ff01fe
04ff08f7
04ff09f6
04ff01fe
04ff08f7
04ff00ff
04ff01fe
04ff08f7
04ff09f6
04ff02fd
04ff03fc
04ff09f6
04ff08f7
04ff01fe
04ff03fc
04ff00ff
04ff09f6
04ff09f6
04ff00ff
04ff08f7
04ff02fd
04ff03fc
04ff03fc
04ff08f7
04ff00ff
04ff01fe
04ff00ff
04ff08f7
04ff09f6
04ff08f7
04ff08f7
04ff09f6
04ff08f7
04ff02fd
04ff02fd
04ff09f6
04ff09f6
04ff09f6
04ff01fe
04ff00ff
04ff09f6
04ff00ff
04ff00ff
04ff03fc
04ff03fc
04ff09f6
04ff02fd
04ff01fe
04ff09f6
04ff02fd
04ff08f7
04ff00ff
04ff08f7
04ff09f6
04ff08f7
04ff00ff
04ff01fe
04ff03fc
04ff03fc
04ff01fe
04ff08f7
04ff00ff
04ff08f7
04ff00ff
04ff08f7
04ff03fc
04ff03fc
04ff08f7
04ff08f7
04ff00ff
04ff03fc
04ff00ff
04ff02fd
04ff00ff
04ff01fe
04ff01fe
04ff08f7
04ff09f6
04ff03fc
04ff01fe
04ff09f6
04ff01fe
04ff08f7
04ff02fd
04ff08f7
04ff03fc
04ff03fc
04ff08f7
04ff00ff
04ff00ff
04ff00ff
04ff02fd
04ff00ff
04ff03fc
04ff01fe
04ff02fd
04ff09f6
04ff00ff
04ff02fd
04ff08f7
04ff02fd
04ff03fc
04ff08f7
04ff02fd
04ff02fd
04ff02fd
04ff03fc
04ff03fc
04ff03fc
04ff08f7
04ff02fd
04ff03fc
04ff00ff
04ff02fd
04ff02fd
04ff08f7
04ff01fe
04ff08f7
04ff00ff
04ff03fc
04ff01fe
04ff01fe
04ff01fe
04ff09f6
04ff01fe
04ff09f6
04ff01fe
04ff00ff
04ff01fe
04ff03fc
04ff01fe
04ff00ff
04ff00ff
04ff03fc
04ff03fc
04ff02fd
04ff09f6
04ff03fc
04ff08f7
04ff01fe
04ff03fc
04ff00ff
04ff08f7
04ff02fd
04ff03fc
04ff03fc
04ff09f6
04ff01fe
04ff02fd
04ff03fc
04ff08f7
04ff02fd
04ff03fc
04ff01fe
04ff03fc
04ff02fd
04ff02fd
04ff08f7
04ff00ff
04ff02fd
04ff03fc
04ff00ff
04ff03fc
04ff00ff
04ff03fc
04ff03fc
04ff02fd
04ff00ff
04ff01fe
04ff02fd
04ff00ff
04ff08f7
04ff08f7
04ff09f6
04ff09f6
04ff0095
04ff03fc
04ff01fe
04ff00ff
04ff02fd
04ff09f6
04ff02fd
04ff01fe
04ff09f6
04ff01fe
04ff08f7
04ff02fd
04ff09f6
04ff03fc
04ff02fd
04ff09f6
04ff00ff
04ff02fd
04ff01fe
04ff01fe
04ff03fc
04ff01fe
04ff09f6
04ff03fc
04ff03fc
04ff03fc
04ff01fe
04ff02fd
04ff08f7
04ff09f6
04ff08f7
04ff03fc
04ff00ff
04ff00ff
04ff03fc
04ff09f6
04ff03fc
04ff02fd
04ff01fe
04ff00ff
04ff09f6
04ff01fe
04ff00ff
04ff08f7
04ff03fc
04ff08f7
04ff09f6
04ff09f6
04ff0220
04ff01fe
04ff00ff
04ff03fc
04ff08f7
04ff03fc
04ff01fe
04ff01fe
```