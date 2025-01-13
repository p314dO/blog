---
title: Intro to AWS IAM Enumeration - PwnedLabs
date: 2024-11-29
description: This lab demonstrates how to use AWS CloudTrail logs to detect malicious activity and perform S3 enumeration. 
isStarred: false
---

![aws_iam_enum.png](aws_iam_enum.png)

Introduction to AWS CLI and IAM User, Role, Group, and Policy Enumeration

## [Lab in PwnedLabs](https://pwnedlabs.io/labs/intro-to-aws-iam-enumeration)

## Scenario

You are a security consultant hired by the global logistics company Huge Logistics. After suspicious activity, your task is to enumerate the IAM user `dev01` and map any potentially compromised resources. Your mission is to list and assess IAM roles, policies, and permissions.

## Real-World Context

IAM (Identity and Access Management) is critical for creating, defending, and attacking cloud services. Both offensive and defensive security professionals need a strong understanding of IAM and how to enumerate permissions: attackers look for overly permissive or misconfigured settings in a potential kill chain, while defenders ensure the enforcement of the least privilege principle and identify any resources or services within the blast radius of a compromised IAM user.

# Tutorial

### How can we interact with AWS?

1. **AWS Management Console**: A web interface that provides a graphical way to manage AWS resources.
2. **AWS CLI**: The command-line interface allows you to manage AWS services via the command line.
3. **SDK**: Software Development Kits are available in various languages like Python (Boto3), Java, .NET, and more, providing programmatic access to AWS services.
4. **API**: Most AWS services expose RESTful APIs that you can interact with directly.

### AWS Management Console

The URL for the AWS Management Console for IAM users looks like the following example. Our AWS account ID `794929857501` is part of the URL, and AWS automatically populates the account ID field on the IAM user login page. Continue, click the link, and log in with the provided IAM user credentials.

![image.png](image.png)

Once logged in, we see a collection of recently visited services. On the top left corner of the window, we can click `All services` and scroll through an alphabetical list of services.

![image.png](image%201.png)

The AWS Console offers a convenient way to access, create, modify, and delete resources.

Click on `GuardDuty` in the alphabetical list or open it after searching for the service. In the GuardDuty console that opens, select the region `us-west-1` from the dropdown list.

![image.png](image%202.png)

Although the AWS Management Console is very convenient, the AWS CLI is very powerful and allows us to process batch commands and perform automation, making our tasks easier.

This lab will focus on the AWS CLI since we want to identify the impact radius of the potentially compromised IAM user and what they might have accessed.

Install AWS CLI:

```csharp
sudo apt install awscli
```

In the AWS CLI, we authenticate using the AWS Access Key and Secret Key.

The access key and secret access key together form a set of credentials used to authenticate API requests and act as a kind of "username and password." These keys are generated through the AWS Management Console and are linked to an IAM user or the AWS root account.

1. **Access Key ID**: A 20-character alphanumeric string (e.g., "AKIAIOSFODNN7EXAMPLE"). It is used to identify the user or account making a programmatic request to an AWS service. It is meant to be shared, like a username.
2. **Secret Access Key**: A 40-character string (e.g., "wJalrXUtnFEMI/K7MEXAMPLEKEYEXAMPLEKEY"), which acts as the "password" to authenticate requests made with the corresponding Access Key ID. Unlike the Access Key ID, the secret access key is confidential and should never be shared or stored insecurely.

We can configure the keys with the AWS CLI command `aws configure`, as shown below.

```csharp
aws configure --profile lab
```

![image.png](image%203.png)

Once the keys are configured, we can interact with AWS in the context of this user.

## Starting the Enumeration

The following command can be considered the equivalent of `whoami` in Windows and Linux. As expected, this reveals that we are in the context of the IAM user named `dev01` in the AWS account `794929857501`.

```python
aws sts get-caller-identity --profile lab
```

![image.png](image%204.png)

Let’s get more information about our potentially compromised user.

