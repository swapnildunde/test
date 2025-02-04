#test
#!/bin/bash

# List all IAM roles
roles=$(aws iam list-roles --query "Roles[*].RoleName" --output text)

for role in $roles; do
  echo "Checking role: $role"

  # Check inline policies
  inline_policies=$(aws iam list-role-policies --role-name "$role" --query "PolicyNames" --output text)
  for policy in $inline_policies; do
    policy_doc=$(aws iam get-role-policy --role-name "$role" --policy-name "$policy")
    if echo "$policy_doc" | jq '.PolicyDocument.Statement[] | select(.Effect=="Allow") | .Action' | grep -q "ec2:CopyImage\|ec2:\*"; then
      echo "Found in INLINE policy: $policy for role: $role"
    fi
  done

  # Check attached managed policies
  attached_policies=$(aws iam list-attached-role-policies --role-name "$role" --query "AttachedPolicies[*].PolicyArn" --output text)
  for policy_arn in $attached_policies; do
    policy_version=$(aws iam get-policy --policy-arn "$policy_arn" --query "Policy.DefaultVersionId" --output text)
    policy_doc=$(aws iam get-policy-version --policy-arn "$policy_arn" --version-id "$policy_version")
    if echo "$policy_doc" | jq '.PolicyDocument.Statement[] | select(.Effect=="Allow") | .Action' | grep -q "ec2:CopyImage\|ec2:\*"; then
      echo "Found in MANAGED policy: $policy_arn for role: $role"
    fi
  done
done
