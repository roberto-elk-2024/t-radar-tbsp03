# Inventário Ansible TESP07

Precisamos criar um inventário com os servidores de cada edge, de forma que possamos aplicar as automações nestes servidores baseado em variáveis e configurações personalizadas de cada ambiente.

Como já temos alguns inventários existentes, para deploy de um novo cluster ELK em um Edge, podemos copiar um repositório de inventário existente e ajustar os arquivos conforme a nova estrutura que será criada. 

```
[roberto@oracle8 inventory]$ cp -rda t-radar-tesp03 t-radar-tesp07
```
Usamos o comando rename para renomear os arquivos do diretório host_vars

```
[roberto@oracle8 inventory]$ cd t-radar-tesp07/host_vars/

[roberto@oracle8 host_vars]$ rename tesp03 tesp07 elk-tesp*

[roberto@oracle8 host_vars]$ ls
elk-tesp07-coord01  elk-tesp07-hot03  elk-tesp07-kibana01    elk-tesp07-master01  elk-tesp07-ml01   elk-tesp07-wrm03
elk-tesp07-coord02  elk-tesp07-hot04  elk-tesp07-kibana02    elk-tesp07-master02  elk-tesp07-ml02   elk-tesp07-wrm04
elk-tesp07-hot01    elk-tesp07-hot05  elk-tesp07-logstash01  elk-tesp07-master03  elk-tesp07-wrm01  elk-tesp07-wrm05
elk-tesp07-hot02    elk-tesp07-hot06  elk-tesp07-logstash02  elk-tesp07-mgmt01    elk-tesp07-wrm02  elk-tesp07-wrm06
```

Também devemos ajustar o nome do Host ESX em cada um desses arquivos. Para facilitar, vamos usar o comando sed para ajustar todos de uma única vez.

```
[roberto@oracle8 host_vars]$ sed -i -- 's/TESP3/TESP2/g' *
```
No diretório raiz do inventário, também vamos precisar ajustar os valores do arquivo hosts.yml, substituindo o nome do edge
```
[roberto@oracle8 host_vars]$ cd ..
[roberto@oracle8 t-radar-tesp07]$ sed -i -- 's/tesp03/tesp07/g' hosts.yml 
```
E também a rede do ELK deste edge:
```
[roberto@oracle8 t-radar-tesp07]$ sed -i -- 's/10.100.20/10.118.20/g' hosts.yml 
```
Agora no diretório group_vars precisamos ajustar alguns arquivos.

## all.yml 
No arquivo all.yml precisamos ajustar a variável ssh_allowed_ip com os IPs que devem ter acesso via SSH nestes servidores, como o servidor de gerenciamento e os servidores do cofre.

Exemplo do tesp07:

# Configure Firewall for SSH
ssh_allowed_ip:
  - 10.118.20.100/32 # elk-tesp07-mgmt
  - 10.118.20.104/32 # elk-tesp07-win
  - 172.27.34.101/32 # CYBR-TESP2-PSM1
  - 172.27.34.102/32 # CYBR-TESP2-PSM2    
  - 172.27.34.201/32 # CYBR-TESP2-PSMP01
  - 172.27.34.202/32 # CYBR-TESP2-PSMP02

Nota: Os Ips dos cofres podem ser encontrados nesse link:
https://tdn.totvs.com/display/CN/PAM+-+CyberArk

## cerebro.yml 
No arquivo cerebro.yml precisamos ajustar as variáveis elasticsearch_cluster_name, elasticsearch_url
 e cerebro_allowed_subnet
```
elasticsearch_cluster_name: "t-radar-tesp07"
elasticsearch_url: "https://10.118.20.101:9200"
cerebro_allowed_subnet: 
  - "10.118.20.0/24" # VLAN ELK tesp07
```

## connectors.yml
No arquivo connectors.yml, ajustamos as variáveis totvs_edge e elasticsearch_output.
```
totvs_edge: "tesp07"
elasticsearch_output:
  - "https://10.118.20.121:9200"
  - "https://10.118.20.122:9200"
```
## connectors_vault.yml
Neste arquivo, precisamos ajustar a variável vault_misp_api_token que deve conter a senha recebida da API do MISP para este edge. Ver seção FILEBEAT CONNECTOR neste mesmo documento. 

O comando para editar o arquivos de vault é
```
[roberto@oracle8]$ ansible-vault edit group_vars/connectors/connectors_vault.yml 
```

## elasticsearch.yml
O próximo arquivo para ajustarmos é o elasticsearch.yml onde encontramos as variáveis para deploy do cluster elasticsearch. Este arquivo é o mais extenso de variáveis, porém muitas delas são retro-alimentadas por outras variáveis, visando padronização e facilitar a configuração do cluster. A seguir demonstro as principais variáveis que devem ser adaptadas para cada TOTVS Edge.

* **elasticsearch_cluster_name**: Define o nome do cluster ELK que será criado
* **elasticsearch_ca_node**: Define o note que será usado para gerar os certificados de todos os nós do cluster
* **elasticsearch_ca_keystore**: Nome do certificado da CA do cluster (sendo usado o mesmo arquivo de CA para toda arquitetura distribuída)
* **elastic_transport_allowed_subnets**: IPs e Redes liberadas para conexão na porta de transporte (9300) do cluster
* **elastic_http_allowed_subnets **: IPs e Redes liberadas para conexão na porta da API HTTP (9200) do cluster

