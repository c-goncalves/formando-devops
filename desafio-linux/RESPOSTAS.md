# Respostas Desafio Linux 1

## 1. Kernel e Boot loader

#### Solução

Crendenciar novas senhas de acesso do root e adicionar o usuario `vagrant` ao grupo `wheel`.

1. Na inicialização da máquina "fisica" (vm), entrar em modo de edição no GRUB e modificar o codigo `ro` para `rw init=/sysroot/bin/sh`;
2. Modificando o diretorio do root pelo chroot (jail): `chroot /sysroot`;
3. Mudando a senha: `passwd root`;
4. Atualizando as novas configurações: `touch /.autorelabel`;
5. Reinicie o sistema e logar como `root`;
6. Ativar o grupo wheel que concede as as permissões `sudo` do CentOS descomentando a linha `%wheel` pelo comando `visudo`;
7. Adicionando o usuario `vagrant`: `usermod -aG wheel vagrant`;
8. Reinicie a maquina.

## 2. Usuários

### 2.1 Criação de usuários

#### Solução

1. Criando o usuario `getup`: `adduser -u 1111 -s /bin/bash -G bin,wheel getup`;
2. Modificando o GID do grupo getup: `groupmod -g 2222 getup`;

## 3. SSH

### 3.1 Autenticação confiável

#### Solução

1. Modificar o arquivo `/etc/ssh/sshd_config.d` na linha `#PasswordAuthentication yes` para `PasswordAuthentication no`;
2. Reiniciar o processo com `systemctl restart sshd`.

### 3.2 Criação de chaves

#### Solução

1. Criando a chave `ssh-keygen -t ecdsa getup`
2. Enviando para o usuario `vagrant`: `ssh-copy-id -i getup.pub vagrant@192.168.0.114`;
3. Reiniciar o processo com `systemctl restart sshd`.

### 3.3 Análise de logs e configurações ssh

#### Solução

1. Extrair private key `cat id_rsa-desafio-linux-devel.gz.b64 | base64 -di | zcat > /home/desafio-getup/.ssh/id_rsa-desafio-linux-devel`;
2. Teste da chave `ssh-keygen -y -i id_rsa-desafio-linux-devel`.
Erro: formato invalido.
3. Converter formato para unix: `dos2unix id_rsa-desafio-linux-devel`
4. Acessando com o ssh:  `ssh devel@192.168.0.114 -i id_rsa-desafio-linux-devel`

## 4. Systemd

#### Erros apresentados pelo `systemctl status nginx`

1. Erro no arquivo `etc/nginx/nginx.conf`;
2. Erro no parametro de execução do systemd `nginx: invalid option "-B"`;
3. Porta já em uso.

#### Solução

1. No arquivo `etc/nginx/nginx.conf`, nas configurações `root` no bloco `server`, inserir `;` no final da linha 44;

    Opcional: Trocar path do `root` para `/var/wwww/html`;

2. Editar o arquivo de execução `/lib/systemd/system/nginx.service` na linha `ExecStart=/usr/sbin/nginx -BROKEN` para `ExecStart=/usr/sbin/nginx` e reiniciar o daemon: `systemctl daemon-reload`;
3. Editar a porta padrão no arquivo `etc/nginx/nginx.conf`, nas configurações `listen` para `80`.
4. Iniciar o nginx com o comando `systemctl start nginx`.

## 5. SSL

### 5.1 Criação de certificados

#### Solução

No diretório `/etc/ssl/certs`:
1. Criando a chave privada: `openssl genrsa -des3 -out desafio.local.private.key 4096`;
2. Criando pedido de certificação com o CN www.desafio.local:`openssl req -new -x509 -days 365 -key desafio.local.private.key -out desafio.local.pem`
3. Criando chave para o server: `openssl genrsa -des3 -out desafio.local.server.key 4096`
4. Criando pedido de assinatura no server com o CN www.desafio.local:`openssl req -newkey rsa:2048 -keyout desafio.local.private.key -out desafio.local.server.csr`
5. Carregando o CSR para o CA: `openssl x509 -req -days 365 -in desafio.local.server.csr -CA desafio.local.pem -CAkey desafio.local.private.key -CAcreateserial -out desafio.local.server.pem`

### 5.2 Uso de certificados

#### Solução

1. Criando os diretórios do site: `mkdir -p /var/www/desafio.local/html`
2. Criando uma página index: `vim /var/www/desafio.local/html/index.hmtl`
3. Alterando as permissões do diretório: `chmod 755 /var/www/desafio.local/html`
4. Criando os diretorios de configuração: `mkdir -p /etc/nginx/sites-available` e `mkdir -p /etc/nginx/sites-enabled`;
5. Arquivo de configuração, `vim /etc/nginx/nginx.conf`:

