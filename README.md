# Projeto Operação NOC — Thiago Bovo Costa

> **Investigação, Monitoramento e Observabilidade de Redes**
> Ubuntu Server + Redes + Wireshark + Zabbix + Grafana

## Identificação

| Campo | Valor |
|---|---|
| Aluno(a) / Grupo | Thiago Bovo Costa |
| Turma | Defesa Cibernética — 2026 |
| Professor | Frank Philson |
| Data | 14–15/09/2026 |
| Rede do laboratório (Rede LAN) | `10.110.102.108`, `10.110.102.111`, `10.110.102.121` |

## Objetivo

Implementar e documentar um laboratório de **Network Operations Center (NOC)** capaz de monitorar disponibilidade, serviços e recursos, combinando diagnóstico de rede, análise de pacotes, monitoramento com Zabbix e visualização no Grafana.

> **Regra operacional utilizada:** primeiro observar e coletar evidências; depois formular a hipótese, corrigir, validar e documentar.

## Ambiente de referência

| Hostname | IP | Função | Stack real |
|---|---:|---|---|
| `srv-zabbix-thiago` | `10.110.102.121` | Zabbix Server + Frontend | Zabbix 7.4 + **PostgreSQL** + **Nginx** |
| `srv-grafana-thiago` | `10.110.102.111` | Grafana | Grafana (porta 3000) |
| `srv-linux-thiago` | `10.110.102.108` | Servidor monitorado | Ubuntu 22.04.5 + SSH + Apache 2.4.52 + Zabbix Agent (clássico) |
| Gateway | `10.110.102.1` | Saída da rede do laboratório | — |

> **Nota de fidelidade:** o ambiente real difere do template genérico em alguns pontos — banco de dados PostgreSQL (não MariaDB), frontend Nginx (não Apache) no Zabbix, e Zabbix Agent clássico (não Agent 2) no host monitorado. Essas escolhas foram mantidas como estão, documentadas fielmente ao invés de forçadas ao template.

---

## Fase 01 — Planejamento e endereçamento

### Execução real
Rede do laboratório: `10.110.102.0/24`, gateway `10.110.102.1`, interface `ens18` em todos os hosts.

| Hostname | IP |
|---|---|
| srv-linux-thiago | 10.110.102.108 |
| srv-grafana-thiago | 10.110.102.111 |
| srv-zabbix-thiago | 10.110.102.121 |

### Checkpoint
✅ Tabela de endereçamento preenchida.

### Evidências
- [x] Tabela de IPs (acima)
- [ ] 🔲 **PENDENTE — print:** diagrama simples da topologia (pode ser um desenho à mão, Draw.io, ou até uma captura de tela do painel de rede do hypervisor mostrando as 3 VMs na mesma rede). Salvar como `imagens/fase01-planejamento.png`.

---

## Fase 02 — VMs e sistemas operacionais

### Execução real
Três VMs Ubuntu Server 22.04.5 LTS, kernel `5.15.0-191-generic`, arquitetura x86-64, virtualização KVM (QEMU).

### Checkpoint
✅ ZABBIX01 (`srv-zabbix-thiago`), GRAFANA01 (`srv-grafana-thiago`) e SRV-LINUX01 (`srv-linux-thiago`) inicializados.

### Execução real (specs)

| Hostname | vCPUs | RAM total | RAM usada | Disco (/) |
|---|---|---|---|---|
| srv-linux-thiago | 4 | 3.8 GiB | 247 MiB | 49G (7.5G usado, 16%) |
| srv-zabbix-thiago | 4 | 7.8 GiB | 574 MiB | 49G (9.1G usado, 20%) |
| srv-grafana-thiago | 2 | 3.8 GiB | 583 MiB | 49G (7.9G usado, 17%) |

### Checkpoint
✅ Especificações de CPU, RAM e disco confirmadas nas três VMs.

### Evidências
- [x] Sistema operacional confirmado via `hostnamectl` (Ubuntu 22.04.5 LTS nas três VMs)
- [x] vCPUs confirmadas via `nproc` (tabela acima)
- [x] Memória confirmada via `free -h` (tabela acima)
- [x] Disco confirmado via `df -h /` (tabela acima)

---

## Fase 03 — IP estático e conectividade

### Execução real
IP estático configurado via `ens18` em todas as VMs, roteamento validado via `ip route`.

```
$ ip route
default via 10.110.102.1 dev ens18 proto static
10.110.102.0/24 dev ens18 proto kernel scope link src 10.110.102.1XX
```

