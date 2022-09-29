# Respostas Desafio AWS

## 1 - Setup de ambiente

Após seguir os passos indicados, a instância foi criada com sucesso.

![alt](/desafio-aws/prints/01.png)

## 2 - Networking

Foram encontrados dois problemas: O range de portas do SecurityGroupIngress não inclui a porta 80 e o CidrIp possui a mascara /1

#### 1. Editei as regras do Security Group

![alt](/desafio-aws/prints/02a.png)

#### 2. Acessei o DNS

![alt](/desafio-aws/prints/02b.png)

## 3 - EC2 Access

### 1 - Acesse a EC2

Para a resolução deste item, criei uma EC2 a partir da AIM (ID: ami-026b57f3c383c2eec) com subnet na mesma sub-região da EC2 do desafio com o key pair criado no passo 1, abaixo.

#### 1. Criando um novo key-pair pelo aws cli e armazenado a private key em um arquivo na máquina local (Ubuntu 20.04) e as permissão de acesso foram atualizada.

```console
$ aws ec2 create-key-pair --key-name desafio-aws-keys --query 'KeyMaterial' --output text > desafio-aws.pem
$ chmod 400 desafio-aws.pem
$ aws ec2 describe-key-pairs --key-name desafio-aws-pair-key
```

![alt](/desafio-aws/prints/03a.png)

#### 2. Parando a instancia a partir do seu ID

```console
$ aws ec2 stop-instances --instance-ids i-03b757ce28143a780
```

#### 3. Detach o volume root da EC2 do desafio e attach na instancia criada

```console
$ aws ec2 detach-volume --volume-id vol-0f75b2cfc5b28ddaa
$ aws ec2 attach-volume --volume-id vol-0f75b2cfc5b28ddaa --instance-id i-07451db4ea4c4b5de --device /dev/sdf
$ aws ec2 describe-volumes
```

![alt](/desafio-aws/prints/03c.png)

#### 4. Conectando com a instancia temporária e montando o novo volume como root

```console
$ ssh -i desafio-aws.pem ec2-user@ec2-54-167-49-54.compute-1.amazonaws.com
$ lsblk
$ sudo su -
$ mount /dev/xvdf1 /mnt/
```

#### 5. Copiando a public key da instancia temporária para a EC2 do desafio

```console
$ cd /mnt/home/ec2-user/.ssh/
$ cat /home/ec2-user/.ssh/authorized_keys > authorized_keys
```

![alt](/desafio-aws/prints/03d.png)

#### 6. Saindo e parando a instancia temporária e associando o volume root na EC2 do desafio

```console
exit
$ aws ec2 stop-instances --instance-ids i-07451db4ea4c4b5de
$ aws ec2 detach-volume --volume-id vol-0f75b2cfc5b28ddaa
$ aws ec2 describe-volumes
    Output: "VolumeId": vol-0f75b2cfc5b28ddaa
$ aws ec2 attach-volume --volume-id vol-0f75b2cfc5b28ddaa --instance-id i-03b757ce28143a780 --device /dev/xvda
```

#### 7. Iniciando a EC2 do desafio e acessando com a nova chave pelo ssh

```console
$ aws ec2 start-instances --instance-ids i-03b757ce28143a780
$ aws ec2 describe-instance-status --instance-id i-03b757ce28143a780
$ ssh -i desafio-aws.pem ec2-user@ec2-54-91-108-20.compute-1.amazonaws.com
```

![alt](/desafio-aws/prints/03e.png)

### 2 - Alterando o texto da página web exibida

```console
$ vim /var/www/html/index.html
$ systemctl start httpd
```

![alt](/desafio-aws/prints/04.png)

## 4 - EC2 troubleshooting

Só foi necessário enable o httpd.

```console
$ systemctl status httpd
$ systemctl enable httpd
$ systemctl start httpd
```

![alt](/desafio-aws/prints/05.png)

## 5 - Balanceamento

#### Solução

Pelo AWS Cli, foram executadas os seguintes passos:


#### 1. Foi criado uma imagem da EC2 do desafio

```console
$ aws ec2 create-image --instance-id i-03b757ce28143a780 --name "AIM Desafio AWS"
    Output: "ImageId": "ami-02b34d15325785caf"
```

#### 2. Uma segunda instancia foi inicada com a imagem criada

