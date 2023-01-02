# Questions

**1. What are the types of cloud computing?**

- Infrastructure as a Service (IaaS)
- Platform as a Service (PaaS)
- Software as a Service (Software as a Service)

**2. What are the models of computing deployment?**

- Public
- Hybrid
- On-premises

**3. What is S3 and what does it mean?**

- S3 stands for Simple Storage Service.
- S3 is on object storage with a simple web interface to store and retrieve any amount of data from anywhere on the web.

**4. What are some usages of S3?**
Can use Amazon S3:

- as primary storage for cloud-native applications
- as a bulk repository, or “data lake,” for analytics
- as a target for backup and recovery and disaster recovery
  with serverless computing.

**5. What is AWS EC2?**
EC2 stands for Amazon Elastic Compute Cloud. It is a web service that provides secure, resizable compute capacity in the cloud. It is designed to make web-scale computing easier for developers.

**6. What is a regions?**
A region is a physical location in the world where we have multiple Availability Zones (AZs).

**7. What is an Availability Zones?**
AZs consist of one or more discrete data centers, each with redundant power, networking,and connectivity, housed in separate facilities.

**8. What is an Edge Location?**

- Edge Locations are endpoints for AWS which are used for caching content.

- Typically this consists of CloudFront, Amazon’s content delivery network.

- There are many more Edge Locations than Regions.

**9. What is IAM?**
Essentially, IAM allows you to manage users and their level of access to the AWS Console.

**10. Critical terms of IAM?**

- Users - End Users (think people)
- Groups - A collection of Users under one set of permissions
- Roles - You create roles and can then be assign them to AWS resources
- Policies - A document that defines one (or more permissions). Can be attached to User/Group/Role.

**11. Size of the files on S3?**
From 0 Bytes to 5 TB

**12. What is the data consistency model for S3?**

- Read after Write consistency for PUTS of new Objects
- Eventual Consistency for overwrite PUTS and DELETES (can take some time to propagate)

**13. What are the two options for controlling access to a S3 bucket?**

- Bucket ACL
- Bucket Policies

**14. Lifecycle Management in S3?**

- Can be used in conjunction with versioning
- Can be applied to current versions and previous versions
- Following actions can now be done:
  - Transition to the Standard IA storage class
  - Archive to Glacier Storage Class
  - Permanently Delete

Updating...
