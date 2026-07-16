# 📚 Anotações — Active Directory

Baseado no projeto homelab de Active Directory. Material de consulta rápida para revisão.

---

## 1. Conceitos fundamentais

### Active Directory (AD)
Serviço da Microsoft que centraliza o gerenciamento de usuários, computadores, permissões e políticas de segurança de uma rede corporativa. O "cérebro" que controla quem pode acessar o quê.

### Domain Controller (DC)
Servidor que hospeda o Active Directory. Autentica logins, aplica políticas e mantém o banco de dados central do domínio.
> Projeto: **DC01-Servidor**

### Domínio
O "nome" da rede administrada pelo AD. Todo usuário, computador e grupo pertence a um domínio.
> Projeto: `adriana.local`

### DNS (Domain Name System)
Serviço que "traduz" nomes (como `adriana.local`) para endereços IP. O Active Directory depende do DNS para funcionar — sem DNS correto, o cliente não localiza o domínio.

---

## 2. Estrutura organizacional

### OU (Organizational Unit)
"Pasta" dentro do domínio usada para organizar usuários, computadores e grupos por departamento, função ou localização. Permite aplicar regras (GPOs) e delegar permissões por grupo, em vez de configurar item por item.
> Projeto: OUs **TI**, **RH**, **Financeiro**, **Suporte**

### Usuários
Contas individuais dentro de uma OU, representando pessoas.
> Projeto: João Silva, Maria Souza, Pedro Santos (na OU Suporte)

### Grupos de segurança
Reúnem vários usuários para aplicar permissões ou políticas de uma vez só.
> Projeto: `GG_Suporte_TI` — prefixo "GG" = "Global Group", convenção comum em ambientes profissionais

---

## 3. Rede: pré-requisito para tudo funcionar

| Conceito | O que é | Configuração usada |
|---|---|---|
| IP fixo | Endereço que não muda, essencial para o servidor ser sempre encontrado | `192.168.56.10` (servidor), `192.168.56.20` (cliente) |
| Máscara de sub-rede | Define o "tamanho" da rede local | `255.255.255.0` |
| DNS do cliente | Precisa apontar pro servidor, senão o domínio não é localizado | `192.168.56.10` |
| Perfil de rede (Pública x Privada) | O Windows aplica regras de Firewall diferentes conforme o perfil | Precisou mudar para "Privada" para liberar as portas do AD |

**Comandos de diagnóstico:**
- `ping <IP>` — testa comunicação básica entre duas máquinas
- `nslookup <domínio>` — testa se a resolução de nomes (DNS) está funcionando
- `ipconfig` — mostra a configuração de rede atual
- `gpupdate /force` — força aplicação imediata de políticas de grupo

---

## 3.5. Windows Server vs. Windows 11

### Propósito de design

| Aspecto | Windows Server | Windows 11 |
|---|---|---|
| Público-alvo | Administradores de TI | Usuários finais |
| Uso típico | Serviços 24/7 em segundo plano (AD, DNS, DHCP, arquivos) | Produtividade pessoal |
| Interface | Pode rodar sem interface gráfica (Server Core) | Sempre com interface gráfica completa |
| Ciclo de reinício | Projetado para longos períodos ligado | Reinicia com frequência normal |

### Diferenças técnicas relevantes

- **Funções (Roles) e Recursos (Features):** exclusivo do Windows Server — é dali que se instala o AD DS e o DNS Server.
- **Requisito de hardware:** Windows Server 2022 rodou com **2GB de RAM**; Windows 11 exigiu no mínimo **4GB**, além de TPM 2.0, EFI e Secure Boot habilitados. O Server é otimizado para serviços leves em segundo plano; o Windows 11 carrega uma camada de interface e recursos muito mais pesada.
- **Licenciamento:** Windows Server usa modelo por núcleos de processador/usuários (CALs); Windows 11 usa licença única.

### Arquitetura do ambiente

```
DC01 (Windows Server)          CLI01 (Windows 11)
     │                                │
     │  hospeda o AD, DNS,            │  representa o
     │  autentica logins,             │  "funcionário":
     │  aplica GPOs                   │  faz login, recebe
     │                                │  políticas do servidor
     └──────────── rede interna ──────┘
```

Windows Server e Windows 11 não são "melhor/pior" — são ferramentas com propósitos diferentes.

---

## 4. Ingresso de um cliente no domínio

