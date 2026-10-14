# Lab OSPF-A — Adjacência

> Lab no CML2: **ENARSI Tshoot OSPF-A Adjacencia** · Plataforma IOL-XE 17.18 · Console sem senha
> Os nós sobem **já com os defeitos** (startup-config). Para recomeçar do zero: *Stop → Wipe → Start* no CML2, ou aplique os blocos **Quebrar** de cada ticket.

## 1. Cenário

OSPF **multiárea**: backbone (CORE, ABR1, ABR2), **área 1 stub** (filiais BR1/BR2) e **área 2** (datacenter DC1).
**Pilha dupla:** OSPFv2 para IPv4 + **OSPFv3 AF** para IPv6. Todos os defeitos estão **só no OSPFv2**: as adjacências OSPFv3 sobem normalmente.
💡 Ordem sugerida: A1 → A3 → A2 → A4 (A3 isola a área 1 do backbone; os tickets A2/A4 são verificados localmente no ABR1).

```
                         CORE   (área 0)
                  e0/0 /      \ e0/1
                 e0/0 /        \ e0/0
                   ABR1          ABR2
            e0/1 /     \ e0/2       \ e0/1
          e0/0  /       \ e0/0       \ e0/0
              BR1        BR2          DC1
            (área 1 stub)          (área 2)
```

## 2. Endereçamento

| Enlace | | IPv4 | IPv6 |
|---|---|---|---|
| CORE e0/0 | ABR1 e0/0 | 10.0.12.0/30 (área 0) | 2001:db8:acad:12::/64 |
| CORE e0/1 | ABR2 e0/0 | 10.0.13.0/30 (área 0) | 2001:db8:acad:13::/64 |
| ABR1 e0/1 | BR1 e0/0 | 10.1.11.0/30 (área 1) | 2001:db8:acad:111::/64 |
| ABR1 e0/2 | BR2 e0/0 | 10.1.12.0/30 (área 1) | 2001:db8:acad:112::/64 |
| ABR2 e0/1 | DC1 e0/0 | 10.2.21.0/30 (área 2) | 2001:db8:acad:221::/64 |

| Roteador | LANs (loopbacks) | Router-ID |
|---|---|---|
| CORE | Lo1 10.0.10.1/24 | RID 1.1.1.1 |
| ABR1 | — | RID 2.2.2.2 |
| ABR2 | — | RID 3.3.3.3 |
| BR1 | Lo1 10.1.1.1/24 | RID 11.11.11.11 |
| BR2 | Lo1 10.1.2.1/24 | RID 12.12.12.12 |
| DC1 | Lo1 10.2.1.1/24 | RID 21.21.21.21 |

Loopback0 de cada nó: `10.255.255.x/32` e `2001:db8:acad:ffff::x/128`.

## 3. Resumo dos tickets

| # | Problema | Categoria ENARSI | Onde |
|---|---|---|---|
| A1 | Hello/Dead diferentes | 6.4 Adjacência | DC1 |
| A3 | MTU diferente | 6.4 Adjacência | ABR1 |
| A2 | Área diferente no enlace | 6.4 Adjacência | BR2 |
| A4 | Flag stub divergente | 6.7 Áreas especiais | BR1 |
| A5 ★ | BÔNUS IPv6 — OSPFv3 tradicional × OSPFv3 AF | 6.4 / OSPFv3 | DC1 |

★ = bônus. Começa **corrigido**; aplique o bloco *Quebrar* se houver tempo.

## 4. Roteiro dos tickets (sintoma → isolar → causa → correção → verificação)

### Ticket A1 — Hello/Dead diferentes

