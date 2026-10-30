# Environment Isolation for Dev / Test / Prod Using VPC Design and IAM

> **Confidentiality note:** This guide was originally built during my internship at CloudThat Technologies. In line with company policy, which does not permit sharing credentials or AWS account IDs, this write-up contains no credentials, account IDs, user identities, company branding or screenshots from that environment. Every identifier below is a placeholder such as `<YOUR_ACCOUNT_ID>`. CIDR ranges are examples only.

## 1. Overview

This project implements strict environment isolation inside a single AWS account. Development, Test/UAT and Production each get their own VPC, and access is controlled through IAM roles that are assumed with AWS STS. This balances governance and security with cost and operational simplicity.

Each environment is isolated at both the network layer and the identity layer, so a failure, misconfiguration or unauthorized access in one environment does not affect the others, especially Production.

## 2. Isolation Boundaries

**Network**
- Each environment has a dedicated VPC with a unique, non overlapping CIDR block.
- No VPC peering, Transit Gateway or shared routing between Dev, Test and Prod.
- Each VPC has its own route tables, subnets, security groups and internet gateway.

**Identity and access**
- No direct IAM user access to Production resources.
- Access is granted only through IAM roles using STS `AssumeRole`.
- One role per environment: `Dev-Access-Role`, `Test-Access-Role`, `Prod-Access-Role`.

**Permissions**
- Write access is restricted with tag based IAM policies.
- A role can modify only resources tagged with its own `Environment` value (for example, the Prod role only touches `Environment=Prod`).
- All access is temporary (1 hour maximum session) and auditable.

## 3. Governance Justification

- Prevents accidental or unauthorized access to Production.
- Enforces separation of duties.
- Removes long lived Production credentials.
- Supports auditing through AWS CloudTrail.
- Reduces the blast radius of mistakes.

## 4. Architecture

_Architecture diagram: to be added._

Three isolated VPCs, each containing:
- Public subnets (for the load balancer)
- Private subnets (application and database workloads)
- A dedicated internet gateway
- Environment specific route tables

There is no peering or routing between the VPCs.

The IAM layer sits on top of the network isolation: a base IAM user or SSO identity assumes an environment specific role, tag based policies limit what that role can change, and read only visibility is available across environments.

**Example CIDR plan (use your own, and make sure the ranges do not overlap):**

| Environment | Example VPC CIDR |
|-------------|------------------|
| Dev         | 10.10.0.0/16     |
| Test        | 10.20.0.0/16     |
| Prod        | 10.30.0.0/16     |

## 5. Build Guide

### 5.1 Network setup

1. Create three VPCs (Dev, Test, Prod) with non overlapping CIDR blocks.
2. In each VPC, create public and private subnets in at least two Availability Zones.
3. Create an internet gateway for each VPC and attach it to that VPC only.
4. Create separate route tables per VPC and associate them with that VPC's subnets. Do not add peering or cross VPC routes.
5. Create one security group per environment. Allow inbound traffic only from within the same environment and add no rules that reference another environment.
6. Launch a sample EC2 instance in each environment.
7. Apply the mandatory tag `Environment` with the value `Dev`, `Test` or `Prod` to every resource, including the instances.

### 5.2 IAM configuration

1. Create the roles `Dev-Access-Role`, `Test-Access-Role` and `Prod-Access-Role`.
2. Configure each role's trust policy to allow `sts:AssumeRole`. Example (replace the placeholder, and prefer a specific user or role over the account root where you can):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<YOUR_ACCOUNT_ID>:root" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

3. Attach the AWS managed `ReadOnlyAccess` policy to each role.
4. Add an inline, tag based policy to each role so it can change only its own environment. Example for the Prod role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "aws:ResourceTag/Environment": "Prod" }
      }
    }
  ]
}
```

5. Grant `sts:AssumeRole` on these roles to the source IAM users or SSO identities that should be able to use them.
6. Set the maximum session duration to 1 hour.

### 5.3 Auditing

Enable an AWS CloudTrail trail for management events so that `AssumeRole` calls and administrative actions across all environments are logged to an S3 bucket.

## 6. Validation and Isolation Testing

**Test 1: Network isolation**
- Confirm the Dev and Prod CIDR blocks do not overlap.
- From a Dev instance, try to reach the private IP of a Prod instance (for example with `ping` or `curl`).
- Expected result: the traffic fails or times out.

**Test 2: Read only visibility**
- Assume `Prod-Access-Role` and view Dev and Test resources.
- Expected result: read only access works.
- Then try to modify a Dev resource while in the Prod role.
- Expected result: `AccessDenied`.

_Record your own results and outputs here after running the tests._

## 7. Cleanup

Terminate the sample instances, and delete any NAT gateways and Elastic IPs when you finish, to avoid ongoing charges.
