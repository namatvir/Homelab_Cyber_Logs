# Fools Mate -- TryHackMe

**Category:** Web Exploitation / Client-Side Logic Bypass
**Tools used:** Firefox DevTools, Burp Suite
**Difficulty:** Easy

## Summary

Chess themed web challenge where mate in one is checked client side and can be bypassed by intercepting and modifying the request which reports the move to server

## Process

Booted up machine and browsed to given IP. Page shows chessboard with mate in one (Ra1:a8). Playing such move returns a message threatening to shut down machine.

First I checked the devtools via inspect and found a function which checks for checkmate (client sided), meaning validation logic lived in browser.

Since the check lived in the browser, I wanted to see what the server actually received when a move was submitted. Thus I opened burp suite and turned on intercept and tried to browse through firefox.

ERROR: Got a connection error through burp suite. First attempt at fixing it was pointing Burp's proxy listener at the DNS-resolved IP, which didn't work. The fix was the other direction: set Firefox's proxy settings to match the IP/port Burp was already listening on. Reloading the page after that produced a GET request in Burp, confirming traffic was flowing.

Once connected, I played a non-winning move (Ra1:a7) which generated a request to the server, which I then caught in Burp and found the field carrying move "{a1:a7}" which I then changed to "{a1:a8}" and forwarded. 

Going back to the browser, I found the winning move was played and the flag was revealed.

## Root Cause

The server trusts the move value sent from the client instead of independently validating game state. Any client-side "you won" logic is decorative if the request that confirms the win can be rewritten before it reaches the server.

## Lessons

Burp 'catches' internet requests, which can then be altered and forwarded, leading to exploits such as this.

## Flag

THM{cl13nt_s1d3_ch3ckm4t3}
