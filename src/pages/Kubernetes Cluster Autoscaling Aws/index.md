---
title: Kubernetes Cluster Autoscaling in AWS
lang: en
date: 2019-03-16
tags: autoscaling, AWS
---
# Kubernetes Cluster Autoscaling in AWS

## Building an AMI

To allow Autoscaling to work properly the Kubernetes and Kubeadm binaries need to be available. The easiest way to accomplish this is to build a new AMI that has the binaries pre installed. Heptio, prior to being acquired by VMWare, has created a tool as part of the Wardroom project that will build the AMIs for this purpose:

To build the new AMI:

* Configure aws cli
  * `aws configure`
* Navigate to the Wardroom codebase
* Build a new AMI:
  * `/path/to/packer build -var-file us-west-2.json -var kubernetes_version="<KUBERNETES_VERSION>" -var kubernetes_cni_version="0.6.0-00" -var build_version=\`git rev-parse HEAD\` packer.json`
    * Replace <KUBERNETES_VERSION> with an actual K8S binary version such as `1.11.3-00`
    * Replace us-west-2.json with a file that fits the region you're deploying to.

This should result in a new AMI being created in the appropriate region. Record the AMI ID for later.

## Create a new Launch Configuration in AWS

A new AWS Launch Configuration will need to be created, as Launch Configurations cannot be edited.

Create a new Launch Configuration by:

* Create a new Launch Configuration using the AWS console.
* Select the `Edit Details` link.
* Modify the Launch Configuration name to something appropriate.
  * For example: `K8S-Node Austoscaler`
* Modify the `User Data` script in the `Advanced Details` section:
  * NOTE: Replace `<K8S APISERVER FQDN>` below with the FQDN used to access your Kubernetes cluster's API server.

```
#!/bin/bash
AZ=$(ec2metadata | grep availability-zone | grep -v "ec2metadata" | awk '{print $2}' | awk '1' RS='.\n')
LOCALHOST=$(ec2metadata | grep local-hostname | grep -v "ec2metadata" | awk -S '{print $2}' | awk -F\. '{print $1}')
hostnamectl set-hostname $LOCALHOST.$AZ.compute.internal

JOINTOKEN=`aws ssm get-parameter --name jointoken --region $AZ | grep Value | awk -S '{print $2}' | awk -F\, '{print $1}' | awk -F\" '{print $2}'`

# disable apt auto update
systemctl stop apt-daily.service
systemctl kill --kill-who=all apt-daily.service

while ! (systemctl list-units --all apt-daily.service | fgrep -q dead)
do
  sleep 1;
done

cat <<EOF | sudo tee /etc/default/kubelet
KUBELET_EXTRA_ARGS=--cloud-provider=aws
EOF

kubeadm join --token $JOINTOKEN <K8S APISERVER FQDN>:443 --node-name $LOCALHOST.$AZ.compute.internal --discovery-token-unsafe-skip-ca-verification
```

* Press the `Skip to Review` button and then `Close` button.
* Select the `Auto Scaling Group` in the EC2 services list.
* Select the Kubernetes Nodes ASG, press the `Actions` button and select `Edit`, or if you haven't created an ASG yet create one now.
* Select the Launch Configuration created above in the `Launch Configuration` drop down menu and select the `Save` button.
* While in the Auto Scaling Group details for the Kubernetes nodes select the `Tags` tab.
* Press the `Add/Edit Tags`.
* Add the following tags:
  * Name: `k8s.io/cluster-autoscaler/enabled` Value: `true` (value is arbitrary)
  * Name: `k8s.io/cluster-autoscaler/<CLUSTER NAME>` Value: `true` (value is arbitrary)
* Press the `Save` button.

## Configure IAM policies

An AWS IAM Policy will need to be created to allow Autoscaling to work properly. This policy will need to be attached to the same IAM Role that the Autoscaling group belongs to.

To set up the new IAM Policy:

* In AWS EC2 Dashboard select one of the Kubernetes node instances and select the `IAM Role` link.
* In the IAM Role summary page select the `Attach policies` button, and select the `create policy` button in the subsequent screen.
* Select the `JSON` tab and enter:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ssm:DescribeParameters",
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:GetParametersByPath",
                "ssm:DeleteParameter",
                "ssm:DeleteParameters",
                "ssm:PutParameter",
                "ssm:UpdateInstanceInformation"
            ],
            "Resource": "*"
        }
    ]
}
```

