# Lab EIGRP-B — Rotas (tabela, stub, filtro, sumarização, RID)

> Lab no CML2: **ENARSI Tshoot EIGRP-B Rotas** · Plataforma IOL-XE 17.18 · Console sem senha
> Os nós sobem **já com os defeitos** (startup-config). Para recomeçar do zero: *Stop → Wipe → Start* no CML2, ou aplique os blocos **Quebrar** de cada ticket.

## 1. Cenário

Empresa com saída de **internet** (EDGE → ISP), núcleo (CORE1/CORE2) e duas filiais **stub** (BR1, BR2).
O EDGE tem rota estática default para o ISP e a **redistribui** no EIGRP (rota externa D*EX, AD 170).
Vizinhanças sobem normalmente: os problemas estão no **conteúdo** das tabelas.
⚠️ **Ordem recomendada: B1 → B2 → B3 → B4** (o B3 mascara o B4: enquanto a filial anuncia uma default interna, o CORE1 tem uma default de qualquer jeito).

```
                         ISP (8.8.8.8)
                          | e0/0
                     e0/2 |
                         EDGE
                  e0/0 /      \ e0/1
               e0/1   /        \  e0/1
               CORE1 e0/0 ──── e0/0 CORE2
              e0/2 |                 | e0/2
              e0/0 |                 | e0/0
                  BR1               BR2
```

## 2. Endereçamento

| Enlace | | IPv4 | IPv6 |
|---|---|---|---|
| CORE1 e0/0 | CORE2 e0/0 | 10.0.12.0/30 | 2001:db8:acad:12::/64 |
| CORE1 e0/1 | EDGE e0/0 | 10.0.13.0/30 | 2001:db8:acad:13::/64 |
| CORE2 e0/1 | EDGE e0/1 | 10.0.23.0/30 | 2001:db8:acad:23::/64 |
| CORE1 e0/2 | BR1 e0/0 | 10.0.14.0/30 | 2001:db8:acad:14::/64 |
| CORE2 e0/2 | BR2 e0/0 | 10.0.15.0/30 | 2001:db8:acad:15::/64 |
| EDGE e0/2 | ISP e0/0 | 203.0.113.0/30 | 2001:db8:ffff::/64 |

| Roteador | LANs (loopbacks) | Router-ID |
|---|---|---|
| CORE1 | Lo1 10.1.1.1/24 | RID 1.1.1.1 |
| CORE2 | Lo1 10.2.1.1/24 | RID 2.2.2.2 |
| EDGE | — | RID 3.3.3.3 |
| BR1 | Lo1 10.11.1.1/24, Lo2 10.11.2.1/24 | RID 11.11.11.11 |
| BR2 | Lo1–Lo4 10.12.0.1–10.12.3.1/24 (sumário 10.12.0.0/22) | RID 12.12.12.12 |
| ISP | Lo1 8.8.8.8/32 (não roda EIGRP) | — |

Loopback0 de cada nó: `10.255.255.x/32` e `2001:db8:acad:ffff::x/128`.

## 3. Resumo dos tickets

| # | Problema | Categoria ENARSI | Onde |
|---|---|---|---|
| B1 | Stub receive-only | 3.6 Stub | BR1 |
| B2 | Distribute-list com prefix-list sem permit final | 3.5 Tabela de roteamento | CORE2 |
| B3 | Sumarização errada: 0.0.0.0/0 em vez de 10.12.0.0/22 | 3.7 Sumarização | BR2 |
| B4 | Router-ID duplicado (EDGE = CORE1) | 3.5 Tabela / Router-ID | EDGE |
| B5 ★ | BÔNUS IPv6 — EDGE não redistribui a default IPv6 | 3.8 EIGRP IPv6 | EDGE |

★ = bônus. Começa **corrigido**; aplique o bloco *Quebrar* se houver tempo.

## 4. Roteiro dos tickets (sintoma → isolar → causa → correção → verificação)

### Ticket B1 — Stub receive-only

