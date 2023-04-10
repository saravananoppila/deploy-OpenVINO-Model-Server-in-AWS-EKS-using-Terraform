# Deploy OpenVINO Model Server using Terraform in Amazon EKS

## Prerequisites
1. [Create and activate an AWS Account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)
2. Select your AWS Region. For the tutorial below, we assume the region to be ```us-east-2```
3. Create an IAM user with Adimistrator access.
4. Ubuntu machine 20.04 LTS
5. Git
6. Visual Studio Code


# step by step tutorial

This tutorial is to deploy OpenVINO Model Server in AWS EKS using Terraform. It requires multiple steps to complete this process, please follow the below steps to complete this.

## Install Terraform
Terraform configuration files in this repository are consistent with Terraform v1.4.4 syntax. It is used to provision the AWS EKS service.
pleae follow the beloe steps to install terraform in your machine. Please find the link for your reference [Install Terraform](https://developer.hashicorp.com/terraform/downloads)

```
$ wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
$ echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
$ sudo apt update && sudo apt install terraform
```
Please check whether you have installed the correct version or not.
``` terraform -v```

##  Install Helm

Helm is package manager for Kubernetes. It uses a package format named charts. A Helm chart is a collection of files that define Kubernetes resources. Please refer the official document for your referrence. [Install helm](https://helm.sh/docs/intro/install/).

```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```


## Install awscli and configure
The AWS Command Line Interface (AWS CLI) is an open source tool that enables you to interact with AWS services using commands in your command-line shell. 

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
After installing awscli please configure the user access key and secret key to interact with AWS services. To do that, please follow the below commands.
```
open terminal
type aws configure and press enter button
now you need to paste access key which you got for your user account and press enter
now you need to paste secret key which you got for your user account and press enter
now you need to paste default region and press enter
now press enter to complete the steps

```

# Use Terraform to create infrastructre

## Clone the ```aws terraform provider``` git repo

open terminal and please follow the below steps
```
git clone https://github.com/hashicorp/terraform-provider-aws.git
cd terraform-provider-aws/examples/eks-getting-started
```




Open Visual Studio Code to change the defaul aws region. From the same path in the terminal. please run ``` code . ``` to open Visual Studio Code and please open ```variable.tf``` and change the aws_region as ```us-east-2``` and save the file.  

switch to terminal and please run the below commands to provision AWS EKS cluster in your account.

```
terraform init   # Prepare your working directory for other commands
terraform validate # Check whether the configuration is valid
terraform plan # Show changes required by the current configuration
terraform apply -auto-approve # Create or update infrastructure
```

Once AWS EKS cluster is provision, please use your user login details to login into AWS Management console to verify whether services are create like VPC, security groups, IAM policy, AWS EKS cluster.
please login into AWS EKS service via termial using the below command

```
aws eks update-kubeconfig --region us-east-2 --name terraform-eks-demo
```


# Deploy OpenVINO Model Server in AWS EKS

## please use the below commands to download sample model files. 
open terminal and please follow the below steps
```
wget https://storage.openvinotoolkit.org/repositories/open_model_zoo/2022.1/models_bin/2/resnet50-binary-0001/FP32-INT1/resnet50-binary-0001.bin -P 1
wget https://storage.openvinotoolkit.org/repositories/open_model_zoo/2022.1/models_bin/2/resnet50-binary-0001/FP32-INT1/resnet50-binary-0001.xml -P 1
```

create a bucket in s3 and copy those above files in the bucket using below commands

```
aws s3api create-bucket --bucket testingcheckfolder --region us-east-2 --create-bucket-configuration LocationConstraint=us-east-2
aws s3 cp yourSubFolder s3://testingcheckfolder/model_files/models/resnet50/1/ --recursive
```

## Clone the ```OpenVINO Model Server Operator``` git repo

open terminal and please follow the below steps. Note: please provide user access and secret key in the helm command(3rd command)
```
$ git clone https://github.com/openvinotoolkit/operator.git
$ cd operator/helm-charts

$ helm install ovms-app11 ovms --set models_settings.model_name=resnet50,models_settings.model_path=s3://testingcheckfolder/model_files/models/resnet50/,models_repository.aws_access_key_id=xxxxxxxxxxx,models_repository.aws_secret_access_key=xxxxxxxxxx,models_repository.aws_region=us-east-2
```


Run ```kubectl get service```, you will get the below result

```
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
kubernetes   ClusterIP   172.20.0.1       <none>        443/TCP             1h
ovms-app11   ClusterIP   172.20.44.157    <none>        8080/TCP,8081/TCP   19s

```

Run the following command to create a client container, launch an interactive session to a pod with Python installed 

```kubectl create deployment client-test --image=python:3.8.13 -- sleep infinity```

Run kubectl get pods, you will get the below result

```
NAME                           READY   STATUS    RESTARTS   AGE
client-test-558d9d6cc8-bphg7   1/1     Running   0          1hs
ovms-app11-6b6c49f7f6-9mw2f    1/1     Running   0          90s

```


Run the following command to get inside the pod ```kubectl exec -it $(kubectl get pod -o jsonpath="{.items[0].metadata.name}" -l app=client-test) -- bash```


From inside the client container, we will connect to the model server API endpoints. A simple curl command lists the served models with their version and status:



```
curl http://ovms-app11:8081/v1/config
{
"resnet50" : 
{
 "model_version_status": [
  {
   "version": "1",
   "state": "AVAILABLE",
   "status": {
    "error_code": "OK",
    "error_message": "OK"
   }
  }
 ]
}
```

Run the following command to install prerequisite for opencv python packages ```apt-get update && apt-get install ffmpeg libsm6 libxext6  -y```

Now let’s use the ovmsclient Python library to process an inference request. Create a virtual environment and install the client with pip:

```
python3 -m venv /tmp/openvino
source /tmp/openvino/bin/activate
pip install ovmsclient opencv-python

```

Download a sample image of a zebra:

```
curl https://raw.githubusercontent.com/openvinotoolkit/model_server/main/demos/common/static/images/zebra.jpeg -o /tmp/zebra.jpeg
```

Before you run the inference make sure input and output metadata names are correct.


```
from ovmsclient import make_grpc_client
client = make_grpc_client("ovms-app11:8080")
model_metadata = client.get_model_metadata(model_name="resnet50")
print(model_metadata)

{'model_version': 1, 
  'inputs': {'0': {'shape': [1, 3, 224, 224], 'dtype': 'DT_FLOAT'}}, 
  'outputs': {'1463': {'shape': [1, 1000], 'dtype': 'DT_FLOAT'}}}
```
press ctrl+ z to exit from python

## To run inference for a single image, please run the below command,

```
cat >> /tmp/predict.py <<EOL
import cv2, numpy as np
from ovmsclient import make_grpc_client
client = make_grpc_client("ovms-app11:8080")
img=cv2.imread("/tmp/zebra.jpeg")
resize_img = cv2.resize(img, (224,224))
image = resize_img[..., np.newaxis]
training_data = np.transpose(image, (3, 2,0,1))
inputs = {"0": np.float32(training_data)}
results = client.predict(inputs=inputs, model_name="resnet50")
print("Detected class:", np.argmax(results))
EOL

python /tmp/predict.py
Detected class: 340
```
## Destroy or delete everything

Firstly, exit from the server, from "helm-chart" path in the terminal 

``` 
kubectl delete pod_name
 ```

From "terraform-provider-aws/examples/eks-getting-started" path open terminal and run the below command

```
terraform destroy -auto-approve
```