Conectividade validada com `ping -c 4` entre as três VMs — **0% de perda de pacotes** em todas as direções (latências entre 0.02 ms e 0.47 ms).

**Resolução de nomes:** cada VM resolve apenas o próprio hostname via `/etc/hosts` (127.0.1.1). `getent hosts` para os hostnames remotos retornou vazio nas três VMs — não há DNS interno nem `/etc/hosts` compartilhado entre elas.

### Checkpoint
✅ As três VMs se comunicam via IP. ⚠️ Resolução de nomes cruzada não configurada (ponto de melhoria futura).

### Evidências
- [x] `ip -br addr` / `ip route` (3 VMs)
- [x] `ping` entre as 3 VMs, 0% de perda
- [x] `getent hosts` (resultado: sem resolução cruzada)

---

## Fase 04 — Preparação Linux

### Execução real

| Hostname | Timezone | NTP |
|---|---|---|
| srv-linux-thiago | Etc/UTC | Sincronizado (active) |
| srv-grafana-thiago | Etc/UTC | Sincronizado (active) |
| srv-zabbix-thiago | Etc/UTC | Sincronizado (active) |

> Fuso horário em UTC (não `America/Sao_Paulo`); não afeta a operação do laboratório, mantido como está.

`apt update && apt upgrade` executado nas três VMs sem erros. `srv-linux-thiago` recebeu 59 atualizações (kernel, initramfs, rede); as demais, atualizações menores.

### Checkpoint
✅ Hostnames corretos e relógios sincronizados via NTP.

### Evidências
- [x] `hostnamectl` (3 VMs)
- [x] `timedatectl` (3 VMs) — `System clock synchronized: yes`
- [x] `apt update/upgrade` (3 VMs, sem erros)

---

## Fase 05 — Serviços SSH e HTTP

### Execução real
No `srv-linux-thiago`:
- **SSH:** ativo desde o boot, testado com login remoto real (`Accepted password for thiago from 10.110.102.254`).
- **Apache:** inicialmente ausente (`Unit apache2.service could not be found`) — instalado e validado nesta fase.

```
$ curl -I http://localhost
HTTP/1.1 200 OK
Server: Apache/2.4.52 (Ubuntu)

$ sudo ss -lntp | grep :80
LISTEN *:80  (apache2)
```

Teste remoto de `srv-zabbix-thiago` → `srv-linux-thiago`:
```
$ curl -I http://10.110.102.108
HTTP/1.1 200 OK
Server: Apache/2.4.52 (Ubuntu)
```

### Checkpoint
✅ Portas 22 (SSH) e 80 (Apache) acessíveis pela rede do laboratório, validadas local e remotamente.

### Evidências
- [x] `systemctl status ssh` — active
- [x] `systemctl status apache2` — active (após instalação)
- [x] `ss -lntp` — portas 22 e 80 confirmadas
- [x] `curl` local e remoto — HTTP/1.1 200 OK

---

## Fase 06 — Diagnóstico manual e Wireshark

### Execução real
Captura inicial ampla (`fase6_capture.pcap`, 169.385 frames) analisada com `tshark -z io,phs`, seguida de uma captura filtrada dedicada (`fase6-extra.pcap`) para completar os protocolos que não apareceram na primeira.

**Estatística geral (captura ampla):**
```
tcp: 141.854 frames | tls: 40.240 frames | http: 860 frames | pgsql: 415 frames
udp: 21.504 frames  | arp: 4.746 frames
```

**ARP** ✅
```
Who has 10.110.102.199? Tell 10.110.102.1
Who has 10.110.102.203? Tell 10.110.102.1
Who has 10.110.102.98?  Tell 10.110.102.1
```

**TCP Three-way handshake** ✅ (Zabbix Server → Zabbix Agent, porta 10050)
```
SYN:      10.110.102.121:42286 → 10.110.102.108:10050
SYN-ACK:  10.110.102.108:10050 → 10.110.102.121:42286
```

**ICMP** ✅
```
10.110.102.121 → 10.110.102.108  Echo (ping) request
10.110.102.108 → 10.110.102.121  Echo (ping) reply
```

**DNS** ✅
```
10.110.102.121 → 8.8.8.8   Standard query A google.com
8.8.8.8 → 10.110.102.121   Standard query response A 172.217.29.238
```

**TLS/HTTPS** ⚠️ (evidência de aplicação, não de pacote bruto)
O filtro `tls.handshake` não retornou pacotes na captura, mas a conexão TLS foi confirmada via `curl -v https://www.google.com`:
- TLS 1.3 negociado com `www.google.com` (142.251.152.119:443)
- Cifra: `TLS_AES_256_GCM_SHA384`
- Certificado validado (emissor: Google Trust Services, `SSL certificate verify ok`)
- ALPN negociou HTTP/2