**Chamado do usuário:** “Da Filial 1 eu acesso tudo, mas ninguém da matriz consegue acessar o servidor da filial.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| BR1 | `ping 10.1.1.1 source 10.11.1.1` | Falha também (o retorno não tem rota!) — mas BR1 *tem* rota para 10.1.1.0 |
| CORE1 | `ping 10.11.1.1 source 10.1.1.1` | Falha |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ip eigrp neighbors` | BR1 (10.0.14.2) **está** como vizinho |
| CORE1 | `show ip route 10.11.1.0` | `% Subnet not in table` |
| BR1 | `show ip route eigrp` | BR1 recebe todas as rotas normalmente |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ip eigrp neighbors detail` | BR1 aparece como **Receive-Only Peer** (stub que não anuncia rotas) |
| BR1 | `show ip protocols | include stub|Stub` | `EIGRP stub, receive-only` |

**4) Correção** (em `configure terminal`)

```
! BR1 — corrige o ticket B1
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  eigrp stub connected summary
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ip eigrp neighbors detail` | `Stub Peer Advertising (CONNECTED SUMMARY ) Routes` |
| CORE1 | `show ip route 10.11.1.0` | D 10.11.1.0/24 via 10.0.14.2 |

**💡 Lição:** Stub **receive-only** não anuncia nada, nem as redes conectadas. O padrão `eigrp stub` = *connected summary*.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! BR1 — reaplica o defeito B1
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  eigrp stub receive-only
end
```
</details>

---

### Ticket B2 — Distribute-list com prefix-list sem permit final

**Chamado do usuário:** “Pediram para bloquear a rede de laboratório 10.100.0.0/16 para a Filial 2. Agora a Filial 2 não alcança nada.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| BR2 | `ping 10.1.1.1 source 10.12.0.1` | Falha |
| BR2 | `show ip route eigrp` | **Nenhuma** rota EIGRP aprendida |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| BR2 | `show ip eigrp neighbors` | CORE2 é vizinho → adjacência OK, o problema é o que ele anuncia |
| CORE2 | `show ip protocols | section eigrp` | `Outgoing update filter list for Et0/2 is FILTRO-BR2` (ou similar) |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `show ip prefix-list FILTRO-BR2` | Só `seq 5 deny 10.1.0.0/16 le 24`: rede errada **e** sem `permit` final → *deny* implícito de todo o resto |

**4) Correção** (em `configure terminal`)

```
! CORE2 — corrige o ticket B2
no ip prefix-list FILTRO-BR2
ip prefix-list FILTRO-BR2 seq 5 deny 10.100.0.0/16 le 24
ip prefix-list FILTRO-BR2 seq 10 permit 0.0.0.0/0 le 32
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| BR2 | `show ip route eigrp` | Rotas da empresa (10.1.1.0/24, 10.11.x...) presentes |

**💡 Lição:** Toda prefix-list termina com **deny implícito**. Filtro de bloqueio precisa de `permit 0.0.0.0/0 le 32` no final.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! CORE2 — reaplica o defeito B2
no ip prefix-list FILTRO-BR2
ip prefix-list FILTRO-BR2 seq 5 deny 10.1.0.0/16 le 24
end
```
</details>

---

### Ticket B3 — Sumarização errada: 0.0.0.0/0 em vez de 10.12.0.0/22

**Chamado do usuário:** “Desde ontem a internet da matriz caiu para quem está no CORE2. O traceroute para 8.8.8.8 morre logo no início.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `traceroute 8.8.8.8 source 10.2.1.1 timeout 1 probe 1 ttl 1 4` | Não sai pelo EDGE (`* * *`) |
| CORE2 | `show ip route 0.0.0.0` | `D* 0.0.0.0/0 [90/...] via 10.0.15.2` → default **interna** (AD 90) vinda da **filial**! |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `show ip eigrp topology 0.0.0.0/0` | Duas entradas: *Internal* via 10.0.15.2 (origem 12.12.12.12) e *External* via EDGE. A interna ganha (AD 90 < 170) |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| BR2 | `show ip route | include Null0` | `D* 0.0.0.0/0 is a summary, Null0` |
| BR2 | `show running-config | section router eigrp` | `af-interface Ethernet0/0` / `summary-address 0.0.0.0 0.0.0.0` |

**4) Correção** (em `configure terminal`)

```
! BR2 — corrige o ticket B3
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/0
   no summary-address 0.0.0.0 0.0.0.0
   summary-address 10.12.0.0 255.255.252.0
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `show ip route 0.0.0.0` | `D*EX 0.0.0.0/0 [170/...] via 10.0.23.2` (EDGE) |
| CORE2 | `show ip route 10.12.0.0` | `D 10.12.0.0/22` (sumário correto) |
| CORE2 | `ping 8.8.8.8 source 10.2.1.1` | !!!!! |

