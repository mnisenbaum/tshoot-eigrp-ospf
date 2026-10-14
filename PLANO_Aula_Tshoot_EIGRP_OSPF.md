# Plano — Aula de Troubleshooting EIGRP e OSPF (CCNP ENARSI v9)

> **Versão 2.** Revisada com dois labs por protocolo, de 6 nós cada.
> **Aula:** 14/10/2026 às 19h · 3h no total: 1h30 de EIGRP e 1h30 de OSPF.
> **Plataforma:** CML2 com IOL-XE. **Pilha dupla:** IPv4 é a prioridade; IPv6 vem configurado e funcionando.
> **Slides:** Claude Slides, depois exportados.
> **Base:** Módulos 3 e 6 do ENARSI v9 e os labs ILM da pasta `ENARSI - Labs`.

---

## 1. Estratégia

- São **4 labs**, dois por protocolo:
  - **Lab A — "Vizinhança"**: problemas de adjacência.
  - **Lab B — "Rotas"**: problemas de tabela de roteamento, caminho, sumarização e default.
- Cada lab tem **4 tickets principais e 1 bônus em IPv6**. No total são 8 tickets por protocolo.
- **Um defeito por ponto da topologia.** Assim um ticket não mascara o outro e a ordem pode mudar.
- **Pilha dupla como recurso didático.** Os defeitos principais ficam só no IPv4, então o IPv6 continua funcionando. Isso leva à pergunta: "Se o IPv6 passa e o IPv4 não, onde está o problema?" (no address-family ou no processo IPv4).
- Cada ticket tem **3 scripts**: `show` (sintoma → isolamento → causa), `fix` e `break` (reaplica o defeito, para repetir ou recuperar).

### Roteiro padrão de cada ticket (~7 min)

**Sintoma** (ticket + ping/traceroute) → **Isolar** (vizinhos/rotas) → **Causa** (show/debug) → **Correção** → **Verificação** → **Lição** (uma frase).

### Tempo por bloco (90 min)

| Etapa | Tempo |
|---|---|
| Slides: metodologia + panorama + 1 slide por ticket | 15 min |
| Lab A: 4 tickets | 30 min |
| Lab B: 4 tickets | 30 min |
| Bônus IPv6, conforme o tempo | 0–10 min |
| Fechamento | 5 min |

---

## 2. EIGRP — named mode `EMPRESA`, AS 100, AF IPv4 + AF IPv6

### Lab EIGRP-A "Vizinhança" (6 nós)

**Nós:** CORE1, CORE2, BR1, BR2, BR3, DC1

**Enlaces:** CORE1–CORE2 · CORE1–BR1 · CORE1–BR3 · CORE2–BR2 · CORE2–DC1. As LANs são simuladas com loopbacks.

| # | Categoria | Defeito (só no AF IPv4) | Onde | Sintoma |
|---|---|---|---|---|
| A1 | 3.9 Autenticação | Key-chain com key-string diferente | BR1 ↔ CORE1 | Filial 1 sem IPv4; o IPv6 funciona |
| A2 | 3.4 Vizinhança | `passive-interface` no uplink | BR2 | Filial 2 isolada no IPv4 |
| A3 | 3.4 Vizinhança | AS 1000 em vez de 100 | BR3 | Filial nova sem acesso |
| A4 | 3.4 Vizinhança | K-values diferentes (`metric weights`) | DC1 | DC fora do ar; log de K-value mismatch |
| A5 ★ | 3.8 IPv6 | `af-interface ... shutdown` no AF IPv6 | CORE2 ↔ DC1 | Só o IPv6 do DC falha |

### Lab EIGRP-B "Rotas" (6 nós)

**Nós:** CORE1, CORE2, EDGE, ISP, BR1, BR2

**Enlaces:** CORE1–CORE2 · CORE1–EDGE · CORE2–EDGE · EDGE–ISP · CORE1–BR1 · CORE2–BR2. O EDGE tem uma rota estática default para o ISP e a redistribui no EIGRP como rota externa.