* Press the `Review` button.
* Enter an appropriate name for the new policy, such as `cluster-autoscaler` and save the new policy.
* Navigate back to the Kubernetes node `Role` and press the `Attach policy` button.
* Use the search bar to find the policy created above, select the new policy, and press the `Attach policy` button.

## Create a Kubeadm Token Cron Job

In order to join new nodes to the cluster a token has to be available for the nodes. To reduce the possibility of an attacker using a Kubeadm join token we will generate tokens that last only an hour. The following script will run every hour on one of the Control Plane nodes to update the AWS System Management Agent(SSM) with a new token.

To use the script:

* SSH into one of the Control Plane nodes
* Create a file called `/etc/cron.hourly/kubeadm-token`
* Add the following to the `kubeadm-token` file above:
  * NOTE: Replace the value for region to the region of your choice

```
#!/bin/bash
export JOINTOKEN=`kubeadm token create --ttl=1h`
aws ssm put-parameter --name jointoken --value $JOINTOKEN --type String --region us-west-2 --overwrite
```

* Make the `kubeadm-token` file executable
  * `chmod 755 /etc/cron.hourly/kubeadm-token`

## Deploying the Cluster Autoscaler

The cluster autoscaler works by reading the number of pods that are in an `Unscheduled` state and validates whether a new Kubernetes Node needs to be added. The cluster autoscaler can also determine if a ndoe is not being used at all and will scale down the cluster (to the limit of the AWS Auto Scaling Group) to reduce unused resources.

To deploy the Cluster Autoscaler:

* Create a file named `~/autoscaler-deployment.yaml`
* Add the following to `autoscaler-deployment.yaml`:

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
- apiGroups: [""]
  resources: ["events","endpoints"]
  verbs: ["create", "patch"]
- apiGroups: [""]
  resources: ["pods/eviction"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["pods/status"]
  verbs: ["update"]
- apiGroups: [""]
  resources: ["endpoints"]
  resourceNames: ["cluster-autoscaler"]
  verbs: ["get","update"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["watch","list","get","update"]
- apiGroups: [""]
  resources: ["pods","services","replicationcontrollers","persistentvolumeclaims","persistentvolumes"]
  verbs: ["watch","list","get"]
- apiGroups: ["extensions"]
  resources: ["replicasets","daemonsets"]
  verbs: ["watch","list","get"]
- apiGroups: ["policy"]
  resources: ["poddisruptionbudgets"]
  verbs: ["watch","list"]
- apiGroups: ["apps"]
  resources: ["statefulsets", "replicasets"]
  verbs: ["watch","list","get"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["watch","list","get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["cluster-autoscaler-status"]
  verbs: ["delete","get","update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      tolerations:
      - effect: NoSchedule
        operator: "Equal"
        value: ""
        key: node-role.kubernetes.io/master
      nodeSelector:
        node-role.kubernetes.io/master: ""
      containers:
        - image: gcr.io/google-containers/cluster-autoscaler:v1.3.5
          name: cluster-autoscaler
          env:
          - name: AWS_REGION
            value: ap-southeast-1
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<CLUSTER NAME>
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
          imagePullPolicy: "Always"
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-certificates.crt"
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
          imagePullPolicy: "Always"
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-certificates.crt"
```

* Change `<CLUSTER NAME>` to match the name of the Kubernetes cluster.
* Deploy the cluster autoscaler into your cluster:
  * `kubectl create -f ~/autoscaler-deployment.yaml`

## Test the Autoscaler

To test the cluster autoscaler:

* `kubectl run autoscaler-test --image=nginx --replicas=<Number greater than your current Node count allows`
* Use `kubectl get nodes` to validate that new nodes are being added to the cluster.
  * Note: Nodes can take up to 15 minutes appear in the cluster. 

## Improvements

* Make the token uploader script run on each master and have it determine if the token needs to be updated, perhaps determing the time since the SSM property has been updated and if it has been longer than the expected time.