**💡 Lição:** Sumário `0/0` vira uma **default interna** (AD 90), que ganha da default externa (AD 170) e “sequestra” o tráfego de internet. Confira sempre o **Originating router** no `show ip eigrp topology`.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! BR2 — reaplica o defeito B3
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/0
   no summary-address 10.12.0.0 255.255.252.0
   summary-address 0.0.0.0 0.0.0.0
end
```
</details>

---

### Ticket B4 — Router-ID duplicado (EDGE = CORE1)

**Chamado do usuário:** “Corrigimos a filial 2, mas o CORE1 e a Filial 1 continuam sem internet. O CORE2 navega normalmente.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `ping 8.8.8.8 source 10.1.1.1` | Falha |
| CORE1 | `show ip route 0.0.0.0` | `% Network not in table` |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ip eigrp neighbors` | EDGE (10.0.13.2) é vizinho normalmente! |
| CORE1 | `show ip eigrp topology 0.0.0.0/0` | `%Entry 0.0.0.0/0 not in topology table` |
| CORE2 | `show ip eigrp topology 0.0.0.0/0` | Entrada *External*… `Originating router is 1.1.1.1` |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ip protocols | include Router-ID|router-id` | `Router-ID: 1.1.1.1` |
| EDGE | `show ip protocols | include Router-ID|router-id` | `Router-ID: 1.1.1.1` → **duplicado**. O CORE1 descarta rotas externas que “ele mesmo” originou |

**4) Correção** (em `configure terminal`)

```
! EDGE — corrige o ticket B4
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 3.3.3.3
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ip route 0.0.0.0` | `D*EX 0.0.0.0/0 [170/...] via 10.0.13.2` |
| BR1 | `traceroute 8.8.8.8 source 10.11.1.1 timeout 1 probe 1` | 10.0.14.1 → 10.0.13.2 → 203.0.113.2 |

**💡 Lição:** RID duplicado no EIGRP **não derruba vizinhança**; afeta só **rotas externas** (o roteador com o mesmo RID as descarta). Diferente do OSPF!

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! EDGE — reaplica o defeito B4
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 1.1.1.1
end
```
</details>

---

### Ticket B5 — BÔNUS IPv6 — EDGE não redistribui a default IPv6 (BÔNUS)

**Chamado do usuário:** “IPv4 navega, IPv6 não.” (aplique o *break* antes)

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `ping 2001:4860:4860::8888 source loopback1` | Falha |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ipv6 route ::/0` | Sem rota |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| EDGE | `show running-config | section router eigrp` | No AF IPv6 não há `redistribute static` |

**4) Correção** (em `configure terminal`)

```
! EDGE — corrige o ticket B5
router eigrp EMPRESA
 address-family ipv6 unicast autonomous-system 100
  topology base
   redistribute static
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ipv6 route ::/0` | `EX ::/0 via ...` |

**💡 Lição:** Cada AF tem sua própria redistribuição: configurar no IPv4 não leva o IPv6 junto.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! EDGE — reaplica o defeito B5
router eigrp EMPRESA
 address-family ipv6 unicast autonomous-system 100
  topology base
   no redistribute static
end
```
</details>

