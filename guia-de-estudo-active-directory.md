# 📚 Minhas Anotações — Active Directory (projeto homelab)

Anotações que fiz enquanto montava meu próprio ambiente de Active Directory do zero. Vou usar isso pra revisar antes de entrevistas ou sempre que precisar relembrar algum conceito.

---

## 1. Conceitos fundamentais que aprendi

### O que é Active Directory (AD)
É o serviço da Microsoft que centraliza o gerenciamento de usuários, computadores, permissões e políticas de segurança de uma rede corporativa. É o "cérebro" que controla quem pode acessar o quê.

### Domain Controller (DC)
É o servidor que hospeda o Active Directory. Ele autentica logins, aplica políticas e mantém o banco de dados central do domínio. No meu projeto: **DC01-Servidor**.

### Domínio
É o "nome" da rede administrada pelo AD (no meu caso, `adriana.local`). Todo usuário, computador e grupo que criei pertence a esse domínio.

### DNS (Domain Name System)
Serviço que "traduz" nomes (como `adriana.local`) para endereços IP. Aprendi na prática que o Active Directory **depende** do DNS pra funcionar — foi exatamente isso que travou meu cliente na hora de entrar no domínio, até eu configurar o DNS certo.

---

## 2. Estrutura organizacional que criei

### OU (Organizational Unit)
É uma "pasta" dentro do domínio que uso pra organizar usuários, computadores e grupos por departamento, função ou localização. Permite aplicar regras (GPOs) e delegar permissões por grupo, em vez de configurar item por item.
> No meu projeto: criei as OUs **TI**, **RH**, **Financeiro**, **Suporte**.

### Usuários
São contas individuais dentro de uma OU, representando pessoas. Criei os usuários fictícios: João Silva, Maria Souza, Pedro Santos.

### Grupos de segurança
Servem pra reunir vários usuários e aplicar permissões ou políticas de uma vez só, em vez de configurar cada usuário separadamente.
> Criei o grupo `GG_Suporte_TI` — o prefixo "GG" indica "Global Group", uma convenção de nomenclatura que aprendi ser comum em ambientes profissionais.

---

## 3. Rede: o que precisei acertar pra tudo funcionar

| Conceito | O que é | Como configurei |
|---|---|---|
| IP fixo | Endereço que não muda, essencial pro servidor ser sempre encontrado no mesmo lugar | `192.168.56.10` (servidor), `192.168.56.20` (cliente) |
| Máscara de sub-rede | Define o "tamanho" da rede local | `255.255.255.0` |
| DNS do cliente apontando pro servidor | Sem isso, o cliente não localiza o domínio | Configurei o DNS do cliente como `192.168.56.10` |
| Perfil de rede (Pública x Privada) | O Windows aplica regras de Firewall diferentes conforme o perfil | Precisei mudar de "Pública" pra "Privada" pra liberar as portas do AD — foi um dos perrengues que resolvi |

**Ferramentas de diagnóstico que usei de verdade:**
- `ping <IP>` — testei se as duas VMs conseguiam se comunicar na rede.
- `nslookup <domínio>` — testei se a resolução de nomes (DNS) estava funcionando.
- `ipconfig` — conferi a configuração de rede de cada máquina.
- `gpupdate /force` — forcei a aplicação imediata da política de grupo, sem esperar o ciclo automático.

---

## 3.5. Windows Server vs. Windows 11 — o que percebi na prática

Isso ficou bem mais claro depois que eu configurei os dois na mão.

### Propósito de design

| Aspecto | Windows Server | Windows 11 |
|---|---|---|
| Público-alvo | Administradores de TI, gerenciando a rede | Usuários finais, no dia a dia |
| Uso típico | Rodar serviços 24/7 em segundo plano (AD, DNS, DHCP, arquivos) | Produtividade pessoal |
| Interface | Pode rodar sem interface gráfica (Server Core) | Sempre com interface gráfica completa |
| Ciclo de reinício | Projetado pra ficar ligado por longos períodos | Reinicia com frequência normal |

### Diferenças técnicas que vivenciei

- **Funções (Roles) e Recursos (Features):** só o Windows Server tem esse conceito de "Adicionar Funções e Recursos" — foi de lá que instalei o AD DS e o DNS Server. O Windows 11 não tem esse papel de servidor de infraestrutura.
- **Requisito de hardware — o detalhe que mais me chamou atenção:** o Windows Server 2022 rodou de boa com **2GB de RAM** no meu lab, enquanto o Windows 11 **exigiu no mínimo 4GB** — e ainda travou até eu ajustar TPM 2.0, EFI e Secure Boot na VM. Isso mostrou na prática que o Server é otimizado pra rodar serviços leves em segundo plano, enquanto o Windows 11 carrega uma camada de interface e recursos pro usuário muito mais pesada.
- **Licenciamento:** Windows Server é licenciado por núcleos de processador/usuários que acessam (CALs), diferente do modelo de licença única do Windows 11.

### Como ficou meu ambiente

```
DC01 (Windows Server)          CLI01 (Windows 11)
     │                                │
     │  hospeda o AD, DNS,            │  representa o
     │  autentica logins,             │  "funcionário":
     │  aplica GPOs                   │  faz login, recebe
     │                                │  políticas do servidor
     └──────────── rede interna ──────┘
```

Entendi que o Windows Server não é "melhor" que o Windows 11 de forma genérica — são ferramentas com propósitos diferentes. O importante é saber quando usar cada um.

