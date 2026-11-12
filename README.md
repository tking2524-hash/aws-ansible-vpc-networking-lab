# AWS VPC Networking with Ansible

## Project Overview

This project demonstrates how to automate AWS networking resources using Ansible. The goal was to create a Virtual Private Cloud (VPC), configure public and private subnets, establish routing through an internet gateway, and implement security group rules.

The project uses four reusable Ansible roles to organize the deployment. I also tested idempotence, verified the resources in the AWS Management Console, and successfully removed the infrastructure using a cleanup playbook.

**Project Status:** Successfully deployed, tested, verified, and removed.

**AWS Region:** US East (Ohio) — `us-east-2`

**GitHub Repository:** https://github.com/tking2524-hash/aws-ansible-vpc-networking-lab

## 1. Project Structure

The project includes the following files:

```text
aws-ansible-vpc-networking-lab/
├── .gitignore
├── README.md
├── deploy.yml
├── destroy.yml
├── inventory.ini
├── group_vars/
│   └── all.yml
└── roles/
    ├── vpc/
    │   └── tasks/main.yml
    ├── subnets/
    │   └── tasks/main.yml
    ├── gateway/
    │   └── tasks/main.yml
    └── security_group/
        └── tasks/main.yml
```

### Ansible Roles

- **vpc:** Creates and configures the AWS VPC.
- **subnets:** Creates the public and private subnets.
- **gateway:** Configures the internet gateway, route tables, routes, and subnet associations.
- **security_group:** Creates the web server security group and configures HTTP, HTTPS, and restricted SSH access.

Using separate roles makes the project easier to maintain, troubleshoot, and reuse.

## 2. Requirements and Environment

The following tools were used:

- AWS account with access to networking resources
- AWS Management Console
- AWS CloudShell
- Python 3.13
- Ansible Core 2.21.6
- Amazon AWS Ansible collection
- Boto3 and Botocore
- Git and GitHub

I used AWS CloudShell as my Ansible control system because it provided a terminal with access to my authenticated AWS environment.

### Installing Ansible

I created a Python virtual environment inside AWS CloudShell:

```bash
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
```

I then installed Ansible and the required AWS dependencies:

```bash
python -m pip install --upgrade pip
python -m pip install ansible boto3 botocore
ansible-galaxy collection install amazon.aws
```

I verified the installation using:

```bash
ansible --version
ansible-galaxy collection list amazon.aws
```

If CloudShell starts a new terminal session, the environment can be activated again using:

```bash
source ~/ansible-venv/bin/activate
```

## 3. AWS Authentication and Credential Security

I authenticated to AWS through the CloudShell environment rather than manually storing AWS access keys in my Ansible files.

I verified the AWS identity using:

```bash
aws sts get-caller-identity
```

The command completed successfully, confirming that my AWS session was authenticated.

To protect my credentials, I did not place AWS secret access keys, session tokens, or GitHub personal access tokens in my playbooks or Git repository.

The project also includes a `.gitignore` file to help prevent unnecessary local files and common credential files from being committed.

For environments outside CloudShell, AWS temporary credentials can be provided securely through environment variables or an approved AWS credentials profile.

## 4. AWS Network Configuration

I deployed the following networking configuration:

| Resource | Configuration |
|---|---|
| Region | `us-east-2` |
| VPC | `10.42.0.0/16` |
| Public subnet | `10.42.1.0/24` |
| Private subnet | `10.42.2.0/24` |
| Public availability zone | `us-east-2a` |
| Private availability zone | `us-east-2b` |
| Internet gateway | Attached to the VPC |
| Route tables | Separate public and private tables |
| Security group | HTTP, HTTPS, restricted SSH |

The configuration values are stored in `group_vars/all.yml` instead of being repeated throughout the playbooks.

### Public and Private Subnets

The public subnet uses a route table with a default route to the internet gateway:

```text
10.42.0.0/16 → local
0.0.0.0/0    → Internet Gateway
```

The private subnet uses only local VPC routing:

```text
10.42.0.0/16 → local
```

This design prevents the private subnet from having a direct internet route. No EC2 instances were deployed as part of this assignment.

### Security Group Configuration

I configured the following inbound rules:

| Type | Port | Source |
|---|---|---|
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |
| SSH | 22 | Restricted IPv4 `/32` |

The restricted SSH source was configured in the project variables rather than allowing SSH access from the entire internet.

## 5. Deploying the Network

I uploaded the Ansible project to AWS CloudShell, extracted it, and configured the project variables for the Ohio region.

Before deploying, I checked the playbook syntax:

```bash
ansible-playbook -i inventory.ini deploy.yml --syntax-check
```

The syntax check completed successfully.

I then deployed the network:

```bash
ansible-playbook -i inventory.ini deploy.yml
```

### Deployment Results

```text
PLAY RECAP
localhost : ok=10 changed=7 unreachable=0 failed=0
```

The deployment completed without errors.

