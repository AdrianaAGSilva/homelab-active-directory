# 🖥️ Homelab de Active Directory — Ambiente Corporativo Simulado

## 🎯 Objetivo do projeto

Simular, do zero, um ambiente de infraestrutura corporativa real utilizando **Windows Server** (Domain Controller) e **Windows 11** (cliente ingressado no domínio), reproduzindo tarefas do dia a dia de um analista de suporte/infraestrutura: gestão de usuários, grupos, políticas de segurança e resolução de problemas de rede.

## 🏗️ Arquitetura

```
┌─────────────────────────┐         ┌─────────────────────────┐
│      DC01-Servidor       │         │      CLI01-Cliente       │
│   Windows Server 2022    │◄───────►│       Windows 11          │
│   Domain Controller      │  Rede   │   Ingressado no domínio   │
│   IP: 192.168.56.10      │ Interna │   IP: 192.168.56.20       │
│   Domínio: adriana.local │ (intnet)│   DNS: 192.168.56.10      │
└─────────────────────────┘         └─────────────────────────┘
        Virtualizado com Oracle VirtualBox
```

## ✅ O que foi implementado

- Instalação e configuração do **Windows Server 2022** como Controlador de Domínio (Active Directory Domain Services + DNS Server)
- Criação do domínio **`adriana.local`**
- Estrutura organizacional com 4 **Unidades Organizacionais (OUs)**: TI, RH, Financeiro e Suporte
- Criação de **usuários fictícios** (João Silva, Maria Souza, Pedro Santos) na OU Suporte
- Criação de **grupo de segurança** (`GG_Suporte_TI`) e associação dos usuários
- Configuração de **IP fixo** e rede interna isolada entre servidor e cliente
- Instalação do **Windows 11 Enterprise** na máquina cliente
- **Ingresso do cliente no domínio** `adriana.local`
- Login validado com credenciais de domínio (`ADRIANA\joao.silva`)

## 📸 Screenshots

> *Prints de cada etapa serão adicionados aqui em breve.*

| Etapa | Screenshot |
|---|---|
| Servidor promovido a Domain Controller | `screenshots/01-dc-promovido.png` |
| OUs criadas (TI, RH, Financeiro, Suporte) | `screenshots/02-ous-criadas.png` |
| Usuários e grupo de segurança criados | `screenshots/03-usuarios-grupos.png` |
| Cliente ingressado no domínio | `screenshots/04-cliente-no-dominio.png` |
| Login com usuário de domínio | `screenshots/05-login-dominio.png` |

## 🧩 Desafios e aprendizados

Documentar os perrengues faz parte do processo — foram eles que geraram o aprendizado mais real:

- **Requisitos de hardware do Windows 11:** a instalação falhou inicialmente por falta de RAM mínima (4GB) e exigiu habilitar TPM 2.0 virtual na configuração da VM.
- **Limitações de hardware do host:** rodar duas VMs simultaneamente em uma máquina com 8GB de RAM causou travamentos recorrentes — foi necessário migrar os arquivos das VMs (`.vbox`/`.vdi`) para um computador com mais recursos, através de exportação/importação no VirtualBox.
- **Rede "Pública" bloqueando o domínio:** o cliente não conseguia localizar o domínio via DNS porque o Windows classificava a rede interna como "Rede Pública" — o Firewall bloqueava as portas necessárias até a rede ser reclassificada como "Privada".
- **Diagnóstico de conectividade:** uso de `ping` e `nslookup` para isolar o problema entre falha de rede básica vs. falha específica de DNS antes de conseguir ingressar o cliente no domínio com sucesso.

## 🛠️ Tecnologias utilizadas

- Windows Server 2022 (Standard, Desktop Experience)
- Windows 11 Enterprise (Evaluation)
- Oracle VirtualBox
- Active Directory Domain Services (AD DS)
- DNS Server
- PowerShell

## 🚀 Próximos passos

- [ ] Aplicar Diretiva de Grupo (GPO) de bloqueio de tela na OU Suporte
- [ ] Adicionar servidor DHCP
- [ ] Configurar compartilhamento de arquivos entre servidor e cliente
- [ ] Implementar políticas de senha mais robustas
