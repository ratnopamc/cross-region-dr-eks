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
    
aws iam create-policy \
    --policy-name VeleroAccessPolicyRestore \
    --policy-document file://velero_policy_restore.json    
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
    --attach-policy-arn=arn:aws:iam::$ACCOUNT:policy/VeleroAccessPolicyRestore \
    --approve \
    --override-existing-serviceaccounts
```


// Install velero through helm chart in primary and recovery clusters

// if velero namespace exists, delete it

helm install velero vmware-tanzu/velero --create-namespace --namespace velero -f config/velero_primary_values.yaml --version 5.2.2


helm install velero vmware-tanzu/velero --create-namespace --namespace velero -f config/velero_recovery_values.yaml --version 5.2.2 



Deploy sample workload on source/primary cluster

```
kubectl apply -f config/sample_workload.yaml
```

It should create a namespace named `nginx` and create Deployment.

Create Backup using velero cli

```
velero create backup eks-bckup-2 --snapshot-move-data --include-namespaces default
```

Check the status of the backup.

Delete the `nginx` namespace. This is to simulate failure on the source/primary cluster

Change context to Recovery cluster.

Run the restore operation

```
velero create restore eks-restore-2 --from-backup eks-bckup-2 --namespace-mappings default:restore
```


## Using Resource Modifiers

Velero provides a generic ability to modify the resources during restore by specifying json patches. The json patches are applied to the resources before they are restored. The json patches are specified in a configmap and the configmap is referenced in the restore command.

In this example, we're going to use Velero Restore Resource Modifiers to update an external service endpoint before restoring from a backup.

### Deploy a sample workload on source cluster

First, let's deploy the example workload in the source EKS cluster using `kubectl apply` command.

```bash
kubectl apply -f resource_modifier/nginx-deployment.yaml
```

Check the status of the resources created in the namespace `demo`.

```bash
$ kubectl -n demo get all
NAME                                   READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-78cff7984-hf4h8   1/1     Running   0          23s
pod/nginx-deployment-78cff7984-n6bmf   1/1     Running   0          23s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   2/2     2            2           23s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-78cff7984   2         2         2       23s
```

Notice, that there's an env variable named `DB_ENDPOINT` that points to an Aurora DB cluster endpoint `mycluster.cluster-1.<source_region>.rds.amazonaws.com:3306`. When restoring the EKS cluster and its workload to a different AWS region, you can use the Velero Restore Resource Modifiers to patch the value of the endpoint at the time of restoring from the backup.

### BackUp the source cluster 

Using the velero cli, take the backup of the source EKS cluster.

```bash
velero create backup eks-backup-demo  --include-namespaces demo
```

Check the status of the backup

```bash
velero get backup

NAME              STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
eks-backup-demo    Completed   0        0          2024-03-04 21:09:01 +0000 UTC   29d       default            <none>
```

### Create Restore Resource Modifer

Now, in your destination or recovery EKS cluster on the other AWS region, create a `ConfigMap` in `velero` namespace to patch the value of the `DB_ENDPOINT` env variable to point to the Aurora cluster on that region.

```bash

kubectl create cm nginx-deploy-cm --from-file resource_modifier/resource_modifer.yaml -n velero
```

Then, restore the cluster from the backup `eks-backup-demo` taken avove.

```bash
velero create restore eks-restore-demo --resource-modifier-configmap nginx-deploy-cm --from-backup eks-backup-demo
```

Check the `velero restore` status

```bash
velero get restore
```

Verify that the resources on the `demo` namespace are in `Running` state.

```bash
kubectl -n demo get all
```

Now, check the value of the DB_ENDPOINT env variable by execing into a Pod.

```bash
kubectl -n demo exec nginx-deployment-849d985c48-xyzhh -- env | grep -i DB_ENDPOINT

DB_ENDPOINT=mycluster.cluster-2.us-east-2.rds.amazonaws.com:3306
```

This way, you can patch any configuration parameter or specific values that your workload uses while restoring from the backup.