I verified the VPC, subnets, internet gateway, route tables, and security group through the AWS Management Console.

## 6. Testing Idempotence

To verify that the playbook was idempotent, I ran the deployment again without changing the configuration:

```bash
ansible-playbook -i inventory.ini deploy.yml | tee idempotence_output.txt
```

### Second Deployment Results

```text
PLAY RECAP
localhost : ok=10 changed=0 unreachable=0 failed=0
```

The second run reported zero changes.

This confirmed that Ansible recognized the existing resources and did not attempt unnecessary modifications or create duplicate networking components.

## 7. AWS Resource Verification

I verified the following resources in the AWS Management Console:

- The VPC was available with CIDR `10.42.0.0/16`.
- Both public and private subnets existed and were available.
- The internet gateway was attached to the correct VPC.
- The public route table had an active default route to the internet gateway.
- The private route table contained only the local VPC route.
- The public and private route tables were associated with the correct subnets.
- The security group contained three inbound rules for HTTP, HTTPS, and restricted SSH.

Screenshots were captured to document the networking configuration.

## 8. Removing AWS Resources

After completing the verification and idempotence test, I used the cleanup playbook to remove the lab infrastructure.

I checked the playbook's resource selection and deletion order before running it:

```bash
ansible-playbook -i inventory.ini destroy.yml --syntax-check
```

I then executed:

```bash
ansible-playbook -i inventory.ini destroy.yml | tee cleanup_output.txt
```

### Cleanup Results

```text
PLAY RECAP
localhost : ok=11 changed=7 unreachable=0 failed=0
```

The cleanup completed successfully.

The playbook removed the dependent resources before deleting the VPC.

### Verifying Deletion

I verified the deletion using:

```bash
aws ec2 describe-vpcs \
  --region us-east-2 \
  --vpc-ids vpc-0dc6829911620a13e
```

AWS returned:

```text
InvalidVpcID.NotFound
```

This confirmed that the lab VPC no longer existed.

## 9. Problems Encountered and Solutions

### Ansible Not Installed

When I first opened AWS CloudShell, the `ansible` command was not recognized.

I resolved this by creating a Python virtual environment and installing Ansible and the required AWS dependencies.

### Virtual Environment Not Active

After opening a new CloudShell session, I received an `ansible-playbook: command not found` error.

I resolved the issue by activating the existing virtual environment again.

### AWS Region and VPC Identification

I needed to make sure the region in my Ansible variables matched the AWS Console region.

I configured the deployment for `us-east-2` and used AWS resource names, tags, and VPC IDs to confirm that I was viewing the correct infrastructure.

### GitHub Authentication

I initially received GitHub authentication errors when attempting to push my project.

I learned that GitHub does not accept regular account passwords for HTTPS Git operations. After creating a personal access token, I also encountered a 403 permission error because the token lacked the required repository Contents permission.

I corrected the token permissions to allow Contents: Read and write, then successfully pushed my project to GitHub.

## 10. GitHub Repository

The completed Ansible project was uploaded to:

https://github.com/tking2524-hash/aws-ansible-vpc-networking-lab

The repository contains the deployment playbook, cleanup playbook, variable configuration, inventory, README, and all four Ansible roles.

Git was used to track and publish the project files.

## 11. Reflection

**Which AWS networking component was the most difficult to automate?**

The route tables were the most challenging part because I had to understand how the public and private subnets use different routes. The public subnet needed access through the internet gateway, while the private subnet needed to remain isolated from direct internet access. I also had to make sure each route table was associated with the correct subnet.

**Why is the order of resource creation and removal important?**

The order is important because AWS networking resources depend on each other. I needed to create the VPC before creating subnets and attaching the internet gateway. During cleanup, I had to remove the security group, route tables, gateway, and subnets before deleting the VPC. Following the correct order prevented dependency errors.

**How did you protect the temporary AWS credentials?**

I used AWS CloudShell, which allowed me to authenticate through my AWS session without storing AWS access keys in the playbooks. I also kept my GitHub authentication token separate from the project and used a `.gitignore` file to reduce the risk of committing sensitive files.

**What would you change before using this network design in production?**

Before using this design in production, I would add more security and monitoring. I would use least-privilege IAM permissions, enable VPC Flow Logs, and consider private administrative access instead of direct SSH. I would also improve high availability by deploying across multiple availability zones and adding redundancy where needed.

## 12. Conclusion

This assignment helped me understand how Ansible can automate AWS networking resources using reusable roles. I successfully created and verified a VPC network with public and private subnets, tested idempotence with zero changes on the second run, and removed the infrastructure using Ansible.

I also gained hands-on experience with AWS CloudShell, Git, GitHub authentication, and troubleshooting cloud automation problems.

## References

- Ansible AWS Collection: https://docs.ansible.com/ansible/latest/collections/amazon/aws/
- AWS VPC Documentation: https://docs.aws.amazon.com/vpc/
- AWS Temporary Credentials: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html
- GitHub Personal Access Tokens: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