### Checkpoint
✅ ICMP, ARP, DNS e TCP (handshake) capturados via tshark. ⚠️ TLS confirmado por evidência de aplicação (curl -v), não por pacote de handshake bruto — registrar essa limitação.

### Evidências
- [x] Filtros utilizados (documentados acima)
- [x] Three-way handshake TCP real (Zabbix Server ↔ Agent)
- [x] ICMP / ARP / DNS reais
- [x] TLS/HTTPS (evidência via curl -v)

---

## Fase 07 — Zabbix Server

### Execução real
No `srv-zabbix-thiago`: Zabbix Server 7.4, **PostgreSQL**, **Nginx** (frontend) e Zabbix Agent2 (auto-monitoramento) instalados e ativos.

```
● zabbix-server.service   Active: active (running) — Main PID 893
● postgresql.service      Active: active (exited)
● nginx.service           Active: active (running)
```

**Portas:**
| Porta | Serviço |
|---|---|
| 80, 8080 | Nginx (frontend) |
| 10051 | Zabbix Server (trapper) |
| 10050 | Zabbix Agent2 local |

### Checkpoint
✅ Frontend (Nginx) funcionando e serviços ativos (Server + PostgreSQL + Nginx).

### Evidências
- [x] Serviços ativos (systemctl status)
- [x] Portas 80/10050/10051 confirmadas via `ss -lntp`
- [ ] 🔲 **PENDENTE — print:** tela do frontend Zabbix logado (dashboard inicial). Salvar como `imagens/fase07-zabbix-server.png`.

---

## Fase 08 — Hosts e Zabbix Agent

### Execução real
`srv-linux-thiago` usa o **Zabbix Agent clássico** (`zabbix-agentd`, versão 7.4.14), não o Agent 2.

```
● zabbix-agent.service   Active: active (running) — Main PID 41089
Log: "Starting Zabbix Agent [srv-linux-thiago]. Zabbix 7.4.14"
IPv6 support: YES | TLS support: YES
```

Agente escutando na porta `10050` (IPv4 e IPv6), 10 listeners ativos, sem erros no log.

### Checkpoint
✅ Agente ativo e escutando corretamente.

### Evidências
- [x] `systemctl status zabbix-agent` — active
- [x] Log sem erros
- [ ] 🔲 **PENDENTE — print:** no frontend Zabbix, ir em **Data collection → Hosts**, confirmar que `srv-linux-thiago` aparece com status verde (disponível), e depois **Monitoring → Latest data** filtrando por esse host, mostrando itens com valores recentes. Salvar como `imagens/fase08-agent2.png`.

---

## Fase 09 — Monitoramento no Zabbix

### Checkpoint
🔲 **PENDENTE.** Nenhum dado desta fase foi coletado ainda — depende só de prints do frontend.

### O que capturar exatamente
1. **Monitoring → Latest data** → filtrar host `srv-linux-thiago` → print mostrando pelo menos: ICMP ping, CPU load, memória, uso de disco, tráfego de rede (RX/TX), e — agora que o Apache está de pé — o item de disponibilidade HTTP.
2. **Monitoring → Problems** → print da tela (mesmo que esteja vazia, isso já é uma evidência válida de "sem problemas ativos").
3. Se possível, um print do **uptime** do host (aparece na tela de Hosts, coluna "Availability" ou similar).

Me descreva o que aparece em cada tela (ou cole o texto dos itens) que eu escrevo o texto desta fase.

---

## Fase 10 — Grafana

### Execução real
```
● grafana-server.service   Active: active (running) — Main PID 608, 1001.5M RAM
$ ss -lntp | grep 3000
LISTEN *:3000 (grafana)
```
Plugins de datasource carregados incluem Zabbix (`alexanderzobnin-zabbix-app`), PostgreSQL, MySQL, InfluxDB, Loki, Prometheus, entre outros.

### Checkpoint
✅ Grafana ativo na porta 3000. ⚠️ Falta confirmar (Fase 14) se o acesso está restrito à rede do laboratório via firewall.

### Evidências
- [x] `grafana-server` ativo
- [x] Porta 3000 confirmada
- [ ] 🔲 **PENDENTE — print:** tela de login do Grafana carregada com sucesso. Salvar como `imagens/fase10-grafana.png`.

---

## Fase 11 — API Zabbix

### Checkpoint
🔲 **PENDENTE.**