**Chamado do usuário:** “O datacenter sumiu depois que alguém ‘acelerou a convergência’ no DC1.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| CORE | `ping 10.2.1.1 source 10.0.10.1` | Falha |
| CORE | `ping 2001:db8:acad:201::1 source loopback1` | Sucesso (IPv6 OK) |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| ABR2 | `show ip ospf neighbor` | Só o CORE (1.1.1.1) — falta 21.21.21.21 |
| ABR2 | `show ospfv3 neighbor` | DC1 aparece em OSPFv3 |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| ABR2 | `show ip ospf interface Ethernet0/1 | include Timer` | `Hello 10, Dead 40` |
| DC1 | `show ip ospf interface Ethernet0/0 | include Timer` | `Hello 5, Dead 20` → diferente |
| ABR2 | `debug ip ospf hello  (undebug all)` | `Mismatched hello parameters from 10.2.21.2` |

**4) Correção** (em `configure terminal`)

```
! DC1 — corrige o ticket A1
interface Ethernet0/0
 no ip ospf hello-interval
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| ABR2 | `show ip ospf neighbor` | 21.21.21.21 FULL |
| CORE | `ping 10.2.1.1 source 10.0.10.1` | !!!!! |

**💡 Lição:** Hello e Dead precisam ser **iguais** nos dois lados. Mudar só o hello muda o dead automaticamente (4×).

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! DC1 — reaplica o defeito A1
interface Ethernet0/0
 ip ospf hello-interval 5
end
```
</details>

---

### Ticket A3 — MTU diferente

**Chamado do usuário:** “As filiais 1 e 2 estão isoladas da matriz. O link CORE–ABR1 está UP/UP e o ping no link funciona.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| CORE | `ping 10.0.12.2` | Sucesso (L3 do link OK) |
| CORE | `show ip route ospf` | Sem rotas da área 1 |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| CORE | `show ip ospf neighbor` | 2.2.2.2 preso em **EXSTART/EXCHANGE** (ou sumindo e voltando) |
| ABR1 | `show logging | include OSPF` | `from EXSTART to DOWN, Neighbor Down: Too many retransmissions` |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip interface Ethernet0/0 | include MTU` | `MTU is 1400 bytes` |
| CORE | `show ip interface Ethernet0/0 | include MTU` | `MTU is 1500 bytes` |

**4) Correção** (em `configure terminal`)

```
! ABR1 — corrige o ticket A3
interface Ethernet0/0
 no ip mtu
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| CORE | `show ip ospf neighbor` | 2.2.2.2 FULL |

**💡 Lição:** MTU diferente: hellos passam, mas a troca de DBD falha → **EXSTART/EXCHANGE**. (Contorno: `ip ospf mtu-ignore`, mas o certo é igualar o MTU.)

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! ABR1 — reaplica o defeito A3
interface Ethernet0/0
 ip mtu 1400
end
```
</details>

---

### Ticket A2 — Área diferente no enlace

**Chamado do usuário:** “A Filial 2 foi reconfigurada por um terceirizado e ficou isolada.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `ping 10.1.2.1` | Falha (sem rota) |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip ospf neighbor` | Sem 12.12.12.12 |
| ABR1 | `show logging | include mismatched` | `Received invalid packet: mismatched area ID from backbone area from 10.1.12.2, Ethernet0/2` |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| BR2 | `show ip ospf interface brief` | Et0/0 na **área 0**; Lo0/Lo1 na área 1 |

**4) Correção** (em `configure terminal`)

```
! BR2 — corrige o ticket A2
interface Ethernet0/0
 ip ospf 1 area 1
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip ospf neighbor` | 12.12.12.12 FULL |
| ABR1 | `show ip route 10.1.2.0` | O 10.1.2.0/24 |

**💡 Lição:** Os dois lados do enlace precisam estar na **mesma área**. O log *mismatched area ID* entrega o problema.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! BR2 — reaplica o defeito A2
interface Ethernet0/0
 ip ospf 1 area 0