---

## 5. Configurações iniciais (com os defeitos) — comentadas

Linhas marcadas com `! >>>>> DEFEITO` são os pontos plantados. As demais linhas `! ---` são comentários didáticos.

### BR1

```
! --- Nome do equipamento
hostname BR1
no ip domain lookup
! --- Habilita roteamento IPv6 (pilha dupla)
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip address 10.255.255.5 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::5/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.11.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:1101::1/64
 no shutdown
! --- LAN simulada (loopback)
interface Loopback2
 ip address 10.11.2.1 255.255.255.0
 ipv6 address 2001:db8:acad:1102::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE1
 ip address 10.0.14.2 255.255.255.252
 ipv6 address 2001:db8:acad:014::2/64
 no shutdown
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0
  eigrp router-id 11.11.11.11
! >>>>> DEFEITO B1: stub receive-only nao anuncia nenhuma rede
  eigrp stub receive-only
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp stub connected summary
  eigrp router-id 11.11.11.11
 exit-address-family
end
```

### BR2

```
! --- Nome do equipamento
hostname BR2
no ip domain lookup
! --- Habilita roteamento IPv6 (pilha dupla)
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip address 10.255.255.6 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::6/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.12.0.1 255.255.255.0
 ipv6 address 2001:db8:acad:1200::1/64
 no shutdown
! --- LAN simulada (loopback)
interface Loopback2
 ip address 10.12.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:1201::1/64
 no shutdown
! --- LAN simulada (loopback)
interface Loopback3
 ip address 10.12.2.1 255.255.255.0
 ipv6 address 2001:db8:acad:1202::1/64
 no shutdown
! --- LAN simulada (loopback)
interface Loopback4
 ip address 10.12.3.1 255.255.255.0
 ipv6 address 2001:db8:acad:1203::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE2
 ip address 10.0.15.2 255.255.255.252
 ipv6 address 2001:db8:acad:015::2/64
 no shutdown
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/0
! >>>>> DEFEITO B3: sumario 0/0 vira default interna (AD 90)
   summary-address 0.0.0.0 0.0.0.0
  exit-af-interface
  network 10.0.0.0
  eigrp router-id 12.12.12.12
  eigrp stub connected summary
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp stub connected summary
  eigrp router-id 12.12.12.12
 exit-address-family
end
```

### CORE1

```
! --- Nome do equipamento
hostname CORE1
no ip domain lookup
! --- Habilita roteamento IPv6 (pilha dupla)
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip address 10.255.255.3 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::3/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.1.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:101::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE2
 ip address 10.0.12.1 255.255.255.252
 ipv6 address 2001:db8:acad:012::1/64
 no shutdown
interface Ethernet0/1
 description Link para EDGE
 ip address 10.0.13.1 255.255.255.252
 ipv6 address 2001:db8:acad:013::1/64
 no shutdown
interface Ethernet0/2
 description Link para BR1
 ip address 10.0.14.1 255.255.255.252
 ipv6 address 2001:db8:acad:014::1/64
 no shutdown
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0
  eigrp router-id 1.1.1.1
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 1.1.1.1
 exit-address-family
end
```

### CORE2

```
! --- Nome do equipamento
hostname CORE2
no ip domain lookup
! --- Habilita roteamento IPv6 (pilha dupla)
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip address 10.255.255.4 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::4/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.2.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:201::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE1
 ip address 10.0.12.2 255.255.255.252
 ipv6 address 2001:db8:acad:012::2/64
 no shutdown
interface Ethernet0/1
 description Link para EDGE
 ip address 10.0.23.1 255.255.255.252
 ipv6 address 2001:db8:acad:023::1/64
 no shutdown
interface Ethernet0/2
 description Link para BR2
 ip address 10.0.15.1 255.255.255.252
 ipv6 address 2001:db8:acad:015::1/64
 no shutdown
! >>>>> DEFEITO B2: rede errada e sem 'permit' final (deny implicito)
ip prefix-list FILTRO-BR2 seq 5 deny 10.1.0.0/16 le 24
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  topology base
   distribute-list prefix FILTRO-BR2 out Ethernet0/2
  exit-af-topology
  network 10.0.0.0
  eigrp router-id 2.2.2.2
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 2.2.2.2
 exit-address-family
end
```

