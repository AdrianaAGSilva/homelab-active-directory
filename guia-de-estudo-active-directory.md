# Guia de Estudo — Active Directory

Este guia resume os conceitos que você já colocou em prática no seu homelab. Use como material de revisão antes de entrevistas ou para relembrar o "porquê" por trás de cada passo.

---

## 1. Conceitos fundamentais

### O que é Active Directory (AD)?
Serviço da Microsoft que centraliza o gerenciamento de usuários, computadores, permissões e políticas de segurança de uma rede corporativa — o "cérebro" que controla quem pode acessar o quê.

### Domain Controller (DC)
O servidor que hospeda o Active Directory. É ele quem autentica logins, aplica políticas e mantém o banco de dados central do domínio. No seu projeto: **DC01-Servidor**.

### Domínio
O "nome" da rede administrada pelo AD (ex: `adriana.local`). Todo usuário, computador e grupo pertence a um domínio.

### DNS (Domain Name System)
Serviço que "traduz" nomes (como `adriana.local`) para endereços IP. O Active Directory **depende** do DNS para funcionar — é assim que um computador cliente "encontra" o servidor do domínio.

---

## 2. Estrutura organizacional

### OU (Organizational Unit)
Uma "pasta" dentro do domínio usada para organizar usuários, computadores e grupos por departamento, função ou localização. Permite aplicar regras (GPOs) e delegar permissões por grupo, em vez de configurar item por item.
> No projeto: OUs **TI**, **RH**, **Financeiro**, **Suporte**.

### Usuários
Contas individuais dentro de uma OU, representando pessoas (funcionários fictícios, no seu caso: João Silva, Maria Souza, Pedro Santos).

### Grupos de segurança
Reúnem vários usuários para aplicar permissões ou políticas de uma vez só, em vez de configurar cada usuário separadamente.
> No projeto: `GG_Suporte_TI` — o prefixo "GG" indica "Global Group", uma convenção de nomenclatura comum em ambientes profissionais.

---

## 3. Rede: pré-requisito para tudo funcionar

| Conceito | O que é | No seu projeto |
|---|---|---|
| IP fixo | Endereço que não muda, essencial para o servidor ser sempre "encontrado" no mesmo lugar | `192.168.56.10` (servidor), `192.168.56.20` (cliente) |
| Máscara de sub-rede | Define o "tamanho" da rede local | `255.255.255.0` |
| DNS do cliente apontando pro servidor | Sem isso, o cliente não consegue localizar o domínio | Cliente configurado com DNS `192.168.56.10` |
| Perfil de rede (Pública x Privada) | O Windows aplica regras de Firewall diferentes conforme o perfil | Precisou mudar de "Pública" para "Privada" para liberar as portas do AD |

**Ferramentas de diagnóstico que você usou:**
- `ping <IP>` — testa se dois computadores conseguem se comunicar na rede.
- `nslookup <domínio>` — testa se a resolução de nomes (DNS) está funcionando.
- `ipconfig` — mostra a configuração de rede atual de uma máquina.
- `gpupdate /force` — força a aplicação imediata de políticas de grupo, sem esperar o ciclo automático.

---

## 3.5. Windows Server vs. Windows 11 — comparação em profundidade

Entender essa diferença a fundo é importante porque mostra que você compreende **papéis distintos** dentro de uma infraestrutura, não só "dois tipos de Windows".

### Propósito de design

| Aspecto | Windows Server | Windows 11 |
|---|---|---|
| Público-alvo | Administradores de TI, gerenciando a rede | Usuários finais, no dia a dia |
| Uso típico | Rodar serviços 24/7 em segundo plano (AD, DNS, DHCP, arquivos, e-mail) | Produtividade pessoal: navegar, editar documentos, jogos |
| Interface | Pode rodar **sem interface gráfica** (Server Core) — só linha de comando | Sempre com interface gráfica completa |
| Ciclo de reinício | Projetado para ficar ligado por longos períodos, com o mínimo de reinicializações | Reinicia com frequência normal (atualizações, uso do dia a dia) |

### Diferenças técnicas relevantes

