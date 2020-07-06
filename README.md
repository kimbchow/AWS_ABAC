# Attribute Based Access Control Workshop
Attribute-based access control (ABAC) is an authorization strategy that defines permissions based on attributes. In AWS, these attributes are called tags. You can attach tags to IAM principals (users or roles) and to AWS resources. You can then define policies that use tag condition keys to grant permissions to your principals based on their tags. When you use tags to control access to your AWS resources, you allow your teams and resources to grow with fewer changes to AWS policies. ABAC policies are more flexible than traditional AWS policies, which require you to list each individual resource.

## Tutorial Overview

This tutorial shows how to create and test a policy that allows IAM roles with principal tags to access resources with matching tags. When a principal makes a request to AWS, their permissions are granted based on whether the principal and resource tags match. This strategy allows individuals to view or edit only the AWS resources required for their jobs.

## Scenario

Assume that you're a lead developer at a large company named Example Corporation, and you're an experienced IAM administrator. You're familiar with creating and managing IAM users, roles, and policies. You want to ensure that your development engineers and quality assurance team members can access the resources they need. You also need a strategy that scales as your company grows.

You choose to use AWS resource tags and IAM role principal tags to implement an ABAC strategy for services that support it, beginning with AWS Secrets Manager. To learn which services support authorization based on tags, see AWS Services That Work with IAM. To learn which tagging condition keys you can use in a policy with each service's actions and resources, see Actions, Resources, and Condition Keys for AWS Services. You can configure your SAML-based or web identity provider to pass session tags to AWS. When your employees federate into AWS, their attributes are applied to their resulting principal in AWS. You can then use ABAC to allow or deny permissions based on those attributes. To learn how using session tags with a SAML federated identity differs from this tutorial, see Using SAML Session Tags for ABAC.

Your Engineering and Quality Assurance team members are on either the Pegasus or Unicorn project. You choose the following 3-character project and team tag values:

* access-project = peg for the Pegasus project
* access-project = uni for the Unicorn project
* access-team = eng for the Engineering team
* access-team = qas for the Quality Assurance team

## Summary of Key Decisions

Employees sign in with IAM user credentials and then assume the IAM role for their team and project. If your company has its own identity system, you can set up federation to allow employees to assume a role without IAM users. For more information, see Using SAML Session Tags for ABAC.

* The same policy is attached to all of the roles. Actions are allowed or denied based on tags.
* Employees can create new resources, but only if they attach the same tags to the resource that are applied to their role. This ensures that employees can view the resource after they create it. Administrators are no longer required to update policies with the ARN of new resources.
* Employees can read resources owned by their team, regardless of the project.
* Employees can update and delete resources owned by their own team and project.
* IAM administrators can add a new role for new projects. They can create and tag a new IAM user to allow access to the appropriate role. Administrators are not required to edit a policy to support a new project or team member.

In this tutorial, you will tag each resource, tag your project roles, and add policies to the roles to allow the behavior previously described. The resulting policy allows the roles Create, Read, Update, and Delete access to resources that are tagged with the same project and team tags. The policy also allows cross-project Read access for resources that are tagged with the same team.

<img width="422" alt="Screen Shot 2020-07-06 at 2 58 21 AM" src="https://user-images.githubusercontent.com/50940575/86540067-9c514d80-bf34-11ea-894a-0b9ac646a0a7.png">

## Prerequisites
To perform the steps in this tutorial, you must do the following: 
* Use the Event Engine Hash for Workshop Number 2

* Step 1 : Retrieve temporary credentials from Event Engine
* Navigate to the Event Engine dashboard
* Enter your team hash code.
* Click AWS Console
* Click Open Console from the Event Engine window

* Step 2: Obtain your 12-digit account ID, which you use to create the roles in step 3. Copy and paste this account ID for use in Step 3. 
* To find your AWS account ID number using the AWS Management Console, choose Support on the navigation bar on the upper right, and then choose Support Center. The account number (ID) appears in the navigation pane on the left.

