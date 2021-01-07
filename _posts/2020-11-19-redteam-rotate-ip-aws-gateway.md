---
title: Rotating Source IPs (Part 1) - AWS API Gateway
author: bigb0ss
date: 2020-11-19 23:36:00 +0800
categories: [RedTeam, Infrastructure]
tags: [aws, api gateway, cloud, rotate ip, redteam]
image: /assets/img/post/redteam/infra/cloud/aws/logo.png
pin: true
---

# Intro

During a security engagement, especially for an evasive/covert type of assessment, you might need to hide your traffic as much as possible. Or if the client has implemented some type of IP based blocking, you might need to rotate your source IPs to bypass it to do something like password spraying, web application enumeration, etc. Also, Microsoft is no longer considering user enumeration as their "feature". So, if you are trying to do a password guessing/user-enumeration against one of their Office365 APIs like ActiveSync or rst2.srf (SOAP API), Microsoft has implemented a defense that after a number of queries, it will start to throw error code saying the user account is locked out whether it is valid or invalid. To bypass this, rotating your source IPs is required. In this blog post, I will show how to use AWS API Gateway to rotate source IPs to access a target URL. 

# AWS API Gateway Setup

Login to your AWS account 
  --> Click `Services` drop-down menu 
  --> Select `API Gateway` under the "Networking & Content Delivery"

![image](/assets/img/post/redteam/infra/cloud/aws/01.png)

Select `REST API` and click `Build`

>**NOTE**: This will allow synchronous communication.

![image](/assets/img/post/redteam/infra/cloud/aws/02.png)

Click `New API`
  --> "API name" : Any Name (e.g., bigb0ss_api_test)
  --> "Endpoint Type": Regional
  --> Click `Create API`

![image](/assets/img/post/redteam/infra/cloud/aws/03.png)

Click `Actions` drop-down menu
  --> Select `Create Method`

![image](/assets/img/post/redteam/infra/cloud/aws/04.png)

Select `ANY`
  --> Click the check mark

![image](/assets/img/post/redteam/infra/cloud/aws/05.png)

Select `HTTP`
  --> Check the `Use HTTP Proxy Integration`
  --> "Endpoint URL" : Your Target URL (e.g., https://mail.victim.com)
  --> Click `Save`   

>**NOTE**: I added my EC2 IP to show the IP rotation later in this blog.

![image](/assets/img/post/redteam/infra/cloud/aws/06.png)

Click `Method Request`

![image](/assets/img/post/redteam/infra/cloud/aws/07.png)

Click the down arrow next to `HTTP Request Headers`
  --> Click `Add header`

![image](/assets/img/post/redteam/infra/cloud/aws/08.png)

Add “X-My-X-Forwarded-For” to `Name` 
  -—> Click the check mark 
  
![image](/assets/img/post/redteam/infra/cloud/aws/09.png) 

Click `Method Execution` to return to the previous page

![image](/assets/img/post/redteam/infra/cloud/aws/10.png) 

Click `Integration Request`

![image](/assets/img/post/redteam/infra/cloud/aws/11.png) 

Click the down arrow next to `HTTP Headers`
  --> Click `Add header`

![image](/assets/img/post/redteam/infra/cloud/aws/12.png)

Add “X-Forwarded-For” to `Name` 
  -—> Add a valid IP in single quote (e.g., '4.4.4.4')
  -—> Click the check mark

>**NOTE**: You can also add a valid IP of your targets to add more obfuscation.

![image](/assets/img/post/redteam/infra/cloud/aws/13.png)

Click `Actions` drop-down menu
  --> Select `Deploy API`

![image](/assets/img/post/redteam/infra/cloud/aws/14.png)

"Deployment stage" : "[New Stage]"
  --> "Stage name" : Any name (e.g., login)
  --> Click `Deploy`

![image](/assets/img/post/redteam/infra/cloud/aws/15.png)

Once deployed, it will create the following `Invoke URL` which is an AWS API gateway URL that will eventually redirect traffic to our endpoint (e.g., http://3.136.22.217). 

![image](/assets/img/post/redteam/infra/cloud/aws/16.png)

# Final Test

Let's test it out if our source IPs get rotated when we are accessing `http://3.136.22.217` via `https://rsdzybrwr7.execute-api.us-east-2.amazonaws.com/login`

I will be running the following command in my local box:

```bash
while true; do curl -k https://rsdzybrwr7.execute-api.us-east-2.amazonaws.com/login; done
```

And this is a web server on the EC2 (http://3.136.22.217). We can see the source IPs rotate for each request.

![image](/assets/img/post/redteam/infra/cloud/aws/17.png)

Thanks for reading!