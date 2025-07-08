# Run this in each MEMBER account cloudshell

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

# Verify Commands

## Verify the role was created
aws iam get-role --role-name GuardDutyHealthCheckRole --query 'Role.{RoleName:RoleName,CreateDate:CreateDate}' --output table

## Verify attached policies
aws iam list-attached-role-policies --role-name GuardDutyHealthCheckRole --output table

## Test the role (from Security Account)
aws sts assume-role --role-arn "arn:aws:iam::333333333333:role/GuardDutyHealthCheckRole" --role-session-name "TestSession"


## Test Result
``` text

{
    "Credentials": {
        "AccessKeyId": "ASIAXXXXXXXXXXXXXXXXXXX",
        "SecretAccessKey": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "SessionToken": "IQoJb3JpZ2luX2VjEHcaCXVzLWVhc3QtMSJHMEUCIQD...",
        "Expiration": "2025-07-15T11:30:00+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAXXXXXXXXXXXXXXXXX:TestSession",
        "Arn": "arn:aws:sts::333333333333:assumed-role/GuardDutyHealthCheckRole/TestSession"
    }
}
```
## Or Full Test  (from Security Account), copy and paste into cloudshell

``` text
# Replace with actual member account ID
MEMBER_ACCOUNT_ID="333333333333"

# Test 1: Basic assume role test
echo "=== Testing AssumeRole ==="
aws sts assume-role \
    --role-arn "arn:aws:iam::${MEMBER_ACCOUNT_ID}:role/GuardDutyHealthCheckRole" \
    --role-session-name "TestSession" \
    --query 'AssumedRoleUser.Arn' \
    --output text

# Test 2: Full functional test
echo "=== Testing Full Functionality ==="
TEMP_CREDS=$(aws sts assume-role \
    --role-arn "arn:aws:iam::${MEMBER_ACCOUNT_ID}:role/GuardDutyHealthCheckRole" \
    --role-session-name "TestSession" 2>/dev/null)

if [ $? -eq 0 ]; then
    echo "✅ Role assumption successful"
    
    # Extract and use credentials
    export AWS_ACCESS_KEY_ID=$(echo $TEMP_CREDS | jq -r '.Credentials.AccessKeyId')
    export AWS_SECRET_ACCESS_KEY=$(echo $TEMP_CREDS | jq -r '.Credentials.SecretAccessKey')
    export AWS_SESSION_TOKEN=$(echo $TEMP_CREDS | jq -r '.Credentials.SessionToken')
    
    echo "Testing permissions in member account..."
    
    # Test GuardDuty access
    DETECTOR_TEST=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text 2>/dev/null)
    if [ "$DETECTOR_TEST" != "None" ] && [ ! -z "$DETECTOR_TEST" ]; then
        echo "✅ GuardDuty access: SUCCESS - Detector: $DETECTOR_TEST"
    else
        echo "⚠️  GuardDuty access: No detector found"
    fi
    
    # Test CloudTrail access
    TRAIL_COUNT=$(aws cloudtrail describe-trails --query 'length(trailList)' --output text 2>/dev/null)
    if [ $? -eq 0 ]; then
        echo "✅ CloudTrail access: SUCCESS - Found $TRAIL_COUNT trail(s)"
    else
        echo "❌ CloudTrail access: FAILED"
    fi
    
    # Test VPC access
    FLOW_LOG_COUNT=$(aws ec2 describe-flow-logs --query 'length(FlowLogs)' --output text 2>/dev/null)
    if [ $? -eq 0 ]; then
        echo "✅ VPC access: SUCCESS - Found $FLOW_LOG_COUNT flow log(s)"
    else
        echo "❌ VPC access: FAILED"
    fi
    
    # Clear credentials
    unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
    echo "✅ Test completed successfully"
else
    echo "❌ Role assumption failed"
fi
```

OR
``` text
# Create the test script
cat > test-guardduty-role.sh << 'EOF'
#!/bin/bash

# Replace with actual member account ID
MEMBER_ACCOUNT_ID="444444444444"

# Test 1: Basic assume role test
echo "=== Testing AssumeRole ==="
aws sts assume-role \
    --role-arn "arn:aws:iam::${MEMBER_ACCOUNT_ID}:role/GuardDutyHealthCheckRole" \
    --role-session-name "TestSession" \
    --query 'AssumedRoleUser.Arn' \
    --output text

# Test 2: Full functional test
echo "=== Testing Full Functionality ==="
TEMP_CREDS=$(aws sts assume-role \
    --role-arn "arn:aws:iam::${MEMBER_ACCOUNT_ID}:role/GuardDutyHealthCheckRole" \
    --role-session-name "TestSession" 2>/dev/null)

if [ $? -eq 0 ]; then
    echo "✅ Role assumption successful"
    
    # Extract and use credentials
    export AWS_ACCESS_KEY_ID=$(echo $TEMP_CREDS | jq -r '.Credentials.AccessKeyId')
    export AWS_SECRET_ACCESS_KEY=$(echo $TEMP_CREDS | jq -r '.Credentials.SecretAccessKey')
    export AWS_SESSION_TOKEN=$(echo $TEMP_CREDS | jq -r '.Credentials.SessionToken')
    
    echo "Testing permissions in member account..."
    
    # Test GuardDuty access
    DETECTOR_TEST=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text 2>/dev/null)
    if [ "$DETECTOR_TEST" != "None" ] && [ ! -z "$DETECTOR_TEST" ]; then
        echo "✅ GuardDuty access: SUCCESS - Detector: $DETECTOR_TEST"
    else
        echo "⚠️  GuardDuty access: No detector found"
    fi
    
    # Test CloudTrail access
    TRAIL_COUNT=$(aws cloudtrail describe-trails --query 'length(trailList)' --output text 2>/dev/null)
    if [ $? -eq 0 ]; then
        echo "✅ CloudTrail access: SUCCESS - Found $TRAIL_COUNT trail(s)"
    else
        echo "❌ CloudTrail access: FAILED"
    fi
    
    # Test VPC access
    FLOW_LOG_COUNT=$(aws ec2 describe-flow-logs --query 'length(FlowLogs)' --output text 2>/dev/null)
    if [ $? -eq 0 ]; then
        echo "✅ VPC access: SUCCESS - Found $FLOW_LOG_COUNT flow log(s)"
    else
        echo "❌ VPC access: FAILED"
    fi
    
    # Clear credentials
    unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
    echo "✅ Test completed successfully"
else
    echo "❌ Role assumption failed"
fi
EOF

# Make it executable and run
chmod +x test-guardduty-role.sh
./test-guardduty-role.sh
```
