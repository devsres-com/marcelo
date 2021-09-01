+++
title = "SSH na AWS sem precisar abrir portas!"
description = "Acessando máquinas sem abrir firewall na AWS"
author = "Marcelo Andrade"
date = "2021-08-31"
tags = ["aws", "ssm", "ssh"]
categories = ["aws", "ssm", "ssh"]
[[images]]
  src = "img/reusable/boxes.jpg"
  alt = "Boxes are containers"
  stretch = ""
+++

* Quem sou eu?
```bash
# Uma outra maneira seria  usar
# aws iam get-user --query='User.Username'
$ AWS_USER=$( aws sts get-caller-identity --query 'Arn' --output text | cut -f2 -d'/' )
```
* Upload da chave SSH:
```bash
$ aws ec2 import-key-pair \
  --key-name "$AWS_USER"  \
  --public-key-material="
  $( cat ~/.ssh/id_ed25519.pub | base64 )"
{
    "KeyFingerprint": "dqDNVFIfUJjiyoW3WGhucBmfzLe0t6tMs0VX7Hhz4CI=",
    "KeyName": "cloud_user",
    "KeyPairId": "key-07c7ea29a43b32a04"
}
```

Hoje vamos criar um VPC.

```bash
$ VPC_ID=$( aws ec2 create-vpc \
  --cidr-block 192.168.0.0/16   \
  --query Vpc.VpcId --output text )

$ echo $VPC_ID
vpc-0cf113c271c7881c8
```
O VPC, por si só, não resolve muita coisa. Precisamos de uma subnet.

```bash
$ SUBNET_ID=$( aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 192.168.0.0/24 --query Subnet.SubnetId --output text)

$ echo $SUBNET_ID
subnet-06c762011fd4b73ab
```

* Criação do Security Group
```bash
$ AWS_SG_ID="$( 
    aws ec2 create-security-group  \
    --group-name 'Allow-NOTHING'   \
    --description 'Allow-NOTHING'  \
    --vpc-id $VPC_ID               \
    --query 'GroupId'              \
    --output text )"

$ echo $AWS_SG_ID
sg-0d02a898f10f974fa    
```

* Que AMI usar?
```bash
$ NS=ami-amazon-linux-latest
$ TYPE=amzn2-ami-hvm-x86_64-ebs
$ QUERY="Parameters[?ends_with(Name,\"$TYPE\")]"
$ QUERY=${QUERY//\"/\'}
$ AWS_IMAGE_ID="$( aws ssm  \
    get-parameters-by-path    \
    --path /aws/service/${NS} \
    --query ${QUERY}.Value \
    --output=text )"

$ echo $AWS_IMAGE_ID
ami-0f5ea7c2783b14c09 
```

* Criação da EC2:
```bash
$ INSTANCE_ID="$( aws ec2          \
  run-instances --count 1          \
  --instance-type t3.medium        \
  --key-name $AWS_USER             \
  --image-id $AWS_IMAGE_ID         \
  --subnet-id $SUBNET_ID           \
  --security-group-ids $AWS_SG_ID  \
  --query 'Instances[].InstanceId' \
  --output text)"

$ echo $INSTANCE_ID
i-066da13197caef1c4
```

Só que... faltam coisas.

Precisamos criar um **instance profile** para nossa Instância EC2.

Como fazer isso? Bem, na linha de comando, é um procecdimento chato - não é difícil, é só chato. Você precisa:

* Criar a **Policy** do IAM;
* Criar a **Role** do IAM;
* Atrelar a **Policy** a **Role** do IAM;
* Atrelar a **Role** a um **Instance Profile**;
* Atrelar o **Instance Profile** à instância EC2.

(Dos passos acima, a criação da Role é facultativa, pois é possível criar uma inline policy, mas vamos deste jeito!)


```bash
$ cat > ssm.policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:UpdateInstanceInformation",
                "ssmmessages:CreateControlChannel",
                "ssmmessages:CreateDataChannel",
                "ssmmessages:OpenControlChannel",
                "ssmmessages:OpenDataChannel"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetEncryptionConfiguration"
            ],
            "Resource": "*"
        }
    ]
}
EOF

POLICY_NAME=ssm-lab-policy
aws iam create-policy --policy-name $POLICY_NAME --policy-document file://ssm.policy.json

cat > arpd.ec2.json << EOF 
{
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
}
EOF

ROLE_NAME=ssm-lab-role

aws iam create-role --role-name $ROLE_NAME --assume-role-policy-document file://arpd.ec2.json


```