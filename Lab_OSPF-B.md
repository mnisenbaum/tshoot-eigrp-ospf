# Lab OSPF-B — Rotas e caminho

> Lab no CML2: **ENARSI Tshoot OSPF-B Rotas** · Plataforma IOL-XE 17.18 · Console sem senha
> Os nós sobem **já com os defeitos** (startup-config). Para recomeçar do zero: *Stop → Wipe → Start* no CML2, ou aplique os blocos **Quebrar** de cada ticket.

## 1. Cenário

OSPF multiárea com **ASBR** (EDGE → ISP), dois ABRs, filial BR1 (área 1, com 4 LANs sumarizadas em 10.1.4.0/22) e datacenter DC1 (área 2) com **link principal de 1 Gbps** (via ABR2) e **link de backup de 100 Mbps** (via ABR1).
Pilha dupla: OSPFv2 + OSPFv3 AF. Defeitos no OSPFv2.
⚠️ **Ordem recomendada: B1 → B2 → B3 → B4** (o B1 esconde o B2: enquanto as rotas da BR1 não entram no ABR1, não há o que sumarizar).

```
                         ISP (8.8.8.8)
                          | e0/0
                     e0/2 |
                         EDGE (ASBR)          área 0
                  e0/0 /      \ e0/1
                 e0/0 /        \ e0/0
                   ABR1 e0/1 ── e0/1 ABR2
              e0/2 |    \ e0/3          | e0/2
              e0/0 |     \ e0/1 (100M) | e0/0 (1G)
                  BR1      \──────── DC1
               (área 1)              (área 2)
```

## 2. Endereçamento

| Enlace | | IPv4 | IPv6 |
|---|---|---|---|
| EDGE e0/0 | ABR1 e0/0 | 10.0.12.0/30 (área 0, 1G) | 2001:db8:acad:12::/64 |
| EDGE e0/1 | ABR2 e0/0 | 10.0.13.0/30 (área 0, 1G) | 2001:db8:acad:13::/64 |
| ABR1 e0/1 | ABR2 e0/1 | 10.0.23.0/30 (área 0, 1G) | 2001:db8:acad:23::/64 |
| ABR1 e0/2 | BR1 e0/0 | 10.1.11.0/30 (área 1, 1G) | 2001:db8:acad:111::/64 |
| ABR2 e0/2 | DC1 e0/0 | 10.2.21.0/30 (área 2, **1G principal**) | 2001:db8:acad:221::/64 |
| ABR1 e0/3 | DC1 e0/1 | 10.2.22.0/30 (área 2, **100M backup**) | 2001:db8:acad:222::/64 |
| EDGE e0/2 | ISP e0/0 | 203.0.113.0/30 (fora do OSPF) | 2001:db8:ffff::/64 |

| Roteador | LANs (loopbacks) | Router-ID |
|---|---|---|
| EDGE | — | RID 1.1.1.1 |
| ABR1 | — | RID 2.2.2.2 |
| ABR2 | — | RID 3.3.3.3 |
| BR1 | Lo1–Lo4 10.1.4.1–10.1.7.1/24 (sumário 10.1.4.0/22) | RID 11.11.11.11 |
| DC1 | Lo1 10.2.1.1/24 | RID 21.21.21.21 |
| ISP | Lo1 8.8.8.8/32 | — |

Loopback0 de cada nó: `10.255.255.x/32` e `2001:db8:acad:ffff::x/128`.

## 3. Resumo dos tickets

| # | Problema | Categoria ENARSI | Onde |
|---|---|---|---|
| B1 | Network type diferente (point-to-point × broadcast) | 6.5 Roteamento | BR1 |
| B2 | area range ... not-advertise | 6.8 Sumarização | ABR1 |
| B3 | ASBR sem default-information originate | 6.5 Roteamento | EDGE |
| B4 | reference-bandwidth padrão (100 Mbps): 1G e 100M com o mesmo custo | 6.6 Seleção de caminho | todos |
| B5 ★ | BÔNUS IPv6 — default só no AF IPv6 | OSPFv3 | EDGE |

★ = bônus. Começa **corrigido**; aplique o bloco *Quebrar* se houver tempo.

## 4. Roteiro dos tickets (sintoma → isolar → causa → correção → verificação)

### Ticket B1 — Network type diferente (point-to-point × broadcast)

