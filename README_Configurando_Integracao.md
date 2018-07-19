# Nomenclatura dos projetos
Todos os projetos devem seguir um modelo de nomenclatura tanto na estrutura de pastas quanto na estrutura do banco de dados.

## Estrutura de pasta
Por padrão todos os servidores devem conter na estrututra raiz, a pasta de SERVICOS que contém os volumes dos sistemas.


```
/SERVICOS/jenkins
         /postgres_principal/
         /seas_sis_8000/
         /seas_principal_7000/
```

## Estrutura banco de dados
* sisrh
  * database: nome_database
  * schema: nome_schema

* principal
  * database: nome_database
  * schema: nome_schema

# Deploy Automatizado
<img src="http://jenkins-ci.org/sites/default/files/jenkins_logo.png"/>


## 1. Integração Contínua Visão Geral

A estrutura de deploy contínuo é feito através do orquestrador de builds Jenkins, que automatiza todo o processo de construção das imagens 'doquerizadas'. A imagem a baixo mostra como a estrutura foi construída, visando otimizar os deploys CI (Continuous Integration) que envia os deploys para o servidor de produção de forma automática.

![estrutura_deploy](https://user-images.githubusercontent.com/37155369/40973755-bf99e7ee-689b-11e8-8f6c-16c99b9d5c4a.png)

### 1.1 Fluxo Jenkins Build

A cada 20 minutos o container Jenkins faz verificação no git buscando mudança nos branch's. Ao encontrar alguma mudança executa os seguintes passos:

1. Download dos fontes no branch _master_
2. Executa o _bundle install_
3. Executa os migrates
4. Roda os testes _rspec_
5. Remove a imagem _seas_principal_7000_ antiga
6. Cria a imagem _seas_principal_7000_ nova
7. Tagueia a nova imagem
8. Envia (push) a nova imagem para o _Server Regitry_

Após a execução dos passos acima o servidor Registry é atualizado com a nova imagens que fica esperando o cron do servidor passar.

### 1.2 Fluxo CRON no servidor de produção

O servidor de produção executa um crontab que, também, a cada 20 minutos puxa (pull) a nova imagem do servidor Regitry. Este cron executa o processo de derrubar e levantar o container da nova imagem contendo as novas atualizações implementadas no SIGI, que executa os seguintes passos:

1. Puxa _(pull)_ a nova imagem do Regitry
2. Para _(stop)_ o container antigo
3. Remove _(rm)_ o container antigo
4. Roda _(RUN)_ o container novo
5. Executa _(exec)_ os migrates
6. Executa os _(exec)_ assets

Com isto, a nova versão do sistema ja esta em produção, e para verifica-la acesse o link:  [http://sigi.seas.ce.gov.br](http://sigi.seas.ce.gov.br)

## 2. Comandos do Fluxo Jenkins Build
As imagens docker, geralmente são grandes, e geram grandes tráfegos na rede. A ETICE cobra por Giga baixados, porisso é importante otimizar todo o trafego de download de arquivos, e pensando nisso exite um servidor registry que armazena todas as imagens que usamos ao longo do tempo. 

A cada imagem nova baixada é preciso enviar para o servidor de registry. Para saber mais como atualizar leia mais a wiki [Atualizando Servidor Regitry](http://intranet.seas.ce.gov.br/wikiseas/doku.php?id=cgti:infra:docker_registry).

![estrutura__servidor_registry](https://user-images.githubusercontent.com/37155369/40973926-564a089a-689c-11e8-8fbf-f3536e480793.png)

### 2.1 Gerando Imagem Jenkins
O script a baixo cria a imagem do jenkins ja integrado com git e o docker instalado, mas antes de tudo, verifique se a imagem do serviço do jenkins esta presente no servidor de imagens Registry. Todas as imagens devem esta presentes no servidor de Registry que é o servidor de imagem do docker, caso não esteja é necessário manter atualizado para evitar tráfego de downloads, tendo em vista que novas imagens gera tráfego gigantes.

Primeiramente clone o projeto https://github.com/edufilgueira/circle-ci-jenkins-rails.
```
git clone https://github.com/edufilgueira/circle-ci-jenkins-rails.git
```

```
docker build . -t seas-jenkins-build:1.0.0
```

### 2.2 Rodando serviço Jenkins

Execute no primpt:
```
 docker run -u 0 -p 8080:8080 --restart always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v $(pwd)/SERVICOS/jenkins/home:/var/jenkins_home \
-v $(pwd)/SERVICOS/jenkins/backup:/var/jenkins_backup \
--name seas-jenkins-build -d seas-jenkins-build:1.0.0 
```

Após rodar o comando verifique se o container esta ativo acessando link [http://192.168.0.236:8080](http://192.168.0.236:8080)

### 2.3 Volumes do Jenkins

Toda a configuração do jenkins como projetos, usuários e plugins instalados ficam localizados no diretório /var/jenkins_home. Então foi criado um volume especifico para mapear este diretório, de modo que seja fácil duplicá-lo caso seja necessário.

```
-v /SERVICOS/jenkins/home:/var/jenkins_home 
```

_NOTA:_ a instrução $(pwd) mostra o caminho atual do diretório de instalação, ideal para quem deseja criar um volume na mesma pasta dos arquivos dockerfile, mas verifique a localização definida no servidor e altere o caminho quando necessário.

### 2.4 Configurando o Jenkins

1. Crie um Novo Job
2. Informe o nome do Job
3. Escolha a opção projeto free-style
4. Clicke no botão "OK"


![jenkins-001](https://user-images.githubusercontent.com/37155369/40920199-3433dc50-67e2-11e8-8c28-3f0b701a27a1.png)

![jenkins-002](https://user-images.githubusercontent.com/37155369/40920377-beb20fdc-67e2-11e8-8732-d99f8a5b9d93.png)

Na tela de detalhe dos Job informe os seguintes dados:

* Nome do Projeto
* Descrição
* Marque a opção _Descartar builds antigos_
* Marque a opção _"GitHub project"_
  * Informe a url do projeto ex: https://github.com/nonatojunior/principal/
* Em Gerenciamento de código fonte marque o Git
  * Informe o Repository URL Ex: https://github.com/nonatojunior/principal.git
  * Caso projeto seja privado informe suas credenciais em _"ADD"_
  * Especifique o branch a ser monitorado Ex: "*/master"
* Trigger de builds marque a opção "_Consultar periodicamente o SCM_" e digite [*/20 * * * *]. Isso significa que a cada 20 miniutos o jenkins fará uma verificação buscando novas alteração no master.
* Na opção Build selecione a opçãp "_Executar Shell_" e escreva os seguintes comandos:
  * bundle install
  * rails db:create db:migrate
  * rspec
  * echo "Criando a imagem" 
  * docker rmi 192.168.0.1:5000/seas_principal_7000
  * docker build . -t seas_principal_7000
  * docker tag seas_principal_7000 192.168.0.1:5000/seas_principal_7000
  * docker push 192.168.0.1:5000/seas_principal_7000
* Salve e aguarde 20 minutos

![jenkins-003](https://user-images.githubusercontent.com/37155369/40920616-7c55de74-67e3-11e8-9fe7-407a7c102899.png)
## 3. Comandos do Fluxo CRON no servidor de produção

Primeiramente para que este fluxo funcione perfeitamente é necessário já ter as imagens do seas_principal_7000, postgres:9.6 e memcached. Caso o servidor esteja sendo criado pela primeira vez execute o comando a baixo para carregar a imagem do repositório registry:

```
docker pull 192.168.0.1:5000/seas_principal_7000
```

Para rodar o postgres execute o comando:
```
docker pull 192.168.0.1:5000/postgres:9.6
```

```
docker run --name postgres_principal -p 5432:5432 -v /SERVICOS/postgres_principal:/var/lib/postgresql/data -e POSTGRES_PASSWORD=pr89e5p1onjiU -d 192.168.0.1:5000/postgres:9.6
```

Para rodar o memcached execute o comando:
```
docker pull 192.168.0.1:5000/memcached
```

```
docker run -p 11211:11211 --name my-memcache -d 192.168.0.1:5000/memcached
```

### 3.1 Estrutura do Crontab

Execute o comando crontab e adicione a linha de comando a abaixo:

```
* * * * * /sbin/backup_routine
*/20 * * * * /sbin/deploy
```

A cada 20 minutos o servidor executa a atualização da imagem.

#### 3.1.1 Arquivo deploy do sigi

```
#!/bin/bash
echo -n "$(date -R): Checking if we need an update..."

APPNAME=seas_principal_7000

docker pull 192.168.0.1:5000/seas_principal_7000 | grep "Status: Downloaded newer image" && sudo docker stop $APPNAME && sudo docker rm $APPNAME && sudo docker run --name $APPNAME --publish 7000:3000 -v /SERVICOS/seas_principal_7000/database-principal.yml:/usr/src/app/config/database.yml -v /SERVICOS/seas_principal_7000/production.rb:/usr/src/config/environments/production.rb -v /SERVICOS/seas_principal_7000/system/:/usr/src/public/system/ --env RAILS_RELATIVE_URL_ROOT="/sigi" --link my-memcache:memcached --link postgres_principal:psql -d 192.168.0.1:5000/seas_principal_7000 && docker exec $APPNAME rake db:migrate && docker exec $APPNAME rake assets:precompile
```

Obs: Caso o servidor seja uma distribuição debian remova o camondo sudo.

```
#### 3.1.2 Arquivo backup

#!/bin/bash

##### SCRIPT DE BACKUP
# Script de backup automatico para a pasta /home/backup/
# copiando dados fundamentais para o sistema para o disco /dev/sdb1
# montado de maneira sincrona.
#####

# Estrutura de servicos docker do servidor no momento da criacao desse arquivo

#arquivos de build e configuracao dos servicos
rsync -avh /SERVICOS/ /home/backup/SERVICOS/

# Arquivos de fotos do Lychee
#rsync -avh /var/lib/docker/volumes/ff550d454280b1dbf17ef10569b62bb0adb8c1300d67857962b930b27cf08012/_data /home/backup/fotos

# Backup da base principal_producao da app principal 
docker exec postgres_principal pg_dump principal_producao -U seas_erp > /home/backup/bd_backup_principal.sql

# Backup da base servico_producao da app materializerh
docker exec postgres_principal pg_dump servico_producao -U seas_erp > /home/backup/bd_backup_servico.sql

SQL_BACKUP_FILENAME=`date +"%s"`_sqlbackup.tar

tar -cvf /tmp/$SQL_BACKUP_FILENAME /home/backup/*.sql
scp /tmp/$SQL_BACKUP_FILENAME root@172.27.0.71:/backups/from_seas_servers/1921680213/
```
