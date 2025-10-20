# 📘 Anotações - EC2 (Elastic Compute Cloud) - AWS

> Este documento contém anotações organizadas sobre a EC2 da AWS, útil para estudos, revisões ou consultas rápidas.

---

## ✅ O que é a EC2?

- **EC2 (Elastic Compute Cloud)** é um serviço da AWS que fornece **servidores virtuais na nuvem**.
- Permite **escalar a capacidade computacional** com facilidade, de forma elástica e sob demanda.
- Suporta múltiplos sistemas operacionais (Linux, Windows, etc.).

---

## 🔧 Componentes principais

### 🖥️ Instâncias
- Equivalente a **máquinas virtuais (VMs)**.
- Podem ser iniciadas, pausadas, paradas e terminadas.
- Baseadas em **AMIs** (Amazon Machine Images).

### 📦 AMI (Amazon Machine Image)
- Imagem usada para criar instâncias.
- Contém SO, aplicações, bibliotecas, etc.
- Tipos:
  - Padrão da AWS
  - Marketplace
  - Personalizada

### 🧠 Tipos de instâncias
- Classificadas por família (uso geral, otimizado para CPU, memória, GPU, etc.)
- Ex: `t3.micro`, `m5.large`, `c6g.xlarge`, etc.
- **Naming convention**:
  - Prefixo da família (ex: `t`, `m`, `c`)
  - Geração (ex: `2`, `3`, `5`)
  - Tamanho (ex: `micro`, `small`, `large`)

### 🌍 Regiões e zonas de disponibilidade (AZ)
- Regiões = locais geográficos (ex: us-east-1)
- AZs = data centers independentes dentro de uma região
- Alta disponibilidade = executar instâncias em múltiplas AZs

---

## 🌐 Conectividade

### 🔒 Key Pairs
- Chaves SSH para acessar instâncias Linux.
- Par de chaves: pública (armazenada na AWS) e privada (baixada pelo usuário).
- Uma vez perdida a chave privada, não é possível acessar a instância.

### 🛡️ Security Groups
- Atuando como **firewall virtual**.
- Controla tráfego de entrada e saída com base em regras (IP, porta, protocolo).
- Pode ser associado a uma ou mais instâncias.

### 🖧 Elastic IP
- Endereço IP fixo que pode ser associado a uma instância.
- Útil para manter um IP estático mesmo após reinicializações.

---

## 💾 Armazenamento

### 📁 Volumes EBS (Elastic Block Store)
- Dispositivos de armazenamento em bloco.
- Persistem após término da instância (se configurado).
- Tipos: SSD, HDD, provisionado, etc.

### 📂 Instance Store
- Armazenamento físico ligado ao host.
- Temporário: dados são perdidos após término da instância.

---

## ⚙️ Auto Scaling e Load Balancing

### 📈 Auto Scaling
- Ajusta automaticamente a quantidade de instâncias com base em métricas (CPU, memória, etc.)
- Usa **Launch Templates** ou **Launch Configurations**.

### 🔄 ELB (Elastic Load Balancer)
- Distribui o tráfego entre múltiplas instâncias.
- Tipos: Application (ALB), Network (NLB), Gateway (GLB)

---

## 💸 Faturamento e Custos

- EC2 é cobrada por:
  - Tipo de instância
  - Tempo de uso (por segundo/minuto/hora)
  - Armazenamento EBS
  - Tráfego de rede (em alguns casos)
- Opções de compra:
  - **On-Demand**: sem compromis