**Chamado do usuário:** “O vizinho da Filial 1 está FULL, mas as redes da filial não aparecem em lugar nenhum.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip route 10.1.4.0 255.255.252.0 longer-prefixes` | Nenhuma rota da BR1 |
| ABR1 | `show ip ospf neighbor` | BR1 (11.11.11.11) **FULL** — parece tudo OK! |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip ospf database router 11.11.11.11` | LSA da BR1 descreve link **point-to-point**; o ABR1 descreve **transit** (DR) |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip ospf interface Ethernet0/2 | include Network Type` | `Network Type BROADCAST` |
| BR1 | `show ip ospf interface Ethernet0/0 | include Network Type` | `Network Type POINT_TO_POINT` → divergente |

**4) Correção** (em `configure terminal`)

```
! BR1 — corrige o ticket B1
interface Ethernet0/0
 no ip ospf network point-to-point
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip route ospf | include 10.1.` | O 10.1.4.0/24 … 10.1.7.0/24 |

**💡 Lição:** Network type diferente **não impede FULL** (timers iguais), mas as LSAs ficam inconsistentes e o SPF descarta as rotas. FULL ≠ rotas na tabela!

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! BR1 — reaplica o defeito B1
interface Ethernet0/0
 ip ospf network point-to-point
end
```
</details>

---

### Ticket B2 — area range ... not-advertise

**Chamado do usuário:** “A filial voltou para o ABR1, mas o resto da empresa ainda não enxerga as redes dela.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| EDGE | `show ip route 10.1.4.0 255.255.252.0 longer-prefixes` | Nada |
| ABR1 | `show ip route ospf | include 10.1.` | Rotas presentes no ABR1 |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| EDGE | `show ip ospf database summary | include Link State ID` | Sem LSA tipo 3 para 10.1.4.0 |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip ospf | section Area 1` | `Area ranges are 10.1.4.0/22 Passive DoNotAdvertise` |

**4) Correção** (em `configure terminal`)

```
! ABR1 — corrige o ticket B2
router ospf 1
 no area 1 range 10.1.4.0 255.255.252.0 not-advertise
 area 1 range 10.1.4.0 255.255.252.0
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| EDGE | `show ip route 10.1.4.0` | `O IA 10.1.4.0/22` |

**💡 Lição:** `not-advertise` serve para **esconder** prefixos de outras áreas: usado sem querer, some com a rede inteira.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! ABR1 — reaplica o defeito B2
router ospf 1
 area 1 range 10.1.4.0 255.255.252.0 not-advertise
end
```
</details>

---

### Ticket B3 — ASBR sem default-information originate

**Chamado do usuário:** “Ninguém da empresa navega na internet. O EDGE pinga o 8.8.8.8 normalmente.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| EDGE | `ping 8.8.8.8` | Sucesso |
| ABR2 | `show ip route 0.0.0.0` | `% Network not in table` |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| ABR2 | `show ip ospf database external` | Sem LSA tipo 5 para 0.0.0.0 |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| EDGE | `show running-config | section router ospf` | Não há `default-information originate` (a rota estática existe) |

**4) Correção** (em `configure terminal`)

```
! EDGE — corrige o ticket B3
router ospf 1
 default-information originate
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| ABR2 | `show ip route 0.0.0.0` | `O*E2 0.0.0.0/0` |
| DC1 | `ping 8.8.8.8 source 10.2.1.1` | !!!!! |

**💡 Lição:** Rota estática no ASBR **não entra** no OSPF sozinha: é preciso `default-information originate` (ou `... always`).

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! EDGE — reaplica o defeito B3
router ospf 1
 no default-information originate
end
```
</details>

---

### Ticket B4 — reference-bandwidth padrão (100 Mbps): 1G e 100M com o mesmo custo

**Chamado do usuário:** “Os backups do datacenter estão lentos. Parece que parte do tráfego está saindo pelo link de backup de 100 Mbps.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| DC1 | `show ip route 10.255.255.2` | **Dois** caminhos (ECMP): via 10.2.21.1 (1G) **e** via 10.2.22.1 (100M) |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| DC1 | `show ip ospf interface brief` | Cost **1** nas duas interfaces (Et0/0 1G e Et0/1 100M) |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| DC1 | `show ip ospf | include Reference` | `Reference bandwidth unit is 100 mbps` → tudo ≥ 100M vira custo 1 |
| DC1 | `show interfaces Ethernet0/1 | include BW` | `BW 100000 Kbit` |

**4) Correção** (em `configure terminal`)

```
! EDGE — corrige o ticket B4
router ospf 1
 auto-cost reference-bandwidth 10000