---

## 4. Como ingressei o cliente no domínio

Quando um computador "entra" no domínio (fiz isso via `sysdm.cpl` → Change → Domain), ele passa a:
- Aceitar login com credenciais do Active Directory (usei `ADRIANA\joao.silva`) em vez de só contas locais.
- Poder receber políticas de grupo (GPOs) definidas no servidor.
- Ficar sujeito às regras de segurança centralizadas da "empresa".

**O que aprendi sobre o requisito técnico:** o cliente precisa conseguir resolver o domínio via DNS — por isso a configuração de rede (IP fixo + DNS apontando pro servidor) é sempre o primeiro ponto que vou verificar se o ingresso falhar de novo no futuro.

---

## 5. GPO (Group Policy Object) — o que entendi a fundo

### O que é, no fim das contas

É um conjunto de configurações que o Active Directory aplica automaticamente a usuários e/ou computadores, sem que eu precise ir máquina por máquina configurando manualmente. Consegui dizer, por exemplo: "todo computador da OU Suporte deve bloquear a tela depois de 5 minutos" — e isso se propagou sozinho.

Entendi que é uma regra centralizada que "desce" do servidor pra máquina, em vez de eu precisar configurar cada computador individualmente — essa é a diferença entre gerenciar 3 computadores manualmente e gerenciar 3.000 automaticamente.

### As duas metades de toda GPO

| Seção | Quando se aplica | Exemplo que usei |
|---|---|---|
| **Computer Configuration** | Aplicada ao computador, não importa quem faça login | Foi essa que usei pra tela de bloqueio |
| **User Configuration** | Aplicada ao usuário, e "segue" ele pra qualquer computador do domínio | Não usei essa dessa vez, mas serviria pra ex: mapear unidade de rede |

Usei **Computer Configuration** pro bloqueio de tela — por isso a regra vale pra qualquer usuário que logar naquele computador específico da OU Suporte.

### Onde a minha GPO ficou dentro do AD

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

### Herança de GPOs (conceito importante pra entrevista)

Aprendi que GPOs seguem uma hierarquia: uma GPO vinculada no domínio inteiro afeta *todas* as OUs abaixo; uma GPO vinculada numa OU específica afeta só ela. Por isso vinculei minha GPO especificamente na OU **Suporte** — se eu tivesse feito no domínio inteiro, teria afetado TI, RH e Financeiro também, o que não era a intenção.

### Como a aplicação acontece na prática

1. Por padrão, o Windows reaplica GPOs a cada **90 minutos** (com uma variação aleatória de até 30 min).
2. Usei `gpupdate /force` pra pular essa espera e testar na hora, sem precisar esperar quase 2 horas.
3. Aprendi que algumas configurações só entram em vigor completamente depois de um reinício ou logoff/login — por isso reiniciei o cliente depois do `gpupdate /force`.

### Por que isso importa numa empresa de verdade

Imaginando uma empresa com 500 computadores: sem GPO, aplicar uma regra nova (tipo "toda tela bloqueia em 5 min por compliance") exigiria visitar computador por computador. Com GPO, configuro uma vez no servidor e a regra se propaga sozinha. Foi exatamente esse tipo de automação que consegui demonstrar no meu projeto.

---

## 6. Boas práticas que percebi no caminho

- Nomenclatura padronizada (tipo `GG_Suporte_TI`) facilita bastante a administração quando tem muitos grupos/usuários.
- Testar com `ping`/`nslookup` antes de assumir que o problema é complexo — a maioria dos meus erros de "não consigo ingressar no domínio" eram, na raiz, problemas simples de rede/DNS.
- Perfil de rede "Privada" é necessário sempre que espero que o Firewall do Windows libere serviços internos como AD e compartilhamento de arquivos.
- RAM é um fator crítico quando estou virtualizando múltiplos servidores — descobri isso na prática quando meu PC travou rodando as duas VMs juntas com só 8GB de RAM.

---

## 7. Perguntas que fiz pra mim mesma (revisão ativa)

1. Qual a diferença entre um Domain Controller e um computador cliente ingressado no domínio?
2. Por que o DNS é essencial pro Active Directory funcionar?
3. O que aconteceria se eu criasse um usuário fora de qualquer OU?
4. Pra que serve um Grupo de Segurança, e por que não aplicar permissões usuário por usuário?
5. Se um cliente não consegue ingressar no domínio, quais os 3 primeiros comandos que eu rodaria pra diagnosticar?
6. O que é uma GPO e onde ela é configurada?
7. Por que a "Rede Pública" no Windows pode impedir a comunicação com o Active Directory?
8. Por que o Windows Server consegue rodar com menos RAM que o Windows 11, mesmo sendo tecnicamente mais robusto em funções?
9. Qual a diferença entre configurar uma GPO em "Computer Configuration" e em "User Configuration"?
10. Se eu vincular uma GPO diretamente no domínio (em vez de numa OU específica), o que muda no alcance dela?
11. Por que o comando `gpupdate /force` é útil durante testes, mesmo sabendo que o Windows já reaplica GPOs automaticamente?

---

## 8. Próximos tópicos que quero aprofundar

- DHCP (atribuição automática de IPs)
- Compartilhamento de arquivos e permissões NTFS
- Políticas de senha (complexidade, expiração, histórico)
- Múltiplos Domain Controllers e replicação
- Grupos aninhados e escopos (Domain Local, Global, Universal)
