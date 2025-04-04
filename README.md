
Este projeto foi proposto para que se fosse estudado e aprimorado os conceitos, técnicas e aprendizados desenvolvidos sobre Docker e AWS. Este projeto visa criar uma estrutura em que as EC2 estejam privadas e sendo montadas por um EFS que também estará na subnet privada, utilize um Classic Load Balancer e que faça uso do monitoramento do Cloudwatch que irá monitorar as EC2 criadas pelo Auto Scalling Group.

## Estágios do Projeto:

### 1. Executar o Wordpress Localmente
### 2. Testar o Wordpress via EC2
### 3. Estruturação das conexões via Security Groups
### 4. Estruturação das conexões via Security Groups
### 5. Criação do RDS (Banco de Dados)
### 6. Criação do EFS (Elastic File System)
### 7. Criação do Load Balancer (Classic Load Balancer)
### 8. Criação do Auto Scalling Group
### 9. Criação do Alarme no Cloudwatch
---- 

## Tecnologias visadas para uso neste projeto:

 - Docker
 - Wordpress
 - AWS (EC2, Script User Data, RDS, EFS, Security Group, Auto-Scalling Group, Target Group, Load Balancer)
 - Excalidraw para abstração de diagramas
----
## 1. Executar o Wordpress Localmente

Primeiramente teremos que fazer a instalação do Docker na sua máquina, no caso do Windows faça a instalação do Docker Desktop. No caso do Linux apenas faça a instalação `sudo apt update && sudo apt install docker`

Para executar o container do Wordpress localmente utilizei o WSL, pois utilizo Windows atualmente, porém se tiver uma máquina que tenha um sistema operacional com Kernel Linux a instalação, criação e execução do container se tornam bem mais simplificadas, pois bem, segue o docker compose que utilizei localmente:


```docker-compose.yml
      services:

  wordpress:
    image: wordpress
    restart: always
    ports:
	  - 8080:80
    environment:
      APACHE_SERVER_NAME: localhost
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html
    depends_on:
      - db
    networks:
      - wordpress_network


  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
    command: --default-authentication-plugin=mysql_native_password
    volumes:
      - db:/var/lib/mysql
    networks:
      - wordpress_network

volumes:
  wordpress:
  db:

networks:
  wordpress_network:

```

Este docker-compose cria um container com Wordpress e MySQL, fazendo uso do MySQL pois o Wordpress necessita de um banco de dados para sua execução, fazendo assim a criação conjunta do container com o Wordpress e o MySQL, fazendo a conexão entre eles via variáveis de ambiente MYSQL_DATABASE, MYSQL_USER, MYSQL_PASSWORD. A porta de acesso para a visualização do wordpress é a porta 8080 e as versões de cada um foram escolhidas por convenção, ambos são exemplos tirados da própria documentação do wordpress no docker hub.

Após a criação do docker-compose.yml faça a inserção do comando `docker-compose up` ou no caso de não querer visualizar os logs do docker-compose `docker-compose up -d`.

