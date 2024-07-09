Based on the GitHub issue and comments you've found, the problem with nodes terminating immediately is likely due to insufficient permissions for the `kms:GenerateDataKeyWithoutPlaintext` action on the KMS key used for EBS encryption. The solution involves ensuring that the IAM role associated with Karpenter's IRSA (IAM Role for Service Accounts) has the correct permissions to use the KMS key.

### Solution: Update IAM Policy for Karpenter's IRSA

1. **Add the necessary permissions to Karpenter's IRSA role**:
   
   Update the IAM policy attached to the Karpenter IRSA role to include permissions for `kms:GenerateDataKeyWithoutPlaintext`, `kms:Decrypt`, and `kms:CreateGrant` actions. This ensures that the Karpenter controller can manage encrypted EBS volumes properly.

   Here’s the IAM policy to add:

   ```hcl
   resource "aws_iam_role_policy" "karpenter_irsa_policy" {
     role = aws_iam_role.karpenter_controller.id
     policy = jsonencode({
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": [
             "kms:GenerateDataKeyWithoutPlaintext",
             "kms:Decrypt",
             "kms:CreateGrant"
           ],
           "Resource": [
             "arn:aws:kms:${var.aws_region}:${data.aws_caller_identity.current.account_id}:key/*"
           ]
         }
       ]
     })
   }
   ```

   Replace `${var.aws_region}` and `${data.aws_caller_identity.current.account_id}` with your specific AWS region and account ID.

2. **Ensure the KMS key policy allows access via EC2 service**:

   Update your KMS key policy as described earlier to allow access through EBS for all principals in the account authorized to use EBS:

   ```json
   {
       "Version": "2012-10-17",
       "Id": "key-default-1",
       "Statement": [
           {
               "Sid": "Allow access through EBS for all principals in the account that are authorized to use EBS",
               "Effect": "Allow",
               "Principal": {
                   "AWS": "*"
               },
               "Action": [
                   "kms:Encrypt",
                   "kms:Decrypt",
                   "kms:ReEncrypt*",
                   "kms:GenerateDataKey*",
                   "kms:CreateGrant",
                   "kms:DescribeKey"
               ],
               "Resource": "*",
               "Condition": {
                   "StringEquals": {
                       "kms:ViaService": "ec2.${AWS_REGION}.amazonaws.com",
                       "kms:CallerAccount": "${AWS_ACCOUNT_ID}"
                   }
               }
           },
           {
               "Sid": "Allow direct access to key metadata to the account",
               "Effect": "Allow",
               "Principal": {
                   "AWS": "arn:aws:iam::${AWS_ACCOUNT_ID}:root"
               },
               "Action": [
                   "kms:Describe*",
                   "kms:Get*",
                   "kms:List*",
                   "kms:RevokeGrant"
               ],
               "Resource": "*"
           }
       ]
   }
   ```

   Replace `${AWS_REGION}` and `${AWS_ACCOUNT_ID}` with your actual AWS region and account ID.

### Example Terraform Configuration

1. **IAM Role and Policy for Karpenter's IRSA**:

   ```hcl
   resource "aws_iam_role" "karpenter_controller" {
     name = "KarpenterControllerRole"
     assume_role_policy = jsonencode({
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Principal": {
             "Service": "ec2.amazonaws.com"
           },
           "Action": "sts:AssumeRole"
         }
       ]
     })
   }

   resource "aws_iam_role_policy" "karpenter_irsa_policy" {
     role = aws_iam_role.karpenter_controller.id
     policy = jsonencode({
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": [
             "kms:GenerateDataKeyWithoutPlaintext",
             "kms:Decrypt",
             "kms:CreateGrant",
             "ec2:DescribeInstances",
             "ec2:DescribeLaunchTemplateVersions",
             "ec2:RunInstances",
             "ec2:TerminateInstances",
             "ec2:CreateFleet",
             "ec2:CreateLaunchTemplateVersion",
             "ec2:DeleteLaunchTemplate",
             "ec2:DeleteLaunchTemplateVersions",
             "ec2:DescribeSubnets",
             "ec2:DescribeSecurityGroups",
             "ec2:DescribeImages",
             "ec2:DescribeInstanceTypes",
             "ec2:DescribeVolumes",
             "ec2:CreateVolume",
             "ec2:DeleteVolume",
             "ec2:AttachVolume",
             "ec2:DetachVolume",
             "ec2:ModifyVolume",
             "sqs:ReceiveMessage",
             "sqs:DeleteMessage",
             "sqs:GetQueueAttributes"
           ],
           "Resource": "*"
         }
       ]
     })
   }
   ```

2. **KMS Key Policy**:

   Update the KMS key policy in the AWS console or through Terraform as shown in the earlier example.

### Apply Configuration and Validate

1. **Apply the Terraform Configuration**:

   ```bash
   terraform apply
   ```

2. **Deploy Node Template Configuration**:

   ```yaml
   apiVersion: karpenter.sh/v1alpha5
   kind: EC2NodeClass
   metadata:
     name: default
   spec:
     amiFamily: AL2
     blockDeviceMappings:
       - deviceName: /dev/xvda
         ebs:
           volumeSize: 20Gi
           volumeType: gp3
           encrypted: true
           kmsKeyId: ${KMS_KEY_ID}
     subnetSelector:
       karpenter.sh/discovery: ${CLUSTER_NAME}
     securityGroupSelector:
       karpenter.sh/discovery: ${CLUSTER_NAME}
     instanceProfile: "KarpenterInstanceProfile"
   ```

   ```bash
   kubectl apply -f node-template.yaml
   ```

3. **Monitor Logs and Events**:

   - Check Karpenter controller logs for issues:

     ```bash
     kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter
     ```

   - Check Kubernetes events for related errors:

     ```bash
     kubectl get events -A
     ```

### Conclusion

By ensuring the necessary permissions are set on the Karpenter IRSA role and the KMS key, you can resolve the issue of nodes terminating immediately due to encrypted EBS volumes. This approach should address the permission issues as identified in the GitHub discussion.