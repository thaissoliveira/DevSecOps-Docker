# Projeto DevSecOps - Docker e WordPress na AWS

A atividade proposta envolve configurar e implementar uma aplicação WordPress na AWS usando máquinas EC2, com boas práticas de infraestrutura e automação! Deve-se, ainda, seguir a estrutura de topologia listada abaixo:

![Captura de tela 2024-12-23 130350](https://github.com/user-attachments/assets/110fe7d6-1bdb-451d-bb73-a5040ea932aa)

### Requisitos do projeto:
(1). Configurar a instância EC2 para que ela suporte contêineres, instalando o Docker ou o Containerd. Este requisito inclui baixar e instalar o Docker ou Containerd, configurar o serviço para ser iniciado automaticamente e garantir que o usuário tenha permissão para usá-lo, normalmente, adicionando o usuário ao grupo apropriado.
   
(2). Utilizar a instalação via script de Start Instance (user_data.sh), o que permite que você execute scripts automaticamente assim que a instância é inicializada.
   
(3). Realizar o deploy da aplicação WordPress utilizando contêineres e o serviço RDS da AWS para o banco de dados.
   
(4). Configuração da utilização do serviço EFS AWS para estáticos do container de aplicação Wordpress.
   
(5). Configurar um Load Balancer da AWS para a aplicação do WordPress, distribuindo, então, o tráfego de rede de forma eficiente entre várias instâncias ou contêineres que estejam executando o WordPress.

# Criando uma VPC (Virtual Private Cloud)

Primeiro, é preciso criar uma VPC na AWS para hospedar a infraestrutura do projeto. Para isso, definimos o CIDR, que são os endereços Ip's que podem ser usados na VPC, a quantidade de sub-redes menores e suas respectivas conexões com a internet.

![image](https://github.com/user-attachments/assets/ff138793-99fb-4dca-8ba1-8b18661bc9af)

A VPC criada para o projeto possui a seguinte estrutura:

- Duas zonas de disponilidade
- **Sub-redes:** Duas públicas e duas privadas
- **CIDR IPv4 da VPC:** 10.0.0.0/16
- **Internet Gateway:** Associado às sub-redes públicas

| Sub-redes  |  Endereços IPV4  |
| ---------- | ----------- |
| Pública us-east-1a | 10.0.0.0/20   |
| Privada us-east-1a | 10.0.128.0/20 |
| Pública us-east-1b | 10.0.16.0/20  |
| Privada us-east-1b | 10.0.144.0/20 |        

| Tabelas de Rotas |
|---------|
| Sub-redes públicas roteando para o IGW. |
| Sub-redes privadas inicialmente isoladas. |

Cada sub-rede está associada a uma zona de disponibilidade (AZ) para distribuir a infraestrutura geograficamente dentro da região AWS, aumentando a resiliência da aplicação.

# Criando os Security Groups

### Grupo de segurança para redes públicas:

![image](https://github.com/user-attachments/assets/0ecbb1e7-83ea-4a59-8d48-d80d2998ec0d)

![image](https://github.com/user-attachments/assets/c26bf811-975f-4dab-ac43-6b6db58fd560)

### Grupo de segurança para redes privadas:

Aqui é importante notar que para o HHTP (porta 80) e o HTTPS (443) tem origem no security group público.

![image](https://github.com/user-attachments/assets/66eebb59-25a1-4ed9-97bb-5a7949db12a1)

![image](https://github.com/user-attachments/assets/73f75630-0aaf-44a6-9083-972c17291c36)

### Grupo de segurança do RDS

# Criando o Elactic File System (EFS)

Para criar o EFS deste projeto, usa-se as seguintes configurações:

### Etapa 1:

- Defina um nome para seu EFS
- Escolha o tipo de sistema como sendo ``Regional``
- Modo de taxa de transferência avançado

### Etapa 2:

- Escolha a VPC do projeto
- Defina as zonas de disponibilidade do EFS
- Nas zonas de disponibilidade, escolhemos as zonas em que estão as instâncias EC2, que no caso são as zonas ``us-east-1a`` e ``us-east-1b``. Além disso, para as duas zonas, selecionamos o security group que criamos para redes privadas.
  
![image](https://github.com/user-attachments/assets/514b93bf-4c3a-4fe2-921c-605eab4d3442)

⚠️ Como este é apenas um projeto para fins de aprendizado, pode-se desabilitar as opções de backup e criptografia, dessa maneira, economizamos recursos de custos.

# Criando uma Sub-Rede Privada para o RDS


# Criando o Relational Database Service (RDS)

Ao criar o RDS, passamos pelas seguintes configurações:

- Método de criação de banco como sendo ``Standard create``
- Opção de motor ``MySQL`` e versão ``MySQL 8.0.39``
- Template ``Free Tier``
- Inserir um nome único da instância do banco de dados
- Digitar um ID de login para o usuário mestre da instância
- Gerenciamento de credenciais como sendo ``Self managed``
- Especificar uma senha para o usuário mestre
- O tupo da inatância RDS deve ser ``db.t3.micro``
- Em conectividade:
  - Vamos manter a opção ``Don’t connect to an EC2 compute resource``
  - Logo em seguida, escolha a VPC do projeto
  - Selecione a sub-rede privada criada para o RDS ⚠️
  - Acesso público marcado como ``No``
  - Escolher o security group das redes privadas (que já deve existir)
  - Deixar as zonas de preferência como ``No prefecence``
- Após isso, podemos criar o RDS!

⚠️ Salve informações como nome do banco, ID do usuário master e senha, pois serão usados futuramente para definir variáveis de ambiente no script ``user_data.sh`` no qual definiremos a conecção da instância com o RDS.

# Criando uma Instância EC2

### Para criar uma isntância EC2, é preciso definir várias configurações:

- Adicionar as tags (ou rótulos);
- Escolher uma AMI (modelo que contém a configuração do software necessária para executar a instância);
- Selecionar um tipo de instância que atenda às suas necessidades de computação, memória, rede ou armazenamento;
- Selecionar um par de chaves, importantes para provar sua identidade ao se conectar a uma instância do Amazon EC2, ou criar um novo;
- Escolher a VPC e a subnet que serão utilizadas;
- Decidir se irá ou não habilitar a criação automática de IP público;
- Definir um grupo de segurança para controlar o tráfego de entrada e saída, ou criar um novo.
- Configurar o armazenamento;
- Se quiser, em ``Detalhes Avançados``, pode-se adicionar um script de comando a ser executado quando você executar a instância (dados do usuário).

### Para este projeto, podemos usar as seguintes configurações:

- AMI Ubuntu
- Instância do tipo t2.micro
- Selecionar um par de chaves existente ou criar um
- Escolher VPC criada para projeto
- Selecionar subnet privada
- Selecionar grupo de segurança
- Defini minhas configurações de armazenamento (1x 25 GiB gp2 Volume raiz)
- E em detalhes avançados, defini o seguinte script de inicialização:

````
#!/bin/bash

sudo yum update -y
sudo yum install -y docker

sudo systemctl start docker
sudo systemctl enable docker

sudo usermod -aG docker ec2-user
newgrp docker

sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

sudo mkdir -p /home/ec2-user/wordpress

cat <<EOF > /home/ec2-user/wordpress/docker-compose.yml
services:
  wordpress:
    image: wordpress
    restart: always
    ports:
      - 80:80
    environment:
      WORDPRESS_DB_HOST: database-1.cbsqoiwwa7q6.us-east-1.rds.amazonaws.com:3306
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: admin1289
      WORDPRESS_DB_NAME: wordpressbd
    volumes:
      - /mnt/efs:/var/www/html
EOF

sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-07c30321be437f1f2.efs.us-east-1.amazonaws.com:/ /mnt/efs

docker-compose -f /home/ec2-user/wordpress/docker-compose.yml up -d
````

Após isso, executei minha instância!   :)

# Conectando a Instância

Para conectar a uma instância, vá até a página de instâncias em execução e selecione a instância que deseja se conectar:

![image](https://github.com/user-attachments/assets/8c759b57-9618-4753-833e-b729b5adfd5b)

Após isso, escolha a maneira como quer se conectar, eu usarei a conecção via SSH:

![image](https://github.com/user-attachments/assets/d48144ff-cbd1-40c1-b3a2-d7af2b8e6f94)

A partir disto, vá até seu terminal e entre no diretório onde está salvo o seu par de chaves .pem e execute os seguintes comandos:
- ``chmod 400 "[nome da sua chave .pem]"``
- ``ssh -i "[nome da sua chave .pem]" ubuntu@ec2-[endereço IPV4 público da sua instância].compute-1.amazonaws.com``

Se seu usuário tiver as permissões necessárias e se estiver no grupo docker, então você conseguirá se conectar com sucesso!