-----
## 2.Testar o Wordpress via EC2 ;
Para executar o Wordpress via EC2 eu primeiramente criei uma EC2 de maneira simples. Criei uma EC2 com a imagem Ubuntu mais recente e fazendo uso da uma chave .PEM, criei a EC2 utilizando uma t2.micro e fiz a configuração de rede com uma VPC que criei da maneira mais rápida possível, apenas para este teste. 
![Pasted image 20250402134310](https://github.com/user-attachments/assets/6afbcd47-cf91-41b7-b0e4-64ed434b1de0)
![Captura de tela 2025-04-01 095609](https://github.com/user-attachments/assets/eeb4c1cc-bfd1-45a2-9d07-3518cbce7f83)


As configurações finais da VPC ficaram assim:
![Pasted image 20250401094855](https://github.com/user-attachments/assets/2f86aec2-422f-415c-a30a-0534ae350ee0)
Por fim para rodarmos a nossa EC2 iremos usar está VPC, fazendo uso dela colocaremos nossa EC2 na subnet pública para que tenhamos acesso via SSH a configuração ficaria desta forma:
![Pasted image 20250403181637](https://github.com/user-attachments/assets/f90ed87f-65b5-42df-9efc-709ee50a18de)
![Pasted image 20250403181659](https://github.com/user-attachments/assets/3e88d7d3-2e80-4681-81e9-0b43d4df591d)
![Pasted image 20250403181823](https://github.com/user-attachments/assets/b3380597-7518-4941-9456-a40cfaadfd93)
A parte de conectividade é criar um Security Group com regra de entrada SSH configurada para o seu próprio IP, desta forma somente você conseguira acessar a EC2 via SSH e aproveitando você irá configurar um Security Group para acesso via seu IP para acessar a máquina via HTTP.
![Captura de tela 2025-04-03 182258](https://github.com/user-attachments/assets/76a590f3-d372-4d40-a19e-84166cba04ba)

De resto podemos manter as configurações padrão e colocar a instância EC2 para rodar.
Ao iniciar a execução da EC2
![Captura de tela 2025-04-03 182737](https://github.com/user-attachments/assets/3b05937a-4d35-43bc-909a-37870c43a0f6)

Eu apaguei o nome da EC2 mas por questão de segurança apenas.

A conexão com a EC2 sendo feita via SSH apenas se atente em caso de ter utilizado uma chave .PEM ou .KSM e caso tenha feito uso de alguma destas chaves na criação da EC2 certifique-se de estar na mesma pasta que a sua chave SSH se encontra, após fazer isto utilizaremos o comando fornecido pela AWS para conectividade via SSH com nossas EC2s:
![Captura de tela 2025-04-03 183420](https://github.com/user-attachments/assets/5a35c513-2f44-43bc-98ca-1112d40a77cc)

Após a inserção do comando de exemplo faremos o download do docker, mas antes iremos subir as nossas permissões para Super User, portanto usaremos o comando `sudo su` e após isso `apt install docker -y` e após isso iremos puxar o arquivo binário do docker compose utilizando o comando a seguir.
`sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose`
e após a inserção deste comando faremos a adequação das permissões para que o docker-compose possa ser executado `sudo chmod +x /usr/local/bin/docker-compose` e então pegaremos o seguinte docker-compose.yml e copiaremos para dentro de um arquivo com o mesmo nome e usaremos o comando `docker-compose up -d` e após isso acessaremos a página do Wordpress via IP publico da nossa EC2 e assim teremos algo semelhante a isto: 

![Pasted image 20250403184159](https://github.com/user-attachments/assets/1070487d-faa3-49ee-8962-a54fec16a506)

E pronto, testado !!

----
## 3.Criação da VPC
A VPC deste projeto deverá ser criada com um NAT GATEWAY acoplado para que as EC2 na subnet privada tenham conectividade com a internet para fazerem o download de todas as pendencias.
A VPC ficará configurada da seguinte forma:
![image](https://github.com/user-attachments/assets/5d6ddae3-e611-4cdb-b935-4f5193ae5c33)

Tendo 1 Nat Gateway, 1 Internet Gateway e 1 VPC Endpoint.

-----
## 4. Estruturação das conexões via Security Groups

Para a criação da estrutura do wordpress via Docker nós iremos utilizar as Security groups para estruturar a base de comunicação entre as cada uma das partes deste projeto.
![Drawing 2025-04-03 10 57 52 excalidraw](https://github.com/user-attachments/assets/be9510ae-59c4-43a9-bf61-454759d90858)
O diagrama acima mostra todas as conexões de forma simples e detalhada, mas devemos ter ciência de que todas as conexões não são unilaterais, de ambos os lados deve-se ter tais conexões, portanto no CLB-SG deve-se ter uma conexão com o WEBSERVER-SG de saída via HTTP pois as requisições viram do CLB e ele receberá de qualquer IP as requisições tendo configuração `0.0.0.0/0` de entrada via HTTP.
Todos esses Security Groups foram criados dentro da VPC anteriormente criada.

---

## 5. Criação do RDS (Banco de Dados)

Para a criação do RDS iremos ter que nos atentar a algumas coisas, o RDS que utilizaremos será com MySQL, o RDS deverá estar privado sem nenhuma possibilidade de conectividade direta a ele.
O RDS funcionará como o banco de dados das instâncias EC2, fazendo a conexão diretamente pelos Security Groups.
![Pasted image 20250403192717](https://github.com/user-attachments/assets/78129d1a-9a25-43d4-9788-9138621f04aa)
![Pasted image 20250403192803](https://github.com/user-attachments/assets/198a07c4-5649-43b0-b350-c9b5099c1fe0)
![Pasted image 20250403192834](https://github.com/user-attachments/assets/20d3440d-ca52-4c9c-9a36-26578e62e7ea)
**Atente-se a senha e a versão do MySQL**
![Pasted image 20250403192922](https://github.com/user-attachments/assets/fbdfa9a1-2570-4a4a-80b5-b28639ca7a04)
Certifique-se destas configurações estarem feitas pois assim não gerará tantos custos para você ao final do projeto.
![Pasted image 20250403193008](https://github.com/user-attachments/assets/198ac86e-3254-4a36-910d-8477e2beb42d)
Estás configurações também devem ser verificadas para não gerar muitos custos ao final do projeto..
![Pasted image 20250403194346](https://github.com/user-attachments/assets/a6d22f84-2591-430e-8320-544a209c9493)
Estás configurações são de conectividade as quais definimos como a VPC a ser usada será a nossa anteriormente criada e o SG será o RDS-SG, acesso público configurado como **NÃO** por padrão e na zona de acessibilidade definimos como sem preferência.
![Pasted image 20250403210552](https://github.com/user-attachments/assets/ecf50e4d-f360-4dd7-9d08-d2582404c26d)
Está é a ultima configuração que faremos, que é da criação do primeiro Database do RDS, o qual precisamos que seja criado anteriormente para que nosso container apenas tenha que fazer o acesso, pois ele não tem permissão de criação dentro do RDS;

---
## 6. Criação do EFS (Elastic File System)

O EFS irá servir como um ponto de backup para as outras EC2 que serão criadas em caso das anteriores terem que ser terminadas. Ele terá as credenciais do Wordpress juntamente com todas as modificações que já foram anteriormente feitas no Wordpress (posts, atualizações, usuários e por ai vai)
Para a criação dele precisaremos tomar cuidado com algumas coisas para que a montagem do nosso EFS seja bem executada:
![Pasted image 20250403201014](https://github.com/user-attachments/assets/594acb7a-1665-43dd-94c7-0456e221f257)
A partir daqui iremos customizar o nosso EFS, a customização será feita para que possamos determinar algumas configurações que nos permitem customizar.
![Pasted image 20250403201224](https://github.com/user-attachments/assets/250ce93e-ff12-403d-868b-db2becd91ce3)
Configuramos aqui o File System como regional, desativamos o backup automático para menos custos e colocamos nenhum em todos os pontos do Lifecycle Management e na performance colocamos como Bursting para termos um throughput alto.

![Pasted image 20250403201416](https://github.com/user-attachments/assets/84463403-fae4-42da-b8e5-f5376872e5be)
Nesta tela iremos configurar as subnets as quais ficara disponivel a montagem do EFS e alocaremos a EFS-SG para comunicação entre o EFS e o WEBSERVER de maneira adequada.
![Pasted image 20250403201509](https://github.com/user-attachments/assets/e665fdd4-ef29-466c-b419-e3d1ee063024)
Nesta parte podemos deixar em branco mesmo, não é necessário configurar nada aqui, e por fim na próxima parte será de revisão e criação, crie em criar e aguarde alguns momentos para que os pontos de montagem sejam criados adequadamente.
Após os pontos de montagem terem sido devidamente criados, verifique-os clicando no EFS e indo para a aba de Network e ali verificaremos os pontos de montagem.
![Pasted image 20250403210426](https://github.com/user-attachments/assets/dba02809-9aae-4dbe-9539-593a9290d8e7)
Um detalhe importante para guardarmos do EFS é o seu comando de montagem que se localiza na página que se abre após clicar em "Attach".


![Captura de tela 2025-04-03 201842](https://github.com/user-attachments/assets/d33a6bf3-62ea-4aa3-9863-de0f0464f6f8)

Retirei a parte do id do EFS por questão de segurança.

EFS criado, partiremos para a criação do Load Balancer;

---
## 7. Criação do Load Balancer (Classic Load Balancer)
O Load Balancer é quem vai fazer a distribuição das requisições feitas as EC2, substituindo a rota de entrada das EC2 pelo CLB.
A configuração do CLB é a seguinte: 
![Pasted image 20250403210426](https://github.com/user-attachments/assets/203bdab4-ec1a-4fb6-b026-334b91ff8190)

![Pasted image 20250403210451](https://github.com/user-attachments/assets/2f65ac5d-da03-4e91-8866-80e7b98aedda)

![Pasted image 20250403210552](https://github.com/user-attachments/assets/fb29b627-fa2c-4cab-b22e-5177a1a9d72d)

![Pasted image 20250403210615](https://github.com/user-attachments/assets/c1d42d76-d2eb-4e2f-b1c9-3234ae6feebb)

![Pasted image 20250403210635](https://github.com/user-attachments/assets/f3d3ea5b-2418-4b38-9078-dd78c8e0e3d0)

Após estás configurações teremos o CLB criado e pré-configurado para uso no Auto Scalling Group.

---
## 8. Criação do Auto Scalling Group

O auto scalling group é quem fará a criação de outras instâncias EC2 para complementar as que forem sendo desligadas (por quaisquer motivo possível).
Para a criação do ASG utilizaremos um Launch Template. 
O launch template é um modelo que será usado para gerar as EC2 através do ASG.
Para criar ele faremos como se estivéssemos criando uma EC2, escolheremos a AMI:
![Pasted image 20250403215013](https://github.com/user-attachments/assets/c9bfb692-6c07-4e4c-827b-61706a222253)
Configurações como o tipo da instância, chave de acesso SSH, conexão de VPC e subnets:
![Pasted image 20250403215122](https://github.com/user-attachments/assets/ec54f2b4-6e24-4e20-a036-d122680d49d2)
e já deixaremos pronto também a configuração do nosso User Data. O User Data é o script que vai ser rodado durante a inicialização da EC2, para configurar as EC2 devidamente configuradas eu preferi criar um User Data e testa-lo até ter a certeza de que eu conseguiria colocar uma EC2 rodar com ele e funcionar excelentemente bem, pois eu não teria acesso a EC2 portanto o único contato que eu deveria ter seria na criação e atualização do launch template e do User Data.
Segue o User Data criado:
```
#!/bin/bash

sudo yum update -y
sudo yum install -y docker wget amazon-efs-utils

sudo service docker start
sudo systemctl enable docker.service
sudo usermod -aG docker ec2-user

sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

sudo mkdir -p /wordpress
sudo mount -t efs -o tls [id do seu EFS]/ /wordpress


wget -O /home/ec2-user/docker-compose.yml [URL do seu docker-compose.yml RAW]
sudo chown ec2-user:ec2-user /home/ec2-user/docker-compose.yml

cd /home/ec2-user
sudo docker-compose up -d
```
Após finalizada a criação do launch template, selecionaremos o launch template criado no ASG:
![Pasted image 20250403215649](https://github.com/user-attachments/assets/1bef9cdb-93e6-46ce-a6df-c3e242d692e3)
Escolhemos a VPC que temos usado desde o começo e colocam os no ASG e definimos duas zonas de disponibilidade para que sejam usadas para alocar as EC2 criadas (**OBS** as subnets devem ser privadas).
![Pasted image 20250403220102](https://github.com/user-attachments/assets/7c570536-4229-4466-a210-4a9fb8c1daaf)
Aqui clicamos em escolher um Load Balancer existente e depois clicamos em Classic Load Balancers e escolhemos o CLB que criamos.
![Pasted image 20250403220347](https://github.com/user-attachments/assets/9f911052-359c-48b1-9ccb-0f07a1b5333a)
Selecionamos apenas está opção, para habilitar o health check do Elastic Load Balancer e configuramos o período de ativação do Health Check.
![Pasted image 20250403221211](https://github.com/user-attachments/assets/bd895204-49d4-4a81-af4e-9c516d272353)
Configuramos o tamanho desejado do grupo para 2, o tamanho mínimo para 2 e máximo para 4.
![Pasted image 20250403221253](https://github.com/user-attachments/assets/1485c60e-e31f-4fb6-a6ab-1e5f3a87e867)

Nesta parte apenas habilitamos o monitoramento por parte do CloudWatch.
As próximas partes apenas clique next e next pois não são relevantes para o projeto em si, mas se preferir criar uma notificação para o acionamento do ASG ou tags para organização do seu ASG.
Após o ASG criado poderemos fazer o acesso ao Wordpress atráves do DNS do Load Balancer, podendo encontrar a seguinte página.
![Pasted image 20250403221640](https://github.com/user-attachments/assets/9230caa2-f7c8-40a1-85e7-d69b0ccc9b74)
E logo em seguida teremos a seguinte tela:
![wpsetup](https://github.com/user-attachments/assets/0fc3b500-57eb-434d-a57f-cd61cb4d7911)
E por fim temos a seguinte tela, logo após a tela de confirmação de criação do usuário:
![Pasted image 20250403222030](https://github.com/user-attachments/assets/21b19a1e-dd5a-420b-bec1-7fd1232ab87b)
Ao entrar teremos a seguinte tela:
![Pasted image 20250403222220](https://github.com/user-attachments/assets/b155d43a-ae61-4f9a-b746-4bb8e5f8a81e)

-----
## 9. Criação do Alarme no Cloudwatch
Para a criação do Alarme do Cloudwatch precisaremos fazer algumas alterações primeiro. Antes de tudo teremos que criar uma politica de escalonamento simples ou passo a passo no ASG.
![image](https://github.com/user-attachments/assets/f005904d-2bb1-4941-89f8-bde7ed35e88d)
E então criamos uma métrica para a qual o alarme usará para poder fazer o monitoramento:
![image](https://github.com/user-attachments/assets/7d9e4c51-c7d4-4980-b4d3-8f9714a847e3)
![image](https://github.com/user-attachments/assets/6c1673be-decc-4c2c-a8a1-dcacf72fc4fe)
![image](https://github.com/user-attachments/assets/12cb8165-08a6-4bbf-abd0-23bb7c927e3f)
![image](https://github.com/user-attachments/assets/b06778fc-eebb-44c6-b479-704babfaee8f)
![image](https://github.com/user-attachments/assets/57eced02-5a83-48ee-881e-8f94cc23ff66)
![image](https://github.com/user-attachments/assets/3df288ae-ca57-4882-ad50-dfa4a3a8d0b0)
![image](https://github.com/user-attachments/assets/c02d4585-c5a6-4160-a7ef-0e93607c7326)
E a ultima parte dessas configurações é apenas uma revisão, após a configuração podemos verificar a criação do alarme na seguinte tela:
![image](https://github.com/user-attachments/assets/4699648f-e676-4fb7-aeb5-c819820975c3)
E assim finalizamos o projeto, muito obrigado!




