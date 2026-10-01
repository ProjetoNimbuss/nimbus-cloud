# Procedimento Técnico: Configuração e Resolução de Acesso Multi-Usuário via Tailscale SSH

Este documento descreve o diagnóstico, a fundamentação técnica e a solução aplicada para viabilizar o acesso seguro de múltiplos membros da equipe ao servidor de produção (`nimbus-server-prod`) na Magalu Cloud via Tailscale SSH.

---

## 1. Contexto Operacional

O servidor `nimbus-server-prod` foi provisionado na Magalu Cloud sem endereço IPv4 público associado (`allocate_public_ipv4 = false`), garantindo isolamento total da internet pública. O tráfego de gerenciamento e administração é roteado exclusivamente por meio da rede privada mesh do Tailscale, com autenticação nativa via Tailscale SSH (`--ssh`).

Durante o processo de integração de novos membros técnicos da equipe ao servidor, foram identificados bloqueios em duas camadas distintas: resolução de rede/organização e políticas de controle de acesso (ACL).

---

## 2. Diagnóstico e Análise de Causa Raiz

### 2.1 Isolamento de Rede e Falha de Resolução de Hostname
* **Sintoma:** O comando `ssh debian@nimbus-server-prod` executado em ambiente Windows PowerShell retornou o erro `Could not resolve hostname nimbus-server-prod: Este host não é conhecido`.
* **Causa Raiz:** O cliente Tailscale no computador do desenvolvedor estava autenticado em sua rede pessoal individual (`desenvolvedor@email.com`), e não na organização proprietária da infraestrutura do projeto (`organizacao-tailnet`). Por estarem em Tailnets distintas e isoladas, o resolvedor local não possuía rota ou visibilidade do nó remoto.

### 2.2 Restrição do Modelo de Propriedade de Dispositivos (Device Ownership)
* **Sintoma:** Após o desenvolvedor alternar para a Tailnet correta, a tentativa de conexão retornou o erro `tailscale: tailnet policy does not permit you to SSH to this node`.
* **Causa Raiz:** Por padrão, o Tailscale implementa a regra restritiva `autogroup:self` na política de SSH, que permite apenas que o próprio usuário criador da máquina estabeleça sessões administrativas. Como a VM foi provisionada pela identidade do proprietário original (`administrador@github`), requisições originadas por outros membros com perfil administrativo eram sumariamente recusadas pelo daemon do nó.

### 2.3 Incompatibilidade de Destino com Identidades de Usuários
* **Sintoma:** Tentativas de autorizar diretamente o e-mail ou o curinga no campo de destino (`dst`) da ACL resultaram nos erros `invalid dst "*"` e `users in dst are only allowed from the same user`.
* **Causa Raiz:** No modelo de segurança do Tailscale, nós vinculados a identidades pessoais de usuários não podem ser declarados como alvos genéricos de SSH para terceiros. O protocolo exige a dissociação do nó da conta pessoal por meio de etiquetas institucionais (`Tags`).

---

## 3. Implementação da Solução Técnica

Para converter a máquina virtual de um recurso pessoal para um ativo compartilhado da organização, foram executadas duas etapas:

### 3.1 Reconfiguração da Política de Controle de Acesso (ACL)
O arquivo de controle de acesso foi atualizado para declarar a posse institucional da tag e conceder permissão de SSH para membros autorizados:

```json
// Example/default ACLs for unrestricted connections.
{
	// Define administradores autorizados a gerenciar tags no parque de servidores
	"tagOwners": {
		"tag:server": ["autogroup:admin"],
	},

	// Regras gerais de conectividade IP na malha
	"grants": [
		{"src": ["*"], "dst": ["*"], "ip": ["*"]},
	],

	// Politica de acesso SSH
	"ssh": [
		{
			"action": "accept",
			"src":    ["autogroup:member"],
			"dst":    ["tag:server", "autogroup:self"],
			"users":  ["autogroup:nonroot", "root", "debian"],
		},
	],
}
```

* **`tagOwners`**: Define que usuários com perfil de administrador podem associar a etiqueta `tag:server` a dispositivos.
* **`action: accept`**: Elimina a necessidade de revalidação interativa constante via navegador durante o fluxo de conexão.
* **`dst: ["tag:server", "autogroup:self"]`**: Autoriza a malha a aceitar tráfego SSH direcionado a qualquer máquina marcada como servidor compartilhado.
* **`users: ["autogroup:nonroot", "root", "debian"]`**: Permite a assunção de sessão diretamente sob o usuário padrão da imagem Linux na nuvem (`debian`).

### 3.2 Transição de Propriedade do Nó (Node Tagging)
1. No console administrativo do Tailscale, o nó `nimbus-server-prod` foi editado para receber a etiqueta `tag:server`.
2. A atribuição da tag revogou a propriedade pessoal do usuário inicial sobre o nó, transferindo o controle do dispositivo formalmente para as regras globais da organização.

---

## 4. Validação de Conectividade

Com a aplicação da política e a marcação do nó, o fluxo de acesso foi estabelecido com sucesso sem necessidade de troca ou distribuição de chaves criptográficas manuais (`id_rsa`):

```powershell
ssh debian@nimbus-server-prod
```
Ou via endereçamento IP direto na malha:
```powershell
ssh debian@100.x.y.z
```

### Resultados Obtidos:
1. **Segurança Zero Trust:** A autenticação é validada pela identidade federada do desenvolvedor (Single Sign-On).
2. **Superfície Zero:** O servidor permanece sem portas expostas e sem IP público na internet.
3. **Governança:** A concessão ou revogação de acesso a novos membros passa a ser gerenciada centralmente pelo painel de usuários da organização, sem necessidade de intervenção no arquivo `authorized_keys` da máquina virtual.
