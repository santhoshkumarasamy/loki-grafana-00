# Deploy Log Monitoring in K8S

## Setup EC2 instance

```sh
aws ec2 run-instances --image-id 'ami-0b6d9d3d33ba97d99' --instance-type 't2.xlarge' --no-ebs-optimized --block-device-mappings '{"DeviceName":"/dev/sda1","Ebs":{"Encrypted":false,"DeleteOnTermination":true,"Iops":3000,"SnapshotId":"snap-0ffe259ee34f7697a","VolumeSize":50,"VolumeType":"gp3","Throughput":125}}' --network-interfaces '{"SubnetId":"subnet-02ccc0775e08a6c67","AssociatePublicIpAddress":true,"DeviceIndex":0,"Groups":["sg-081806af9eb6e3482"]}' --credit-specification '{"CpuCredits":"standard"}' --tag-specifications '{"ResourceType":"instance","Tags":[{"Key":"Name","Value":"log-monitoring"}]}' --iam-instance-profile '{"Arn":"arn:aws:iam::272577611462:instance-profile/my-ssm-role"}' --instance-market-options '{"MarketType":"spot"}' --metadata-options '{"HttpEndpoint":"enabled","HttpPutResponseHopLimit":2,"HttpTokens":"required"}' --private-dns-name-options '{"HostnameType":"ip-name","EnableResourceNameDnsARecord":false,"EnableResourceNameDnsAAAARecord":false}' --count '1' 
```

This creates a spot instance with ubuntu OS. I have selected an existing SG you may have to update it based on your needs.


## Setup the Minikube with Argo-CD


```sh
01_k8s_setup.sh
```

Run the commands in the script to setup the base things

- Docker
- Minikube
- Argo-CD

These will get installed.
Lets get it exposed and get password.

```sh
# Get the password
kubectl -n default get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# Expose the service to outside world
kubectl port-forward service/my-argo-cd-argocd-server -n default 8080:80 --address 0.0.0.0
```

## Setup the ArgoCD Application

You can find all the argo-cd Application manifest under ``argo_cd_apps`` folder

```sh
kubectl apply -f grafana.yaml
kubectl apply -f test_app.yaml
```

You might want to update the ``test_app.yaml`` file to point to your own github repo.
Once deployed everything, expose the grafana and get admin password
```sh
kubectl -n grafana get secret grafana -o jsonpath={.data.admin-password} | base64 -d
kubectl -n grafana port-forward --address 0.0.0.0 svc/grafana 8081:80
```


# Now lets setup loki and alloy 

## Setup Loki and alloy with some references

```sh
kubectl apply -f loki-v1.yaml
```

Here loki didnt use minio for storage cause its going deprecated.

```sh
kubectl apply -f alloy-v1.yaml
```
https://devopscube.com/setup-grafana-loki/#step-3-deploy-loki

This configs I took references form the above blog. Refer for more details.


## Setting up Loki with s3 storage

Okay so the above setup is not good for prod level setup

so lets setup loki aws s3 as a storage

deploy argo cd and the test application.

setup the namespace of the loki and the secret containing the aws access key

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: loki-s3-credentials
  namespace: default # change if using a different namespace
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "YOUR_ACTUAL_ACCESS_KEY"
  AWS_SECRET_ACCESS_KEY: "YOUR_ACTUAL_SECRET_KEY"
```

this part you can even improve this access
- this can done using IRSA
- In EKS use the POD Identity

