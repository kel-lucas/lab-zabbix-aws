# Guia Definitivo: Monitoramento Zabbix + DHCP na AWS (Custo Zero)

Para garantir **Custo Zero (Free Tier)** e funcionalidade real, esta arquitetura foi alterada para evitar armadilhas comuns de cobrança na AWS.

## 🚨 As 3 Melhores Importantes (Correções Críticas)

1.  **A Armadilha da "Subnet Privada":**
    * *Problema:* A solução tradicional sugere subnets privadas. Porém, para instalar pacotes nelas, você precisaria de um *NAT Gateway*, que custa ~$30 USD/mês (sem free tier).
    * *Solução:* Usaremos apenas **Subnets Públicas**. A segurança será feita via Firewall (Security Groups), garantindo custo zero.

2.  **O Problema do DHCP na AWS:**
    * *Problema:* A AWS bloqueia tráfego de broadcast. Um servidor DHCP criado por você não atribuirá IPs reais para outras instâncias.
    * *Ajuste:* Instalaremos o serviço DHCP apenas para **simular o processo** e monitorar se o serviço está rodando (Up/Down).

3.  **Zabbix Database:**
    * *Ajuste:* Rodaremos o banco de dados na mesma máquina do Zabbix (Localhost) para economizar recursos e evitar a complexidade/custo do RDS neste momento.

---

## 🛠️ Guia Passo a Passo

### ⚠️ Passo 0: O "Cinto de Segurança" (Billing Alarm)
Antes de tocar em qualquer servidor, configure um alerta.

1.  No console AWS, vá em **Billing and Cost Management**.
2.  Em **Budgets**, crie um orçamento de "Zero Spend" ou defina um limite de **$1.00**.
3.  Insira seu e-mail. Se configurar algo errado, a AWS avisa imediatamente.

---

### 🧱 Passo 1: A Rede (VPC Simples)
Vamos criar a rede onde os servidores ficarão.

1.  Vá para **VPC** -> **Create VPC**.
2.  Selecione **VPC and more**.
3.  **Configurações:**
    * **Name:** `lab-zabbix`
    * **IPv4 CIDR:** `10.0.0.0/16`
    * **Number of Availability Zones:** `1`
    * **Number of Public Subnets:** `1`
    * **Number of Private Subnets:** `0` (**Crucial para custo zero**)
    * **NAT Gateways:** `None` (**Crucial para custo zero**)
    * **VPC Endpoints:** `None`
4.  Clique em **Create VPC**.

> **Justificativa:** Sem subnets privadas e sem NAT Gateway, eliminamos custos de infraestrutura de rede. O acesso é controlado via firewall.

---

### 🛡️ Passo 2: Segurança (Security Groups)
Vá em **VPC** -> **Security Groups**.

#### Grupo 1: `sg_zabbix_server`
* **Inbound Rules (Entrada):**
    * `SSH (22)` -> Source: **My IP** (Seu acesso).
    * `HTTP (80)` -> Source: **My IP** (Seu acesso ao painel web).
    * `Custom TCP (10051)` -> Source: `10.0.0.0/16` (Permite agentes da rede enviarem dados).

#### Grupo 2: `sg_dhcp_alvo`
* **Inbound Rules (Entrada):**
    * `SSH (22)` -> Source: **My IP**.
    * `Custom TCP (10050)` -> Source: Selecione o grupo `sg_zabbix_server`.

---

### 💻 Passo 3: Criar as Instâncias (EC2)
Vá para **EC2** -> **Launch Instances**.

#### Servidor 1: Zabbix Server
* **Name:** `SRV-ZABBIX`
* **OS:** Ubuntu Server 24.04 LTS.
* **Instance Type:** `t2.micro` ou `t3.micro` (Free tier eligible).
* **Key Pair:** Crie uma nova (ex: `chave-lab`) e baixe o `.pem`.
* **Network Settings:**
    * VPC: `lab-zabbix`
    * Subnet: Public Subnet
    * Auto-assign Public IP: **Enable**
    * Security Group: `sg_zabbix_server`
