# Criando uma Instãncia EC2
## Para criar uma isntância EC2, é preciso definir as seguintes configurações:
- Adicionar as tags (ou rótulos);
- Escolher uma AMI (modelo que contém a configuração do software necessária para executar a instância);
- Selecionar um tipo de instância que atenda às suas necessidades de computação, memória, rede ou armazenamento;
- Selecionar um par de chaves, importantes para provar sua identidade ao se conectar a uma instância do Amazon EC2, ou criar um novo;
- Escolher a VPC e a subnet que serão utilizadas;
- Decidir se irá ou não habilitar a criação automática de IP público;
- Definir um grupo de segurança para controlar o tráfego de entrada e saída, ou criar um novo.
- Configurar o armazenamento;
- Se quiser, em ``Detalhes Avançados``, pode-se adicionar um script de comando a ser executado quando você executar a instância (dados do usuário).

## Na minha EC2, usei as seguintes configurações:
- AMI Ubuntu
- Instância do tipo t2.micro
- Criei um par chaves
- Escolhi uma VPC que criei com duas subnets privadas e outras duas públicas
- Selecionei uma subnet privada
- Criei um grupo de segurança
- Defini minhas configurações de armazenamento (1x 25 GiB gp2 Volume raiz)
- E em detalhes avançados, defini o seguinte script de inicialização:
````
  #!/bin/bash
sudo apt-get update -y
sudo apt-get upgrade -y

sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

sudo usermod -aG docker ubuntu

sudo systemctl enable docker

sudo apt-get install -y git

docker --version
git --version

sudo reboot
````
Configuração do meu Security Group:

![WhatsApp Image 2024-12-16 at 19 50 28](https://github.com/user-attachments/assets/d67b30b3-267a-4325-a8f3-309ca2a4634b)

Configuração da minha VPC recém criada:

![image](https://github.com/user-attachments/assets/e6030d29-f670-4a34-8ec6-42da6faf2357)

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

# Instalando o Nginx

Para instalar o Nginx, execute os seguintes comandos em seu terminal:

- ``sudo apt update -y`` - Atualiza a lista de pacotes disponíveis 
- ``sudo apt upgrade -y`` - Atualiza todos os pacotes instalados para as versões mais recentes
- ``sudo apt install nginx -y`` - Instala o servidor web Nginx
- ``sudo systemctl start nginx`` - Inicia o serviço do Nginx
- ``sudo systemctl enable nginx`` - Habilita o Nginx para iniciar automaticamente no boot

Aqui, se você pesquisar o IP Público da sua instância no navegador, verá que o Nginx está funcionando:

![image](https://github.com/user-attachments/assets/96607cde-5c7a-4d20-89fe-4ec8608bab08)