end
```
</details>

---

### Ticket A4 — Flag stub divergente

**Chamado do usuário:** “A Filial 1 não sobe vizinhança com o ABR1. O cabo e os IPs foram conferidos.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `ping 10.1.1.1` | Falha (sem rota) |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip ospf neighbor` | Sem 11.11.11.11 |
| ABR1 | `show ospfv3 neighbor` | BR1 aparece em OSPFv3 |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip ospf | include Area|stub` | `Area 1` … `It is a stub area` |
| BR1 | `show ip ospf | include normal|stub` | `1 normal 0 stub 0 nssa` → BR1 acha que a área 1 é **normal** |
| ABR1 | `debug ip ospf hello  (undebug all)` | `Hello from 10.1.11.2 with mismatched Stub/Transit area option bit` |

**4) Correção** (em `configure terminal`)

```
! BR1 — corrige o ticket A4
router ospf 1
 area 1 stub
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| ABR1 | `show ip ospf neighbor` | 11.11.11.11 FULL |
| BR1 | `show ip route 0.0.0.0` | `O*IA 0.0.0.0/0` (default injetada pelo ABR na área stub) |

**💡 Lição:** O flag stub (bit E) vai no hello: **todos** os roteadores da área precisam concordar que ela é stub.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! BR1 — reaplica o defeito A4
router ospf 1
 no area 1 stub
end
```
</details>

---

### Ticket A5 — BÔNUS IPv6 — OSPFv3 tradicional × OSPFv3 AF (BÔNUS)

**Chamado do usuário:** “Migraram o DC1 usando um modelo antigo de configuração IPv6.” (aplique o *break* antes)

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| ABR2 | `show ospfv3 neighbor` | Sem 21.21.21.21 |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| DC1 | `show ospfv3` | Processo `ipv6 router ospf` (tradicional) em vez de `router ospfv3` AF |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| DC1 | `show running-config | section ospf` | `ipv6 router ospf 1` e `ipv6 ospf 1 area 2` nas interfaces |

**4) Correção** (em `configure terminal`)

```
! DC1 — corrige o ticket A5
no ipv6 router ospf 1
router ospfv3 1
 address-family ipv6 unicast
  router-id 21.21.21.21
interface Ethernet0/0
 ospfv3 1 ipv6 area 2
interface Loopback0
 ospfv3 1 ipv6 area 2
interface Loopback1
 ospfv3 1 ipv6 area 2
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| ABR2 | `show ospfv3 neighbor` | 21.21.21.21 FULL |

**💡 Lição:** (Caso do lab oficial 9.1.1.1) Não misture os dois estilos no mesmo domínio. ⚠️ Bônus **não testado** no lab — teste antes se for usar.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! DC1 — reaplica o defeito A5
no router ospfv3 1
ipv6 router ospf 1
 router-id 21.21.21.21
interface Ethernet0/0
 ipv6 ospf 1 area 2
interface Loopback0
 ipv6 ospf 1 area 2
interface Loopback1
 ipv6 ospf 1 area 2
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
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 0
 ip address 10.255.255.2 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::2/128
 no shutdown
interface Ethernet0/0
 description Link para CORE
 ip address 10.0.12.2 255.255.255.252
 ipv6 address 2001:db8:acad:012::2/64
 ip ospf 1 area 0
! >>>>> DEFEITO A3: MTU 1400 x 1500 no CORE -> EXSTART/EXCHANGE
 ip mtu 1400
 ospfv3 1 ipv6 area 0
 no shutdown
interface Ethernet0/1
 description Link para BR1
 ip address 10.1.11.1 255.255.255.252
 ipv6 address 2001:db8:acad:111::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 no shutdown
interface Ethernet0/2
 description Link para BR2
 ip address 10.1.12.1 255.255.255.252
 ipv6 address 2001:db8:acad:112::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 no shutdown
! --- OSPFv2 (IPv4)
router ospf 1
 router-id 2.2.2.2
 area 1 stub
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 2.2.2.2
  area 1 stub
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
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 0
 ip address 10.255.255.3 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::3/128
 no shutdown
