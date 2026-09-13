Security:

What are the different services in AWS which can help you secure the application and the AWS platform on which you are running you app?

IAM - Provide security for authorization, authorization
e.g: 

ec2 needs read and write access to S3 bucket for my app
usually what people do is:
    - they will create a overly permissive policy
    - read and write on all s3 buckets
    - attach this policy to the EC2

What should be done:
    - first get the name/s of the S3 buckets which the app needs access to
    - then create a policy which can provide access to only those
    - restrictive policies


WAF (Web Application Firewall) - Filter the traffic based on various parameters.
    - Route53 -> WAF -> ALB -> 5-10 EC2 (auto-scaling) -> RDS

    - Hackers may do something like DDOS attack on your website

Do we actually need a firewall on top of it?

    - Geographical restriction
    - filter and restrict malicious traffic
    - IP with bad reputation
    - IP whitelisting
    - Rate limiting: limiting the number of requests which is allowed in certain time duration:
        - Allow 500 request in 2 mins from same IP


How do you secure / encrypt your data in transit?
    - TLS / SSL certs can be implemented in AWS via AWS certificate manager

How do you secure / encrypt your data at rest?
    - Data at rest can be at:
        - Database (RDS) inside its storage
        - EC2 inside the EBS volumes
        - S3 buckets
    - In AWS, there KMS (Key Management service)

    Server side and customer managed keys

to have proper backups in place


Security Groups
-------





If the application is accessed by users in different countries, then how do you deploy your app?

- Servers where you have the highest traffic from
- Cloudfront - CDN (Content Delivery Network) - USA, Singapore, Australia - edge servers which will cache the website content


when would you actually need to provision servers in both the regions where your users are?

- Where the content is different
- When most of the content is dynamic
- Compliance, data residency part

Hotel website is hosted in USA, now they are expanding to UK.. will they be able to use the same servers in USA and expand it by CDN?

- Yes, if they are not storing any PI (Personal Information) data 
- No, if they are storing any form of PI data.. As UK has GDPR compliance, and there is huge penalty if you are storing their citizens PI data to any other countries servers

---

Traffic is normally low but rises sharply when ticket sales open

- from compute side: Autoscaling 
- from database side:
    - baseline your database instance to serve higher traffic
    - the app should be created in such a way that it does not bombard the database, write database scripts in in an optimized way
    - Read Replicas inside the RDS Database
        - It is a readonly copy of your database
        - update your application to point to Read replica for any read request and point to the main rds endpoint for any write request
    - Use of the queue servers

---

CloudFront - Handson

---