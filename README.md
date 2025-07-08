# Run this in each MEMBER account

# Step 1: Set your Security Account ID
SECURITY_ACCOUNT_ID="333333333333"  # Replace with actual ID

# Step 2: Create trust policy file manually

``` text
cat > trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::333333333333:root"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

# Step 3: Create role
aws iam create-role --role-name GuardDutyHealthCheckRole --assume-role-policy-document file://trust-policy.json

# Step 4: Attach policies one by one
aws iam attach-role-policy --role-name GuardDutyHealthCheckRole --policy-arn arn:aws:iam::aws:policy/AmazonGuardDutyReadOnlyAccess

aws iam attach-role-policy --role-name GuardDutyHealthCheckRole --policy-arn arn:aws:iam::aws:policy/AWSCloudTrail_ReadOnlyAccess

aws iam attach-role-policy --role-name GuardDutyHealthCheckRole --policy-arn arn:aws:iam::aws:policy/AmazonVPCReadOnlyAccess

echo "GuardDutyHealthCheckRole created successfully"

```