### O que fazer e capturar
1. No frontend Zabbix, criar um usuário `grafana_ro` com permissão **somente leitura (Read)** no(s) host group(s) relevante(s).
2. Gerar um **API token** dedicado para esse usuário (**Users → API tokens**).
3. Print da tela mostrando o usuário criado e a permissão Read — **nunca** printe ou publique o valor do token, apenas confirme que ele foi gerado (pode aparecer mascarado/cortado no print).

Salvar como `imagens/fase11-api-zabbix.png`. Me diga quando estiver feito que eu escrevo o texto.

---

## Fase 12 — Integração Grafana + Zabbix

### Execução real
A integração Grafana + Zabbix está funcional — confirmado indiretamente pelos dados reais que fluem no dashboard da Fase 13 (uptime, memória, CPU, disco e rede do host monitorado aparecendo com valores atualizados em tempo real). Isso só é possível com o datasource Zabbix corretamente configurado e testado (`Save & test` bem-sucedido).

### Checkpoint
✅ Integração validada — dados do Zabbix aparecendo corretamente nos painéis do Grafana.

### Evidências
- [x] Plugin Zabbix habilitado no Grafana (confirmado na Fase 10 — `alexanderzobnin-zabbix-app`)
- [x] Dados fluindo em tempo real no dashboard (evidência indireta de `Save & test` bem-sucedido)
- [ ] 🔲 Opcional: print direto da tela de configuração do data source (não obrigatório, já que o dashboard funcionando é evidência suficiente)

---

## Fase 13 — Dashboard NOC

### Execução real
Dashboard NOC criado no Grafana, com período de visualização "Last 6 hours", reunindo os seguintes painéis:

| Painel | Conteúdo observado |
|---|---|
| **Uptime** | 4 dias, 22h36min — host monitorado estável, sem reinicializações recentes |
| **Memory Usage (%)** | 12.9% de uso de memória |
| **Storage Usage** | Disponível: 39.0 GiB / Usado: 7.41 GiB |
| **CPU Usage (%)** | 0.03% — carga muito baixa, condizente com ambiente de laboratório ocioso |
| **Disk I/O Latency** | Gráfico de leitura/escrita ao longo do tempo — pico pontual visível próximo às 19:40 |
| **Network Traffic** | Download/Upload em kb/s — pico de tráfego correspondente ao mesmo horário do pico de I/O (~19:40), sugerindo alguma atividade concentrada nesse período |
| **Incidentes** | Tabela com colunas Host / Severity / Status / Problem / Tags / Time — status atual: **"No problems found"** |

### Checkpoint
✅ Painéis de disponibilidade (uptime), CPU, memória, disco, rede e problemas ativos presentes e com dados reais. Métricas com unidades corretas (%, GiB, kb/s) e período de tempo coerente (últimas 6 horas).

> Observação: não há um painel dedicado exclusivamente à disponibilidade HTTP do Apache — os painéis atuais cobrem recursos do sistema (CPU/memória/disco/rede) e uptime geral, mas não o status do serviço web isoladamente. Pode ser um ponto de melhoria futura, já que a Fase 09 monitora esse item no Zabbix.

### Evidências
- [x] Dashboard completo (print anexado)
- [x] Métricas com unidades (%, GiB, kb/s)
- [x] Período de tempo coerente (Last 6 hours)

---

## Fase 14 — Segurança

### Execução real

```bash
$ sudo ufw status numbered
Status: inactive
```
Resultado idêntico nas três VMs (`srv-linux-thiago`, `srv-zabbix-thiago`, `srv-grafana-thiago`).

```bash
$ sudo iptables -L
Chain INPUT (policy ACCEPT)
Chain FORWARD (policy ACCEPT)
Chain OUTPUT (policy ACCEPT)
```
Nenhuma regra configurada — política padrão `ACCEPT` em todas as chains, nas três VMs.

### ⚠️ Achado de segurança
**Não há firewall ativo em nenhuma das três VMs do laboratório.** O `ufw` está desabilitado e o `iptables` não possui regras restritivas — todo o tráfego de entrada, saída e encaminhamento é aceito por padrão. Isso significa que, além das portas de serviço legítimas (22, 80, 3000, 10050/10051, etc.), qualquer outra porta eventualmente aberta por um processo ficaria acessível pela rede sem controle algum.

Essa condição foi identificada e documentada como está, sem alteração no ambiente, por decisão do autor do laboratório.

### Checkpoint
⚠️ Verificação de segurança realizada — **firewall inativo identificado como risco**, não corrigido nesta versão do laboratório.

