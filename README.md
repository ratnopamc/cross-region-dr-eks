         ___        ______     ____ _                 _  ___  
        / \ \      / / ___|   / ___| | ___  _   _  __| |/ _ \ 
       / _ \ \ /\ / /\___ \  | |   | |/ _ \| | | |/ _` | (_) |
      / ___ \ V  V /  ___) | | |___| | (_) | |_| | (_| |\__, |
     /_/   \_\_/\_/  |____/   \____|_|\___/ \__,_|\__,_|  /_/ 
 ----------------------------------------------------------------- 


Hi there! Welcome to AWS Cloud9!

To get started, create some files, play with the terminal,
or visit https://docs.aws.amazon.com/console/cloud9/ for our documentation.

Happy coding!
# cross-region-dr-eks


## Create Primary and Recovery Cluster


## Export envs

export CLUSTER_PRIMARY=primary
export CLUSTER_RECOVERY=recovery
export CTX_PRIMARY=
export CTX_RECOVERY=

## Install other components

// Set context to primary cluster

eksctl utils associate-iam-oidc-provider \
  --region=us-east-2 \
  --cluster=$CLUSTER_PRIMARY \
  --approve
  
 
# Create a service account
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --region=us-east-2 \
  --cluster $CLUSTER_PRIMARY \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole  
  
  
# add ebs csi driver add on
eksctl create addon --name aws-ebs-csi-driver --cluster $CLUSTER_PRIMARY --region us-east-2 \
    --service-account-role-arn arn:aws:iam::200202725330:role/AmazonEKS_EBS_CSI_DriverRole --force
  

eksctl get addon --name aws-ebs-csi-driver --cluster $CLUSTER_PRIMARY --region us-east-2  


// change context to Recovery cluster



eksctl utils associate-iam-oidc-provider \
  --region=us-east-1 \
  --cluster=$CLUSTER_RECOVERY \
  --approve
  
 
# Create a service account
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --region=us-east-1 \
  --cluster $CLUSTER_RECOVERY \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole2  
  
  
# add ebs csi driver add on
eksctl create addon --name aws-ebs-csi-driver --cluster $CLUSTER_RECOVERY --region us-east-1 \
    --service-account-role-arn arn:aws:iam::200202725330:role/AmazonEKS_EBS_CSI_DriverRole2 --force
    
# check addon status
eksctl get addon --name aws-ebs-csi-driver --cluster $CLUSTER_RECOVERY --region us-east-1  


// create s3 buckets
BUCKET=aratnch-eks-velero-dr
REGION=us-east-2
aws s3 mb s3://$BUCKET --region $REGION


```bash
cd config
cat > velero_policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}"
            ]
        }
    ]
}
EOF

aws iam create-policy \
    --policy-name VeleroAccessPolicy \
    --policy-document file://velero_policy.json
```

// create service accounts for both clusters

```
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

eksctl create iamserviceaccount \
    --cluster=$CLUSTER_PRIMARY \
    --region=us-east-2 \
    --name=velero-server \
    --namespace=velero \
    --role-name=eks-velero-primary-role \
    --attach-policy-arn=arn:aws:iam::$ACCOUNT:policy/VeleroAccessPolicy \
    --approve \
    --override-existing-serviceaccounts


eksctl create iamserviceaccount \
    --cluster=$CLUSTER_RECOVERY \
    --region=us-east-1 \
    --name=velero-server \
    --namespace=velero \
    --role-name=eks-velero-recovery-role \
    --attach-policy-arn=arn:aws:iam::$ACCOUNT:policy/VeleroAccessPolicy \
    --approve \
    --override-existing-serviceaccounts
```


// Install velero through helm chart in primary and recovery clusters

helm install velero vmware-tanzu/velero  --namespace velero -f velero_primary_values.yaml --version 5.2.2 --set serviceAccount.server.create=false

helm install velero vmware-tanzu/velero  --namespace velero     -f velero_primary_values.yaml --version 5.2.2 --set serviceAccount.server.create=false