* **Storage:** 20 GB gp3.

#### Servidor 2: DHCP Server (Monitorado)
* **Name:** `SRV-DHCP`
* **OS/Instance Type/Key Pair:** Mesmos do acima.
* **Network Settings:**
    * Security Group: `sg_dhcp_alvo`
* **Storage:** 10 GB gp3.

---

### ⚙️ Passo 4: Instalar o Zabbix Server
Acesse o `SRV-ZABBIX` via SSH.

1.  **Baixe e instale o repositório Zabbix 7.0:**
    ```bash
    wget [https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb](https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb)
    sudo dpkg -i zabbix-release_7.0-2+ubuntu24.04_all.deb
    sudo apt update
    ```

2.  **Instale Server, Frontend, Agente e MySQL:**
    ```bash
    sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent mysql-server -y
    ```

3.  **Configure o Banco de Dados:**
    ```bash
    sudo mysql -uroot -e "create database zabbix character set utf8mb4 collate utf8mb4_bin;"
    sudo mysql -uroot -e "create user zabbix@localhost identified by 'senha_segura_aqui';"
    sudo mysql -uroot -e "grant all privileges on zabbix.* to zabbix@localhost;"
    sudo mysql -uroot -e "set global log_bin_trust_function_creators = 1;"
    ```

4.  **Importe os dados iniciais:**
    ```bash
    zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p'senha_segura_aqui' zabbix
    ```

5.  **Configure a senha no arquivo:**
    * Edite: `sudo nano /etc/zabbix/zabbix_server.conf`
    * Mude para: `DBPassword=senha_segura_aqui`

6.  **Reinicie tudo:**
    ```bash
    sudo systemctl restart zabbix-server zabbix-agent apache2
    sudo systemctl enable zabbix-server zabbix-agent apache2
    ```

---

### 🎯 Passo 5: Configurar o Servidor Alvo (DHCP)
Acesse o `SRV-DHCP` via SSH.

1.  **Instale o serviço DHCP:**
    ```bash
    sudo apt update
    sudo apt install isc-dhcp-server -y
    ```
    *(Nota: Falhará ao iniciar por falta de config. Normal)*

2.  **Instale o Agente Zabbix:**
    ```bash
    wget [https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb](https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb)
    sudo dpkg -i zabbix-release_7.0-2+ubuntu24.04_all.deb
    sudo apt update
    sudo apt install zabbix-agent -y
    ```

3.  **Configure o Agente:**
    * Edite: `sudo nano /etc/zabbix/zabbix_agentd.conf`
    * `Server=IP_PRIVADO_DO_ZABBIX` (ex: 10.0.X.X)
    * `Hostname=SRV-DHCP`

4.  **Reinicie o agente:**
    ```bash
    sudo systemctl restart zabbix-agent
    sudo systemctl enable zabbix-agent
    ```

---

### 🖥️ Passo 6: Configuração Visual (Frontend)

1.  Acesse `http://IP-PUBLICO-DO-ZABBIX/zabbix`.
2.  Siga o setup (Login: `Admin` / `zabbix`).
3.  Vá em **Data collection** -> **Hosts** -> **Create Host**.
    * **Host name:** `SRV-DHCP`
    * **Templates:** `Linux by Zabbix agent`
    * **Interfaces:** Add -> Agent -> **IP PRIVADO do SRV-DHCP** (Porta 10050).
4.  Clique em **Add**.

**Para monitorar o processo DHCP:**
1.  No host SRV-DHCP, vá em **Items** -> **Create Item**.
2.  **Name:** `Status do DHCP`
3.  **Key:** `proc.num[dhcpd]`

---

### 💡 Dica de Ouro para Economia

* **Pare as instâncias:** Quando terminar de estudar, vá no console EC2 -> Instance State -> **Stop Instance**.
* **Custo:** Máquina parada não cobra processamento, apenas o armazenamento (que é barato e coberto pelo Free Tier até 30GB).