# Criando Cluster Mongodb / ReplicaSet

------------------------------------------------------------------------------------
--- 00 - MEU AMBIENTE E OBSERVAÇÕES
------------------------------------------------------------------------------------

* 3 VMs com Ubuntu Server 20.04. </br>
* O replicaSet precisa de pelo menos 3 máquinas para operar, uma sendo o Master e dois Nodes. </br>
* Cada VM possui uma partição secundária de Dados de 1 TB (/Dados). </br>
* MongoDb 4.4. </br>
* Meu diretório Path do Mongo será migrado para a partição de Dados. </br>
* Testado usando Vagrant e no Azure Virtual Machines. </br>
* OS PASSO 1, 2, 3, 4, 5 e 8, VOCÊ PRECISA REALIZAR NAS 3 MÁQUINAS. </br>

------------------------------------------------------------------------------------
--- 01 - CRIAR DIRETÓRIOS DADOS NO DISCO SECUNDÁRIO
------------------------------------------------------------------------------------

```
cd /Dados
sudo mkdir MongoDB
cd MongoDB
sudo mkdir mongodb 
``` 
</br>

------------------------------------------------------------------------------------
--- 02 - INSTALAR O MONGO DB 4.4
------------------------------------------------------------------------------------

```
sudo curl -fsSL https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add - && 
sudo echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list && 
sudo apt update -y && 
sudo apt install mongodb-org -y 
```
</br>

-- INICIAR SERVIÇO E VALIDAR INSTALAÇÃO: </br>

```
sudo systemctl start mongod.service && sudo systemctl status mongod && sudo systemctl enable mongod
```
||
|---|
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/1eff1757f6fec0bf78f94e20c21d208501f9b577/img/passo02.1.png" width="700" height="200"/>|

------------------------------------------------------------------------------------
--- 03 - ALTERAR O DIRETÓRIO DE DADOS DO MONGO
------------------------------------------------------------------------------------

```
sudo service mongod stop
```

-- ALTERAR O PATH DE DADOS, NO ARQUIVO DE CONFIGURAÇÃO DO MONGO: </br>

```
sudo nano /etc/mongod.conf
```

-- ENCONTE A LINHA e ALTERE O PATH: </br>

```
storage:
  dbPath: /Dados/MongoDB/mongodb
```

||
|---|
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/6908f188bf24df00813149c22bbc5d83530a6d60/img/passo03.1.png" width="700" height="200"/>|

-- AJUSTAR PEMISSÕES NO NOVO DIRETÓRIO DE DADOS: </br>

```
sudo chmod 600 -R /Dados/MongoDB/mongodb && sudo chown mongodb:mongodb /Dados/MongoDB/mongodb/*
```

-- REINICIAR O SERVIÇO: </br>

```
sudo service mongod restart && sudo service mongod status
```

------------------------------------------------------------------------------------
--- 04 - CONFIGURAR REPLICASET NO ARQUIVO DE CONFIGURAÇÃO DO MONGO
------------------------------------------------------------------------------------
-- EDITAR O ARQUIVO DE CONFIGURAÇÃO:

```
sudo service mongod stop && sudo nano /etc/mongod.conf
```

-- ADICIONE AS LINHAS: </br>

```
replication:
  replSetName: rs0
```
  
||
|---|
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/6908f188bf24df00813149c22bbc5d83530a6d60/img/passo04.1.png" width="700" height="200"/>|

-- REINICIAR O SERVIÇO: </br>

```
sudo service mongod restart && sudo service mongod status
```

------------------------------------------------------------------------------------
--- 05 - AJUSTAR HOSTNAME E FILE HOSTS DAS MÁQUINAS
------------------------------------------------------------------------------------
-- VM MASTER: </br>

```
sudo echo "mdb01.mydomain.com" > /etc/hostname
```  

-- VM SLAVE 01: </br>

```
sudo echo "mdb02.mydomain.com" > /etc/hostname
```

-- VM SLAVE 02: </br>

```
sudo echo "mdb03.mydomain.com" > /etc/hostname
```

-- EM TODAS A VMs: </br>

```
sudo echo "192.168.50.10 mdb01.mydomain.com" >> /etc/hosts
sudo echo "192.168.50.11 mdb02.mydomain.com" >> /etc/hosts
sudo echo "192.168.50.12 mdb03.mydomain.com" >> /etc/hosts
```

-- AJUSTAR O BIND IP EM TODAS AS VMs (LIBERA O ACESSO EXTERNO AO MONGO): </br>

```
sudo service mongod stop && sudo nano /etc/mongod.conf
```

||
|---|
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/726dd5e977630b5ddcf74fe41c80baf31c5509e9/img/passo05.1.png" width="700" height="200"/>|