Exemplo destas variáveis para o cluster do tesp07:
```
elasticsearch_cluster_name: "t-radar-tesp07"

elasticsearch_ca_node: elk-tesp07-master01

elasticsearch_ca_keystore: t-radar-trd-ca.p12

elastic_transport_allowed_subnets: # 9300
  - "10.108.20.0/24" # VLAN ELK TESP05 - CCS - Arq Distribuida
  - "10.100.20.0/24" # VLAN ELK TESP03 - CCS - Arq Distribuida
  - "10.118.20.0/24" # VLAN ELK tesp07 - CCS - Arq Distribuida
  - "172.27.20.0/24" # ELK TRD - CCS - Arq Distribuida
  
elastic_http_allowed_subnets: # 9200 ## API - Index, Search, Config
  - "10.108.20.0/24" # VLAN ELK TESP05 - CCS - Arq Distribuida 
  - "10.100.20.0/24" # VLAN ELK TESP03 - CCS - Arq Distribuida 
  - "10.118.20.0/24" # VLAN ELK tesp07 - CCS - Arq Distribuida 
  - "172.27.20.0/24" # ELK TRD - CCS - Arq Distribuida 
  - "172.17.144.0/24" # VLAN SLB ELK tesp07
  - "172.17.145.0/24" # VLAN SNAT POOL SLB ELK tesp07
  - "172.18.149.0/24" # VLAN MONITORAMENTO ZABBIX - VLAN CYBERARK TRD
```
## elasticsearch_vault.yml

Neste arquivo, ajustamos as variáveis com conteúdo sensíveis como senhas para o elasticsearch.

Podemos gerar senhas aleatórias via o porta getmypassword da TOTVS https://getmypassword.cloudtotvs.com.br/home.

Escolher opções com 20 caracteres, sem símbolos, pois alguns símbolos geram problema na hora de cadastrar no cofre.

Neste arquivo precisamos ajustar as seguintes variáveis: 

* **vault_internal_users**: Contém as senhas para os usuários built-in do serviço elasticsearch e os componentes princiais como Kibana, beats e logstash. Cada senha deve ser complexa e única. 
* **vault_s3_access_key**: Access Key para acesso ao bucket na AWS
* **vault_s3_secret_key**: Secret Key para acesso ao bucket na AWS
* **vault_api_enrich_tcloudinsights_password**: Senha para integração com a automação do Rod, para enriquecimento dos logs. Uma para cada cluster
* **vault_zbx_password**: Senha para monitoramento via Zabbix. Uma para cada cluster

O comando para editar o arquivos de vault é
```
[roberto@oracle8 t-radar-tesp07]$ ansible-vault edit group_vars/elasticsearch/elasticsearch_vault.yml
```
## kibana.yml
Este arquivo contém as variáveis para implementação do kibana. 

* **kibana_https_cert**: Nome do arquivo de certificado para uso do Kibana - Ver procedimento para gerar certificado na sessão KIBANA neste documento.[ELK] - Implementação novo cluster nos Edges
* **kibana_allowed_subnet**:Lista com endereços de rede que terão acesso ao Kibana
* **elasticsearch_hosts**: Lista dos endereços dos coordinators nodes do cluster sendo implementado
* **server_publicBaseUrl**: Principal URL de acesso ao Kibana


## kibana_vault.yml

Como estamos usando as mesmas senhas para gerenciamento dos certificados dos clusters ELK, em geral não há necessidade de alterar este arquivo. Para analisar se realmente nenhuma variável precisa ser ajustada, executar o seguinte comando:

```
[roberto@oracle8 t-radar-tesp07]$ ansible-vault edit group_vars/kibana/kibana_vault.yml 
```
## logstash.yml

Este é o arquivo com as variáveis para deploy e configuração dos servidores Logstash. As principais variáveis que são alteradas nestes arquivo são:

* **ssl_certificate_authorities_file**: Arquivo com o certificado PEM que será usado para validação da comunicação com os beats. Nesta variável estamos usando o arquivo do certificado piloto para a maioria dos edges (pois os agentes foram implementados antes da arquitetura distribuida) e no caso do TESP05, foi gerado novos certificados durante a implementação do edge. Utilizado arquivo PEM da CA t-radar-trd.pem, atualizado em todos os logstash e nos beats.
* **ssl_certificate_file**: Arquivo certificado gerado para o logstash para comunicação com os Beats
* **ssl_key_file**: Arquivo de chave de certificado para o logstash para comunicação com os Beats
* **elasticsearch_ca_cert**: Certificado da CA do cluster ELK para relação de confiança com o cluster
* **elasticsearch_output**: IPs dos servidores coordinators do cluster ELK
* **totvs_edge**: Indica o edge que será implementado

## logstash_vault.yml
Como estamos usando as mesmas senhas para gerenciamento dos certificados dos clusters ELK, em geral não há necessidade de alterar este arquivo. Para analisar se realmente nenhuma variável precisa ser ajustada, executar o seguinte comando:
```
[roberto@oracle8 t-radar-tesp07]$ ansible-vault edit group_vars/logstash/logstash_vault.yml
```
Após ajustarmos todo o inventário, inicializamos um repositório git para controle destes arquivos e realizamos o primeiro commit.
```
[roberto@oracle8 t-radar-tesp07]$ git init 
[roberto@oracle8 t-radar-tesp07]$ git status
[roberto@oracle8 t-radar-tesp07]$ git add .
[roberto@oracle8 t-radar-tesp07]$ git commit -m "Repositório ELK tesp07 - Arquitetura Distribuida"
```

# Criação repositório Github
Após isso, será necessário criar o repositório no github (atualmente usando a organização totvscloud-seginf).

Criado, configurar remote cluster com os seguintes comandos

```
[roberto@oracle8 t-radar-tesp07]$ git remote add origin git@github.com:totvscloud-seginf/t-radar-tesp07.git
[roberto@oracle8 t-radar-tesp07]$ git branch -M main
[roberto@oracle8 t-radar-tesp07]$ git push -u origin main
```

---

By: Roberto - jose.rsilva

# t-radar-tbsp03