end
```
```
! ABR1 — corrige o ticket B4
router ospf 1
 auto-cost reference-bandwidth 10000
end
```
```
! ABR2 — corrige o ticket B4
router ospf 1
 auto-cost reference-bandwidth 10000
end
```
```
! BR1 — corrige o ticket B4
router ospf 1
 auto-cost reference-bandwidth 10000
end
```
```
! DC1 — corrige o ticket B4
router ospf 1
 auto-cost reference-bandwidth 10000
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| DC1 | `show ip ospf interface brief` | Et0/0 cost 10, Et0/1 cost 100 |
| DC1 | `show ip route 10.255.255.2` | Um único caminho via 10.2.21.1 |

**💡 Lição:** Com links ≥ 100M, ajuste `auto-cost reference-bandwidth` **igual em todos** os roteadores. O OSPFv3 deste lab já estava certo: compare!

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! EDGE — reaplica o defeito B4
router ospf 1
 no auto-cost reference-bandwidth 10000
end
```
```
! ABR1 — reaplica o defeito B4
router ospf 1
 no auto-cost reference-bandwidth 10000
end
```
```
! ABR2 — reaplica o defeito B4
router ospf 1
 no auto-cost reference-bandwidth 10000
end
```
```
! BR1 — reaplica o defeito B4
router ospf 1
 no auto-cost reference-bandwidth 10000
end
```
```
! DC1 — reaplica o defeito B4
router ospf 1
 no auto-cost reference-bandwidth 10000
end
```
</details>

---

### Ticket B5 — BÔNUS IPv6 — default só no AF IPv6 (BÔNUS)

**Chamado do usuário:** “IPv4 navega, IPv6 não.” (aplique o *break* antes)

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| ABR2 | `show ipv6 route ::/0` | Sem rota |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| EDGE | `show ospfv3 database external` | Sem LSA de ::/0 |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| EDGE | `show running-config | section ospfv3` | AF IPv6 sem `default-information originate` |

**4) Correção** (em `configure terminal`)

```
! EDGE — corrige o ticket B5
router ospfv3 1
 address-family ipv6 unicast
  default-information originate
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| ABR2 | `show ipv6 route ::/0` | `OE2 ::/0` |

**💡 Lição:** Cada address-family tem sua própria configuração de default.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! EDGE — reaplica o defeito B5
router ospfv3 1
 address-family ipv6 unicast
  no default-information originate
end
```
</details>

---

## 5. Configurações iniciais (com os defeitos) — comentadas

Linhas marcadas com `! >>>>> DEFEITO` são os pontos plantados. As demais linhas `! ---` são comentários didáticos.

### ABR1

```
! --- Nome do equipamento
hostname ABR1
no ip domain lookup
! --- Habilita roteamento IPv6 (pilha dupla)
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ip address 10.255.255.3 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::3/128
 no shutdown
interface Ethernet0/0
 description Link para EDGE
 ip address 10.0.12.2 255.255.255.252
 ipv6 address 2001:db8:acad:012::2/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/1
 description Link para ABR2
 ip address 10.0.23.1 255.255.255.252
 ipv6 address 2001:db8:acad:023::1/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/2
 description Link para BR1
 ip address 10.1.11.1 255.255.255.252
 ipv6 address 2001:db8:acad:111::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 bandwidth 1000000
 no shutdown
interface Ethernet0/3
 description Link para DC1
 ip address 10.2.22.1 255.255.255.252
 ipv6 address 2001:db8:acad:222::1/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 bandwidth 100000
 no shutdown
! --- OSPFv2 (IPv4)
router ospf 1
 router-id 2.2.2.2
! >>>>> DEFEITO B2: not-advertise esconde a filial das outras areas
 area 1 range 10.1.4.0 255.255.252.0 not-advertise
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 2.2.2.2
  auto-cost reference-bandwidth 10000
 exit-address-family
end
```

### ABR2

```
! --- Nome do equipamento
hostname ABR2
no ip domain lookup
! --- Habilita roteamento IPv6 (pilha dupla)
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ip address 10.255.255.4 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::4/128
 no shutdown