```csharp
aws iam get-user --profile lab
```

![image.png](image%205.png)

This also provides the creation date of the IAM user and the associated tag. A tag is a key-value pair that can be attached to AWS resources, including IAM users. Tags make it easier to manage, search, and filter resources.

We can also issue the following command to list any groups the user may belong to. A user group is a collection of IAM users, allowing multiple users to inherit the permissions assigned to the group. We found that the IAM user is not a member of any group.

```csharp
aws iam list-groups-for-user --user-name dev01
```

![image.png](image%206.png)

## **List Attached User Policies**

Instead, we can try to list the attached user policies.

```csharp
aws iam list-attached-user-policies --user-name dev01
```

![image.png](image%207.png)

It seems we have permissions to list them, and this reveals that the Amazon managed policy `AmazonGuardDutyReadOnlyAccess` and a customer-managed policy named `dev01` have been attached to our IAM user. GuardDuty is a threat detection service that continuously monitors malicious or unauthorized behaviors to help protect your AWS accounts and workloads. The `PolicyArn` refers to the Amazon resource name, which is a combination of the AWS account ID, resource type, and resource name. The ARN is a globally unique reference to the object.

## List Inline Policies

Next, let’s check for any inline policies. An inline policy in AWS IAM is a policy that is embedded directly into a single user, group, or role in IAM and cannot be reused or managed separately.

```csharp
aws iam list-user-policies --user-name dev01
```

![image.png](image%208.png)

This reveals an interesting inline policy called `S3_Access`. Let’s investigate and examine the policy to see what permissions we have!

## Diving into the Policies

First, let’s list the versions of the `AmazonGuardDutyReadOnlyAccess` policy. Both Amazon-managed and customer-managed policies can have multiple versions, allowing you to preserve, review, and revert to previous policy versions. Inline policies do not support versioning.

```csharp
aws iam list-policy-versions --policy-arn arn:aws:iam::aws:policy/AmazonGuardDutyReadOnlyAccess
```

![image.png](image%209.png)

The latest version is v4. Running the following command shows that all `Describe`, `Get`, and `List` actions are allowed.

```csharp
aws iam get-policy-version --policy-arn arn:aws:iam::aws:policy/AmazonGuardDutyReadOnlyAccess --version-id v4
```

![image.png](image%2010.png)

![image.png](image%2011.png)

`Describe` allows us to retrieve details about GuardDuty resources. It will permit anything that begins with "Describe" in the GuardDuty API operations, such as describing detectors, findings, or configurations. `Get` allows us to obtain findings, threat intelligence sets, or filter details, while `List` allows us to enumerate detectors, members, or findings. Generally, it is used to list resources but not their detailed information.

Next, let’s list the versions of the customer-managed policy named `dev01` and see that the latest version is v7.

```csharp
aws iam list-policy-versions --policy-arn arn:aws:iam::794929857501:policy/dev01 --profile lab
```

![image.png](image%2012.png)

Let’s also retrieve the JSON document for this policy. It shows several allowed actions that apply to a user, role, and policies in the Resource section.

```csharp
aws iam get-policy-version --policy-arn arn:aws:iam::794929857501:policy/dev01 --version-id v7 --profile lab
```

![image.png](image%2013.png)

We can map the allowed actions as they apply to the various resources listed in the Resource section of the JSON document.

1. IAM User (`arn:aws:iam::794929857501:user/dev01`):
- `iam:GetUser`
- `iam:ListAttachedUserPolicies`
- `iam:ListUserPolicies`
- `iam:GetUserPolicy`
- `iam:ListGroupsForUser`
2. IAM Role (`arn:aws:iam::794929857501:role/BackendDev`):
- `iam:GetRole`
- `iam:GetRolePolicy`
- `iam:ListAttachedRolePolicies`
3. Customer Managed Role Policy (`arn:aws:iam::794929857501:policy/BackendDevPolicy`):
- `iam:GetPolicy`
- `iam:GetPolicyVersion`
- `iam:ListPolicyVersions`
4. Customer Managed Policy (`arn:aws:iam::794929857501:policy/dev01`):
- `iam:GetPolicy`
- `iam:GetPolicyVersion`
- `iam:ListPolicyVersions`
5. Amazon Managed Policy (`arn:aws:iam::aws:policy/AmazonGuardDutyReadOnlyAccess`):
- `iam:GetPolicy`
- `iam:GetPolicyVersion`
- `iam:ListPolicyVersions`


