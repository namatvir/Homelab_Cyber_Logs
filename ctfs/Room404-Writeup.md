# Room 404 -- TryHackMe
 
**Category:** Web / Directory Enumeration
**Tools used:** git-dumper, Forexbuster, Nmap
**Difficulty:** Easy
 
## Summary
 
Web room themed around a hotel guest-experience app that got shipped with a live .git directory still exposed on port 8080. Whole challenge is just recon -> spot the exposed repo -> dump it -> read the source for the flag.
 
## Process
 
Started with an nmap scan using -sC to run default scripts against the target. Port 8080 came back open running the web app, and the http-enum style output flagged .git/ returning a 200 instead of a 404 which was confirmation the git metadata directory was being served by the web root instead of blocked.
 
Grabbed a common wordlist off the box with find to use for enumeration, then ran feroxbuster against the site with that wordlist. Feroxbuster came back and confirmed .git/HEAD was also a 200, which is means that this is a working repo sitting in the webroot with git internals intact and readable over http.

Instead of brute manual brute force, I installed git-dumper and pointed it at the target. It reconstructed the whole repository locally by pulling every object git-dumper can find referenced from HEAD and the pack/loose object files, same as if you'd cloned it.

Once the dump finished, ran ls -la in the output directory to see what actually got pulled down. A README was sitting right there. Opened it and the flag was written plaintext:

## Root Cause
 
Web server was serving app's working directory directly with .git instead of pointing the docroot at a build output, since git doesnt protect its own contents from being read. Git assumes all access is local, not public. So if .git/HEAD returns 200 (as it did), the entire commit history can be reconstructed
 
## Lessons
 
- .git/HEAD returning 200 on a web app gives user/hacker a lot of power
- nmaps flags: -sC, -oN, -sV
- git-dumper to automate manual /git/objects

## Flag:

THM{byt3_l0tus_n3v3r_f0rg3ts}