- **Funções (Roles) e Recursos (Features):** só o Windows Server tem o conceito de "Adicionar Funções e Recursos" (Add Roles and Features) — é dali que instalamos o AD DS e o DNS Server. O Windows 11 não tem esse papel de "servidor de infraestrutura".
- **Licenciamento:** Windows Server é licenciado por núcleos de processador/usuários que acessam (CALs), diferente do modelo de licença única do Windows 11.
- **Requisitos de hardware:** curiosamente, o Windows Server 2022 rodou bem com **2GB de RAM** no seu lab, enquanto o Windows 11 **exige no mínimo 4GB** — porque o Server é otimizado para rodar serviços leves em segundo plano, enquanto o Windows 11 carrega uma camada de interface e recursos para o usuário muito mais pesada (TPM 2.0, Secure Boot, Windows Hello, etc. — tudo isso você viu na prática ao configurar a VM CLI01).
- **Atualizações e suporte:** Windows Server tem ciclos de suporte muito mais longos (geralmente 10 anos) comparado ao Windows 11 (ciclos mais curtos por versão).

### Na prática, no seu projeto

```
DC01 (Windows Server)          CLI01 (Windows 11)
     │                                │
     │  hospeda o AD, DNS,            │  representa o
     │  autentica logins,             │  "funcionário":
     │  aplica GPOs                   │  faz login, recebe
     │                                │  políticas do servidor
     └──────────── rede interna ──────┘
```

O Windows Server **não é "melhor"** que o Windows 11 de forma genérica — são ferramentas com propósitos diferentes. Um recrutador vai valorizar que você entende **quando usar cada um**, não que um é superior ao outro.

---

## 4. Ingresso de um cliente no domínio

Quando um computador "entra" no domínio (via `sysdm.cpl` → Change → Domain), ele passa a:
- Aceitar login com credenciais do Active Directory (ex: `ADRIANA\joao.silva`) em vez de só contas locais.
- Poder receber políticas de grupo (GPOs) definidas pelo administrador do domínio.
- Ficar sujeito às regras de segurança centralizadas da "empresa".

**Requisito técnico:** o cliente precisa conseguir resolver o domínio via DNS — por isso a configuração de rede (IP fixo + DNS apontando pro servidor) é sempre o primeiro ponto a verificar quando o ingresso falha.

---

## 5. GPO (Group Policy Object) — Diretiva de Grupo, em profundidade

### O que é, de fato

Uma GPO é um **conjunto de configurações** que o Active Directory aplica automaticamente a usuários e/ou computadores, sem que ninguém precise ir máquina por máquina configurando manualmente. É a ferramenta que permite a um administrador dizer, por exemplo: "todo computador da OU Suporte deve bloquear a tela depois de 5 minutos" — e isso se propaga sozinho para todas as máquinas daquela OU.

Pense nela como uma **regra centralizada** que "desce" do servidor para as máquinas, em vez de alguém precisar configurar cada computador individualmente — essa é literalmente a diferença entre gerenciar 3 computadores manualmente e gerenciar 3.000 automaticamente.

### As duas metades de toda GPO

Toda GPO é dividida em duas grandes seções, e é importante saber quando usar cada uma:

| Seção | Quando se aplica | Exemplo |
|---|---|---|
| **Computer Configuration** | Aplicada ao **computador**, independente de quem faça login nele | Configurações de firewall, serviços do Windows, tela de bloqueio (como no seu projeto) |
| **User Configuration** | Aplicada ao **usuário**, e o "segue" para qualquer computador do domínio em que ele logar | Papel de parede padrão, unidades de rede mapeadas, restrições de menu Iniciar |

No seu projeto, você usou **Computer Configuration** para o bloqueio de tela — por isso a regra vale para qualquer usuário que logar naquele computador específico da OU Suporte.

### Onde as GPOs "moram" dentro do AD

```
Domínio (adriana.local)
   └── OU (Suporte)
         └── GPO vinculada (GPO_Bloqueio_Tela)
               ├── Computer Configuration
               │     └── Policies → Administrative Templates
               │           └── Control Panel → Personalization
               │                 ├── Screen saver timeout
               │                 └── Password protect the screen saver
               └── User Configuration (não usada nesse caso)
```