interface Ethernet0/0
 description Link para EDGE
 ip address 10.0.13.2 255.255.255.252
 ipv6 address 2001:db8:acad:013::2/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/1
 description Link para ABR1
 ip address 10.0.23.2 255.255.255.252
 ipv6 address 2001:db8:acad:023::2/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/2
 description Link para DC1
 ip address 10.2.21.1 255.255.255.252
 ipv6 address 2001:db8:acad:221::1/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 bandwidth 1000000
 no shutdown
! --- OSPFv2 (IPv4)
router ospf 1
 router-id 3.3.3.3
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 3.3.3.3
  auto-cost reference-bandwidth 10000
 exit-address-family
end
```

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
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ip address 10.255.255.5 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::5/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.1.4.1 255.255.255.0
 ipv6 address 2001:db8:acad:104::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 no shutdown
! --- LAN simulada (loopback)
interface Loopback2
 ip address 10.1.5.1 255.255.255.0
 ipv6 address 2001:db8:acad:105::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 no shutdown
! --- LAN simulada (loopback)
interface Loopback3
 ip address 10.1.6.1 255.255.255.0
 ipv6 address 2001:db8:acad:106::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 no shutdown
! --- LAN simulada (loopback)
interface Loopback4
 ip address 10.1.7.1 255.255.255.0
 ipv6 address 2001:db8:acad:107::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 no shutdown
interface Ethernet0/0
 description Link para ABR1
 ip address 10.1.11.2 255.255.255.252
 ipv6 address 2001:db8:acad:111::2/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 bandwidth 1000000
! >>>>> DEFEITO B1: point-to-point x broadcast no ABR1
 ip ospf network point-to-point
 no shutdown
! --- OSPFv2 (IPv4)
router ospf 1
 router-id 11.11.11.11
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 11.11.11.11
  auto-cost reference-bandwidth 10000
 exit-address-family
end
```

### DC1

```
! --- Nome do equipamento
hostname DC1
no ip domain lookup
! --- Habilita roteamento IPv6 (pilha dupla)
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ip address 10.255.255.6 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::6/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.2.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:201::1/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 ip ospf network point-to-point
 ospfv3 network point-to-point
 no shutdown
interface Ethernet0/0
 description Link para ABR2
 ip address 10.2.21.2 255.255.255.252
 ipv6 address 2001:db8:acad:221::2/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 bandwidth 1000000
 no shutdown
interface Ethernet0/1
 description Link para ABR1
 ip address 10.2.22.2 255.255.255.252
 ipv6 address 2001:db8:acad:222::2/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 bandwidth 100000
 no shutdown
! --- OSPFv2 (IPv4)
! >>>>> DEFEITO B4: sem 'auto-cost reference-bandwidth 10000' (vale para TODOS os roteadores)
router ospf 1
 router-id 21.21.21.21
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 21.21.21.21
  auto-cost reference-bandwidth 10000
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
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ip address 10.255.255.2 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::2/128
 no shutdown
interface Ethernet0/0
 description Link para ABR1
 ip address 10.0.12.1 255.255.255.252
 ipv6 address 2001:db8:acad:012::1/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/1
 description Link para ABR2
 ip address 10.0.13.1 255.255.255.252
 ipv6 address 2001:db8:acad:013::1/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/2
 description Link para ISP
 ip address 203.0.113.1 255.255.255.252
 ipv6 address 2001:db8:ffff::1/64
 no shutdown
! --- Default estatica para o ISP
ip route 0.0.0.0 0.0.0.0 203.0.113.2
ipv6 route ::/0 2001:db8:ffff::2
! --- OSPFv2 (IPv4)
! >>>>> DEFEITO B3: falta 'default-information originate' no OSPFv2
router ospf 1
 router-id 1.1.1.1
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 1.1.1.1
  auto-cost reference-bandwidth 10000
  default-information originate
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

<details><summary>ABR1</summary>

```
hostname ABR1
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback0
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ip address 10.255.255.3 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::3/128
 no shutdown
interface Ethernet0/0
 description Link para EDGE
 ip address 10.0.12.2 255.255.255.252
 ipv6 address 2001:db8:acad:012::2/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/1
 description Link para ABR2
 ip address 10.0.23.1 255.255.255.252
 ipv6 address 2001:db8:acad:023::1/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/2
 description Link para BR1
 ip address 10.1.11.1 255.255.255.252
 ipv6 address 2001:db8:acad:111::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 bandwidth 1000000
 no shutdown
