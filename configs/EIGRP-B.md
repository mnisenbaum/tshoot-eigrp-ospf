# Configurações IOS — Lab EIGRP-B

Lab EIGRP-B — Rotas (tabela, stub, filtro, sumarização, RID)

Configurações completas de todos os nós, testadas no CML2 com IOL-XE 17.18. Para aplicar, use `configure terminal` e cole o bloco do nó.
Para EVE-NG, PNETLab ou GNS3, veja a tabela de interfaces em [`README.md`](README.md).

![Topologia](../topologias_png/EIGRP-B_SOLUCAO.png)

## Cabeamento

| Lado A | Lado B | IPv4 | IPv6 |
|---|---|---|---|
| CORE1 e0/0 | CORE2 e0/0 | 10.0.12.0/30 | 2001:db8:acad:12::/64 |
| CORE1 e0/1 | EDGE e0/0 | 10.0.13.0/30 | 2001:db8:acad:13::/64 |
| CORE2 e0/1 | EDGE e0/1 | 10.0.23.0/30 | 2001:db8:acad:23::/64 |
| CORE1 e0/2 | BR1 e0/0 | 10.0.14.0/30 | 2001:db8:acad:14::/64 |
| CORE2 e0/2 | BR2 e0/0 | 10.0.15.0/30 | 2001:db8:acad:15::/64 |
| EDGE e0/2 | ISP e0/0 | 203.0.113.0/30 | 2001:db8:ffff::/64 |

## Defeitos plantados

| Ticket | Problema | Onde |
|---|---|---|
| B1 | Stub receive-only | BR1 |
| B2 | Distribute-list com prefix-list sem permit final | CORE2 |
| B3 | Sumarização errada: 0.0.0.0/0 em vez de 10.12.0.0/22 | BR2 |
| B4 | Router-ID duplicado (EDGE = CORE1) | EDGE |

Índice: [Com problemas](#1-configurações-com-problemas) · [Solução](#2-configurações-da-solução)

## 1. Configurações com problemas

Cada defeito está marcado com `! >>>>> DEFEITO`.

### BR1 (com problemas)

```
! ==========================================================
! Lab EIGRP-B - BR1 - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### BR2 (com problemas)

```
! ==========================================================
! Lab EIGRP-B - BR2 - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### CORE1 (com problemas)

```
! ==========================================================
! Lab EIGRP-B - CORE1 - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### CORE2 (com problemas)

```
! ==========================================================
! Lab EIGRP-B - CORE2 - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### EDGE (com problemas)

```
! ==========================================================
! Lab EIGRP-B - EDGE - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### ISP (com problemas)

```
! ==========================================================
! Lab EIGRP-B - ISP - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

## 2. Configurações da solução

Todos os tickets corrigidos.

### BR1 (solução)

```
! ==========================================================
! Lab EIGRP-B - BR1 - SOLUCAO - todos os tickets corrigidos
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### BR2 (solução)

```
! ==========================================================
! Lab EIGRP-B - BR2 - SOLUCAO - todos os tickets corrigidos
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### CORE1 (solução)

```
! ==========================================================
! Lab EIGRP-B - CORE1 - SOLUCAO - todos os tickets corrigidos
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### CORE2 (solução)

```
! ==========================================================
! Lab EIGRP-B - CORE2 - SOLUCAO - todos os tickets corrigidos
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### EDGE (solução)

```
! ==========================================================
! Lab EIGRP-B - EDGE - SOLUCAO - todos os tickets corrigidos
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### ISP (solução)

```
! ==========================================================
! Lab EIGRP-B - ISP - SOLUCAO - todos os tickets corrigidos
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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
