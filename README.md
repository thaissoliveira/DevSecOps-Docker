# Projeto DevSecOps - Docker e WordPress na AWS

A atividade proposta envolve configurar e implementar uma aplicação WordPress na AWS usando máquinas EC2, com boas práticas de infraestrutura e automação! Deve-se, ainda, seguir a estrutura de topologia listada abaixo:

![Captura de tela 2024-12-23 130350](https://github.com/user-attachments/assets/110fe7d6-1bdb-451d-bb73-a5040ea932aa)

### Requisitos do projeto:
(1). Configurar a instância EC2 para que ela suporte contêineres, instalando o Docker ou o Containerd. Este requisito inclui baixar e instalar o Docker ou Containerd, configurar o serviço para ser iniciado automaticamente e garantir que o usuário tenha permissão para usá-lo, normalmente, adicionando o usuário ao grupo apropriado.
   
(2). Utilizar a instalação via script de Start Instance (user_data.sh), o que permite que você execute scripts automaticamente assim que a instância é inicializada.
   
(3). Realizar o deploy da aplicação WordPress utilizando contêineres e o serviço RDS da AWS para o banco de dados.
   
(4). Configuração da utilização do serviço EFS AWS para estáticos do container de aplicação Wordpress.
   
(5). Configurar um Load Balancer da AWS para a aplicação do WordPress, distribuindo, então, o tráfego de rede de forma eficiente entre várias instâncias ou contêineres que estejam executando o WordPress.

# Iniciando Implantação do Projeto: Criando uma VPC 

O primeiro passo para a execução deste projeto é a criação de uma Rede Privada Virtual (Virtual Private Cloud), é preciso criar esta VPC na AWS para hospedar a infraestrutura do projeto. Portanto, no ambiente da AWS pesquise na barra de pesquisa por "VPC", clique e inicie a criação de uma; a partir disto, definimos o bloco CIDR (que é o intervalo de endereços IPv4 para a VPC), a quantidade de sub-redes menores e suas respectivas conexões com a internet.

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

![image](https://github.com/user-attachments/assets/ff138793-99fb-4dca-8ba1-8b18661bc9af)

Se você seguiu as instruções listadas acima, sua VPC vai ter o esquema de fluxo como o da imagem.

# Criando os Security Groups

Os grupos de segurança na AWS são firewalls virtuais que controlam o tráfego de entrada e saída de instâncias e outros recursos na nuvem, baseando-se em regras que definem quais protocolos, portas e endereços IP são permitidos. São baseados em estado, o que significa que permitem automaticamente o tráfego de retorno para conexões ativas. Por padrão, bloqueiam todo o tráfego de entrada e permitem todo o tráfego de saída, mas podem ser configurados conforme as necessidades. Para nosso projeto, vamos criar um grupo de segurança para cada serviço, a fim de deixar mais organizado e preciso. Vá até a área de EC2 na AWS e pesquise por ``Security groups`` para criar os grupos.

### Grupo de segurança para redes públicas:

Regra de entrada em rede pública:

![image](https://github.com/user-attachments/assets/fe6feede-b4a7-48ab-af09-d7a1530df6c2)

Regra de saída: All traffic.

### Grupo de segurança para redes privadas:

Regra de entrada:

Aqui é importante notar que para o HHTP (porta 80) e o HTTPS (443) tem origem no security group público.

![image](https://github.com/user-attachments/assets/80f70b3a-f92c-4e6f-96c3-eda9043d5072)

Regra de saída: All traffic.

### Grupo de segurança para Elastic File System:

Regra de entrada:

![image](https://github.com/user-attachments/assets/7fb945ad-2b6b-47e6-9075-68927da5c929)

Regra de saída: All traffic.

### Grupo de segurança para Load Balancer:

Regra de entrada:

![image](https://github.com/user-attachments/assets/f18aa9b5-2172-481c-a728-c1412a54fd46)

Regra de saída: All traffic.

### Grupo de segurança para RDS:

Regra de entrada:

![image](https://github.com/user-attachments/assets/19c9a9bd-6989-4d31-bb5b-ebdf6009b3ca)

Regra de saída: All trafic.

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
  - Acesso público marcado como ``No``
  - Escolher o security group das redes privadas (que já deve existir)
  - Deixar as zonas de preferência como ``No prefecence``
- Após isso, podemos criar o RDS!

⚠️ Salve informações como nome do banco, ID do usuário master e senha, pois serão usados futuramente para definir variáveis de ambiente no script ``user_data.sh`` no qual definiremos a conecção da instância com o RDS. Além disso, em ``Configuração adicional`` podemos desativar opções de backup, monitoramento e criptografia para evitar gastos.

# Criando uma Bastion Host

O bastion host é um servidor intermediário usado para acessar máquinas em uma rede privada com segurança. Ele funciona como uma ponte entre a internet e os recursos internos, garantindo que as outras máquinas na rede privada não fiquem expostas diretamente à internet. Resumindo, vamos acessar nossas instâncias privadas através de uma instância pública que acessa a internet, que é o bastion host!

Configurações da bastion host:

Lembre-se que seu bastion host precisa ter um edereço público, por isso, atenção quando for configurar a rede.

- Defina um nome como ``wordpress-bastion-host``
- Escolha a imagem de sistema ``Ubuntu`` na versão ``24.04 LTS``
- Tipo de instância ``t2.micro``
- Selecione seu par de chaves .pem ou crie um
- Selecione a VPC do projeto
- Escolha uma sub-rede pública (⚠️ passo importante)
- Escolha o grupo de segurança para redes públicas criado anteriormente (⚠️ passo importante)
- Habilite a atribuição de IP Público automático (⚠️ passo importante)
- Mantenha as configurações de armazenamento como estão

Lembre-se que para editar as ``Configurações de Rede``, é preciso clicar em ``Editar`` no canto superior direito do bloco:

![image](https://github.com/user-attachments/assets/b389aa49-1628-41f3-a504-e70d19ebf832)

Agora, conecte-se a sua instância bastion host via protocolo SSH!

# Conectando a Instância

Para conectar a uma instância, vá até a página de instâncias em execução, selecione a instância que deseja se conectar e clique no botão ``Conectar`` no canto superior esquerdo:

![image](https://github.com/user-attachments/assets/8c759b57-9618-4753-833e-b729b5adfd5b)

Após isso, escolha a maneira como quer se conectar, eu usarei a conecção via SSH:

![image](https://github.com/user-attachments/assets/d48144ff-cbd1-40c1-b3a2-d7af2b8e6f94)

A partir disto, vá até seu terminal e navegue até o diretório onde está salvo o seu par de chaves .pem e execute os seguintes comandos:

- ``chmod 400 "[nome da sua chave .pem]"``
- ``ssh -i "[nome da sua chave .pem]" ubuntu@ec2-[endereço IPV4 público da sua instância].compute-1.amazonaws.com``
  
Se seu usuário tiver as permissões necessárias e se estiver no grupo docker, então você conseguirá se conectar com sucesso!

Futuramente, vamos precisar acessar nossas instâncias privadas através da instância bastion host (para isso que criamos ela!), portanto, vamos precisar copiar nosso par de chaves (que está na máquina local) para a máquina virtual.

No terminal da sua máquina local, ainda no diretório onde está sua chave e execute o seguinte comando:

````
scp -i <chave.pem> <arquivo_local> <usuario>@<ip_remoto>:<caminho_destino_remoto>
````

- ``chave.pem`` É o arquivo da sua chave.
- ``arquivo_local`` É o diretório local, o caminho, onde está sua chave.
- ``usuario`` É o nome do usuário da máquina virtual, no nosso caso, como estamos usando uma AMI Ubuntu, o nome sera "ubuntu".
- ``ip_remoto`` É o IPV4 atribuído a sua máquina virtual (você pode verificar na AWS)
- ``caminho_destino_remoto``

Agora você estará conectado e sua chave estrá copiada.

# Criando uma Instância EC2



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

# Atualizar pacotes do sistema
sudo apt update -y
sudo apt upgrade -y

# Instalar Docker, curl e suporte a NFS
sudo apt install -y docker.io
sudo apt install -y nfs-common

# Iniciar e habilitar o serviço Docker
sudo systemctl start docker
sudo systemctl enable docker

# Adicionar o usuário ao grupo Docker
sudo usermod -aG docker $(whoami)

# Baixar e instalar Docker Compose
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Criar diretório para o WordPress
sudo mkdir -p /home/$(whoami)/wordpress

# Criar o arquivo docker-compose.yml com configurações do WordPress
cat <<EOF > /home/$(whoami)/wordpress/docker-compose.yml
version: '3.1'

services:
  wordpress:
    image: wordpress
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: database-1.cbsqoiwwa7q6.us-east-1.rds.amazonaws.com:3306
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: admin1289
      WORDPRESS_DB_NAME: wordpressdb
    volumes:
      - /mnt/efs:/var/www/html
EOF

# Montar o EFS na máquina local
sudo mkdir -p /mnt/efs
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0ff145e2edac09127.efs.us-east-1.amazonaws.com:/ /mnt/efs

# Rodar o Docker Compose para iniciar o WordPress
sudo docker-compose -f /home/$(whoami)/wordpress/docker-compose.yml up -d
````

Após isso, executei minha instância!   :)

# Teste de integração com Load Balancer (LB)

Um Load Balancer (Balanceador de Carga) na AWS é um serviço gerenciado que distribui automaticamente o tráfego de entrada de uma aplicação entre várias instâncias, contêineres ou recursos computacionais, garantindo alta disponibilidade, escalabilidade e resiliência. Portanto, ele balanceia o tráfego de entrada para evitar sobrecarga em um único recurso, no caso, em uma única zona. Além disso, ele integra-se ao Auto Scaling, garantindo que novos recursos sejam adicionados ou removidos conforme a demanda, o que será muito importante no futuro do projeto.

Na área de EC2, pesquise por ``Load balancers`` e inicie a criação de um grupo seguindo as seguintes características:

- Escolha o tipo do LB como sendo ``Classic Load Balancer ``
- Dê um nome ao seu LB
- Mantenha o ``Scheme`` como ``Internet-facing``
- Selecione a VPC do projeto e em seguida as duas zonas disponíveis que aparecerão, assim como as sub-redes públicas destas zonas (É importante que as sub-redes sejam as públicas)

  ![image](https://github.com/user-attachments/assets/a417cd55-a233-454b-b96b-576f2e117fb2)

- Escolha o security group criado anteriormente para o LB
- No campo ``Health checks`` adicione o caminho de ping path: ``/wp-admin/index.php``
- Em ``Instâncias`` selecione as instâncias privadas que foram criadas anteriormente

  ![image](https://github.com/user-attachments/assets/d23492aa-88b4-4ab5-bff0-095a6e5cf4b5)

- Clique em ``Create load balancer``

Para verificar se suas instâncias passaram do teste de integração, vá no campo ``Target instances`` e veja se as instâncias estão em serviço.

![image](https://github.com/user-attachments/assets/2e00ab11-8d4c-4af1-8012-79fff9f82ca4)

# Criando um Grupo de Auto Scaling

O Auto Scaling Group (ASG) da AWS é um serviço que ajusta automaticamente a capacidade computacional da aplicação de acordo com a demanda, garantindo que ela tenha o desempenho necessário enquanto otimiza custos. Ele opera principalmente com grupos de Auto Scaling associados a instâncias EC2. O Auto Scaling garante que o site continue funcional mesmo durante picos de tráfego, escalando horizontalmente, enquanto que durante períodos de baixa, o número de instâncias é reduzido para economizar. Pode ser integrado a balanceadores de carga (ELB) para distribuir o tráfego uniformemente entre as instâncias escaladas.

Na área de EC2, pesquise por ``Auto Scaling Groups`` e inicie a criação de um grupo com as seguintes configurações:

- Dê um nome para o grupo
- Em ``Launch Template``, selecione o modelo de execução criado para instâncias privadas e clique em ``Next``
- Na próxima página, escolha a VPC do projeto e selecione as sub-redes públicas e clique em ``Next``
- Em ``Load Balancing`` escolha a opção ``Attach to an existing load balancer`` e logo em seguida selecione ``Choose from Classic Load Balancers``
- Selecione o Classic Load Balancer que foi criado na etapa anterior e clique em ``Next``
  
    ![image](https://github.com/user-attachments/assets/2b748b93-3176-4457-af48-2f824e6ad605)

- Defina as variáveis:
   - Capacidade Mínima ``(min size)``: 2 (o ASG nunca terá menos de duas instâncias).
   - Capacidade Desejada ``(desired capacity)``: 2 (o ASG inicia com duas instâncias).
   - Capacidade Máxima ``(max size)``: 4 (o ASG pode escalar até um total de três instâncias).
- O restante das configurações, mantenha como estão.

Se você for até a página de instâncias, verá que o ASG está subindo novas instâncias!

![image](https://github.com/user-attachments/assets/2c27e3b1-db5e-4013-b9aa-c7dc8495c89d)

# Conclusão