| # | Categoria | Defeito | Onde | Sintoma |
|---|---|---|---|---|
| B1 | 3.6 Stub | `eigrp stub receive-only` | BR1 | BR1 alcança tudo, mas ninguém alcança as LANs dela |
| B2 | 3.5 Tabela | Distribute-list bloqueando as LANs da BR1 | CORE2 (saída para BR2) | Só a BR2 não alcança a BR1 |
| B3 | 3.7 Sumarização | `summary-address 0.0.0.0/0` em vez de `10.2.0.0/22` | BR2 | O sumário vira uma default interna (AD 90), que ganha da externa do EDGE (AD 170). O CORE2 manda o tráfego de internet para a filial |
| B4 | 3.5 / RID | Router-ID do EDGE igual ao do CORE1 | EDGE | Vizinhança OK, mas o CORE1 descarta a default externa. O lado do CORE1 fica sem internet |
| B5 ★ | 3.8 IPv6 | Prefix-list IPv6 errada na distribute-list | ❓ | Só o IPv6 é afetado |

---

## 3. OSPF — OSPFv2 (IPv4) + OSPFv3 AF (IPv6)

### Lab OSPF-A "Adjacência" (6 nós)

**Nós:** CORE, ABR1, ABR2, BR1, BR2, DC1

| Área | Enlaces |
|---|---|
| Área 0 | CORE–ABR1 · CORE–ABR2 |
| Área 1 (stub) | ABR1–BR1 · ABR1–BR2 |
| Área 2 | ABR2–DC1 |

| # | Categoria | Defeito (só no OSPFv2) | Onde | Estado visível |
|---|---|---|---|---|
| A1 | 6.4 | Hello/dead diferentes | DC1 | Sem vizinho; o IPv6 está OK |
| A2 | 6.4 | Área diferente no enlace (0 em vez de 1) | BR2 | Log "mismatched area" |
| A3 | 6.4 | MTU diferente (`ip mtu`) | ABR1 ↔ CORE | Vizinho preso em EXSTART/EXCHANGE |
| A4 | 6.7 | Falta `area 1 stub` | BR1 | Sem vizinho (stub flag mismatch) |
| A5 ★ | 6.4 IPv6 | Router com OSPFv3 tradicional × OSPFv3 AF (caso do lab oficial 9.1.1.1) | ❓ | Só o IPv6 falha |

Alternativa ao A5: autenticação MD5 divergente entre CORE e ABR2.

### Lab OSPF-B "Rotas e caminho" (6 nós)

**Nós:** EDGE (ASBR), ISP, ABR1, ABR2, BR1, DC1

| Área | Enlaces |
|---|---|
| Área 0 | EDGE–ABR1 · EDGE–ABR2 · ABR1–ABR2 |
| Área 1 | ABR1–BR1 (BR1 com 4 LANs em loopback) |
| Área 2 | DC1–ABR2 (link principal "1G") · DC1–ABR1 (link de backup "100M") |

| # | Categoria | Defeito | Onde | Estado visível |
|---|---|---|---|---|
| B1 | 6.5 | Network type p2p × broadcast | BR1 ↔ ABR1 | Vizinho FULL, mas **as rotas não entram** |
| B2 | 6.8 | `area 1 range ... not-advertise` | ABR1 | As LANs da BR1 somem da área 0 |
| B3 | 6.5 | Falta `default-information originate` | EDGE | Ninguém sai para a internet |
| B4 | 6.6 | `auto-cost reference-bandwidth` padrão (100 Mb) com links de 1G e 100M | todos | Custos iguais; o tráfego do DC passa pelo backup lento (ECMP) |
| B5 ★ | IPv6 | `default-information originate` ausente só no AF IPv6, ou outro defeito em aberto | EDGE | Só o IPv6 fica sem default |

---

## 4. Entregáveis

