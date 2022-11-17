# ebs-volume-mount-in-amazon-eks
Troubleshooting issues with my ebs volume mount in amazon eks https://aws.amazon.com/premiunsupport/knowledge-center/eks-troubleshoot-ebs-volume-mounts/

## Step 1
1. Verify that you have the required AWS Identity and Access Management (IAM) permission for your "ebs-csi-controller-sa" service account IAM role.

2. The ebs-csi-controller has a service account and the service account should have the required IAM roles attached to it.

3. Therefore you must have Amazon EBS CSI driver installed in your cluster. The Amazon EBS Container Storage Interface (CSI) driver allows Amazon Elastic Kubernates Service (Amazon EKS) clusters to manage the lifecycle of Amazon EBS volumes for persistence volumes.

4. To use the driver, you must add it as an Amazon EKS add-on or as a self-managed add-on.

5. To add it as an Amazon EKS add-on, you must have:
    a. a cluster 
    b. an existing AWS Identity and Access Management (IAM) OpenID Connect (OIDC) provider for your cluster.

6. To determine whether you already have one or to create one. Your cluster has an OpenID  Connect (OIDC) issuer URL associated with it. To use AWS IAM roles for service accounts, an IAM OIDC provider must exist for your cluster.

## Step 2
To create an IAM OIDC identity provider for your cluster with the AWS Management Console

1. Open the Amazon EKS console at https://console.aws.amazon.com/eks/home#/clusters.

2. In the left pane, select **Clusters**, and then select the name of your cluster on the **Clusters** page.

3. In the **Details** section on the **Overview** tab, note the value of the **OpenID Connect provider URL**.

4. Open the IAM console at https://console.aws.amazon.com/iam/.

5. In the left navigation pane, choose **Identity Providers** under **Access management**. If a Provider is listed that matches the URL for your cluster, then you already have a provider for your cluster. If a provider isn't listed that matches the URL for your cluster, then you must create one.

6. To create a provider, choose **Add provider**.

7. For **Provider type**, select **OpenID Connect**.

8. For **Provider URL**, enter the OIDC provider URL for your cluster, and then choose **Get thumbprint**.

9. For **Audience**, enter 'sts.amazonaws.com' and choose **Add provider**.

Copy the Cluster's OIDC provider ID
## Step 3
### Creating the Amazon EBS CSI driver IAM role for service accounts.

The Amazon EBS CSI plugin requires IAM permissions to make calls to AWS APIs on your behalf. For more information, see Set up driver permission on GitHub https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/docs#set-up-driver-permission.

When the plugin is deployed, it creates and is configured to use a service account that's named ebs-csi-controller-sa. The service account is bound to a Kubernetes clusterrole that's assigned the required Kubernetes permissions.

**Note**: No matter if you configure the Amazon EBS CSI plugin to use IAM roles for service accounts, the pods have access to the permissions that are assigned to the IAM role. This is the case except when you block access to IMDS. For more information, see Security best practices for Amazon EKS https://docs.aws.amazon.com/eks/latest/userguide/security-best-practices.html.

Prerequisites

    1. An existing cluster.

    2. An existing AWS Identity and Access Management (IAM) OpenID Connect (OIDC) provider for your cluster. To determine whether you already have one, or to create one, see Creating an IAM OIDC provider for your cluster https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html.

#### Create an IAM role and attach the required AWS managed policy to it. You can use eksctl, the AWS Management Console, or the AWS CLI.

 #### Using AWS Management Console

To create your Amazon EBS CSI plugin IAM role with the AWS Management Console

1. Open the IAM console at https://console.aws.amazon.com/iam/.

2. In the left navigation pane, choose **Roles**.

3. On the **Roles** page, choose **Create role**.

4. On the Select trusted entity page, do the following:

    a. In the **Trusted entity type** section, choose **Web identity**.

    b. For **Identity provider**, choose the **OpenID Connect provider URL** for your cluster (as shown under **Overview** in Amazon EKS).

    c. For **Audience**, choose `sts.amazonaws.com`.

    d. Choose **Next**.