```console
$ aws ec2 run-instances \
$ >     --image-id ami-02b34d15325785caf \
$ >     --instance-type t2.micro \
$ >     --key-name desafio-aws-keys \
$ >     --security-group-ids sg-02d9fe585bd3d11d4 \
$ >     --subnet-id subnet-0c51959edc1be3c7d \
$ >     --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value='2 - Formando DevOps - AWS Challenge EC2'}]"
    Output: "InstanceId": "i-04bedcbabe72db59b"
```

#### 3. Foi criado um target group do tipo aplicação

```console
aws elbv2 create-target-group \
$ >     --name desafio-aws-tg-grp \
$ >     --protocol HTTP \
$ >     --port 80 \
$ >     --target-type instance \
$ >     --vpc-id vpc-07c63844ea92fa082
    Output: 
    "TargetGroupArn":
        "arn:aws:elasticloadbalancing:us-east-1:601618353109:targetgroup/desafio-aws-tg-grp/2a6550d9432bde37",
    "TargetGroupName": 
        "desafio-aws-tg-grp",
```

#### 4. As duas instancias foram registradas no target group

```console
$ aws elbv2 register-targets \
$ >     --target-group-arn arn:aws:elasticloadbalancing:us-east-1:601618353109:targetgroup/desafio-aws-tg-grp/2a6550d9432bde37 \
$ >     --targets Id=i-03b757ce28143a780 Id=i-04bedcbabe72db59b
```

#### 5. Foi criado um novo security group para o load balancer e inbound rule na porta 80

```console
$ aws ec2 create-security-group /
$ > --description "LBSecurityGroupDesafio" /
$ > --group-name lb-secgrp /
$ > --vpc-id vpc-07c63844ea92fa082

$ aws ec2 authorize-security-group-ingress /
$ > --group-id sg-058caac40dce2c80a /
$ > --protocol tcp 
$ > --port 80 
$ > --cidr 0.0.0.0/0
```

#### 6. Foi criado o load-balancer com duas subnets nas regiões us-east-1a e u-east-1d e o novo security group

```console
$ aws elbv2 create-load-balancer \
$ >     --name lb-desafio-aws \
$ >     --subnets subnet-0c51959edc1be3c7d subnet-05389f5ab47790966 \
$ >     --security-groups sg-058caac40dce2c80a 
    Output: "LoadBalancerArn": "arn:aws:elasticloadbalancing:us-east-1:601618353109:loadbalancer/app/lb-desafio-aws/744112ed0bc52629"
```

#### 7. Por fim, o load balancer foi associado ao target group

```console
$ > aws elbv2 create-listener \
$ > --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:601618353109:loadbalancer/app/lb-desafio-aws/744112ed0bc52629 \
$ >  --protocol HTTP --port 80  \
$ >  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:601618353109:targetgroup/desafio-aws-tg-grp/2a6550d9432bde37

```

> Para fim de teste, foram modificados os textos em [/var/www/html](https://) em cada instancia.

#### 1. Teste com as duas instancias ligadas:

<center>

![alt](/desafio-aws/prints/06a.png)

</center>
<center>

![alt](/desafio-aws/prints/06b.png)

</center>

#### 2. Teste com a segunda instancia desligada.

<center>

![alt](/desafio-aws/prints/06d.png)

</center>
<center>

![alt](/desafio-aws/prints/06c.png)

</center>

## 6 - Segurança

Para impedir o acesso externo ao endereço das instancias, as regras da porta 80 do security group devem ser modificadas para somente ser associada com o load balancer. Por questão de segurança, optei por deletar a regra e implementar uma nova.

```console
# Descobrindo o ID da regra
$ aws ec2 describe-security-group-rules \
>     --filter Name="group-id",Values="sg-02d9fe585bd3d11d4"
    Output: "SecurityGroupRuleId": sgr-0c159380ad90fc1e3

# Removendo a regra pelo ID
$ aws ec2 revoke-security-group-ingress --group-id sg-02d9fe585bd3d11d4 --security-group-rule-ids sgr-0c159380ad90fc1e3

# Criando uma nova inbound rule na porta 80, protocolo tcp e o id do security group do load balancer.
$ aws ec2 authorize-security-group-ingress --group-id sg-02d9fe585bd3d11d4 --protocol tcp --port 80 --source-group sg-058caac40dce2c80a

```

#### Nova regra implementada

<center>

![alt](/desafio-aws/prints/07b.png)

</center>

#### Teste com o dns das duas instancias ligadas:

<center>

![alt](/desafio-aws/prints/07a.png)

</center>
