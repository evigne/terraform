The error message "Malformedpolicydocument: Federated principles must be valid domain names or SAML metadata ARNs" indicates an issue with the format or content of the trust policy document you're using to create the IAM role. Specifically, it suggests that the `Principal` section of your trust policy document contains an invalid value.

Here's how you can resolve this:

1. **Check the Trust Policy**: Ensure that the `Principal` in your trust policy document is a valid domain name or SAML metadata ARN. For IRSA, it should be an OIDC provider URL.

2. **OIDC Provider**: If you're using IRSA, make sure you've correctly specified the OIDC provider URL for your EKS cluster.

Here's an example of a correct trust policy for IRSA:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::your-account-id:oidc-provider/oidc.eks.region.amazonaws.com/id/your-cluster-id"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.region.amazonaws.com/id/your-cluster-id:sub": "system:serviceaccount:namespace:service-account-name"
        }
      }
    }
  ]
}
```

### Steps to Create the IAM Role with a Valid Trust Policy

1. **Retrieve OIDC Provider URL**:
   - You can get the OIDC provider URL for your EKS cluster using the AWS CLI:
     ```sh
     aws eks describe-cluster --name your-cluster-name --query "cluster.identity.oidc.issuer" --output text
     ```

2. **Create the Trust Policy**:
   - Replace `your-account-id`, `region`, `your-cluster-id`, `namespace`, and `service-account-name` with the appropriate values.

3. **Create the IAM Role**:
   - Use the AWS CLI or the AWS Management Console to create the IAM role with the correct trust policy.

Using the AWS CLI, the command to create the role would look like this:

```sh
aws iam create-role --role-name your-role-name --assume-role-policy-document file://trust-policy.json
```

### Example Trust Policy JSON (trust-policy.json)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub": "system:serviceaccount:default:my-service-account"
        }
      }
    }
  ]
}
```

Ensure all fields are correctly replaced with your actual values.

If you provide more specific details about your trust policy or the exact configuration you're using, I can give more targeted advice.