# Configurar Servidor de Email em Debian (sem certificados) — VM

Guia passo a passo para configurar um servidor de email básico (Postfix + Dovecot) numa máquina virtual Debian, sem TLS/SSL. Indicado apenas para testes locais ou ambiente isolado, não para produção exposta à internet.

## Pré-requisitos

- Máquina virtual com Debian instalado (11 ou 12)
- Acesso root ou sudo
- Hostname definido (ex: mail.local)
- Rede interna configurada

## 1. Atualizar o sistema

```
sudo apt update
sudo apt upgrade -y
```

## 2. Definir o hostname

```
sudo hostnamectl set-hostname mail.local
```

Editar o ficheiro hosts:

```
sudo nano /etc/hosts
```

Adicionar a linha:

```
127.0.1.1   mail.local mail
```

## 3. Instalar o Postfix (servidor SMTP)

```
sudo apt install postfix -y
```

Durante a instalação, escolher:

- Tipo de configuração: Internet Site
- Mail name: mail.local

## 4. Configurar o Postfix

Editar o ficheiro de configuração principal:

```
sudo nano /etc/postfix/main.cf
```

Confirmar/ajustar estas linhas:

```
myhostname = mail.local
mydomain = local
myorigin = /etc/mailname
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
home_mailbox = Maildir/
```

Reiniciar o serviço:

```
sudo systemctl restart postfix
```

## 5. Instalar o Dovecot (POP3/IMAP)

```
sudo apt install dovecot-imapd dovecot-pop3d -y
```

## 6. Configurar o Dovecot

Editar o ficheiro de protocolos:

```
sudo nano /etc/dovecot/dovecot.conf
```

Confirmar a linha:

```
protocols = imap pop3
```

Editar a configuração de autenticação:

```
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

Alterar:

```
disable_plaintext_auth = no
auth_mechanisms = plain login
```

Editar a configuração do mail location:

```
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

Definir:

```
mail_location = maildir:~/Maildir
```

Editar a configuração SSL para desativar certificados:

```
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

Alterar:

```
ssl = no
```

Reiniciar o serviço:

```
sudo systemctl restart dovecot
```

## 7. Criar utilizador de teste

```
sudo adduser teste
```

Seguir as instruções para definir a password.

## 8. Testar o envio de email localmente

```
echo "Corpo da mensagem" | mail -s "Assunto de teste" teste@mail.local
```

Verificar a caixa de correio:

```
sudo su - teste
mail
```

## 9. Testar a ligação IMAP/POP3

A partir de outro terminal ou máquina na mesma rede:

```
telnet <ip-da-vm> 143
```

Deve aparecer uma resposta do Dovecot a confirmar que o serviço está ativo.

## 10. Verificar o estado dos serviços

```
sudo systemctl status postfix
sudo systemctl status dovecot
```

## Notas finais

- Esta configuração usa autenticação e transmissão em texto simples, sem encriptação.
- Adequado apenas para redes internas, testes ou laboratório.
- Para produção, deve ser adicionado TLS/SSL com certificados válidos.