### EDGE

```
! --- Nome do equipamento
hostname EDGE
no ip domain lookup
! --- Habilita roteamento IPv6 (pilha dupla)
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip address 10.255.255.2 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::2/128
 no shutdown
interface Ethernet0/0
 description Link para CORE1
 ip address 10.0.13.2 255.255.255.252
 ipv6 address 2001:db8:acad:013::2/64
 no shutdown
interface Ethernet0/1
 description Link para CORE2
 ip address 10.0.23.2 255.255.255.252
 ipv6 address 2001:db8:acad:023::2/64
 no shutdown
interface Ethernet0/2
 description Link para ISP
 ip address 203.0.113.1 255.255.255.252
 ipv6 address 2001:db8:ffff::1/64
 no shutdown
! --- Default estatica para o ISP
ip route 0.0.0.0 0.0.0.0 203.0.113.2
ipv6 route ::/0 2001:db8:ffff::2
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute static
  exit-af-topology
  network 10.0.0.0
! >>>>> DEFEITO B4: RID igual ao do CORE1 -> CORE1 descarta as externas do EDGE
  eigrp router-id 1.1.1.1
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  af-interface Ethernet0/2
   shutdown
  exit-af-interface
  topology base
   redistribute static
  exit-af-topology
  eigrp router-id 3.3.3.3
 exit-address-family
end
```

### ISP

```
! --- Nome do equipamento
hostname ISP
no ip domain lookup
! --- Habilita roteamento IPv6 (pilha dupla)
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
! --- LAN simulada (loopback)
interface Loopback1
 ip address 8.8.8.8 255.255.255.255
 ipv6 address 2001:4860:4860::8888/128
interface Ethernet0/0
 description Link para EDGE
 ip address 203.0.113.2 255.255.255.252
 ipv6 address 2001:db8:ffff::2/64
 no shutdown
! --- ISP: rota de retorno para a empresa
ip route 10.0.0.0 255.0.0.0 203.0.113.1
ipv6 route 2001:db8:acad::/48 2001:db8:ffff::1
end
```

## 6. Configurações finais (golden — todos os tickets resolvidos)

Referência do estado correto. Diferenças em relação à seção 5 = exatamente as correções dos tickets.

<details><summary>BR1</summary>

```
hostname BR1
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback0
 ip address 10.255.255.5 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::5/128
 no shutdown
interface Loopback1
 ip address 10.11.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:1101::1/64
 no shutdown
interface Loopback2
 ip address 10.11.2.1 255.255.255.0
 ipv6 address 2001:db8:acad:1102::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE1
 ip address 10.0.14.2 255.255.255.252
 ipv6 address 2001:db8:acad:014::2/64
 no shutdown
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0
  eigrp router-id 11.11.11.11
  eigrp stub connected summary
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp stub connected summary
  eigrp router-id 11.11.11.11
 exit-address-family
end
```
</details>

<details><summary>BR2</summary>

```
hostname BR2
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback0
 ip address 10.255.255.6 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::6/128
 no shutdown
interface Loopback1
 ip address 10.12.0.1 255.255.255.0
 ipv6 address 2001:db8:acad:1200::1/64
 no shutdown
interface Loopback2
 ip address 10.12.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:1201::1/64
 no shutdown
interface Loopback3
 ip address 10.12.2.1 255.255.255.0
 ipv6 address 2001:db8:acad:1202::1/64
 no shutdown
interface Loopback4
 ip address 10.12.3.1 255.255.255.0
 ipv6 address 2001:db8:acad:1203::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE2
 ip address 10.0.15.2 255.255.255.252
 ipv6 address 2001:db8:acad:015::2/64
 no shutdown
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/0
   summary-address 10.12.0.0 255.255.252.0
  exit-af-interface
  network 10.0.0.0
  eigrp router-id 12.12.12.12
  eigrp stub connected summary
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp stub connected summary
  eigrp router-id 12.12.12.12
 exit-address-family
end
```
</details>