```console
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*.conf;
```

6. Arquivo de configuração, `vim /etc/nginx/sites-available/desafio.local.conf`:

```console
server {
    listen 443 ssl default_server;
    ssl_certificate /ect/ssl/certs/desafio.local.server.pem;
    ssl_certificate_key /ect/ssl/certs/desafio.local.server.pem;

   listen 80;

   server_name desafio.local www.desafio.local;

   location / {
      root /var/www/desafio.local/html;
      index index.html index.htm;
      try_files $uri $uri/ =404;
   }

   error_page 500 502 503 504 /50x.html;
   location = /50x.html {
      root html;
   }
}
```

7. Habilitando o site: `ln -s /etc/nginx/sites-available/desafio.local.conf /etc/nginx/sites-enabled/desafio.local.conf`;
8. Reiniciando o nginx: `systemctl restart nginx.service`;
9. Atualizando os hosts em `vim /etc/hosts`;
10. Teste `curl www.desafio.local` realizado com sucesso.

## 6. Rede

### 6.1 Firewall

O comando `ping 8.8.8.8` não apresenta erros.

### 6.2 HTTP

#### Solução

`curl https://httpbin.org/response-headers?hello=world -v`

```console
*   Trying 34.227.213.82...
* TCP_NODELAY set
* Connected to httpbin.org (34.227.213.82) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/pki/tls/certs/ca-bundle.crt
  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=httpbin.org
*  start date: Nov 21 00:00:00 2021 GMT
*  expire date: Dec 19 23:59:59 2022 GMT
*  subjectAltName: host "httpbin.org" matched cert's "httpbin.org"
*  issuer: C=US; O=Amazon; OU=Server CA 1B; CN=Amazon
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x5631ee295690)
> GET /response-headers?hello=world HTTP/2
> Host: httpbin.org
> User-Agent: curl/7.61.1
> Accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
< HTTP/2 200 
< date: Wed, 07 Sep 2022 20:46:14 GMT
< content-type: application/json
< content-length: 89
< server: gunicorn/19.9.0
< hello: world
< access-control-allow-origin: *
< access-control-allow-credentials: true
< 
{
  "Content-Length": "89", 
  "Content-Type": "application/json", 
  "hello": "world"
}
* Connection #0 to host httpbin.org left intact

```

## Logs

#### Solução

1. Criar arquivo de configuração em `/etc/logrotate.d/nginx' com os parametros desejados como no exemplo abaixo:

```console
/var/log/nginx/*.log {
        create 0644 nginx nginx
        rotate 15
        daily
        missingok
        notifempty
        compress
        sharedscripts
           postrotate
               invoke-rc.d nginx rotate >/dev/null 2>&1
           endscript
       }
```

2. Reinicie o `logrotate`;

## 7. Filesystem

### 7.1 Expandir partição LVM

### Solução

1. Consultando os volumes e partições com `lsblk`;
2. Editando a partição `sdb` com o `fdisk /dev/sdb`:

```console
+   p       #listando as partições

+   n       #nova partição
+   p       #primaria
+   1       #nº da partição
+           #first sector default
+   +5G     #last sector 5Gi

+   t       #formato
+   8e      $hexadecimal para LVM

+   w       #save
```

2. Atualizando a tabela de particionamento: `partprobe`;
3. Extendendo o grupo do volume: `lvextend -l 100%FREE data_vg`
4. Atualizando o filesystem em `ext4`: `resize2fs data_vg`

### 7.2 Criar partição LVM

#### Solução

1. Criando a particao `sd2` com o `fdisk /dev/sdb`:

```console

fdisk /dev/sdb
+   n       #nova partição
+   p       #primaria
+   2       #nº da partição
+           #first sector default
+           #last sector default 5Gi

+   t       #formato
+   8e      $hexadecimal para LVM

+   w       save
```

2. Atualizando a tabela de particionamento: `partprobe`
3. Criando o volume com `lvcreate /dev/sd2`;
3. Extendendo o grupo do volume: `lvextend -l 100%FREE data_vg`
4. Atualizando o filesystem em `ext4`: `resize2fs data_vg`

### 7.3 Criar partição XFS

#### Solução

1. Criando o disco: `vgcreate data-vg /dev/sdc`;
2. Criando a LV: `lvcreate -l 100%FREE -v -n data-lv data-vg`
3. Confirmando: `vgs data-vg`
4. Formatando para `xfs`: `makes t -xfs /dev/data-vg/data-lv`.
