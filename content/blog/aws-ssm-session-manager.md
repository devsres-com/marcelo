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

Algum tempo atrás, postei na minha página do Instagram um roteiro para começar a usar a AWS cli e criar uma instância EC2 na AWS. No fim, para mostrar que tudo funcionou, acessei a instância via SSH.

Embora eu tenha restringido o Security Group da instância para meu endereço IP, no mundo ideal não devemos abrir a porta SSH dos hosts diretamente para Internet. A recomendação oficial é que se use um "jumpbox" como Bastion Host.

A AWS oferece um recurso adicional: que tal acessar a sua máquina por meio da própria API? Isso é possível por meio do uso do **AWS Systems Manager**.

O caminho é um pouco burocrático para chegar lá - especialmente se você quiser seguir o caminho ideal de hosts em subnet privadas sendo acessados via VPC endpoint. Tendo isso em consideração, vamos começar do jeito mais simples: uma máquina EC2 criada exatamente no mesmo modelo do primeiro post, com a única diferença de não contar com um **Security Group** com a porta 22 aberta.

## Foreplay

Para viabilizar o acesos da instância EC2 por meio do **Systems Manager**, vamos precisar gerenciar o IAM. 

Estas operações são bem mais simples usando a interface gráfica; justamente por isso, vamos fazer tudo pela **linha de comando**!

Normalmente, para que uma instância EC2 faça uso de um serviço da AWS, é necessário atribuir uma **Role** do **IAM** para ela garantindo essas permissões. Por exemplo: sua aplicação irá acessar um **bucket S3** e o **DynamoDB**; a melhor maneira de viabilizar esse acesso é criar uma **Role** com as **Policies** apropriadas liberando os acessos. Ao fazer essa atribuição, a instância acessará os serviços sem precisar de credenciais na máquina ou algo assim.

Precisaremos fazer isso neste laboratório. Descrevendo o procedimento etapa por etapa, é necessário:

* Criar a **Policy** do IAM;
* Criar a **Role** do IAM;
* Atrelar a **Policy** a **Role** do IAM;
* Atrelar a **Role** a um **Instance Profile**;
* Atrelar o **Instance Profile** à instância EC2.

**Instance Profile** é uma coisa que você não vê a partir da Console - a AWS cria implicitamente para você quando uma **Role** é associada a uma instância. Não é o caso pela linha de comando: você tem que saber que isso existe e sev irar.

* Vamos criar a policy garantindo os acessos necessários para o **Systems Manager**:

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

$ POLICY_NAME=ssm-lab-policy
$ POLICY_ARN="$( aws iam create-policy     \
--policy-name $POLICY_NAME                 \
--policy-document file://ssm.policy.json   \
--query 'Policy.Arn' --output text)"
```

* Agora é a vez da **Role**. Note que é necessário passar um **Assume Role Policy Document** para o comando - na Console AWS, ele te oferece uma lista de botões para selecionar o serviço para ocultar a complexidade abaixo:
```bash
$ cat > arpd.ec2.json << EOF 
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

$ ROLE_NAME=ssm-lab-role

$ ROLE_ARN="$( aws iam          \
  create-role                   \
  --role-name $ROLE_NAME        \
  --assume-role-policy-document \
  file://arpd.ec2.json          \
  --query 'Role.Arn'            \
  --output text)"
```

* Não há qualquer associação da **Role** com a **Policy**. Você precisa fazê-la:
```
$ aws iam \ 
  attach-role-policy         \
  --role-name "$ROLE_NAME"   \
  --policy-arn "$POLICY_ARN"
```
> Observação: caso não tivéssemos criado uma Policy "nomeada", poderíamos passar a Policy no modo **inline** para a Role. Seria uma alternativa tranquila sem impacto aqui.

* Chegou a vez do **Instance Profile**:
```
$ IPROF_NAME=ssm-lab-irpof
$ IPROF_ARN=$( aws iam               \ 
create-instance-profile              \ 
--instance-profile-name $IPROF_NAME  \ 
--query 'InstanceProfile.Arn'        \
--output text)