interface Ethernet0/3
 description Link para DC1
 ip address 10.2.22.1 255.255.255.252
 ipv6 address 2001:db8:acad:222::1/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 bandwidth 100000
 no shutdown
router ospf 1
 router-id 2.2.2.2
 auto-cost reference-bandwidth 10000
 area 1 range 10.1.4.0 255.255.252.0
router ospfv3 1
 address-family ipv6 unicast
  router-id 2.2.2.2
  auto-cost reference-bandwidth 10000
 exit-address-family
end
```
</details>

<details><summary>ABR2</summary>

```
hostname ABR2
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback0
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ip address 10.255.255.4 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::4/128
 no shutdown
interface Ethernet0/0
 description Link para EDGE
 ip address 10.0.13.2 255.255.255.252
 ipv6 address 2001:db8:acad:013::2/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/1
 description Link para ABR1
 ip address 10.0.23.2 255.255.255.252
 ipv6 address 2001:db8:acad:023::2/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/2
 description Link para DC1
 ip address 10.2.21.1 255.255.255.252
 ipv6 address 2001:db8:acad:221::1/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 bandwidth 1000000
 no shutdown
router ospf 1
 router-id 3.3.3.3
 auto-cost reference-bandwidth 10000
router ospfv3 1
 address-family ipv6 unicast
  router-id 3.3.3.3
  auto-cost reference-bandwidth 10000
 exit-address-family
end
```
</details>

<details><summary>BR1</summary>

```
hostname BR1
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback0
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ip address 10.255.255.5 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::5/128
 no shutdown
interface Loopback1
 ip address 10.1.4.1 255.255.255.0
 ipv6 address 2001:db8:acad:104::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 no shutdown
interface Loopback2
 ip address 10.1.5.1 255.255.255.0
 ipv6 address 2001:db8:acad:105::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 no shutdown
interface Loopback3
 ip address 10.1.6.1 255.255.255.0
 ipv6 address 2001:db8:acad:106::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 no shutdown
interface Loopback4
 ip address 10.1.7.1 255.255.255.0
 ipv6 address 2001:db8:acad:107::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 no shutdown
interface Ethernet0/0
 description Link para ABR1
 ip address 10.1.11.2 255.255.255.252
 ipv6 address 2001:db8:acad:111::2/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 bandwidth 1000000
 no shutdown
router ospf 1
 router-id 11.11.11.11
 auto-cost reference-bandwidth 10000
router ospfv3 1
 address-family ipv6 unicast
  router-id 11.11.11.11
  auto-cost reference-bandwidth 10000
 exit-address-family
end
```
</details>

<details><summary>DC1</summary>

```
hostname DC1
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback0
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ip address 10.255.255.6 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::6/128
 no shutdown
interface Loopback1
 ip address 10.2.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:201::1/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 ip ospf network point-to-point
 ospfv3 network point-to-point
 no shutdown
interface Ethernet0/0
 description Link para ABR2
 ip address 10.2.21.2 255.255.255.252
 ipv6 address 2001:db8:acad:221::2/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 bandwidth 1000000
 no shutdown
interface Ethernet0/1
 description Link para ABR1
 ip address 10.2.22.2 255.255.255.252
 ipv6 address 2001:db8:acad:222::2/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 bandwidth 100000
 no shutdown
router ospf 1
 router-id 21.21.21.21
 auto-cost reference-bandwidth 10000
router ospfv3 1
 address-family ipv6 unicast
  router-id 21.21.21.21
  auto-cost reference-bandwidth 10000
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
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ip address 10.255.255.2 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::2/128
 no shutdown
interface Ethernet0/0
 description Link para ABR1
 ip address 10.0.12.1 255.255.255.252
 ipv6 address 2001:db8:acad:012::1/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/1
 description Link para ABR2
 ip address 10.0.13.1 255.255.255.252
 ipv6 address 2001:db8:acad:013::1/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 bandwidth 1000000
 no shutdown
interface Ethernet0/2
 description Link para ISP
 ip address 203.0.113.1 255.255.255.252
 ipv6 address 2001:db8:ffff::1/64
 no shutdown
ip route 0.0.0.0 0.0.0.0 203.0.113.2
ipv6 route ::/0 2001:db8:ffff::2
router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 10000
 default-information originate
router ospfv3 1
 address-family ipv6 unicast
  router-id 1.1.1.1
  auto-cost reference-bandwidth 10000
  default-information originate
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

