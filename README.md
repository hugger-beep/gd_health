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
### Note SECURITY_ACCOUNT_ID="333333333333" # Replace with actual ID

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

 ## Or Save as Script (Reusable) and run from Security Account
 ### Note SECURITY_ACCOUNT_ID="333333333333" # Replace with actual ID
 
``` text
# Create the test script
cat > test-guardduty-role.sh << 'EOF'
#!/bin/bash

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
EOF

# Make it executable and run
chmod +x test-guardduty-role.sh
./test-guardduty-role.sh
```
## Cloudshell on Security Account 

``` text

# Create health check script
cat > guardduty-health-check.sh << 'EOF'
#!/bin/bash

echo "=== GuardDuty Health Check Report ==="
echo "Generated: $(date)"
echo ""

DETECTOR_ID=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)

echo "=== Detector Status ==="
aws guardduty get-detector --detector-id $DETECTOR_ID --query '{Status:Status,PublishingFrequency:FindingPublishingFrequency,UpdatedAt:UpdatedAt}' --output table

echo "=== Member Account Status ==="
aws guardduty list-members --detector-id $DETECTOR_ID --query 'Members[?RelationshipStatus!=`Enabled`].{AccountId:AccountId,Status:RelationshipStatus,InvitedAt:InvitedAt}' --output table

echo "=== Data Sources Status ==="
aws guardduty get-detector --detector-id $DETECTOR_ID --query 'DataSources.{CloudTrail:CloudTrail.Status,DNSLogs:DnsLogs.Status,FlowLogs:FlowLogs.Status,S3Logs:S3Logs.Status}' --output table

echo "=== Recent Findings Count ==="
aws guardduty list-findings --detector-id $DETECTOR_ID --finding-criteria '{
  "Criterion": {
    "updatedAt": {
      "Gte": '$(date -d '24 hours ago' +%s)000'
    }
  }
}' --query 'length(FindingIds)' --output text

# Cross-Account Member Validation
echo "=== Member Account Configuration Validation ==="
echo "Note: Requires cross-account roles in member accounts"
echo ""

# Get list of enabled member accounts
MEMBER_ACCOUNTS=$(aws guardduty list-members --detector-id $DETECTOR_ID --query 'Members[?RelationshipStatus==`Enabled`].AccountId' --output text)

if [ ! -z "$MEMBER_ACCOUNTS" ]; then
    echo "Checking member account configurations..."
    echo ""
    
    for ACCOUNT_ID in $MEMBER_ACCOUNTS; do
        echo "=== Member Account: $ACCOUNT_ID ==="
        
        # Try to assume cross-account role (if configured)
        ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/GuardDutyHealthCheckRole"
        
        # Check if we can assume the role
        TEMP_CREDS=$(aws sts assume-role --role-arn "$ROLE_ARN" --role-session-name "GuardDutyHealthCheck" 2>/dev/null)
        
        if [ $? -eq 0 ]; then
            # Extract credentials
            export AWS_ACCESS_KEY_ID=$(echo $TEMP_CREDS | jq -r '.Credentials.AccessKeyId')
            export AWS_SECRET_ACCESS_KEY=$(echo $TEMP_CREDS | jq -r '.Credentials.SecretAccessKey')
            export AWS_SESSION_TOKEN=$(echo $TEMP_CREDS | jq -r '.Credentials.SessionToken')
            
            # Check member account GuardDuty status
            MEMBER_DETECTOR=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text 2>/dev/null)
            
            if [ "$MEMBER_DETECTOR" != "None" ] && [ ! -z "$MEMBER_DETECTOR" ]; then
                echo "  ✅ GuardDuty Enabled: $MEMBER_DETECTOR"
                
                # Check underlying services
                CLOUDTRAIL_COUNT=$(aws cloudtrail describe-trails --query 'length(trailList[?IsLogging==`true`])' --output text 2>/dev/null)
                FLOW_LOGS_COUNT=$(aws ec2 describe-flow-logs --query 'length(FlowLogs[?FlowLogStatus==`ACTIVE`])' --output text 2>/dev/null)
                
                echo "  Active CloudTrail: $CLOUDTRAIL_COUNT"
                echo "  Active Flow Logs: $FLOW_LOGS_COUNT"
                
                if [ "$CLOUDTRAIL_COUNT" -eq 0 ]; then
                    echo "  ❌ WARNING: No active CloudTrail - API monitoring disabled"
                fi
                
                if [ "$FLOW_LOGS_COUNT" -eq 0 ]; then
                    echo "  ❌ WARNING: No active VPC Flow Logs - Network monitoring disabled"
                fi
            else
                echo "  ❌ GuardDuty Not Enabled in member account"
            fi
            
            # Clear temporary credentials
            unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
        else
            echo "  ⚠️  Cannot assume cross-account role - member account validation skipped"
            echo "  💡 To enable: Create role 'GuardDutyHealthCheckRole' in member account"
        fi
        
        echo ""
    done
else
    echo "No enabled member accounts found"
fi

echo "=== Health Check Complete ==="
echo ""
echo "💡 To enable member account validation:"
echo "   1. Create 'GuardDutyHealthCheckRole' in each member account"
echo "   2. Grant AssumeRole permission to Security Account"
echo "   3. Attach GuardDuty and basic read permissions"
EOF
```

![GuardDuty Health Check Results]
(result.PNG)
