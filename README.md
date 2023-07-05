# Guia de Auto Scaling Jitsi, Jibri-docker na AWS

[![Powered By Deyvi-dev](https://img.shields.io/static/v1?label=Powered%20By&message=Deyvi-dev&color=blue&style=flat-square)](https://github.com/Deyvi-dev)

# Descrição

Este guia é um passo a passo que apresenta o processo de auto scaling para Jitsi e Jibri-docker na AWS. Utilizaremos instâncias EC2 para o Jitsi e o Jibri, com cada instância do Jibri contendo 4 contêineres. O auto scaling será realizado de forma horizontal, aproveitando recursos como Alarme CloudWatch, Amazon EC2, Auto Scaling Groups e AWS S3.

No servidor do Jitsi, será implantado um script cronjob  para verificar a disponibilidade de instâncias do Jibri. Quando apenas uma instância do Jibri estiver disponível, um alerta será enviado para a AWS, solicitando o início de uma nova instância do Jibri.

Além disso, no servidor do Jibri, um script cronjob será configurado para consultar a cada minuto se os contêineres estão em uso. Caso não estejam em uso e a API do Jitsi retorne que há mais de 5 instâncias do Jibri disponíveis, a instância será automaticamente encerrada, mantendo o fluxo do auto scaling.

# Documentação

- [# Criação de Máquinas EC2 do Jitsi](#criação-de-máquinas-ec2-do-jitsi)
- [# Instalação do Jitsi](#instalação-do-jitsi)
- [# Ajustando um ambiente Jitsi Meet para o Jibri](#ajustando-um-ambiente-jitsi-meet-para-o-jibri)
- [# Criação de Máquina EC2 do Jibri](#criação-de-máquina-ec2-do-jibri)
- [# Instalação do Jibri-docker](#instalação-do-jibri-docker)
- [# Configurando Alarme CloudWatch](#configurando-alarme-cloudWatch)
- [# Configurando AWS Auto Scaling Groups](#configurando-aws-auto-scaling-groups)
- [# Script cronjob jitsi](#script-cronjob-jitsi)
- [# Script cronjob jibri](#script-cronjob-jibri)
- [# Envio dos videos para AWS S3](#envio-dos-videos-para-aws-s3)

# Criação de Máquinas EC2 do Jitsi

Para criar as máquinas EC2 do Jitsi, siga as etapas abaixo:

* Acesse o [Console da AWS](https://us-east-1.console.aws.amazon.com/console/home?region=us-east-1).
* Pesquise por "EC2" na barra de pesquisa do console.
* Selecione a região desejada, no meu caso foi a us-east-1.
* Clique em "executar instâncias", mostrado na imagem abaixo:

![Texto alternativo](images/jitsi/instancia.png)

* Coloque um nome para a instância no meu caso é o jitsi

![nome jitsi](images/jitsi/name-jitsi.png)

* escolha uma imagem. Neste guia, usaremos a imagem Ubuntu Server 22.04 LTS:

![imagem ubuntu](images/jitsi/ami-jitsi.png)

* Selecione um tipo de instância t3.medium.

* Gere um par de chaves para acessar via SSH.

* Crie um grupo de segurança com intervalo de portas 5280, 443, 22, 5349, 10000, 3478, 80, 5222 e 8888, com origem 0.0.0.0/0. Veja o exemplo na imagem abaixo:

![grupo-port](images/jitsi/grupo-port.png)

* Adicione um IP elástico à instância. Você precisará de um domínio e deverá apontar o DNS para o IP escolhido.

![elastic-ip](images//jitsi/elastic-ip.png)

# Instalação do Jitsi

### Para instalação Jitsi, usaremos o guia de instalação do jitsi [Self-Hosting Guide - Debian/Ubuntu server](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-quickstart):


## Pré requisitos:
### Acesse o servidor Jitsi via SSH usando a chave criada

* Abra o terminal no seu computador.

* Mude as permissões do arquivo de chave para garantir que apenas você possa acessá-lo. Substitua `jitsi.pem` pelo nome do seu arquivo de chave, se necessário.

```bash
chmod 400 jitsi.pem
```

* Conecte-se ao servidor Jitsi usando SSH e a chave privada. Substitua `jitsi.pem` pelo nome do seu arquivo de chave e `seu_endereco_ip` pelo endereço IP do servidor Jitsi.

```bash
ssh -i jitsi.pem ubuntu@seu_endereco_ip
```

### Atualize as versões mais recentes dos pacotes em todos os repositórios, Em seguida instale o suporte para repositórios do apt servidos via HTTPS:

```bash
sudo apt update && sudo apt -y upgrade
sudo apt install apt-transport-https
```

* Em sistemas Ubuntu, o Jitsi requer dependências do Ubuntu universerepositório de pacotes. Para garantir que isso esteja ativado, execute este comando: 

```bash
sudo apt-add-repository universe
```

* Recupere as versões de pacote mais recentes em todos os repositórios `sudo apt update`

### Configure o nome de domínio totalmente qualificado (FQDN)

```bash
sudo hostnamectl set-hostname seudominio.com
```
* Em seguida, adicione o mesmo FQDN no /etc/hosts no arquivo: 
`sudo nano /etc/hosts`

```bash
127.0.0.1 localhost
x.x.x.x seudominio.com
```

:warning: **Observação:** Substitua `x.x.x.x` pelo endereço IP público do seu servidor.

### Adicione o repositório de pacotes Prosody

#### Ubuntu 18.04 e 20.04 

```bash
echo deb http://packages.prosody.im/debian $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list
wget https://prosody.im/files/prosody-debian-packages.key -O- | sudo apt-key add -
apt install lua5.2
```

#### Ubuntu 22.04

```bash
curl -sL https://prosody.im/files/prosody-debian-packages.key | sudo tee /etc/apt/keyrings/prosody-debian-packages.key
echo "deb [signed-by=/etc/apt/keyrings/prosody-debian-packages.key] http://packages.prosody.im/debian $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/prosody-debian-packages.list
apt install lua5.2
```

### Adicione o repositório de pacotes Jitsi

#### Ubuntu 18.04 e 20.04 

```bash
curl https://download.jitsi.org/jitsi-key.gpg.key | sudo sh -c 'gpg --dearmor > /usr/share/keyrings/jitsi-keyring.gpg'
echo 'deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/' | sudo tee /etc/apt/sources.list.d/jitsi-stable.list > /dev/null
```

#### Ubuntu 22.04

```bash
curl -sL https://download.jitsi.org/jitsi-key.gpg.key | gpg --dearmor | sudo tee /usr/share/keyrings/jitsi-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/" | sudo tee /etc/apt/sources.list.d/jitsi-stable.list
```

* Atualize todas as fontes do pacote `sudo apt update`

#### Instale e configure seu firewall 
As mesmas portas que foram abertas na aws tem que ser abertas dentro do seu servidor jitsi
use esse comando para abrir

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 10000/udp
sudo ufw allow 22/tcp
sudo ufw allow 3478/udp
sudo ufw allow 5349/tcps
sudo ufw allow 5280/tcp
sudo ufw allow 5222/tcp
sudo ufw allow 8888/tcp
sudo ufw enable
```

Verifique o status do firewall com: 

```bash
sudo ufw status verbose
```

## Instale o Jitsi Meet 

```bash
sudo apt install jitsi-meet
```

Ao executar este comando, primeiro será solicitado o nome do domínio.  Basta digitar o endereço DNS do seu servidor:

Em seguida, você será questionado sobre o certificado.  Escolha o certificado autoassinado por enquanto:

### Registro SSL
registre um certificado SSL válido no serviço Let's Encrypt.  É gratuito e bem implementado nas ferramentas Jitsi.  Basta executar o seguinte comando em seu servidor via SSH:

```bash
sudo /usr/share/jitsi-meet/scripts/install-letsencrypt-cert.sh
```

# Ajustando um ambiente Jitsi Meet para o Jibri

### Configurando um ambiente do Jitsi Meet para o Jibri requer a ativação de certas configurações. Isso inclui a configuração de virtualhosts e contas no Prosody, bem como ajustes específicos para a interface web do Jitsi Meet

## Configure o Prosody

### Crie a entrada do componente MUC interno. Isso é necessário para que os clientes do Jibri possam ser descobertos pelo Jicofo em um MUC que não seja acessível externamente pelos usuários do Jitsi Meet

* execute 

```bash
sudo nano /etc/prosody/prosody.cfg.lua
```

* Adicione o seguinte no arquivo

```bash
-- internal muc component, meant to enable pools of jibri and jigasi clients
Component "internal.auth.seudominio.com" "muc"
    modules_enabled = {
      "ping";
    }
    -- storage should be "none" for prosody 0.10 and "memory" for prosody 0.11
    storage = "memory"
    muc_room_cache_size = 1000
```

* quase no final do arquivo na area resevada para virtual host adcione isso

```bash
VirtualHost "recorder.seudominio.com"
  modules_enabled = {
    "ping";
  }
  authentication = "internal_plain"
```

* Salve o arquivo e saia dele

:warning: Certifique-se de substituir seudominio.com pelo domínio adequado do seu ambiente Jitsi Meet.
Essa configuração permitirá que apenas as sessões autenticadas do Jibri Chrome sejam participantes ocultos na conferência que está sendo gravada, fornecendo restrições de acesso adequadas

### Pelo seu terminal Configure as duas contas que o jibri usará.

```bash
prosodyctl register jibri auth.seudominio.com jibriauthpass
prosodyctl register recorder recorder.seudominio.com jibrirecorderpass
```

A primeira conta é aquela que o Jibri usará para fazer login no MUC de controle (onde o Jibri enviará seu status e aguardará os comandos). A segunda conta é aquela que a Jibri usará como cliente no Selenium ao ingressar na chamada para que possa ser tratada de maneira especial pela IU da Web do Jitsi Meet

* De um reload no Jicofo

```bash
sudo systemctl reload jicofo
```

## Configure o Jicofo

### Editar `/etc/jitsi/jicofo/jicofo.conf`, defina o MUC apropriado para procurar os controladores Jibri. Reinicie o Jicofo após definir esta propriedade. Também é sugerido definir o tempo limite pendente para 90 segundos, para permitir que o Jibri tenha algum tempo para inicializar antes de ser marcado como com falha

* execute 

```bash
sudo nano /etc/jitsi/jicofo/jicofo.conf
```

* Adicione o seguinte no arquivo

```bash
jibri {
    brewery-jid = "JibriBrewery@internal.auth.seudominio.com"
    pending-timeout = 90 seconds
  }
```

* ficará algo parecido com isso

![config jicofo](images/jitsi/config-jicofo.png)

## Configure UI o Jitsi meet

### Para habilitar botões de gravações e transmissão ao vivo

* Edite o `/etc/jitsi/meet/seudominio.com-config.js` arquivo, adicione/defina as seguintes propriedades: 

```bash
recordingService: {
  enabled: true,
},

fileRecordingsEnabled: true, // If you want to enable file recording
liveStreamingEnabled: true, // If you want to enable live streaming
hiddenDomain: 'recorder.seudominio.com',
```

* Por fim reinicie o jicofo jitsi e prosody

```bash
sudo /etc/init.d/prosody restart
sudo /etc/init.d/jicofo restart
sudo /etc/init.d/jitsi-videobridge2 restart
```

### Reincie seu servidor `sudo reboot`e sua instacia. Você ja poderá acessa o jitsi na web  pelo seu host

exemplo: `seudominio.com`

# Criação de Máquina EC2 do Jibri

Para criar as máquinas EC2 do Jibri-docker, siga as etapas abaixo:
* Acesse o [Console da AWS](https://us-east-1.console.aws.amazon.com/console/home?region=us-east-1).
* Pesquise por "EC2" na barra de pesquisa do console.
* Selecione a região desejada, no meu caso foi a us-east-1.
* Clique em "executar instâncias", mostrado na imagem abaixo:

![instancia.png](images/jitsi/instancia.png)

* Coloque um nome para a instância no meu caso é o jibri-docker

![nome jibri-dockr](images/jitsi/name-jibri-docker.png)

* escolha uma imagem. Neste guia, usaremos a imagem Ubuntu Server 18.04 Bionic:

![imagem ubuntu](images/jitsi/ami-jibri.png)

* Selecione um tipo de instância t2.xlarge.

* Gere um par de chaves para acessar via SSH.

* Crie um grupo de segurança com intervalo de portas 5280, 443, 22, 5349, 10000, 3478, 80, 5222 e 8888, com origem 0.0.0.0/0. Veja o exemplo na imagem abaixo:

![grupo-port](images/jitsi/grupo-port.png)

* Adicione um IP elástico à instância. Você precisará de um domínio e deverá apontar o DNS para o IP escolhido.

![elastic-ip](images//jitsi/elastic-ip.png)

# Instalação do Jibri-docker

### Para instalação Jibri, usaremos  Guia de auto-hospedagem - Docker, porém utilizando so as imagens e recursos do Jibri  [Self-Hosting Guide - Docker](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-quickstart):

## Pré requisitos:

### Acesse o servidor Jibri via SSH usando a chave criada

* Abra o terminal no seu computador.

* Mude as permissões do arquivo de chave para garantir que apenas você possa acessá-lo. Substitua `jibri.pem` pelo nome do seu arquivo de chave, se necessário.

```bash
chmod 400 jitsi.pem
```

* Conecte-se ao servidor Jibri usando SSH e a chave privada. Substitua `jibri.pem` pelo nome do seu arquivo de chave e `seu_endereco_ip` pelo endereço IP do servidor Jibri.

```bash
ssh -i jibri-docker.pem ubuntu@seu_endereco_ip
```

### Atualizações de pacotes

* No console, escreva o seguinte comando para atualizar seu sistema:

```bash
sudo apt update && sudo apt -y upgrade
```

### Baixe e extraia [latest release](https://github.com/jitsi/docker-jitsi-meet/releases/latest) nesse gui foi utilizado a versao stable-8719: release

#### Execute esse comando para fazer o download do arquivo e extraia no seu servidor jibri

```bash
wget https://github.com/jitsi/docker-jitsi-meet/archive/refs/tags/stable-8719.tar.gz
```

```bash
tar -zxvf stable-8719.tar.gz
```

* Acesse a pasta com 

```bash
cd docker-jitsi-meet-stable-8719  
```

* Crie um `.env` arquivo copiando e ajustando env.example

```bash
cp env.example .env
```
* Seu `.env` tem que se parecer com esse aqui:

```bash
`# Exposed HTTP port`
```
HTTP_PORT=80

# Exposed HTTPS port
HTTPS_PORT=443

# System time zone
TZ=UTC

# Public URL for the web service (required)
PUBLIC_URL=https://seudominio.com
#configurações deyvi
XMPP_BOSH_URL_BASE=http://xmpp.seudominio.com
XMPP_RECORDER_DOMAIN=recorder.seudominio.com
XMPP_AUTH_DOMAIN=auth.seudominio.com
JIBRI_BREWERY_MUC=JibriBrewery
XMPP_INTERNAL_MUC_DOMAIN=internal.auth.seudominio.com
XMPP_DOMAIN=seudominio.com
XMPP_SERVER=seudominio.com


# JaaS Components (beta)
# https://jaas.8x8.vc

# Enable JaaS Components (hosted Jigasi)
# NOTE: if Let's Encrypt is enabled a JaaS account will be automatically created, using the provided email in LETSENCRYPT_EMAIL
#ENABLE_JAAS_COMPONENTS=0

#
# Let's Encrypt configuration
#

# Enable Let's Encrypt certificate generation
# ENABLE_LETSENCRYPT=1

# Domain for which to generate the certificate
LETSENCRYPT_DOMAIN=seudominio.com

# E-Mail for receiving important account notifications (mandatory)
LETSENCRYPT_EMAIL=seuemail@gmail.com

# Use the staging server (for avoiding rate limits while testing)
#LETSENCRYPT_USE_STAGING=1


#
# Etherpad integration (for document sharing)
#

# Set etherpad-lite URL in docker local network (uncomment to enable)
#ETHERPAD_URL_BASE=http://etherpad.meet.jitsi:9001

# Set etherpad-lite public URL, including /p/ pad path fragment (uncomment to enable)
#ETHERPAD_PUBLIC_URL=https://etherpad.my.domain/p/

# Name your etherpad instance!
ETHERPAD_TITLE=Video Chat

# The default text of a pad
ETHERPAD_DEFAULT_PAD_TEXT="Welcome to Web Chat!\n\n"

# Name of the skin for etherpad
ETHERPAD_SKIN_NAME=colibris

# Skin variants for etherpad
ETHERPAD_SKIN_VARIANTS="super-light-toolbar super-light-editor light-background full-width-editor"


#
# Basic Jigasi configuration options (needed for SIP gateway support)
#

# SIP URI for incoming / outgoing calls
#JIGASI_SIP_URI=test@sip2sip.info

# Password for the specified SIP account as a clear text
#JIGASI_SIP_PASSWORD=passw0rd

# SIP server (use the SIP account domain if in doubt)
#JIGASI_SIP_SERVER=sip2sip.info

# SIP server port
#JIGASI_SIP_PORT=5060

# SIP server transport
#JIGASI_SIP_TRANSPORT=UDP


#
# Authentication configuration (see handbook for details)
#

# Enable authentication
#ENABLE_AUTH=1

# Enable guest access
ENABLE_GUESTS=1

# Select authentication type: internal, jwt, ldap or matrix
#AUTH_TYPE=internal

# JWT authentication
#

# Application identifier
#JWT_APP_ID=my_jitsi_app_id

# Application secret known only to your token generator
#JWT_APP_SECRET=my_jitsi_app_secret

# (Optional) Set asap_accepted_issuers as a comma separated list
#JWT_ACCEPTED_ISSUERS=my_web_client,my_app_client

# (Optional) Set asap_accepted_audiences as a comma separated list
#JWT_ACCEPTED_AUDIENCES=my_server1,my_server2

# LDAP authentication (for more information see the Cyrus SASL saslauthd.conf man page)
#

# LDAP url for connection
#LDAP_URL=ldaps://ldap.domain.com/

# LDAP base DN. Can be empty
#LDAP_BASE=DC=example,DC=domain,DC=com

# LDAP user DN. Do not specify this parameter for the anonymous bind
#LDAP_BINDDN=CN=binduser,OU=users,DC=example,DC=domain,DC=com

# LDAP user password. Do not specify this parameter for the anonymous bind
#LDAP_BINDPW=LdapUserPassw0rd

# LDAP filter. Tokens example:
# %1-9 - if the input key is user@mail.domain.com, then %1 is com, %2 is domain and %3 is mail
# %s - %s is replaced by the complete service string
# %r - %r is replaced by the complete realm string
#LDAP_FILTER=(sAMAccountName=%u)

# LDAP authentication method
#LDAP_AUTH_METHOD=bind

# LDAP version
#LDAP_VERSION=3

# LDAP TLS using
#LDAP_USE_TLS=1

# List of SSL/TLS ciphers to allow
#LDAP_TLS_CIPHERS=SECURE256:SECURE128:!AES-128-CBC:!ARCFOUR-128:!CAMELLIA-128-CBC:!3DES-CBC:!CAMELLIA-128-CBC

# Require and verify server certificate
#LDAP_TLS_CHECK_PEER=1

# Path to CA cert file. Used when server certificate verify is enabled
#LDAP_TLS_CACERT_FILE=/etc/ssl/certs/ca-certificates.crt

# Path to CA certs directory. Used when server certificate verify is enabled
#LDAP_TLS_CACERT_DIR=/etc/ssl/certs

# Wether to use starttls, implies LDAPv3 and requires ldap:// instead of ldaps://
# LDAP_START_TLS=1


#
# Security
#
# Set these to strong passwords to avoid intruders from impersonating a service account
# The service(s) won't start unless these are specified
# Running ./gen-passwords.sh will update .env with strong passwords
# You may skip the Jigasi and Jibri passwords if you are not using those
# DO NOT reuse passwords
#

# XMPP password for Jicofo client connections
JICOFO_AUTH_PASSWORD=NixlzFm0x4kJ4SWO

# XMPP password for JVB client connections
JVB_AUTH_PASSWORD=NixlzFm0x4kJ4SWO

# XMPP password for Jigasi MUC client connections
JIGASI_XMPP_PASSWORD=NixlzFm0x4kJ4SWO

# XMPP recorder password for Jibri client connections
JIBRI_RECORDER_PASSWORD=jibrirecorderpass

# XMPP password for Jibri client connections
JIBRI_XMPP_PASSWORD=jibriauthpass

#
# Docker Compose options
#

# Container restart policy
RESTART_POLICY=unless-stopped

# Jitsi image version (useful for local development)
JITSI_IMAGE_VERSION=stable-8719
#jibri recording and livestreaming
ENABLE_RECORDING=1
ENABLE_LIVESTREAMING=1
ENABLE_SUBDOMAINS=1
JIBRI_RECORDING_RESOLUTION=1920x1080
JIBRI_STRIP_DOMAIN_JID=conference
# JIBRI_RECORDING_DIR=/config/recordings
JIBRI_FINALIZE_RECORDING_SCRIPT_PATH="/config/finalize.sh"
DISPLAY=:0=
```

## Crie config  de diretorios do jibri

### nesse diretorio config jibri é aonde deve estar o scritp finalize.sh

* execute essse comando
```bash
mkdir -p ~/.jitsi-meet-cfg/{web,transcripts,prosody/config,prosody/prosody-plugins-custom,jicofo,jvb,jigasi,jibri}
```

## Altere o Dockerfile do Jibri
### Edite o dockerfile para ficar dessa forma aqui para que permita usar comandos aws nos containeres jibri:

```bash
ARG JITSI_REPO=jitsi
ARG BASE_TAG=stable-8719
FROM ${JITSI_REPO}/base-java:${BASE_TAG}

LABEL org.opencontainers.image.title="Jitsi Broadcasting Infrastructure (jibri)"
LABEL org.opencontainers.image.description="Components for recording and/or streaming a conference."
LABEL org.opencontainers.image.url="https://github.com/jitsi/jibri"
LABEL org.opencontainers.image.source="https://github.com/jitsi/docker-jitsi-meet"
LABEL org.opencontainers.image.documentation="https://jitsi.github.io/handbook/"

ARG TARGETPLATFORM
ARG USE_CHROMIUM=0
# ARG CHROME_RELEASE=latest
# ARG CHROMEDRIVER_MAJOR_RELEASE=latest
ARG CHROME_RELEASE=114.0.5735.90
ARG CHROMEDRIVER_MAJOR_RELEASE=114

COPY rootfs/ /

    # Atualiza a lista de pacotes 
RUN apt-get update && apt-get -y install apt-transport-https ca-certificates curl && \
    curl -fsSL https://deb.nodesource.com/setup_14.x | bash -
    # Instalação do Python 3
RUN apt-get update && \
    apt-get install -y python3 && \
    ln -s /usr/bin/python3 /usr/bin/python
# Instalação do AWS CLI
RUN apt-get update && \
    apt-get install -y python3-pip && \
    pip3 install --upgrade awscli

RUN chmod +x /usr/bin/install-chrome.sh

RUN apt-get update && \
    apt-get install -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" jibri libgl1-mesa-dri procps jitsi-upload-integrations jitsi-autoscaler-sidecar jq pulseaudio dbus dbus-x11 rtkit unzip && \
    /usr/bin/install-chrome.sh && \
    apt-cleanup && \
    adduser jibri rtkit


VOLUME /config

```
## Altere o jibri.yml
### Adcione jribi2 e jibri3 e jibri4 respectivamente com seus volumes com isso terá 4 containers:
* O jibri.yml ficarar dessa forma:

```bash
version: '3.5'

services:
    jibri:
        image: my-jibri
        restart: ${RESTART_POLICY:-unless-stopped}
        volumes:
        - ${CONFIG}/jibri:/config:Z
        shm_size: '7gb'
        cap_add:
            - SYS_ADMIN
        environment:
            - AUTOSCALER_SIDECAR_KEY_FILE
            - AUTOSCALER_SIDECAR_KEY_ID
            - AUTOSCALER_SIDECAR_GROUP_NAME
            - AUTOSCALER_SIDECAR_INSTANCE_ID
            - AUTOSCALER_SIDECAR_PORT
            - AUTOSCALER_SIDECAR_REGION
            - AUTOSCALER_URL
            - CHROMIUM_FLAGS
            - DISPLAY=:0
            - ENABLE_STATS_D
            - JIBRI_WEBHOOK_SUBSCRIBERS
            - JIBRI_HTTP_API_EXTERNAL_PORT
            - JIBRI_HTTP_API_INTERNAL_PORT
            - JIBRI_RECORDING_RESOLUTION
            - JIBRI_USAGE_TIMEOUT
            - JIBRI_XMPP_USER
            - JIBRI_XMPP_PASSWORD
            - JIBRI_BREWERY_MUC
            - JIBRI_RECORDER_USER
            - JIBRI_RECORDER_PASSWORD
            - JIBRI_RECORDING_DIR
            - JIBRI_FINALIZE_RECORDING_SCRIPT_PATH
            - JIBRI_STRIP_DOMAIN_JID
            - JIBRI_STATSD_HOST
            - JIBRI_STATSD_PORT
            - LOCAL_ADDRESS
            - PUBLIC_URL
            - TZ
            - XMPP_AUTH_DOMAIN
            - XMPP_DOMAIN
            - XMPP_INTERNAL_MUC_DOMAIN
            - XMPP_MUC_DOMAIN
            - XMPP_RECORDER_DOMAIN
            - XMPP_SERVER
            - XMPP_PORT
            - XMPP_TRUST_ALL_CERTS
    jibri2:
        extends:
            service: jibri
        volumes:
            - ${CONFIG}/jibri2:/config:Z
        environment:
            - JIBRI_ID=jibri2

    jibri3:
        extends:
            service: jibri
        volumes:
            - ${CONFIG}/jibri3:/config:Z
        environment:
            - JIBRI_ID=jibri3
    jibri4:
        extends:
            service: jibri
        volumes:
            - ${CONFIG}/jibri4:/config:Z
        environment:
            - JIBRI_ID=jibri4
volumes:
    jibri1_data:
    jibri2_data:
```
## No diretorio ` ~/.jitsi-meet-cfg/jibri ` crie os diretorios dos jibris
### Irá ficar dessa forma:
![elastic-ip](images//jitsi/diretorio-jibri.png)

## executando o jibri docker:
### Faça a instalação do docker e docker-compose

* Esteja dentro do diretorio raiz `docker-jitsi-meet` e execute esse comando para inciar os containeres:

```bash
docker-compose -f jibri.yml up -d
```

# Configurando Alarme CloudWatch
### Acesse o console da aws e vá em [CloudWatch Alames](https://us-east-1.console.aws.amazon.com/cloudwatch/home?region=us-east-1#alarmsV2:)
* Crie um alarme seguindo essas etapas:
1. crie uma metrica personalizada
2. Adcione esatica maxima,periodo 1 minuto, valor limite = 1
![elastic-ip](images//jitsi/metrica1.png)
![elastic-ip](images//jitsi/metrica2.png)
3. Adicione o nome do alarme e finalize as etapas de criação
# Configurando AWS Auto Scaling Groups
### 
# Script cronjob jitsi

# Script cronjob jibri

# Envio dos videos para AWS S3
