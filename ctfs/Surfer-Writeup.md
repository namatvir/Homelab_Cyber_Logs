# Surfer -- TryHackMe
 
**Category:** Password Attachs
**Tools used:** Burp Suite
**Difficulty:** Easy
 
## Summary
 
An admin panel page used default credentials and an internal only page which was restricted from direct access, but could be reached by intercepting and modifying a different allowed request
 
## Process

First I went to the given IP, and since it was a webpage, I instinctively opened Burp Suite. Nothing looked out of the ordinary so I guessed the log in. My first guess was admin:admin and that worked.

Once inside, I looked around to find a message saying the flag was in /internal/admin.php, and when trying to open it directly, I was faced with "local access only."

Back inside the page, there was a PDF button, which opened the same information as the website, but in a PDF. I turned intercept on and pressed the button and a new URL field was in the raw code. This field ended with .php, thus I replaced the given url (specfically the ending only) with /internal/admin.php and forwarded the request, which presented me the PDF containing the flag.

## Root Cause
 
Admin panel was left with default crednetials, and restriction on /internal/admin.php wasn't properly enforced, but rather blocked direct browser access and did not account for requests made through server-side functions. Allowing for the request to be manipulated and forwarded.
 
## Lessons
 
- Don't leave default passwords
- "Local Access Only" needs to be enforced consistently on the server-side

## Flag:

flag{6255c55660e292cf0116c053c9937810}