5. On the **Add permissions** page, do the following:

    a. In the Filter policies box, enter `AmazonEBSCSIDriverPolicy`.

    b. Select the check box to the left of the `AmazonEBSCSIDriverPolicy` returned in the search.

    c. Choose **Next**.

6. On the **Name, review, and create page**, do the following:

    a. For **Role name**, enter a unique name for your role, such as **AmazonEKS_EBS_CSI_DriverRole**.

    b. Under **Add tags (Optional)**, add metadata to the role by attaching tags as key–value pairs. For more information about using tags in IAM, see Tagging IAM Entities in the IAM User Guide.

    c. Choose **Create role**.

7. After the role is created, choose the role in the console to open it for editing.

8. Choose the **Trust relationships** tab, and then choose **Edit trust policy**.

9. Find the line that looks similar to the following line:

    ```
    "oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
    ```

    Add a comma to the end of the previous line, and then add the following line after the previous line. Replace **region-code** with the AWS Region that your cluster is in. Replace **EXAMPLED539D4633E53DE1B71EXAMPLE** with your cluster's OIDC provider ID.

    ```
    "oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
    ```

10. Choose **Update policy** to finish.

11. If you use a custom KMS key for encryption on your Amazon EBS volumes, customize the IAM role as needed. For example, do the following:

    a. In the left navigation pane, choose Policies.

    b. On the Policies page, choose Create Policy.

    c. On the Create policy page, choose the JSON tab.

    d. Copy and paste the following code into the editor, replacing custom-key-id with the custom KMS key ID:

    ```
    {
    "Version": "2012-10-17",
    "Statement": [
        {
        "Effect": "Allow",
        "Action": [
            "kms:CreateGrant",
            "kms:ListGrants",
            "kms:RevokeGrant"
        ],
        "Resource": ["custom-key-id"],
        "Condition": {
            "Bool": {
            "kms:GrantIsForAWSResource": "true"
            }
        }
        },
        {
        "Effect": "Allow",
        "Action": [
            "kms:Encrypt",
            "kms:Decrypt",
            "kms:ReEncrypt*",
            "kms:GenerateDataKey*",
            "kms:DescribeKey"
        ],
        "Resource": ["custom-key-id"]
        }
    ]
    }
    ```

    e. Choose Next: Tags.

    f. On the Add tags (Optional) page, choose Next: Review.

    g. For Name, enter a unique name for your policy (for example, KMS_Key_For_Encryption_On_EBS_Policy).

    h. Choose Create policy.

    i. In the left navigation pane, choose Roles.

    j. Choose the AmazonEKS_EBS_CSI_DriverRole in the console to open it for editing.

    k. From the Add permissions drop-down list, choose Attach policies.

    l. In the Filter policies box, enter KMS_Key_For_Encryption_On_EBS_Policy.

    m. Select the check box to the left of the KMS_Key_For_Encryption_On_EBS_Policy that was returned in the search.

    n. Choose Attach policies.

12. Annotate the ebs-csi-controller-sa Kubernetes service account with the ARN of the IAM role. Replace 111122223333 with your account ID and AmazonEKS_EBS_CSI_DriverRole with the name of the IAM role.
```
kubectl annotate serviceaccount ebs-csi-controller-sa -n kube-system eks.amazonaws.com/role-arn=arn:aws:iam::111122223333:role/AmazonEKS_EBS_CSI_DriverRole
```

## Step 4
### Adding the Amazon EBS CSI add-on
**Important**: Before adding the Amazon EBS CSI add-on, confirm that you don't self-manage any settings that Amazon EKS will start managing. To determine which settings Amazon EKS manages, see Amazon EKS add-on configuration.

You can use eksctl, the AWS Management Console, or the AWS CLI to add the Amazon EBS CSI add-on to your cluster .

#### Using the AWS Management Console

To add the Amazon EBS CSI add-on using the AWS Management Console

1. Open the Amazon EKS console at https://console.aws.amazon.com/eks/home#/clusters.

2. In the left navigation pane, choose **Clusters**.

3. Choose the name of the cluster that you want to configure the Amazon EBS CSI add-on for.

4. Choose the **Add-ons** tab.

5. Choose **Add new**.

