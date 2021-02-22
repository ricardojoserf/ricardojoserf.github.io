---
layout: post
title: Ntds-Analyzer - Tool to analyze Ntds.dit files
---


<!-- ![_config.yml]({{ site.baseurl }}/images/config.png) -->

## Presentation

Ntds-analyzer is a tool to extract and analyze the hashes in Ntds.dit files after cracking the LM and NTLM hashes in it. It offers relevant information about the Active Directory’s passwords, such as the most common used ones or which accounts use the username as password.
Also, it offers an extra functionality: it calculates the NTLM hash value from the LM hash when only the latter has been cracked (we will explain this later!). 

## What is the ntds.dit file?

First, let us quickly introduce what the Ntds.dit files are. These files contain the passwords of all accounts in an Active Directory in a hash format, which may be only NTLM or both LM and NTLM. 

The Ntds.dit files are located in systems named “Domain Controllers”, which authenticate and verify users in the network. To get access to them it is usually necessary to first get privileges in the domain as an account of a high privilege group, like the “Domain Admins” or “Enterprise Admins” group. Once you have that access, you can use tools like Mimikatz, Impacket or ntdsutil to get the Ntds.dit file.

Once dumped, we have one line per account of the Active Directory, with a format similar to these two hashes:

```
springfield.local\homer:500:e52cac67419a9a224a3b108f3fa6cb6d:7b41dcec483f64b9319288d1b265387c:::
springfield.local\bart:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0::: 
```

In these lines we have first the account username, then the account ID and then two hashes: the first one is a LM hash and the second is a NTLM hash.

## LM algorithm

LM is a weak algorithm which we can find in the Ntds.dit files hashes. Basically, this algorithm splits the password into two 7-byte blocks, converts all characters to uppercase and fills unused bytes with zeroes. This makes it a very easy to crack hash: you will always be able to crack the password of an account whose LM hash is not null! (in other words, that the LM hash is different to “aad3b435b51404eeaad3b435b51404ee”).

Unfortunately, even that it is becoming less and less common over time, it is not rare to find accounts with a not null LM hash. For example, checking the previous hashes, we can see that user homer has a not null LM hash, because it is different to “aad3b435b51404eeaad3b435b51404ee”. We can use Hashcat to crack it using a brute force attack:

```
hashcat -m 3000 -a 3 hash 
```

![Test](https://raw.githubusercontent.com/ricardojoserf/ntds-analyzer/main/images/image4.png)

The password gets cracked in no time, and we get the value “PASSWORD”:

![Test](https://raw.githubusercontent.com/ricardojoserf/ntds-analyzer/main/images/image5.png)
 
However, we have a problem: we try to log in as the user homer with the password “PASSWORD” and it does not seem to work. Also, we calculate the NTLM hash of “PASSWORD” and it is not “7b41dcec483f64b9319288d1b265387c“, the users’s NTLM hash. 

This is because the LM algorithm, as we explained before, starts converting all characters to uppercase, so any combination of lower and uppercase characters in “password” would have the same LM hash:

-	‘password’ LM hash = e52cac67419a9a224a3b108f3fa6cb6d
-	‘Password’ LM hash = e52cac67419a9a224a3b108f3fa6cb6d
-	‘pAssword’ LM hash = e52cac67419a9a224a3b108f3fa6cb6d
…
-	‘PASSWORD’ LM hash = e52cac67419a9a224a3b108f3fa6cb6d

In this case, being a simple password, we could crack the NTLM hash with Hashcat. But what if this was a very difficult password? It could take too much time to crack it with Hashcat!

The solution is that, given that we have the NTLM hash related to the LM hash and the uppercase password, we can calculate all the permutations of the original password and their related NTLM hash, and see which one is the same that the one in the Ntds.dit file.

For that, we can use Python’s itertools to get all the combinations of the uppercase password:

![Test](https://raw.githubusercontent.com/ricardojoserf/ntds-analyzer/main/images/image6.png)

And we can use the hashlib library to calculate each NTLM possible hash and compare it with the dumped NTLM hash:
 
![Test](https://raw.githubusercontent.com/ricardojoserf/ntds-analyzer/main/images/image7.png)

And so, with this program, we can automatically get that the password of the account homer, with NTLM hash “7b41dcec483f64b9319288d1b265387c”, was in fact “PassworD” and not “PASSWORD”. And we can do this in all the cases where we have cracked the LM hash but not the NTLM one!

## How to use it

We need three elements:

-	The Ntds.dit file
-	A file with the LM cracked hashes in hash:password format
-	A file with the NTLM cracked hashes in hash:password format

The syntax is:

```
python3 analyzer.py -f NTDS.DIT -n NTLM_CRACKED_HASHES -l LM_CRACKED_HASHES
```

The easiest way to get the hashes files in hash:password format is to use Hashcat to crack the Ntds.dit file (with option “-m 3000” for LM and option “-m 1000” for NTLM hashes) and then use the “--show” option to generate them.

But, to test the script, we can use the files in the “test_files” folder of the repository and use the following command:

```
python3 analyzer.py -f test_files/ntds.dit -n test_files/ntlm_cracked.txt -l test_files/lm_cracked.txt
```


We can see we start with 4 NTLM and 8 LM cracked hashes and the Ntds.dit file:

![Test](https://raw.githubusercontent.com/ricardojoserf/ntds-analyzer/main/images/image0.png)

The script shows the number of hashes, the most common ones and which accounts use the same username and password:
 
![Test](https://raw.githubusercontent.com/ricardojoserf/ntds-analyzer/main/images/image0.png)

Then it shows the cracked hashes. Before we had 4 NTLM hashes cracked and now we have 10! That is because we calculated the password from the related LM hashes:
 
![Test](https://raw.githubusercontent.com/ricardojoserf/ntds-analyzer/main/images/image2.png)

Finally we get the list of users and their passwords in user:password format and the result is dumped to the "credentials.txt" file:
 
![Test](https://raw.githubusercontent.com/ricardojoserf/ntds-analyzer/main/images/image3.png)

## Installation

We must download it from Github and install the needed libraries:

```
git clone https://github.com/ricardojoserf/ntds-analyzer
cd ntds-analyzer
pip3 install argparse
```

## Conclusion

This tool can be useful from a defensive approach, to show the dangers of having a bad password policy and the usage of LM hashes; and from an offensive approach to retrieve the maximum number of credentials from these files.

It is an open-source project, so it may have more functionalities in the future. That is why pull requests are more than welcome! :)

## References

https://github.com/ricardojoserf/ntds-analyzer

https://github.com/ricardojoserf/LM_original_password_cracker