![ABACTutorial](https://user-images.githubusercontent.com/50940575/86540279-86448c80-bf36-11ea-882c-1cf8a4cc2826.png)

## Step 1: Create abac-assume-role permissions policy
In this step, you will create create four IAM users with permissions to assume roles with the same tags. This makes it easier to add more users to your teams. When you tag the users, they automatically get access to assume the correct role. You don't have to add the users to the trust policy of the role if they work on only one project and team.

* Navigate to the IAM service, and click on policies on the left hand side of the console. 

<img width="886" alt="Screen Shot 2020-07-06 at 5 15 10 AM" src="https://user-images.githubusercontent.com/50940575/86542359-b9dbe280-bf47-11ea-84b9-5e80c34c9b1d.png">

* Click on "Create Policies" and then "JSON" 

* Add the following AssumeRole Policy: Assume Any ABAC Role, But Only When the User and Role Tags Match

The following policy allows a user to assume any role in your account with the access-name prefix. The role must also be tagged with the same project, team, and cost center tags as the user.


```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "TutorialAssumeRole",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::123456789012:role/access-*",
            "Condition": {
                "StringEquals": {
                    "iam:ResourceTag/access-project": "${aws:PrincipalTag/access-project}",
                    "iam:ResourceTag/access-team": "${aws:PrincipalTag/access-team}",
                    "iam:ResourceTag/cost-center": "${aws:PrincipalTag/cost-center}"
                }
            }
        }
    ]
}
```
* Add the JSON policy above, changing the AWS Account Number to the account number that you obtained in the Prerequisite section of this workshop. 

![JSON Policy](https://user-images.githubusercontent.com/50940575/86540600-3d420780-bf39-11ea-95ad-bb7300add57f.png)

* Name this policy **ABAC-assume-role** and click Save. 

## Step 2: Create ABAC Group
Create a group called **ABAC** You will assign the **ABAC-assume-role** policy you created in step 1 to this Group.

* Navigate back to the IAM Console, and click on Groups. Click on "Create new Group", and create a group called "ABAC".

<img width="591" alt="Screen Shot 2020-07-06 at 3 51 36 AM" src="https://user-images.githubusercontent.com/50940575/86541057-5b5d3700-bf3c-11ea-8627-3595884f5c01.png">

* Attach the **ABAC-assume-role** policy to the Group and click Create Group. 

<img width="862" alt="Screen Shot 2020-07-06 at 3 54 57 AM" src="https://user-images.githubusercontent.com/50940575/86541085-9bbcb500-bf3c-11ea-8ff9-5f8dc53735a0.png">

## Step 3: Create Test Users
In this step you will create the following IAM users, and attach them to the **ABAC Group** created in the previous step, and add the following tags.

| UserName  | User Tags |
| ------------- | ------------- |
| access-Arnav-peg-eng  | access-project = peg, access-team = eng , cost-center = 987654  |
| access-Mary-peg-qas  | access-project = peg, access-team = qas, cost-center = 987654  |
| access-Saanvi-uni-eng  | access-project = uni, access-team = eng, cost-center = 123456  |
| access-Carlos-uni-qas  | access-project = uni, access-team = qas, cost-center = 123456 |

* Click on "Users", and "Create Users" on the left hand side of the IAM Console. Create users. Please create a custom password for these users and keep a copy of them. You will use them later when logging into the console in step 6. 

<img width="494" alt="Screen Shot 2020-07-06 at 5 17 19 AM" src="https://user-images.githubusercontent.com/50940575/86542394-01fb0500-bf48-11ea-9ca5-fc3eafe5fcaa.png">

* Click on "Add User to Group", and add users to the **ABAC** group you created in step 2. This will now give the user the permissions you set up in step 2. 

<img width="848" alt="Screen Shot 2020-07-06 at 4 01 49 AM" src="https://user-images.githubusercontent.com/50940575/86541187-81cfa200-bf3d-11ea-868c-102fdc7905c6.png">

* Now, add the **Tags** for each of the users within the table above. 

![Tags](https://user-images.githubusercontent.com/50940575/86541244-edb20a80-bf3d-11ea-98aa-46cd9b06103a.png)

* Click "Create User" 

* Repeat for each of the 4 users listed in the table. 

* You should end up with 4 users created. 

![Users](https://user-images.githubusercontent.com/50940575/86541439-7da48400-bf3f-11ea-9958-6e441d1d70e8.png)




## Step 4: Create ABAC Policy

* Create the following policy named **access-same-project-team**. You will add this policy to the roles in a later step. 

**ABAC Policy: Access Secrets Manager Resources Only When the Principal and Resource Tags Match

The following policy allows principals to create, read, edit, and delete resources, but only when those resources are tagged with the same key-value pairs as the principal. When a principal creates a resource, they must add access-project, access-team, and cost-center tags with values that match the principal's tags. The policy also allows adding optional Name or OwnedBy tags.

```
{
 "Version": "2012-10-17",
 "Statement": [
     {
         "Sid": "AllActionsSecretsManagerSameProjectSameTeam",
         "Effect": "Allow",
         "Action": "secretsmanager:*",
         "Resource": "*",
         "Condition": {
             "StringEquals": {
                 "aws:ResourceTag/access-project": "${aws:PrincipalTag/access-project}",
                 "aws:ResourceTag/access-team": "${aws:PrincipalTag/access-team}",
                 "aws:ResourceTag/cost-center": "${aws:PrincipalTag/cost-center}"
             },
             "ForAllValues:StringEquals": {
                 "aws:TagKeys": [
                     "access-project",
                     "access-team",
                     "cost-center",
                     "Name",
                     "OwnedBy"
                 ]
             },
             "StringEqualsIfExists": {
                 "aws:RequestTag/access-project": "${aws:PrincipalTag/access-project}",
                 "aws:RequestTag/access-team": "${aws:PrincipalTag/access-team}",
                 "aws:RequestTag/cost-center": "${aws:PrincipalTag/cost-center}"
             }
         }
     },
     {
         "Sid": "AllResourcesSecretsManagerNoTags",
         "Effect": "Allow",
         "Action": [
             "secretsmanager:GetRandomPassword",
             "secretsmanager:ListSecrets"
         ],
         "Resource": "*"
     },
     {
         "Sid": "ReadSecretsManagerSameTeam",
         "Effect": "Allow",
         "Action": [
             "secretsmanager:Describe*",
             "secretsmanager:Get*",
             "secretsmanager:List*"
         ],
         "Resource": "*",
         "Condition": {
             "StringEquals": {
                 "aws:ResourceTag/access-team": "${aws:PrincipalTag/access-team}"
             }
         }
     },
     {
         "Sid": "DenyUntagSecretsManagerReservedTags",
         "Effect": "Deny",
         "Action": "secretsmanager:UntagResource",
         "Resource": "*",
         "Condition": {
             "StringLike": {
                 "aws:TagKeys": "access-*"
             }
         }
     },
     {
         "Sid": "DenyPermissionsManagement",
         "Effect": "Deny",
         "Action": "secretsmanager:*Policy",
         "Resource": "*"
     }
 ]
}
```

![ABACpolicy](https://user-images.githubusercontent.com/50940575/86541482-decc5780-bf3f-11ea-9c30-bb45cb9ad594.png)

## Step 5: Create IAM Roles 

* Create the following IAM roles and attach the **access-same-project-team** policy that you created in the previous step. 


| Job Function | Role Tags | Role Name | Role Description | 
| ------------ | ----------- | ----------| ----------------|
| Project Pegasus Engineering | access-project = peg, access-team = eng, cost-center = 987654 | access-peg-engineering | Allows engineers to read all engineering resources and create and manage Pegasus engineering resources.| 
| Project Pegasus Quality Assurance | access-project = peg, access-team = qas, cost-center = 987654 | access-peg-quality-assurance | Allows the QA team to read all QA resources and create and manage all Pegasus QA resources.
| Project Unicorn Engineering | access-project = uni, access-team = eng, cost-center = 123456 | access-uni-engineering | Allows engineers to read all engineering resources and create and manage Unicorn engineering resources.
| Project Unicorn Quality Assurance | access-project = uni, access-team = qas, cost-center = 123456 | access-uni-quality-assurance | Allows the QA team to read all QA resources and create and manage all Unicorn QA resources.

* Click on Roles, and Create Roles. 

* Choose Another AWS Account. Add in the Account ID that you obtained in Step 1. 

<img width="948" alt="Screen Shot 2020-07-06 at 5 20 56 AM" src="https://user-images.githubusercontent.com/50940575/86542475-9ebda280-bf48-11ea-9e65-e62fdc3dd78c.png">


* Attach the **access-same-project-team** policy to the role.

![Rolepermission](https://user-images.githubusercontent.com/50940575/86541794-5c916280-bf42-11ea-9ebd-a18d72f51b39.png)

* Enter the role details in the table above. Click Create Role. 

<img width="581" alt="Screen Shot 2020-07-06 at 11 29 21 AM" src="https://user-images.githubusercontent.com/50940575/86552959-03debb80-bf7c-11ea-817c-4b46b7b68eca.png">

* Repeat these steps for all 4 roles. You should now have created these 4 roles. 

<img width="905" alt="Screen Shot 2020-07-06 at 11 32 09 AM" src="https://user-images.githubusercontent.com/50940575/86553117-6041db00-bf7c-11ea-886c-1abfcd2a9102.png">


## Step 6: Test Attribute-based access to Secrets Manager
The permissions policy attached to the roles allows the employees to create secrets. This is allowed only if the secret is tagged with their project, team, and cost center. Confirm that your permissions are working as expected by signing in as your users, assuming the correct role, and testing activity in Secrets Manager.

**To test creating a secret with and without the required tags:**

* Sign out of your current AWS account, and login as the **access-Arnav-peg-eng** IAM user, using the user credentials you created in step 1. 

<img width="786" alt="Screen Shot 2020-07-06 at 4 51 52 AM" src="https://user-images.githubusercontent.com/50940575/86542023-7469e600-bf44-11ea-8987-aeaad1b2d52e.png">

* Navigate to Secrets Manager from the console. Note that you have access to secrets manager using this role. 

* Switch roles to the **access-uni-engineering role** from the Secrets Manager Console. 

<img width="293" alt="Screen Shot 2020-07-06 at 4 55 55 AM" src="https://user-images.githubusercontent.com/50940575/86542102-0245d100-bf45-11ea-9192-5b823c60befa.png">

<img width="925" alt="Screen Shot 2020-07-06 at 4 56 43 AM" src="https://user-images.githubusercontent.com/50940575/86542108-20133600-bf45-11ea-9da2-7d2166fb0d29.png">

* Note that this operation fails because the access-project and cost-center tag values do not match for the access-Arnav-peg-eng user and access-uni-engineering role.

<img width="1002" alt="Screen Shot 2020-07-06 at 4 58 01 AM" src="https://user-images.githubusercontent.com/50940575/86542122-4d5fe400-bf45-11ea-8873-3125008c5e06.png">

* Click Cancel. 

* Try assuming roles into each of the other accounts. 

## Conclusion
You've now successfully completed all of the steps necessary to use tags for attribute-based access control (ABAC). You've learned how to define a tagging strategy. You applied that strategy to your principals and resources. You created and applied a policy that enforces the strategy for Secrets Manager. You also learned that ABAC scales easily when you add new projects and team members. As a result, you are able to sign in to the IAM console with your test roles and experience how to use tags for ABAC in AWS.