interface Ethernet0/0
 description Link para CORE
 ip address 10.0.13.2 255.255.255.252
 ipv6 address 2001:db8:acad:013::2/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 no shutdown
interface Ethernet0/1
 description Link para DC1
 ip address 10.2.21.1 255.255.255.252
 ipv6 address 2001:db8:acad:221::1/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 no shutdown
! --- OSPFv2 (IPv4)
router ospf 1
 router-id 3.3.3.3
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 3.3.3.3
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
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 1
 ip address 10.255.255.4 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::4/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.1.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:101::1/64
 ip ospf 1 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 1
 no shutdown
interface Ethernet0/0
 description Link para ABR1
 ip address 10.1.11.2 255.255.255.252
 ipv6 address 2001:db8:acad:111::2/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 no shutdown
! --- OSPFv2 (IPv4)
! >>>>> DEFEITO A4: falta 'area 1 stub' no OSPFv2 (no OSPFv3 esta correto)
router ospf 1
 router-id 11.11.11.11
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 11.11.11.11
  area 1 stub
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
 ip ospf 1 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 1
 ip address 10.255.255.5 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::5/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.1.2.1 255.255.255.0
 ipv6 address 2001:db8:acad:102::1/64
 ip ospf 1 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 1
 no shutdown
interface Ethernet0/0
 description Link para ABR1
 ip address 10.1.12.2 255.255.255.252
 ipv6 address 2001:db8:acad:112::2/64
! >>>>> DEFEITO A2: enlace na area 0 (ABR1 usa area 1)
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 1
 no shutdown
! --- OSPFv2 (IPv4)
router ospf 1
 router-id 12.12.12.12
 area 1 stub
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 12.12.12.12
  area 1 stub
 exit-address-family
end
```

### CORE

```
! --- Nome do equipamento
hostname CORE
no ip domain lookup
! --- Habilita roteamento IPv6 (pilha dupla)
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip ospf 1 area 0
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 0
 ip address 10.255.255.1 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::1/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.0.10.1 255.255.255.0
 ipv6 address 2001:db8:acad:010::1/64
 ip ospf 1 area 0
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 0
 no shutdown
interface Ethernet0/0
 description Link para ABR1
 ip address 10.0.12.1 255.255.255.252
 ipv6 address 2001:db8:acad:012::1/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 no shutdown
interface Ethernet0/1
 description Link para ABR2
 ip address 10.0.13.1 255.255.255.252
 ipv6 address 2001:db8:acad:013::1/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 no shutdown
! --- OSPFv2 (IPv4)
router ospf 1
 router-id 1.1.1.1
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 1.1.1.1
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
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 2
 ip address 10.255.255.6 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::6/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.2.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:201::1/64
 ip ospf 1 area 2
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 2
 no shutdown
interface Ethernet0/0
 description Link para ABR2
 ip address 10.2.21.2 255.255.255.252
 ipv6 address 2001:db8:acad:221::2/64
 ip ospf 1 area 2
! >>>>> DEFEITO A1: hello 5 / dead 20 (ABR2 usa 10/40)
 ip ospf hello-interval 5
 ospfv3 1 ipv6 area 2
 no shutdown
! --- OSPFv2 (IPv4)
router ospf 1
 router-id 21.21.21.21
! --- OSPFv3 com address-family (IPv6)
router ospfv3 1
 address-family ipv6 unicast
  router-id 21.21.21.21
 exit-address-family
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
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 0
 ip address 10.255.255.2 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::2/128
 no shutdown
interface Ethernet0/0
 description Link para CORE
 ip address 10.0.12.2 255.255.255.252
 ipv6 address 2001:db8:acad:012::2/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 no shutdown
interface Ethernet0/1
 description Link para BR1
 ip address 10.1.11.1 255.255.255.252
 ipv6 address 2001:db8:acad:111::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 no shutdown
interface Ethernet0/2
 description Link para BR2
 ip address 10.1.12.1 255.255.255.252
 ipv6 address 2001:db8:acad:112::1/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 no shutdown
