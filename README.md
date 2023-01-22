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
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/0407fb0fd4933b28c66180b921a661388aeb6bef/img/passo02.1.png" width="800" height="300"/>|

  
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

-- ENCONTE A LINHA: </br>

storage: </br>
  dbPath: /Dados/MongoDB/mongodb </br>

||
|---|
|<img src="https://github.com/maicondevops/criar-cluster-mongodb-replicaset/blob/0407fb0fd4933b28c66180b921a661388aeb6bef/img/passo03.1.png" width="800" height="300"/>|

-- AJUSTAR PEMISSÕES NO DIRETÓRIO DE DADOS DO MONGO: </br>

```
sudo chmod chown mongodb:mongodb /Dados/MongoDB/mongodb
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

-- ADICIONE A AS LINHAS: </br>
replication: </br>
  replSetName: rs0 </br>

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
sudo echo "24.145.54.110 mdb01.mydomain.com" >> /etc/hosts
sudo echo "3.101.87.5 mdb02.mydomain.com" >> /etc/hosts
sudo echo "19.786.35.28 mdb03.mydomain.com" >> /etc/hosts
```

OBSERVAÇÃO:

1 - ESSES IPS SÃO FICTICIOS. </br>
2 - VOCÊ PRECISA CRIAR OS MESMOS REGISTROS NO SEU DNS EXTERNO, CASO SEU MONGO SEJA ACESSADO FORA DA SUA REDE LOCAL. </br>

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
 { </br>
 user: "mongoadmin", 
 pwd: "321321",</br>
 roles: [ "userAdminAnyDatabase",
          "dbAdminAnyDatabase", 
          "readWriteAnyDatabase"]
 } 
) 

```
-- APÓS CONFIGURE O REPLICASET: </br>

```
rs.initiate() 
var cfg = rs.conf() 
cfg.members[0].host="mdb01.mydomain.com:27017" 
rs.reconfig(cfg) 
```

```
rs.add("mdb02.mydomain.com:27017")
rs.add("mdb03.mydomain.com:27017")
```

-- VALIDAR: </br>

```
rs.status()
```

------------------------------------------------------------------------------------
--- 07 - TESTAR LOGON NO CLUSTER
------------------------------------------------------------------------------------

```
mongodb://mdb01.mydomaincom:27017,mdb02.mydomain.com:27017,mdb03.mydomain.com:27017/?replicaSet=rs0 
```

------------------------------------------------------------------------------------
--- 08 - GERAR KEY E ATIVAR AUTHORIZATION
------------------------------------------------------------------------------------
-- EXECUTAR NO MASTER: </br>

```
sudo bash -c "openssl rand -base64 756 > /Dados/MongoDB/mongodb/mongodb-key" && sudo chmod 600 /Dados/MongoDB/mongodb/mongodb-key && sudo chown mongodb:mongodb /Dados/MongoDB/mongodb/mongodb-key 
```

-- COPIAR A HASH DO MASTER: </br>

```
sudo cat /Dados/MongoDB/mongodb/mongodb-key
```

-- CRIAR O FILE PARA KEY E COLAR A HASH COPIADA DO MASTER: </br>

```
sudo nano /Dados/MongoDB/mongodb/mongodb-key
```

-- AJUSTAR PERMISSÕES: </br>

```
sudo chmod 600 /Dados/MongoDB/mongodb/mongodb-key && sudo chown mongodb:mongodb /Dados/MongoDB/mongodb/mongodb-key
```

-- ATIVAR AUTHORIZATION EM TODOS AS MÁQUINAS: </br>

-- EDITAR: : </br>

```
sudo nano /etc/mongod.conf
```

-- ADICIONE A AS LINHAS: </br>   
security:
   authorization: enabled
   keyFile: /Dados/MongoDB/mongodb/mongodb.key

-- REINICIAR O SERVIÇO: </br>

```
sudo service mongod restart && sudo systemctl restart mongod && sudo systemctl status mongod
```


------------------------------------------------------------------------------------
--- 08 - TESTAR LOGON NO CLUSTER COM AUTHORIZATION
------------------------------------------------------------------------------------

```
mongodb://mongoadmin:321321@mdb01.mydomaincom:27017,mdb02.mydomain.com:27017,mdb03.mydomain.com:27017/?replicaSet=rs0
```