$ echo $IPROF_ARN
arn:aws:iam::533426116307:instance-profile/ssm-lab-iprof
```
* E da associação da Role ao Instance Profile:
```
$ aws iam                             \ 
  add-role-to-instance-profile        \ 
  --instance-profile-name $IPROF_NAME \
  --role-name $ROLE_NAME
```

## "E se minha máquina já existir?"

Sem bronca!

Você precisa recuperar o ID da instância e fazer a atribuição com o comando abaixo.

```bash
$ aws ec2 associate-iam-instance-profile --instance-id $INSTANCE_ID --iam-instance-profile Name=$IPROF_NAME

An error occurred (IncorrectState) when calling the AssociateIamInstanceProfile operation: There is an existing association for instance i-0bc64fb2224bb30a9
```
Obviamente, a saída do meu deu erro porque ele já contava com um Instance Profile associado. Se você receber esse erro, é porque a instância já conta com um Instance Profile e você não pode fazer a atribuição sem destituir o original.

"Ah, mas eu quero substituir mesmo assim!"

Não recomendo ir por aí; a saída deu erro porque já existe uma atribuição. E se existe uma atribuição, provavelmente é porque o Instance Profile **serve para alguma coisa**. Se o resto dos procedimentos para acecsso não funcionar, você pode averiguar a possibilidade de fazer um **attach** da **Policy** à **Role** associada ao **Instance Profile**.

## Criação da VM

* Garanta que você é quem você espera:
```bash
# Alternativa mais segura ao aws iam get-user
$ AWS_USER=$( aws sts         \ 
  get-caller-identity         \ 
  --query 'Arn' --output text \ 
  | cut -f2 -d'/' )

$ echo $AWS_USER
cloud_user
```

* Criação do Security Group
```bash
$ AWS_SG_ID="$( 
    aws ec2 create-security-group  \
    --group-name 'Allow-NOTHING'   \
    --description 'Allow-NOTHING'  \
    --query 'GroupId'              \
    --output text )"

$ echo $AWS_SG_ID
sg-0d02a898f10f974fa    
```

* Vocês lembram como recuperar o nome da AMI mais recente da region em uso? Não esqueçam, AMI IDs são diferentes em cada região.
```bash
$ NS=ami-amazon-linux-latest
$ TYPE=amzn2-ami-hvm-x86_64-ebs
$ QUERY="Parameters[?ends_with(Name,\"$TYPE\")]"
$ QUERY=${QUERY//\"/\'}
$ AWS_IMAGE_ID="$( aws ssm    \
    get-parameters-by-path    \
    --path /aws/service/${NS} \
    --query ${QUERY}.Value    \
    --output=text )"

$ echo $AWS_IMAGE_ID
ami-0f5ea7c2783b14c09 
```

* Criação da EC2:
```bash
$ INSTANCE_ID="$( aws ec2          \
  run-instances --count 1          \
  --iam-instance-profile Name=$IPROF_NAME \
  --instance-type t3.medium        \
  --image-id $AWS_IMAGE_ID         \
  --security-group-ids $AWS_SG_ID  \
  --query 'Instances[].InstanceId' \
  --output text)"

$ echo $INSTANCE_ID
i-066da13197caef1c4
```
* Recuperando o IP público:
```bash
$ QUERY="Reservations[].Instances[].PublicIpAddress"
$ INSTANCE_IP="$( aws ec2      \
  describe-instances         \
  --instance-id $INSTANCE_ID \
  --query $QUERY --output text )"
```
* Testando a conectividade com o host:
```bash
$ ssh ec2_user@$INSTANCE_IP
ssh: connect to host 44.195.32.134 port 22: Connection timed out
```

Nada feito. Ótimo!

Para acessá-la, vamos usar o **Systems Manager**:

```bash
$ aws ssm start-session --target $INSTANCE_ID

Starting session with SessionId: cloud_user-0af04a7a1abc2b0c8

sh-4.2$ sudo -i
[root@ip-172-31-10-222 ~]#
```

Win!