router ospf 1
 router-id 2.2.2.2
 area 1 stub
router ospfv3 1
 address-family ipv6 unicast
  router-id 2.2.2.2
  area 1 stub
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
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 0
 ip address 10.255.255.3 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::3/128
 no shutdown
interface Ethernet0/0
 description Link para CORE
 ip address 10.0.13.2 255.255.255.252
 ipv6 address 2001:db8:acad:013::2/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 no shutdown
interface Ethernet0/1
 description Link para DC1
 ip address 10.2.21.1 255.255.255.252
 ipv6 address 2001:db8:acad:221::1/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 no shutdown
router ospf 1
 router-id 3.3.3.3
router ospfv3 1
 address-family ipv6 unicast
  router-id 3.3.3.3
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
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 1
 ip address 10.255.255.4 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::4/128
 no shutdown
interface Loopback1
 ip address 10.1.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:101::1/64
 ip ospf 1 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 1
 no shutdown
interface Ethernet0/0
 description Link para ABR1
 ip address 10.1.11.2 255.255.255.252
 ipv6 address 2001:db8:acad:111::2/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 no shutdown
router ospf 1
 router-id 11.11.11.11
 area 1 stub
router ospfv3 1
 address-family ipv6 unicast
  router-id 11.11.11.11
  area 1 stub
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
 ip ospf 1 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 1
 ip address 10.255.255.5 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::5/128
 no shutdown
interface Loopback1
 ip address 10.1.2.1 255.255.255.0
 ipv6 address 2001:db8:acad:102::1/64
 ip ospf 1 area 1
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 1
 no shutdown
interface Ethernet0/0
 description Link para ABR1
 ip address 10.1.12.2 255.255.255.252
 ipv6 address 2001:db8:acad:112::2/64
 ip ospf 1 area 1
 ospfv3 1 ipv6 area 1
 no shutdown
router ospf 1
 router-id 12.12.12.12
 area 1 stub
router ospfv3 1
 address-family ipv6 unicast
  router-id 12.12.12.12
  area 1 stub
 exit-address-family
end
```
</details>

<details><summary>CORE</summary>

```
hostname CORE
no ip domain lookup
ipv6 unicast-routing
line con 0
 exec-timeout 0 0
 logging synchronous
interface Loopback0
 ip ospf 1 area 0
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 0
 ip address 10.255.255.1 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::1/128
 no shutdown
interface Loopback1
 ip address 10.0.10.1 255.255.255.0
 ipv6 address 2001:db8:acad:010::1/64
 ip ospf 1 area 0
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 0
 no shutdown
interface Ethernet0/0
 description Link para ABR1
 ip address 10.0.12.1 255.255.255.252
 ipv6 address 2001:db8:acad:012::1/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 no shutdown
interface Ethernet0/1
 description Link para ABR2
 ip address 10.0.13.1 255.255.255.252
 ipv6 address 2001:db8:acad:013::1/64
 ip ospf 1 area 0
 ospfv3 1 ipv6 area 0
 no shutdown
router ospf 1
 router-id 1.1.1.1
router ospfv3 1
 address-family ipv6 unicast
  router-id 1.1.1.1
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
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 2
 ip address 10.255.255.6 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::6/128
 no shutdown
interface Loopback1
 ip address 10.2.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:201::1/64
 ip ospf 1 area 2
 ip ospf network point-to-point
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 2
 no shutdown
interface Ethernet0/0
 description Link para ABR2
 ip address 10.2.21.2 255.255.255.252
 ipv6 address 2001:db8:acad:221::2/64
 ip ospf 1 area 2
 ospfv3 1 ipv6 area 2
 no shutdown
router ospf 1
 router-id 21.21.21.21
router ospfv3 1
 address-family ipv6 unicast
  router-id 21.21.21.21
 exit-address-family
end
```
</details>

