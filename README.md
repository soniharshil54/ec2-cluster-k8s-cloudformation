aws cloudformation create-stack --stack-name ec2-cluster-k8s --template-body file://cf-template.yml  --profile harshil-us-east-2

aws cloudformation update-stack --stack-name ec2-cluster-k8s --template-body file://cf-template.yml  --profile harshil-us-east-2

aws cloudformation delete-stack --stack-name ec2-cluster-k8s --profile harshil-us-east-2

ssh ec2-user@3.145.169.131 -i sharks.pem