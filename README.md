# Atividade_docker
## Requisitos da atividade:
- Instalação e configuração do DOCKER ou CONTAINER no host EC2.

- Ponto adicional para o trabalho: Utilizar a instalação via script de Start Instance (user_data.sh).

- Efetuar deploy de uma aplicação WordPress com container de aplicação RDS database MySQL.

- Configuração da utilização do serviço EFS AWS para estáticos do container de aplicação WordPress.

- Configuração do serviço de Load Balancer AWS para a aplicação WordPress.

## Pontos de atenção:
- Não utilizar IP público para saída dos serviços WordPress (Evitem publicar o serviço WordPress via IP público).

- Sugestão para o tráfego: Internet sair pelo LB (Load Balancer Classic).

- Pastas públicas e estáticos do WordPress sugestão de utilizar o EFS (Elastic File System).

- Fica a critério de cada integrante usar Dockerfile ou Docker Compose.

- Necessário demonstrar a aplicação WordPress funcionando (tela de login).

- Aplicação WordPress precisa estar rodando na porta 80 ou 8080.

- Utilizar repositório git para versionamento.

- Criar documentação.

# Instruções de execução

### 1. Configuração de Rede

**1.1 Criar VPC:**

- Acesse o console AWS > VPC > Suas VPCs > Criar VPC
- Configurações:
    - Recursos a criar: VPC e mais
    - Nome da tag: "project-docker-vpc"
    - Número de Zonas de Disponibilidade (AZs): 2
    - Gateways NAT: Em 1 AZ
    - Endpoints da VPC: Nenhum
    - Clique em Criar VPC

### 2. Configuração de Grupos de Segurança

**2.1 Criar Grupos de Segurança:**

- Acesse o console AWS > EC2 > Grupos de Segurança > Criar grupo de segurança
- Crie e configure os seguintes grupos e regras:
    - **Load Balancer**
        - Tipo: HTTP, Protocolo: TCP, Faixa de Portas: 80, Fonte: 0.0.0.0/0
    - **Servidor Web EC2**
        - Tipo: SSH, Protocolo: TCP, Faixa de Portas: 22, Fonte: EC2 ICE
        - Tipo: HTTP, Protocolo: TCP, Faixa de Portas: 80, Fonte: Load Balancer
    - **EC2  (saída)**
        - Tipo: SSH, Protocolo: TCP, Faixa de Portas: 22, Fonte: Servidor Web EC2
    - **RDS**
        - Tipo: MYSQL/Aurora, Protocolo: TCP, Faixa de Portas: 3306, Fonte: Servidor Web EC2
    - **EFS**
        - Tipo: NFS, Protocolo: TCP, Faixa de Portas: 2049, Fonte: Servidor Web EC2

### 3. Criar Elastic File System (EFS)

**3.1 Configurar EFS:**

- Acesse o console AWS > EFS > Criar sistema de arquivos > Personalizar
    - **Etapa 1 - Configurações do sistema de arquivos:**
        - Nome: "efs-docker"
        - Clique em Próximo
    - **Etapa 2 - Acesso à rede:**
        - VPC: VPC criada anteriormente
        - Subnets: Subnets privadas de cada AZ
        - Grupos de segurança: Grupo "EFS" criado anteriormente
        - Clique em Próximo

### 4. Criar Relational Database Service (RDS)

**4.1 Configurar RDS:**

- Acesse o console AWS > RDS > Criar banco de dados
    - **Opções de motor:** MySQL
    - **Modelos:** Free tier
    - **Configurações de Credenciais:** Adicionar senha mestre
    - **Conectividade:** VPC criada anteriormente, Grupo de segurança "RDS"
    - **Nome do banco de dados inicial:** "DOCKERDB"

### 5. Criar Load Balancer

**5.1 Configurar Load Balancer:**

- Acesse o console AWS > EC2 > Load Balancers > Criar load balancer > Classic Load Balancer > Criar
    - De um nome ao load balance
    - **Mapeamento de rede:** VPC criada anteriormente, subnets públicas
    - **Grupos de segurança:** Grupo "Load Balancer"
    - **Verificações de integridade:** Caminho de ping "/wp-admin/install.php"
    

### 7. Criar Modelo de Execução

**7.1 Configurar Template de Inicialização:**

- Acesse o console AWS > EC2 > Modelos de Inicialização > Criar modelo de inicialização
    - **Nome:** de sua preferencia
    - **Descrição da versão do template:** "docker-wordpress"
    - **AMI:** Amazon Linux 2023
    - **Tipo de instância:** t3.small
    - **Nome do par de chaves:** "chave-projeto"
    - **Grupos de segurança:** "Servidor Web EC2"
    - Em **Advanced details**, no campo **User data** adicionei o script abaixo

```bash
bashCopiar código
#!/bin/bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo yum install nfs-utils -y
sudo systemctl start nfs-utils
sudo systemctl enable nfs-utils
sudo mkdir /efs
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport ID-EFS:/ efs
sudo echo "ID-EFS:/ /efs nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0" >> /etc/fstab
sudo mkdir /efs/wordpress
sudo cat <<EOL > /efs/docker-compose.yaml
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: RDS-Endpoint
      WORDPRESS_DB_USER: RDS-Master username
      WORDPRESS_DB_PASSWORD: RDS-Master password
      WORDPRESS_DB_NAME: RDS-Initial database name
      WORDPRESS_TABLE_CONFIG: wp_
    volumes:
      - /efs/wordpress:/var/www/html
EOL
docker-compose -f /efs/docker-compose.yaml up -d

```

- Clique em Criar modelo de inicialização

### 8. Criar Auto Scaling Group

**8.1 Configurar Auto Scaling Group:**

- Acesse o console AWS > EC2 > Grupos de Auto Scaling > Criar Auto Scaling group
    - **Nome:** de sua preferencia
    - **Rede:** VPC criada anteriormente, subnets privadas
    - **Balanceamento de carga:** Conectar a um load balancer existente.
    - **Tamanho do grupo:** Capacidade desejada: 2, Min: 2, Máx: 4
    - **Política de escalonamento:** Utilização média da CPU, Valor alvo: 75

### 9. Configuração do Endpoint de Conexão do EC2 Instance Connect

**9.1 Configurar Endpoint:**

- Acesse o console AWS > VPC > Endpoints > Criar endpoint
    - **Categoria do serviço:** Endpoint de Conexão do EC2 Instance
    - **VPC:** VPC criada anteriormente
    - **Grupos de segurança:** "EC2 ICE"
    - **Subnet:** Subnet privada
    - Clique em Criar endpoint
