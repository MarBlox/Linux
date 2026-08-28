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

> **Nota:** este guia foi testado com Postfix e Dovecot 2.4. Nesta versão, vários parâmetros antigos do Dovecot foram renomeados (ex: `disable_plaintext_auth` deixou de existir). As secções abaixo já usam a sintaxe correta para 2.4.

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

Alterar (sintaxe Dovecot 2.4):

```
auth_allow_cleartext = yes
auth_mechanisms = plain login
auth_username_format = %{user | username}
```

`auth_allow_cleartext` substitui o antigo `disable_plaintext_auth` (que já não existe no Dovecot 2.4). `auth_username_format` garante que um login como `utilizador@dominio` é tratado apenas como `utilizador` pelo sistema, evitando erros de "user unknown".

Editar a configuração do mail location:

```
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

Definir (sintaxe Dovecot 2.4 — o antigo `mail_location` foi dividido em duas configurações):

```
mail_driver = maildir
mail_path = ~/Maildir
```

Editar a configuração SSL para desativar certificados:

```
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

Alterar:

```
ssl = no
```

Verificar se a configuração está correta antes de reiniciar:

```
sudo doveconf -n
```

Se não houver erros, reiniciar o serviço:

```
sudo systemctl restart dovecot
```

## 7. Configurar autenticação SMTP (SASL) no Postfix

Sem isto, o Outlook (e outros clientes modernos) recusam ligar ao SMTP porque o Postfix não anuncia nenhum método de autenticação.

Instalar suporte SASL:

```
sudo apt install libsasl2-modules -y
```

Editar o Dovecot para disponibilizar o socket de autenticação ao Postfix:

```
sudo nano /etc/dovecot/conf.d/10-master.conf
```

Dentro do bloco `service auth {`, adicionar:

```
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
}
```

Editar o Postfix:

```
sudo nano /etc/postfix/main.cf
```

Adicionar no fim do ficheiro:

```
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
broken_sasl_auth_clients = yes
smtpd_recipient_restrictions = permit_sasl_authenticated, permit_mynetworks, reject_unauth_destination
```

Reiniciar os dois serviços:

```
sudo systemctl restart dovecot
sudo systemctl restart postfix
```

Confirmar que o Postfix já anuncia autenticação:

```
telnet localhost 25
EHLO teste
```

Deve aparecer uma linha `250-AUTH PLAIN LOGIN` na resposta.

## 9. Criar utilizador de teste

```
sudo adduser teste
```

Seguir as instruções para definir a password.

## 10. Testar o envio de email localmente

```
echo "Corpo da mensagem" | mail -s "Assunto de teste" teste@mail.local
```

Verificar a caixa de correio:

```
sudo su - teste
mail
```

## 11. Testar a ligação IMAP/POP3

A partir de outro terminal ou máquina na mesma rede:

```
telnet <ip-da-vm> 143
```

Deve aparecer uma resposta do Dovecot a confirmar que o serviço está ativo.

## 12. Verificar o estado dos serviços

```
sudo systemctl status postfix
sudo systemctl status dovecot
```

## 13. Configurar a conta no Outlook

Sem certificados, usar as portas sem encriptação:

- Servidor de entrada (IMAP): IP da VM, porta 143, encriptação: Nenhuma
- Servidor de entrada (POP3, alternativa): IP da VM, porta 110, encriptação: Nenhuma
- Servidor de saída (SMTP): IP da VM, porta 25, encriptação: Nenhuma
- Autenticação: password normal (login = utilizador criado, ex: teste)

Nota: o Outlook novo (o cliente moderno da Microsoft) pode recusar ligações sem SSL/TLS independentemente da configuração do servidor. Se isso acontecer, usar o Outlook clássico (desktop) ou outro cliente como o Thunderbird, que permitem desativar a encriptação manualmente.

## Notas finais

- Esta configuração usa autenticação e transmissão em texto simples, sem encriptação.
- Adequado apenas para redes internas, testes ou laboratório.
- Para produção, deve ser adicionado TLS/SSL com certificados válidos.
