# AWS VPC Networking with Ansible

## Purpose
Create one AWS VPC, a public and a private subnet, an internet gateway, two subnet-associated route tables, and a web security group using four reusable Ansible roles. Run a second deployment to evaluate idempotence and remove everything with `destroy.yml`.

## Project layout
- `deploy.yml`: runs `vpc`, `subnets`, `gateway`, and `security_group` in dependency order.
- `destroy.yml`: removes resources in reverse dependency order.
- `group_vars/all.yml`: regional settings, CIDR blocks, availability zones, names, and restricted SSH source.
- `roles/vpc/tasks/main.yml`: provisions the VPC.
- `roles/subnets/tasks/main.yml`: provisions the public and private subnets.
- `roles/gateway/tasks/main.yml`: attaches an internet gateway and manages public/private routes and associations.
- `roles/security_group/tasks/main.yml`: permits inbound HTTP (80), HTTPS (443), and SSH (22) from one trusted public IPv4 address.

## Requirements and setup
Use an Ansible controller with Python, Ansible Core, AWS CLI v2, and internet access to the AWS APIs. Linux or WSL Ubuntu is recommended; run the playbook locally rather than against an EC2 instance. The controller must have authorization to create/delete VPC networking resources. Restricted AWS Educate/AWS Academy labs may limit regions, AZs, or IAM actions.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install ansible-core -r requirements.txt
ansible-galaxy collection install -r requirements.yml
ansible --version
ansible-galaxy collection list amazon.aws
aws --version
```

If you already have Ansible installed, only install boto3/botocore into the Python interpreter running your local Ansible tasks and install the collection.

### Temporary AWS credentials
Start the AWS Educate/lab session, copy the **temporary** AWS access key ID, secret access key, and session token from the lab's credentials panel to the controller environment. For Linux/WSL, enter the following commands directly in a trusted terminal, replacing placeholders privately; **do not write real credentials to files in the Git project**:

```bash
export AWS_ACCESS_KEY_ID='YOUR_TEMPORARY_KEY'
export AWS_SECRET_ACCESS_KEY='YOUR_TEMPORARY_SECRET'
export AWS_SESSION_TOKEN='YOUR_TEMPORARY_SESSION_TOKEN'
export AWS_DEFAULT_REGION='us-east-1'
aws sts get-caller-identity
```

Alternatively, use the lab-provided AWS CLI profile in `~/.aws/credentials` (outside this Git repository). Refresh all three temporary credential values if the session expires. Never commit credentials, terminal transcripts with secrets, or screenshots revealing tokens. The `.gitignore` excludes common local secret files. The `ansible` AWS modules use Boto3's credential provider chain; credentials are not hardcoded in any playbook.

## Required variables
Edit `group_vars/all.yml` *before running*:

| Variable | Example | Meaning |
| --- | --- | --- |
| `aws_region` | `us-east-1` | Lab-permitted AWS region |
| `project_name` | `student-ansible-vpc-lab` | Unique prefix for resources |
| `vpc_cidr` | `10.42.0.0/16` | VPC network |
| `public_subnet_cidr` | `10.42.1.0/24` | Public subnet range |
| `private_subnet_cidr` | `10.42.2.0/24` | Private subnet range |
| `public_az` | `us-east-1a` | Public subnet AZ |
| `private_az` | `us-east-1b` | Private subnet AZ |
| `ssh_allowed_cidr` | `YOUR_PUBLIC_IP/32` | Required restricted inbound SSH source |

Set the AZs to ones available in your specific lab. Public and private subnet CIDRs must not overlap. To find your public IPv4 address, use a trusted IP-checking service; append `/32`. The bundled documentation-only `203.0.113.10/32` is intentionally blocked by a preflight assertion.

## Deployment and verification
Run from the **project root**:

```bash
ansible-playbook -i inventory.ini deploy.yml | tee evidence/deployment.txt
```

In the AWS Console, use **VPC → Your VPCs**, **Subnets**, **Internet gateways**, **Route tables**, and **Security groups**; confirm names and region. The gateway should be attached to the lab VPC. The public route table should have `0.0.0.0/0 -> igw-...` and an association with the public subnet. The private route table should be associated with the private subnet and have only the local VPC route. The public subnet has `Auto-assign public IPv4` enabled; the private subnet has it disabled. No EC2 instance is deployed, so internet connectivity itself is not tested by this assignment.

**AWS CLI checks:** use the name/ID printed by the playbook, replacing `VPC_ID` with your actual VPC ID:

```bash
aws ec2 describe-vpcs --filters 'Name=tag:Name,Values=student-ansible-vpc-lab-vpc' --region us-east-1
aws ec2 describe-subnets --filters 'Name=vpc-id,Values=VPC_ID' --region us-east-1
aws ec2 describe-internet-gateways --filters 'Name=attachment.vpc-id,Values=VPC_ID' --region us-east-1
aws ec2 describe-route-tables --filters 'Name=vpc-id,Values=VPC_ID' --region us-east-1
aws ec2 describe-security-groups --filters 'Name=vpc-id,Values=VPC_ID' --region us-east-1
```

## Idempotence
Run exactly the same command again without changing any variables or resources:

```bash
ansible-playbook -i inventory.ini deploy.yml | tee evidence/idempotence.txt
```

Review the play recap. Expected result is no duplicate resources and ideally `changed=0` on the second run. The read-only debug/recording steps should also report `ok`. If a cloud module still reports `changed`, inspect the task results and the AWS Console before assuming the operation was not idempotent.

## Cleanup
**Capture deployment and idempotence screenshots before cleanup.** Then run:

```bash
ansible-playbook -i inventory.ini destroy.yml | tee evidence/cleanup.txt
```

The destroy playbook looks up the VPC by its exact project Name tag and refuses to delete anything if multiple VPCs match. The order is: web security group → public/private route tables → internet gateway → subnets → VPC. It does not delete the default/main VPC route table; AWS removes that when deleting the VPC. It does **not** remove unrelated resources, and VPC deletion can fail if EC2 instances or other dependencies have been added. Verify that the project VPC no longer appears under **Your VPCs** and check the corresponding views for related resources. If the lab session credentials have expired, refresh them before cleanup.

## Screenshots and evidence checklist
Save screenshots of the VPC details, two subnets, attached gateway, public/private routes **and associations**, security-group inbound rules, deployment recap, idempotence recap, and VPC absence after cleanup. Keep actual CLI/playbook logs in `evidence/` locally; they are ignored from Git by default so you can review them for sensitive information before deciding what to upload separately to Moodle.

## Common problems / troubleshooting
- `Unable to locate credentials` or `ExpiredToken`: set all three temporary AWS environment variables again in the same terminal.
- `AccessDenied` / `UnauthorizedOperation`: your classroom account does not have the required API permission. Ask the instructor; do not attempt to bypass lab policies.
- `InvalidParameterValue` for availability zone: choose permitted AZs within your configured region.
- AWS resources missing from console: switch the console to the same AWS region as `aws_region`.
- `DependencyViolation` on cleanup: locate added EC2 instances, ENIs, or other resources blocking deletion; remove only resources you own.
- Python `boto3` errors: install requirements into the Python environment used by Ansible on localhost.
- Second run unexpectedly changes resources: inspect task output, resource names/tags, drift, and route/SG rules.

## Git workflow and submission
```bash
git init
git add ansible.cfg inventory.ini requirements.yml requirements.txt .gitignore .env.example group_vars roles deploy.yml
git commit -m 'Add modular AWS VPC deployment roles'
git add destroy.yml README.md
git commit -m 'Add AWS cleanup automation and documentation'
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO.git
git push -u origin main
```

For a preexisting repository, omit `git init` and use its remote URL. Do not claim live deployment/idempotence evidence until it has actually run in AWS. Submit the project archive, real Git URL, README, real task output, screenshots, deletion evidence, and the short reflection below.

## Reflection (edit after completing the lab)
The route tables and subnet associations were the most challenging parts to automate because the public subnet requires a default route through the internet gateway while the private subnet must not have one. Resource creation order matters because a VPC must exist before subnets and gateways can attach, and resources must be detached and removed before the VPC can be deleted. I protected the temporary AWS credentials by storing them outside the repository, using environment variables, and avoiding passwords or tokens in Git commits and screenshots. For production, I would use separate environments, stricter access controls, infrastructure review/testing, monitoring and flow logs, least-privilege IAM, and a high-availability network design. **Replace this example with your real experiences and any problems you actually encounter.**

## References
- Ansible amazon.aws collection: https://docs.ansible.com/projects/ansible/latest/collections/amazon/aws/
- AWS temporary credentials: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html