| # | Entregável | Formato |
|---|---|---|
| 1 | Slides EIGRP: capa, metodologia, panorama e 8 tickets | Claude Slides |
| 2 | Slides OSPF: capa, metodologia, panorama e 8 tickets | Claude Slides |
| 3.1 | 4 labs no CML2 com os defeitos plantados | Lab no CML2 + `.yaml` exportado na pasta |
| 3.2 | Roteiro do instrutor, um por protocolo, com as saídas reais coletadas nos testes | `.md` |
| 3.3 | Scripts por ticket: `show` / `fix` / `break` | `.txt` |

**Convenções:**
- Interfaces IOL-XE: `Ethernet0/x`.
- Endereços IPv4: `10.<lab>.x.x`. Endereços IPv6: `2001:db8:acad:<x>::/64`. O detalhe sai na montagem.
- Senha dos equipamentos: `cisco12345`.

---

## 5. Cronograma de hoje (até as 19h)

| Horário | Atividade |
|---|---|
| 15:45–16:00 | Fechar o plano (este documento) |
| 16:00–16:45 | Montar e testar os labs EIGRP-A e EIGRP-B no CML2 |
| 16:45–17:30 | Montar e testar os labs OSPF-A e OSPF-B |
| 17:30–17:55 | Scripts show/fix/break e roteiros do instrutor |
| 17:55–18:25 | Slides EIGRP e OSPF |
| 18:25–18:50 | Revisão do Moises e ensaio rápido |
| 18:50–19:00 | Reaplicar os defeitos (`break`) e deixar tudo pronto |

> Se o tempo apertar, cortamos primeiro os bônus IPv6 (A5/B5) e depois o B4 de cada protocolo.

---

## 6. Decisões em aberto ❓

- [ ] Os 4 labs (24 nós IOL) cabem no servidor ao mesmo tempo, ou ligamos só os 2 do bloco em andamento?
- [ ] O servidor CML2 está ligado e acessível agora, via MCP `cml2-pyats`?
- [ ] Aprovar os tickets dos 4 labs, incluindo os bônus IPv6.

---

## 7. Status final (14/10, ~16h45)

| Item | Status |
|---|---|
| 4 labs COM PROBLEMAS no CML2 (EIGRP-A, EIGRP-B, OSPF-A, OSPF-B) | ✅ testados ponta a ponta, defeitos reaplicados, desligados |
| 4 labs SOLUÇÃO no CML2 (mesmos nomes + "SOLUCAO") | ✅ testados (vizinhos, rotas, pings IPv4/IPv6 e internet), desligados |
| Roteiro do instrutor + scripts show/fix/break | ✅ `Lab_EIGRP-A.md`, `Lab_EIGRP-B.md`, `Lab_OSPF-A.md`, `Lab_OSPF-B.md` |
| Slides EIGRP e OSPF (Claude Slides, 14 cada) | ✅ |
| Topologias draw.io (8 arquivos: PROBLEMAS e SOLUCAO) | ✅ pasta `topologias_drawio/` |
| Backup das topologias CML (8 arquivos importáveis) | ✅ pasta `topologias_cml/` |
| Bônus IPv6 (A5/B5) | ⚠️ não testados no lab |

### Na hora da aula
1. **Licença CML2 = 20 nós.** Ligue só 2 labs por vez (6 nós cada).
2. Bloco EIGRP: ligar **EIGRP-A** e **EIGRP-B** (com problemas). Esperar ~1 min para convergir.
3. Bloco OSPF: parar os labs EIGRP, ligar **OSPF-A** e **OSPF-B**. OSPF leva ~40 s (estado WAIT) para convergir.
4. Ordem dos tickets: EIGRP-A A1→A4 · EIGRP-B B1→B2→B3→B4 · OSPF-A A1→A3→A2→A4 · OSPF-B B1→B2→B3→B4.
5. Para recomeçar um lab do zero: Stop → Wipe → Start (volta à configuração com os defeitos).