This customer-managed policy attached to the `dev01` IAM user grants permissions to retrieve details about itself, any group it belongs to, and its attached policy. It also grants permission to retrieve details about the `BackendDev` role, list the policies attached to the role, and retrieve information about a policy called `BackendDevPolicy`, which we assume is associated with the role. It also allows us to retrieve information about an Amazon-managed policy.

Let's confirm our assumption about `BackendDev` and `BackendDevPolicy`.

```csharp
aws iam list-attached-role-policies --role-name BackendDev --profile lab
```

![image.png](image%2014.png)

Next, let's continue by listing the `BackendDev` role and `BackendDevPolicy` policy (ARNs listed below).

```csharp
arn:aws:iam::794929857501:role/BackendDev
arn:aws:iam::794929857501:policy/BackendDevPolicy
```

We can use the `get-role` IAM action to retrieve information about the role. This reveals that the purpose of the role is to allow developers to assume it. Assuming a role can be considered somewhat like the `sudo` command in Linux, as it provides temporary credentials that allow us to act in the role’s privileged context. Currently, only the IAM user `dev01` is allowed to assume the role.

```csharp
aws iam get-role --role-name BackendDev --profile lab
```

![image.png](image%2015.png)

Let’s continue our investigation and list the policy. The `get-policy` action reveals that the current version of the policy is v1.

```csharp
aws iam get-policy --policy-arn arn:aws:iam::794929857501:policy/BackendDevPolicy --profile lab
```

![image.png](image%2016.png)

We can use `get-policy-version` to retrieve the associated JSON policy document for this version (currently the only version). If we assume the role, we’ll be able to retrieve information about all EC2 instances in the AWS account and list all the secrets currently stored in SecretsManager. We will also be able to describe and retrieve the secret value for `prod/Customers`!

```csharp
aws iam get-policy-version --policy-arn arn:aws:iam::794929857501:policy/BackendDevPolicy --version-id v1 --profile lab
```

![image.png](image%2017.png)

Finally, let’s examine the inline user policy using the `get-user-policy` action. This shows that we can list and retrieve any content in the potentially confidential S3 bucket called `hl-dev-artifacts`.

```csharp
aws iam get-user-policy --user-name dev01 --policy-name S3_Access --profile lab
```

![image.png](image%2018.png)

Let’s visually map what someone using this potentially compromised account could access.

![image.png](image%2019.png)

```csharp
aws s3 ls s3://hl-dev-artifacts --profile lab
```

![image.png](image%2020.png)

![image.png](image%2021.png)

## Extra Mile

Assume the role and access the Secrets Manager secret.

[How assume a role?](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security/aws-privilege-escalation/aws-sts-privesc#sts-assumerole)

![image.png](image%2022.png)

![image.png](image%2023.png)

[Secrets Manager Enumeration](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security/aws-services/aws-secrets-manager-enum#enumeration)

![image.png](image%2024.png)

![image.png](image%2025.png)

---

## References

- [Lab in PwnedLabs](https://pwnedlabs.io/labs/intro-to-aws-iam-enumeration)
- [IAM Enumeration Hacktricks](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security/aws-services/aws-iam-enum#enumeration)
- [STS: Assume a Role](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security/aws-privilege-escalation/aws-sts-privesc#sts-assumerole)
- [Secrets Manaer Enumeration](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security/aws-services/aws-secrets-manager-enum#enumeration)
