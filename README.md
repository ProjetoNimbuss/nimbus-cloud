# 🌩️ Nimbus Cloud — Infraestrutura, Redes e Segurança

Este repositório contém todo o código de **Infraestrutura como Código (IaC)**, provisionamento de nuvem, segurança, VPN e redes da **Plataforma Nimbus**.

---

## 🏗️ Estrutura do Repositório

```
nimbus-cloud/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   ├── backend.tf
│   ├── terraform.tfvars.example
│   └── modules/
│       ├── gcp_data/         # Buckets GCS, Datasets BigQuery, Repositório Dataform
│       ├── mgc_compute/      # Provisionamento da VM na MagaluCloud
│       └── cloudflare_dns/   # DNS, Domínios, SSL e Tunnels
├── docs/
│   ├── Documentação_Terraform.md
│   ├── Documentação_Configuração_Acesso_Equipe_Tailscale.md
│   ├── Documentação_Tailscale.md
│   └── Documentação_de_Acesso_tailscale.md
├── .gitignore
└── README.md
```

---

## 🚀 Como Utilizar o Terraform

### 1. Pré-requisitos
- [Terraform >= 1.5](https://www.terraform.io)
- Credenciais configuradas para GCP (Google Cloud SDK / Service Account)
- Credenciais configuradas para MagaluCloud (MGC API Key)
- Credenciais configuradas para Cloudflare (API Token)

### 2. Configuração de Variáveis
Copie o arquivo de exemplo e preencha com as credenciais reais:
```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
```

> ⚠️ **Atenção:** O arquivo `terraform.tfvars` contém segredos e está bloqueado no `.gitignore`. Nunca o envie para o repositório.

---

## 📚 Documentação
Consulte a pasta [`docs/`](docs/) para orientações detalhadas de configuração do Tailscale, acessos da equipe e arquitetura da infraestrutura.
