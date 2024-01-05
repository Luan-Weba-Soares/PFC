# PFC
Documentação Passo a Passo para instalar softwares realizados na prática no PFC

### Passo a Passo instalação Greenbone

#### Passo 1) Instalar GVM

```shell
# Faz update e upgrade dos pacotes
sudo apt-get update
sudo apt-get upgrade
#Instala o gvm
sudo apt-get install gvm
# Faz download do banco de dados
sudo gvm-setup
# Força sincronização com banco de dados
sudo gvm-feed-update
# Script que checa se a instalação foi bem sucedida
sudo gvm-check-setup
# Script que inicia o gvm
sudo gvm-start
# Comando nmap para criar um arquivo com todos IPs de uma rede doméstica e salvar em um arquivo
nmap -sP 192.168.1.0/24 | awk '/is up/ {print up}; {gsub(/\(|\)/,""); up = $NF}' > network
```
### Passo a Passo instalação ELK

#### Passo 1: Instalar Elastic

```shell

# Importar a chave GPG e converte essa chave em apt
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch |sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
# Source para o APT buscar nova origem, no caso a origem será versão 8 do elastic
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
# Agora baixar os pacotes vindo da source do elastic para a biblioteca do SO
sudo apt update

# instalar elastic, no final da instalação elastic vai gerar um usuário e senha, salve esse usuário pois ele é o root
sudo apt install elasticsearch

# Altere configuração do elastic para localhost
sudo vi /etc/elasticsearch/elasticsearch.yml
	network.host: localhost
# Iniciar e habilitar elastic
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch

# Teste conexão com elastic
curl -X GET -k https://elastic:[SENHA_ELASTIC]@localhost:9200

# Deve aparecer algo como abaixo, se aparecer significa que elastic está rodando como deveria

{
  "name" : "Elasticsearch",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "n8Qu5CjWSmyIXBzRXK-j4A",
  "version" : {
    "number" : "8.4.3",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "de7261de50d90919ae53b0eff9413fd7e5307301",
    "build_date" : "2022-03-28T15:12:21.446567561Z",
    "build_snapshot" : false,
    "lucene_version" : "9.3.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
} 

```

#### Passo 2: Instalar Kibana

```shell
# instalar kibana
sudo apt install kibana
# Criar um token do elastic para o kibana, para realizar autenticação entre os dois serviços
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
# Adiciona o token criado no input do comando abaixo
sudo /usr/share/kibana/bin/kibana-setup
# Inicie e Habilite Kibana
systemctl start kibana
systemctl enable kibana
# Ao iniciar, espere um pouco e cheque se a porta 5601 está aberta com o comando
ss -lntp
```

#### Passo 3: Instalar Fleet Server

1) Entre com a url http://IP_KIBANA:5601
2) Logue no kibana com o usuário e senha do elastic
3) Vá nas 3 barras a esquerda -> Management -> Integration -> procure por Fleet Server
4) Adicione a integração, pode deixar default as configurações do nome (pode mudar se quiser)
5) Salve e continue
6) Adicione o servidor fleet
7) coloque nome(que voce quiser) e a url para comunicação, a url deve ser algo como https://IP_OU_HOSTNAME:8220
8) Gere o token
9) Instale fleet 
```shell
# Após configurar Fleet, rode o comando gerado pela web para instalar o fleet no servidor, exemplo temos o comando:
elastic-agent enroll --fleet-server-es=https://kali:9200 --fleet-server-service-token=AAEAAWVsYXN0aWMvZmxlZXQtc2VydmVyL3Rva2VuLTE3MDAzNDc0MzM1Mzc6TjVodmZ4MzBUVnVZX2o0SjZPNjR5Zw --fleet-server-policy=fleet-server-policy --fleet-server-es-ca-trusted-fingerprint=a820111acc72ce87d118468c8434c812d49c717f0343ea0f3a97b236dee2e5b7 --fleet-server-port=8220
```


#### Passo 4: Instalar Agent

1) Vá nas 3 barras a esquerda -> Management -> Fleet -> Agents -> Add Agent
2) Crie uma nova Policy (Ex: Policy 2)
3) Instale o agent no devido sistema operacional

#### Windows

##### Adicionando nome do servidor no hosts do windows

	Elastic funciona melhor quando resolve nome dns do servidor, para a workstation resolver o nome do servidor, edite o arquivo C://Windows/System32/Drivers/etc/hosts
adicione
```shell 
	IP_SERVIDOR NOME_SERVIDOR
```
	Teste fazendo um ping para o servidor usando o nome ao invés do IP

##### Instalando o Agent
```shell
#Para Windows, use as seguintes flags, mas use os mesmos valores que está no seu web
elastic-agent enroll --fleet-server-es=https://NOME_SERVIDOR:9200 --fleet-server-service-token=AAEAAWVsYXN0aWMvZmxlZXQtc2VydmVyL3Rva2VuLTE3MDAzNDc0MzM1Mzc6TjVodmZ4MzBUVnVZX2o0SjZPNjR5Zw --fleet-server-policy=fleet-server-policy --fleet-server-es-ca-trusted-fingerprint=a820111acc72ce87d118468c8434c812d49c717f0343ea0f3a97b236dee2e5b7 --fleet-server-port=8220 --insecure
```

#### Passo 5: Integrações

##### Endpoint and Cloud Security

1) Vá nas 3 barras a esquerda -> Management -> Integration -> Endpoint and Cloud Security
2) Adicione ela na policy 2

#### Windows

1) Vá nas 3 barras a esquerda -> Management -> Integration -> Windows
2) Add Windows
3) Vá em advanced e cheque se "sysmon logs" está habiitado
4) Adicione ela na policy 2
5) Save and deploy changes


#### Passo 6: Importando Alertas

Obs: Recomendável instalar a integração Endpoint and Cloud Security para habilitar mais alertas

1) Vá nas 3 barras a esquerda -> Security -> Manage -> Rules
2) Se aparecer erro de chave de integração API é necessário:
	1) Entre no servidor e faça os seguintes comandos

```shell
sudo /usr/share/kibana/bin/kibana-encryption-keys generate
# Copie as 3 chaves, devem ter nomes
xpack.encryptedSavedObjects.encryptionKey: [CHAVE]
xpack.reporting.encriptionKey:[CHAVE]
xpack.security.encryptionKey:[CHAVE]

#Copie as 3 chaves para o arquivo de configuração do kibana (etc/kibana/kibana.yml) e reinicie o kibana
```

3) Após serviço reinicar, faça o passo 1
4) Clique em "Load Elastic prebuilt rules and timeline templates"


#### Passo 7: Checando Logs e alertas Alertas

1) Vá nas 3 barras a esquerda -> Analytics-> Discover
	1) Se aparecer logs significa que está coletando com sucesso
2) Vá nas 3 barras a esquerda -> Security -> Alerts
	1) Se aparecer algum alerta, significa que está funcionando corretamente
