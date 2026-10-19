# Corridor -- TryHackMe
 
**Category:** IDOR Vulnerabilities
**Tools used:** Burp Suite, MD5 Hash Converters
**Difficulty:** Easy
 
## Summary
 
Flag located in a room not intended for user to access
 
## Process

Upon reaching the target ip, I was presented with 13 rooms, using Burp (or simply the inspect tool), each room had a hash. 

Using an MD5 hash converter, I see that these hashes represent numbers 1 through 13.

I reversed the converter to find the hash for '0' and forwarded that hash through burp suite. Thus presented the flag in room 0.

## Root Cause
 
Website blocked access to room 0 by not showing it in the gui, however by bypassing this via burp and hash generators, room 0 is accessible
 
## Lessons
 
If a part of a website needs to be blocked off from users, enforce security measures on server/host side. 

## Flag:

flag{2477ef02448ad9156661ac40a6b8862e} 