a. Select **Amazon EBS CSI Driver** for **Name**.

b. Select the **Version** you'd like to use.

c. For **Service account role**, select the name of an IAM role that you attached the IAM policy to.

d. If you select **Override existing configuration for this add-on on the cluster**., one or more of the settings for the existing add-on can be overwritten with the Amazon EKS add-on settings. If you don't enable this option and there's a conflict with your existing settings, the operation fails. You can use the resulting error message to troubleshoot the conflict. Before selecting this option, make sure that the Amazon EKS add-on doesn't manage settings that you need to self-manage. For more information about managing Amazon EKS add-ons, see Amazon EKS add-on configuration.

e. Choose **Add**.

#### Updating the Amazon EBS CSI driver as an Amazon EKS add-on
Amazon EKS doesn't automatically update Amazon EBS CSI for your cluster when new versions are released or after you update your cluster to a new Kubernetes minor version. To update Amazon EBS CSI on an existing cluster, you must initiate the update and then Amazon EKS updates the add-on for you.

**Important**: Update your cluster and nodes to a new Kubernetes minor version before you update Amazon EBS CSI to the same minor version.

#### AWS Management Console
To update the Amazon EBS CSI add-on using the AWS Management Console

1. Open the Amazon EKS console at https://console.aws.amazon.com/eks/home#/clusters.

2. In the left navigation pane, choose **Clusters**.

3. Choose the name of the cluster that you want to update the Amazon EBS CSI add-on for.

4. Choose the **Add-ons** tab.

5. Select the radio button in the upper right of the **aws-ebs-csi-driver** box.

6. Choose **Edit**.

    a. Select the **Version** of the Amazon EKS add-on that you want to use.

    b. For **Service account role**, select the name of the IAM role that you've attached the Amazon EBS CSI driver IAM policy to.

    c. For **Conflict resolution method**, select one of the options. For more information about Amazon EKS add-on configuration management, see Amazon EKS add-on configuration.

    d. Select Update.

#### Removing the Amazon EBS CSI add-on
You have two options for removing an Amazon EKS add-on.

- **Preserve add-on software on your cluster** – This option removes Amazon EKS management of any settings. It also removes the ability for Amazon EKS to notify you of updates and automatically update the Amazon EKS add-on after you initiate an update. However, it preserves the add-on software on your cluster. This option makes the add-on a self-managed add-on, rather than an Amazon EKS add-on. With this option, there's no downtime for the add-on. The commands in this procedure use this option.

- **Remove add-on software entirely from your cluster** – We recommend that you remove the Amazon EKS add-on from your cluster only if there are no resources on your cluster that are dependent on it. To do this option, delete `--preserve` from the command you use in this procedure.

If the add-on has an IAM account associated with it, the IAM account isn't removed.

You can use eksctl, the AWS Management Console, or the AWS CLI to remove the Amazon EBS CSI add-on.

#### AWS Management Console
To remove the Amazon EBS CSI add-on using the AWS Management Console

1. Open the Amazon EKS console at https://console.aws.amazon.com/eks/home#/clusters.

2. In the left navigation pane, choose **Clusters**.

3. Choose the name of the cluster that you want to remove the Amazon EBS CSI add-on for.

4. Choose the **Add-ons** tab.

5. Select the radio button in the upper right of the **aws-ebs-csi-driver** box.

6. Choose **Remove**.

7. Select **Preserve on cluster** if you want Amazon EKS to stop managing settings for the add-on. Do this if you want to retain the add-on software on your cluster. This is so that you can manage all of the settings of the add-on on your own.

8. Enter `aws-ebs-csi-driver`.

9. Select **Remove**.


## Step 5
The Amazon EBS CSI driver consists of controller pods that run as a deployment and node pods that run as a daemonset. Run the following commands to verify if these pods are running in your cluster:

```
kubectl get all -l app.kubernetes.io/name=aws-ebs-csi-driver -n kube-system
```
**Note**: The Amazon EBS CSI driver isn't supported on Windows worker nodes or EKS Fargate.

Make sure that the installed Amazon EBS CSI driver version is compatible with your cluster's Kubernetes version.