<details><summary>CORE1</summary>

```
hostname CORE1
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback0
 ip address 10.255.255.3 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::3/128
 no shutdown
interface Loopback1
 ip address 10.1.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:101::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE2
 ip address 10.0.12.1 255.255.255.252
 ipv6 address 2001:db8:acad:012::1/64
 no shutdown
interface Ethernet0/1
 description Link para EDGE
 ip address 10.0.13.1 255.255.255.252
 ipv6 address 2001:db8:acad:013::1/64
 no shutdown
interface Ethernet0/2
 description Link para BR1
 ip address 10.0.14.1 255.255.255.252
 ipv6 address 2001:db8:acad:014::1/64
 no shutdown
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0
  eigrp router-id 1.1.1.1
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 1.1.1.1
 exit-address-family
end
```
</details>

<details><summary>CORE2</summary>

```
hostname CORE2
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback0
 ip address 10.255.255.4 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::4/128
 no shutdown
interface Loopback1
 ip address 10.2.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:201::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE1
 ip address 10.0.12.2 255.255.255.252
 ipv6 address 2001:db8:acad:012::2/64
 no shutdown
interface Ethernet0/1
 description Link para EDGE
 ip address 10.0.23.1 255.255.255.252
 ipv6 address 2001:db8:acad:023::1/64
 no shutdown
interface Ethernet0/2
 description Link para BR2
 ip address 10.0.15.1 255.255.255.252
 ipv6 address 2001:db8:acad:015::1/64
 no shutdown
ip prefix-list FILTRO-BR2 seq 5 deny 10.100.0.0/16 le 24
ip prefix-list FILTRO-BR2 seq 10 permit 0.0.0.0/0 le 32
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  topology base
   distribute-list prefix FILTRO-BR2 out Ethernet0/2
  exit-af-topology
  network 10.0.0.0
  eigrp router-id 2.2.2.2
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 2.2.2.2
 exit-address-family
end
```
</details>

<details><summary>EDGE</summary>

```
hostname EDGE
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback0
 ip address 10.255.255.2 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::2/128
 no shutdown
interface Ethernet0/0
 description Link para CORE1
 ip address 10.0.13.2 255.255.255.252
 ipv6 address 2001:db8:acad:013::2/64
 no shutdown
interface Ethernet0/1
 description Link para CORE2
 ip address 10.0.23.2 255.255.255.252
 ipv6 address 2001:db8:acad:023::2/64
 no shutdown
interface Ethernet0/2
 description Link para ISP
 ip address 203.0.113.1 255.255.255.252
 ipv6 address 2001:db8:ffff::1/64
 no shutdown
ip route 0.0.0.0 0.0.0.0 203.0.113.2
ipv6 route ::/0 2001:db8:ffff::2
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute static
  exit-af-topology
  network 10.0.0.0
  eigrp router-id 3.3.3.3
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  af-interface Ethernet0/2
   shutdown
  exit-af-interface
  topology base
   redistribute static
  exit-af-topology
  eigrp router-id 3.3.3.3
 exit-address-family
end
```
</details>

<details><summary>ISP</summary>

```
hostname ISP
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback1
 ip address 8.8.8.8 255.255.255.255
 ipv6 address 2001:4860:4860::8888/128
interface Ethernet0/0
 description Link para EDGE
 ip address 203.0.113.2 255.255.255.252
 ipv6 address 2001:db8:ffff::2/64
 no shutdown
ip route 10.0.0.0 255.0.0.0 203.0.113.1
ipv6 route 2001:db8:acad::/48 2001:db8:ffff::1
end
```
</details>

