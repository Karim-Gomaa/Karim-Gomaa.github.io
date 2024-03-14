---
title: "Cyber Champions "
classes: wide
header:
  teaser: /assets/images/CTF/Cyber%20Champions/zinad.png
ribbon: DodgerBlue
description: "Writeup for Cyber Champions Reverse challenges ..."
categories:
  - CTF
toc: true
---

# French Man
**French Man** is a  PE32+ executable (console) x86-64, for MS Windows.

first running **Floss** tool got those interesting text:

![floos](/assets/images/CTF/Cyber%20Champions/Floss.png)

here as you can see the Base64 ending with the padding **"="** 
decoding this got **YqEoaye{XF4gsG_4@$a_CIS4c4Z4}** which still doesn't make sense 

running the file got this:

![running](/assets/images/CTF/Cyber%20Champions/running.png)

with some general information and search found that the French man who made Cipher is **Vigenere** so trying to decode the output of b64 with Vigenere found that I need a key from strings and found the only valid key is **"zichampion"** we found previous 

I tried that combination and voila

![The Flag](/assets/images/CTF/Cyber%20Champions/The%20Flag.png)

# SecretsSafe

**SecretsSafe** is a PE32+ executable (console) x86-64, for MS Windows as the previous.

Doing the same floos trick found something maybe useful

![floss](/assets/images/CTF/Cyber%20Champions/secret_floss.png)

decoding this b46 as well gave **"G!v3_M3_Th3_F1@G"**

so let's run the file and see:

![run](/assets/images/CTF/Cyber%20Champions/secret_run.png)

but sadly this was just misleading one 

so here is **IDA** time, I opened and first searched for printed strings but as I thought those were resolved on runtime I chose to look for the only thing I have **"[-] Wrong Password!"** and found it printed after some checks:

![wrong](/assets/images/CTF/Cyber%20Champions/wrong.png)

here is the time of debugging, set breakpoint at main and let's see.

After the function prologue there is a loop which  is responsible for the printing: 

![print](/assets/images/CTF/Cyber%20Champions/print.png)

then it performs some unimportant operations and asks for input, giving it the input we tried first "G!v3_M3_Th3_F1@G" This time i understood that if I provided this it will close so let's try any thing like Password and see, after the message "[-] Wrong Password!" is printed there is three printing function:

![fun](/assets/images/CTF/Cyber%20Champions/prints.png)

here is the time to edit flags to control the flow.

toggling the ZF takes the flow from the wrong one to the home where the function target prints the flag but as b64

![target](/assets/images/CTF/Cyber%20Champions/target.png)

![b64](/assets/images/CTF/Cyber%20Champions/b64.png)

decoding this i got the flag **"ZiChmp{Y0uR_$3cR3t_!$_$@f3!!}"**.

# TrustedApp

**TrustedApp** is a PE32 executable (console) Intel 80386 Mono/.Net assembly, for MS Windows.

as it's a .NET I chose to use **dnspy**

![trust](/assets/images/CTF/Cyber%20Champions/trust.png)

this code takes input compares it with the key then prints an array of characters, but there is some part called hint which sounds important:

![hint](/assets/images/CTF/Cyber%20Champions/hint.png)

doesn't seem meaningful but leave it now.
Running the app with a breakpoint at the entry to get the key to compare with:

![key](/assets/images/CTF/Cyber%20Champions/key.png)

that's the desired input **"128a3c5f-554a-47ab-9b29-f284f5f4d72b"**
let's try again with it.

![final](/assets/images/CTF/Cyber%20Champions/final.png)

that prints an encrypted message, but not b64.
Now the hint started to make sense with some focus you can notice that only three letters are capital case **"A E S"** so let's search for the key to decrypt this using **"Am Enjoying Sure"** as key with no IV the output is **"Congratulations! You have successfully decrypted the message."**



#### Written by

## *Karim Gomaa*