Ao ingressar um computador no domínio (`sysdm.cpl` → Change → Domain), ele passa a:
- Aceitar login com credenciais do Active Directory (ex: `ADRIANA\joao.silva`)
- Poder receber políticas de grupo (GPOs) do servidor
- Ficar sujeito às regras de segurança centralizadas

**Requisito técnico:** o cliente precisa resolver o domínio via DNS — configuração de rede (IP fixo + DNS apontando pro servidor) é sempre o primeiro ponto a verificar se o ingresso falhar.

---

## 5. GPO (Group Policy Object)

### O que é
Conjunto de configurações que o Active Directory aplica automaticamente a usuários e/ou computadores, sem necessidade de configuração manual máquina por máquina. Regra centralizada que "desce" do servidor — diferença entre gerenciar 3 computadores manualmente e gerenciar 3.000 automaticamente.

### As duas metades de toda GPO

| Seção | Quando se aplica | Exemplo usado no projeto |
|---|---|---|
| **Computer Configuration** | Aplicada ao computador, independente de quem faça login | Bloqueio de tela |
| **User Configuration** | Aplicada ao usuário, "segue" ele para qualquer computador do domínio | Não usada neste projeto (ex de uso: mapear unidade de rede) |

### Estrutura da GPO no projeto

```
Domínio (adriana.local)
   └── OU (Suporte)
         └── GPO vinculada (GPO_Bloqueio_Tela)
               └── Computer Configuration
                     └── Policies → Administrative Templates
                           └── Control Panel → Personalization
                                 ├── Screen saver timeout
                                 └── Password protect the screen saver
```

### Herança de GPOs
GPOs seguem hierarquia: uma vinculada no domínio inteiro afeta todas as OUs abaixo; uma vinculada numa OU específica afeta só ela. Por isso a GPO do projeto foi vinculada especificamente na OU Suporte, restringindo o efeito só a esse departamento.

### Ciclo de aplicação
1. Por padrão, o Windows reaplica GPOs a cada **90 minutos** (variação aleatória de até 30 min).
2. `gpupdate /force` pula essa espera e força verificação imediata.
3. Algumas configurações só entram em vigor após reinício ou logoff/login.

### Relevância em escala
Numa empresa com 500 computadores, aplicar uma regra nova (ex: bloqueio de tela por compliance) sem GPO exigiria configurar máquina por máquina. Com GPO, a configuração é feita uma vez no servidor e se propaga automaticamente.

---

## 6. Boas práticas identificadas

- Nomenclatura padronizada (ex: `GG_Suporte_TI`) facilita administração em ambientes com muitos grupos/usuários.
- `ping`/`nslookup` antes de assumir causas complexas — a maioria dos erros de "não consigo ingressar no domínio" são, na raiz, problemas simples de rede/DNS.
- Perfil de rede "Privada" é necessário sempre que se espera que o Firewall libere serviços internos como AD e compartilhamento de arquivos.
- RAM é fator crítico ao virtualizar múltiplos servidores — Windows Server e Windows 11 têm requisitos mínimos diferentes, e ultrapassar a capacidade do hardware causa instabilidade.

---

## 7. Perguntas de revisão

1. Qual a diferença entre um Domain Controller e um computador cliente ingressado no domínio?
2. Por que o DNS é essencial para o Active Directory funcionar?
3. O que aconteceria com um usuário criado fora de qualquer OU?
4. Para que serve um Grupo de Segurança, e por que não aplicar permissões usuário por usuário?
5. Diante de um cliente que não consegue ingressar no domínio, quais os 3 primeiros comandos de diagnóstico?
6. O que é uma GPO e onde ela é configurada?
7. Por que a "Rede Pública" no Windows pode impedir a comunicação com o Active Directory?
8. Por que o Windows Server roda com menos RAM que o Windows 11, mesmo sendo tecnicamente mais robusto em funções?
9. Qual a diferença entre configurar uma GPO em "Computer Configuration" e em "User Configuration"?
10. Se uma GPO for vinculada diretamente no domínio (em vez de numa OU específica), o que muda no alcance dela?
11. Por que o comando `gpupdate /force` é útil durante testes, mesmo o Windows já reaplicando GPOs automaticamente?

---

## 8. Tópicos para aprofundar

- DHCP (atribuição automática de IPs)
- Compartilhamento de arquivos e permissões NTFS
- Políticas de senha (complexidade, expiração, histórico)
- Múltiplos Domain Controllers e replicação
- Grupos aninhados e escopos (Domain Local, Global, Universal)