```
sudo service mongod start
```

OBSERVAÇÃO:

1 - SE O ACESSO AO MONGO FOR EXTERNO, VOCÊ PRECISA CRIAR OS REGISTROS NO SEU DNS EXTERNO. </br>

------------------------------------------------------------------------------------
--- 06 - ATIVAR REPLICASET E INGRESSAR NODES
------------------------------------------------------------------------------------
-- EXECUTAR NO MASTER: </br>

```
mongo
```

-- CRIAR UM USUÁRIO ADMINISTRADOR </br>

```
db.createUser(
 {
 user: "mongoadmin",
 pwd: "mongoadmin",
 roles: [ "userAdminAnyDatabase",
          "dbAdminAnyDatabase",
          "readWriteAnyDatabase"]
 }
)

```
-- ATIVAR O REPLICASET: </br>

```
rs.initiate() 
var cfg = rs.conf() 
cfg.members[0].host="mdb01.mydomain.com:27017" 
rs.reconfig(cfg) 
```

-- ADICIONAR NODES: </br>

```
rs.add("mdb02.mydomain.com:27017")
rs.add("mdb03.mydomain.com:27017")
```

-- VALIDAR: </br>

```
rs.status()
```

||
|---|
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/6908f188bf24df00813149c22bbc5d83530a6d60/img/passo06.1.png" width="700" height="400"/>|

------------------------------------------------------------------------------------
--- 07 - TESTAR LOGON NO CLUSTER
------------------------------------------------------------------------------------

```
mongodb://mdb01.mydomaincom:27017,mdb02.mydomain.com:27017,mdb03.mydomain.com:27017/?replicaSet=rs0 
```

||
|---|
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/6908f188bf24df00813149c22bbc5d83530a6d60/img/passo07.1.png" width="700" height="200"/>|

------------------------------------------------------------------------------------
--- 08 - GERAR KEY E ATIVAR AUTHORIZATION
------------------------------------------------------------------------------------
-- EXECUTAR APENAS NO MASTER: </br>

```
sudo bash -c "openssl rand -base64 756 > /Dados/MongoDB/mongodb/mongodb-key" && sudo chmod 600 /Dados/MongoDB/mongodb/mongodb-key && sudo chown mongodb:mongodb /Dados/MongoDB/mongodb/mongodb-key 
```

-- COPIAR A HASH DO MASTER: </br>

```
sudo cat /Dados/MongoDB/mongodb/mongodb-key
```

||
|---|
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/6908f188bf24df00813149c22bbc5d83530a6d60/img/passo08.1.png" width="700" height="200"/>|

-- CRIAR O FILE PARA KEY E COLAR A HASH COPIADA DO MASTER: </br>

```
sudo nano /Dados/MongoDB/mongodb/mongodb-key
```

-- AJUSTAR PERMISSÕES: </br>

```
sudo chmod 600 /Dados/MongoDB/mongodb/mongodb-key && sudo chown mongodb:mongodb /Dados/MongoDB/mongodb/mongodb-key
```

-- ATIVAR AUTHORIZATION EM TODAS AS MÁQUINAS: </br>

```
sudo nano /etc/mongod.conf
```

-- ADICIONE A AS LINHAS (EM TODAS AS VMs): </br>   

```
security:
   authorization: enabled
   keyFile: /Dados/MongoDB/mongodb/mongodb.key
```
   
||
|---|
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/6908f188bf24df00813149c22bbc5d83530a6d60/img/passo08.2.png" width="700" height="200"/>|

-- REINICIAR O SERVIÇO: </br>

```
sudo service mongod restart && sudo systemctl restart mongod && sudo systemctl status mongod
```

------------------------------------------------------------------------------------
--- 09 - TESTAR LOGON NO CLUSTER COM AUTHORIZATION
------------------------------------------------------------------------------------

```
mongodb://mongoadmin:mongoadmin@mdb01.mydomaincom:27017,mdb02.mydomain.com:27017,mdb03.mydomain.com:27017/?replicaSet=rs0
```

||
|---|
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/6908f188bf24df00813149c22bbc5d83530a6d60/img/passo09.1.png" width="700" height="200"/>|

------------------------------------------------------------------------------------
--- 10 - POSSÍVEIS ERROS E DICAS
------------------------------------------------------------------------------------

1 - Erro de permissão no diretório "/Dados/MongoDB/mongodb".
2 - Erro de permissão no arquivo "/Dados/MongoDB/mongodb/mongodb.key".
3 - Erro no acesso externo da VM, editar BindIp no arquivo "/etc/mongod.conf".
4 - Sempre leia o arquivo de log "/var/log/mongodb/mongod.log". A resposta para a maioria dos problemas estará nele.