### Herança de GPOs (um conceito que costuma cair em entrevista)

GPOs seguem uma **hierarquia de herança**: uma GPO vinculada no nível do domínio afeta *todas* as OUs abaixo dele; uma GPO vinculada numa OU específica afeta só aquela OU (e subpastas dela, se houver). Se houver conflito entre regras, a mais "próxima" da OU final geralmente prevalece (a não ser que a GPO de nível superior esteja marcada como "Enforced").

No seu projeto: se você tivesse criado a GPO no nível do domínio inteiro, ela afetaria também TI, RH e Financeiro — por isso ela foi vinculada especificamente na OU **Suporte**, para restringir o efeito só ao departamento desejado.

### Como a aplicação acontece na prática

1. Por padrão, o Windows verifica e reaplica GPOs automaticamente a cada **90 minutos** (com uma variação aleatória de até 30 min para evitar que todos os PCs consultem o servidor ao mesmo tempo).
2. O comando `gpupdate /force` (que você usou) **pula essa espera** e força a verificação imediatamente — extremamente útil para testar se uma GPO nova está funcionando sem precisar esperar quase 2 horas.
3. Algumas configurações (como as de segurança) só entram em vigor completamente após um **reinício** do computador ou **logoff/login** do usuário — foi por isso que reiniciamos o cliente depois do `gpupdate /force`.

### Por que isso importa numa empresa real

Imagine uma empresa com 500 computadores. Sem GPO, aplicar uma nova regra de segurança (ex: "toda tela deve bloquear em 5 minutos por questões de compliance") exigiria visitar computador por computador. Com GPO, o administrador configura **uma vez**, no servidor, e a regra se propaga sozinha para todas as máquinas certas — é exatamente esse tipo de automação/centralização que você demonstrou saber fazer no seu projeto.

> No projeto: `GPO_Bloqueio_Tela`, vinculada à OU Suporte, forçando bloqueio automático de tela após 5 minutos de inatividade — simulando uma política de segurança corporativa real, validada com sucesso através de `gpupdate /force` e teste prático de bloqueio.

---

## 6. Boas práticas percebidas no projeto

- **Nomenclatura padronizada** (ex: `GG_Suporte_TI`) facilita a administração em ambientes com muitos grupos/usuários.
- **Testar com `ping`/`nslookup` antes de assumir que o problema é complexo** — grande parte dos erros de "não consigo ingressar no domínio" são, na raiz, problemas simples de rede/DNS.
- **Perfil de rede "Privada" é necessário** sempre que se espera que o Firewall do Windows libere serviços internos como AD e compartilhamento de arquivos.
- **RAM é um fator crítico** ao virtualizar múltiplos servidores — Windows Server e Windows 11 têm requisitos mínimos diferentes, e rodar VMs além da capacidade do hardware causa instabilidade.

---

## 7. Revisão ativa

1. Qual a diferença entre um Domain Controller e um computador cliente ingressado no domínio?
2. Por que o DNS é essencial para o Active Directory funcionar?
3. O que aconteceria se você criasse um usuário fora de qualquer OU?
4. Para que serve um Grupo de Segurança, e por que não aplicar permissões usuário por usuário?
5. Se um cliente não consegue ingressar no domínio, quais os 3 primeiros comandos que você rodaria para diagnosticar?
6. O que é uma GPO e onde ela é configurada?
7. Por que a "Rede Pública" no Windows pode impedir a comunicação com o Active Directory?
8. Por que o Windows Server consegue rodar com menos RAM que o Windows 11, mesmo sendo tecnicamente mais "robusto" em funções?
9. Qual a diferença entre configurar uma GPO em "Computer Configuration" e em "User Configuration"?
10. Se você vincular uma GPO diretamente no domínio (em vez de numa OU específica), o que muda no alcance dela?
11. Por que o comando `gpupdate /force` é útil durante testes, mesmo sabendo que o Windows já reaplica GPOs automaticamente?

---
