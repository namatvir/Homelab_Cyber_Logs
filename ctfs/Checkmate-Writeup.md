# Checkmate -- TryHackMe
 
**Category:** Password Attacks
**Tools used:** Cupp, Hydra, vim, Hashcat, Crunch
**Difficulty:** Easy
 
## Summary
 
System admin made internal services with weak passwords
 
## Process

First I had to add the ip into Vim with the domain so I could access the website

For port 5001, I simply guessed the password within minutes

For port 5002, again I guessed passwords using the keywords from the previous page to find the correct password

For port 5003, here I discovered (via other writeups) of a tool called cupp, after cloning and deploying repo, I ran using: ``` python3 cupp.py -i ``` which prompted multple inputs. after it completed, I used hydra with this newly created text file against the website

For port 5004, I used hashcat to brute force the original file name

For port 22 (aka SSH), I used crunch with the known parameters into another text file which I used with hydra to brute force the final password

## Root Cause
 
System admin made weak, predicable password, causing every one to either be guessed or brute forced
 
## Lessons
 
- Make strong, unpredictable password which use a combination of numbers and characters.
- Tools learned: Cupp, Hydra, Crunch and Hashcat, all useful for brute forcing passwords
- Accessing certain ports on an ip address that were once locked can be bypassed by altering vim

## Flag:

- 12345
- excellence
- Bianchi2495
- family
- Security2024!