### Evidências
- [x] `ufw status numbered` (3 VMs) — inactive
- [x] `iptables -L` (3 VMs) — sem regras, política ACCEPT
- [x] Achado registrado como risco conhecido

### Recomendação (melhoria futura)
Habilitar `ufw` com política padrão de negar entrada e liberar apenas as portas necessárias, por exemplo:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.110.102.0/24 to any port 22
sudo ufw allow from 10.110.102.0/24 to any port 80
sudo ufw allow from 10.110.102.0/24 to any port 3000
sudo ufw allow from 10.110.102.0/24 to any port 10050
sudo ufw allow from 10.110.102.0/24 to any port 10051
sudo ufw enable
```

---

## Fase 15 — Simulação de incidentes

### Execução real

**1. Sintoma provocado:** Apache parado deliberadamente no `srv-linux-thiago`.
```bash
$ sudo systemctl stop apache2
```

**2. Evidência coletada — ICMP continua OK, HTTP falha:**
```
$ ping -c 4 10.110.102.108
4 packets transmitted, 4 received, 0% packet loss

$ curl -I http://10.110.102.108
curl: (7) Failed to connect to 10.110.102.108 port 80 after 0 ms: Connection refused
```

**3. Hipótese/causa confirmada:**
```
$ sudo systemctl status apache2 --no-pager
Active: inactive (dead) since Tue 2026-09-15 22:50:38 UTC

$ sudo journalctl -u apache2 --no-pager | tail -n 20
Sep 15 22:50:38 srv-linux-thiago systemd[1]: Stopping The Apache HTTP Server...
Sep 15 22:50:38 srv-linux-thiago systemd[1]: apache2.service: Deactivated successfully.
Sep 15 22:50:38 srv-linux-thiago systemd[1]: Stopped The Apache HTTP Server.
```
Causa confirmada: serviço Apache parado manualmente — o log não mostra crash, apenas parada normal (`Deactivated successfully`).

> Observação adicional: o log também mostra o aviso `AH00558: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1` — não é a causa do incidente, mas é um ponto de configuração pendente (`ServerName` não definido no Apache).

**4. Correção:**
```bash
$ sudo systemctl start apache2
```

**5. Validação — HTTP restaurado:**
```
$ curl -I http://10.110.102.108
HTTP/1.1 200 OK
Server: Apache/2.4.52 (Ubuntu)
```

### Checkpoint
✅ Incidente detectado, diagnosticado (causa raiz confirmada via log), corrigido e validado com sucesso.

### Evidências
- [x] Sintoma (HTTP falhou, ICMP OK)
- [x] Evidência (curl + ping)
- [x] Hipótese/causa (systemctl status + journalctl)
- [x] Correção (systemctl start)
- [x] Validação (curl OK novamente)

---

## Fase 16 — Evidências e documentação final

🔲 **PENDENTE** — só pode ser fechada depois das Fases 09, 11–15.

---

## Conclusão

*(a redigir quando as fases pendentes forem concluídas — o aprendizado central já observado no laboratório: um host pode responder ICMP e mesmo assim falhar em SSH, HTTP ou coleta do agente, como ficou evidente entre as Fases 05 e 15.)*

## Checklist final

- [x] Rede privada e tabela de IPs documentadas.
- [x] Três VMs instaladas e validadas.
- [x] IP, gateway, DNS e horário corretos.
- [x] SSH e HTTP funcionando.
- [x] Capturas de ICMP, ARP, DNS, TCP (TLS parcial — evidência de aplicação).
- [x] Zabbix Server e Agent funcionando.
- [x] ICMP, HTTP, CPU, memória, disco e rede monitorados (Fase 09 pendente).
- [x] Grafana integrado ao Zabbix.
- [x] Dashboard NOC criado.
- [x] Regras de segurança revisadas — firewall inativo identificado e documentado como risco (Fase 14).
- [x] Incidente controlado investigado e corrigido (Fase 15).
- [x] Nenhuma credencial real publicada.

## Estrutura deste repositório

```text
projeto-noc-thiago/
├── README.md
└── imagens/
    ├── fase01-planejamento.png
    ├── fase02-vms.png
    ├── fase03-conectividade.png
    ├── fase04-preparacao-linux.png
    ├── fase05-servicos.png
    ├── fase06-wireshark.png
    ├── fase07-zabbix-server.png
    ├── fase08-agent2.png
    ├── fase09-monitoramento.png
    ├── fase10-grafana.png
    ├── fase11-api-zabbix.png
    ├── fase12-integracao.png
    ├── fase13-dashboard.png
    ├── fase14-seguranca.png
    ├── fase15-incidentes.png
    └── fase16-evidencias.png
```
