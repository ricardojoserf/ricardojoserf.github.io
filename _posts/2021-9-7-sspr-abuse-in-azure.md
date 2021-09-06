---
layout: post
title: Abusing SSPR in Azure to get a Domain Admin
excerpt_separator: <!--more-->
---

In this post I will explain a simplified scenario in which we could abuse the SSPR functionality in Azure in a Red Team assessment. Knowing the expired password of a "Domain Admin" of the local AD, it is possible to update its password by updating the Azure AD password with SSPR (as far as the AD and the Azure AD are synchronized). 

<!--more-->


#### TL;DR

1. The Active Directory and Azure AD were synced

2. Compromising one domain it was possible to become Global Administrator of the Azure tenant

3. A "Domain Admin" password was obtained in cleartext in the original domain, and that user had the same password in the target AD domain, where it is also a "Domain Admin"

4. The credentials in the target domain were expired, but it was possible to access the Azure portal with those credentials

5. The user is not part of the SSPR (Self-Service Password Reset) group, so it is unable to update its own password. But it is a privileged account, so the Global Administrator could not update its password

6. Using the Global Administrator account in Azure it was possible to add the user in the target domain to a new group and set that group as the SSPR selected group

7. The user can update its password in the Azure portal, updating the password in the target Active Directory, and now it is a valid and not expired password so the domain is compromised



#### Initial attack scenario

In order to exploit this scenario, we had first achieved these goals:

1. The Spanish branch of the company had been fully compromised, dumping all the account hashes of the Active Domain domain SPAIN_DOMAIN after carrying out a DCSync attack

2. The LM and NTLM hashes were cracked, getting the cleartext password of one "Domain Admin" user: John Doe, whose account was *SPAIN_DOMAIN\jdoe*

3. The password of the account *SPAIN_DOMAIN\O365Admin* was also cracked. It was not relevant in the Active Directory... but it will become useful later


Our goal at this moment is to reach GLOBAL_DOMAIN domain and dump all the password hashes. There is a trust relationship between SPAIN_DOMAIN and GLOBAL_DOMAIN, so we can list information of our objective. We have good news:

- There is an account for the user "John Doe", which in this case is *GLOBAL_DOMAIN\john.doe*

- The account *GLOBAL_DOMAIN\john.doe* has the same password that *SPAIN_DOMAIN\jdoe*

- The account *GLOBAL_DOMAIN\john.doe* has "Domain Admin" privileges in GLOBAL_ADMIN (John is Domain Admin in both domains!)

- The Domain Controller of GLOBAL_DOMAIN is accessible from our point in the network

However, when we try to dump the credentials we have a bad new:

- ... the password has expired



#### Recon phase in Azure

Accessing the Azure Portal using the email account o365admin@spain_domain.com, we see the account has Global Administrator privileges:

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/azure-sspr/image1.png)

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/azure-sspr/image2.png)

These privileges apply to the whole Azure tenant, which not only has the SPAIN_DOMAIN domain but also others such as GLOBAL_DOMAIN. We can see this for example when we start creating a new user in the tenant, we can see a long list of possible domains for it:

![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/azure-sspr/image3.png)

With the privileges of O365Admin account (o365admin@spain_domain.com) we can even update the password of the users

![img4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/azure-sspr/image4.png)

However, this only works for non-privileged users, so we will not be able to update the password of *GLOBAL_DOMAIN\john.doe*.



#### Abusing SSPR

Using just the O365Admin account (o365admin@spain_domain.com), it seemed impossible to get access to a privileged account of GLOBAL_ADMIN, and *GLOBAL_DOMAIN\john.doe* was expired in the (local) Active Directory. However, what about Azure portal? 

![img5](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/azure-sspr/image5.png)

Yes, we have access! So the account is expired in Active Directory but not in Azure. 


After some research, the magic letters appeared: SSPR or Self-Service Password Reset. From [this link](https://docs.microsoft.com/en-us/azure/active-directory/authentication/concept-sspr-howitworks), SSPR *"gives users the ability to change or reset their password, with no administrator or help desk involvement. If a user's account is locked or they forget their password, they can follow prompts to unblock themselves and get back to work. This ability reduces help desk calls and loss of productivity when a user can't sign in to their device or an application"*.

SSPR can be enabled for:

- Everyone - All users can update their own password

- Selected - Only users of one specific group can update their own password

- None - None can update its own password


In this case, the second option is implemented in this tenant:

![img7](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/azure-sspr/image7.png)


Initially, the user is not part of this group so it is not able to update its password:

![img6](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/azure-sspr/image6.png)


To avoid adding the user to a privileged group (maybe that is monitorized) I created a new group, adding John Doe (john.doe@global_admin.com) to it:

![img8](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/azure-sspr/image8.png)

It was possible to change the SSPR selected group to "TestTeam2". As John Doe is part of the SSPR group, with that account credentials is possible to update its own password:

![img9](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/azure-sspr/image9.png)

And so, the Azure password was changed and, at the same time, the *GLOBAL_DOMAIN\john.doe* account password was updated (because, as we mentioned before, Azure AD is synced with the Active Directory of the organizarion). 

As the Domain Controller was accessible from our start point, we could carry out a DCSync attack and dump all the hashes of the main domain, GLOBAL_DOMAIN, completing our final goal!