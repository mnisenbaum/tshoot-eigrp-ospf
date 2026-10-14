# Configurações IOS — Lab OSPF-B

Lab OSPF-B — Rotas e caminho

Configurações completas de todos os nós, testadas no CML2 com IOL-XE 17.18. Para aplicar, use `configure terminal` e cole o bloco do nó.
Para EVE-NG, PNETLab ou GNS3, veja a tabela de interfaces em [`README.md`](README.md).

![Topologia](../topologias_png/OSPF-B_SOLUCAO.png)

## Cabeamento

| Lado A | Lado B | IPv4 | IPv6 |
|---|---|---|---|
| EDGE e0/0 | ABR1 e0/0 | 10.0.12.0/30 (área 0, 1G) | 2001:db8:acad:12::/64 |
| EDGE e0/1 | ABR2 e0/0 | 10.0.13.0/30 (área 0, 1G) | 2001:db8:acad:13::/64 |
| ABR1 e0/1 | ABR2 e0/1 | 10.0.23.0/30 (área 0, 1G) | 2001:db8:acad:23::/64 |
| ABR1 e0/2 | BR1 e0/0 | 10.1.11.0/30 (área 1, 1G) | 2001:db8:acad:111::/64 |
| ABR2 e0/2 | DC1 e0/0 | 10.2.21.0/30 (área 2, **1G principal**) | 2001:db8:acad:221::/64 |
| ABR1 e0/3 | DC1 e0/1 | 10.2.22.0/30 (área 2, **100M backup**) | 2001:db8:acad:222::/64 |
| EDGE e0/2 | ISP e0/0 | 203.0.113.0/30 (fora do OSPF) | 2001:db8:ffff::/64 |

## Defeitos plantados

| Ticket | Problema | Onde |
|---|---|---|
| B1 | Network type diferente (point-to-point × broadcast) | BR1 |
| B2 | area range ... not-advertise | ABR1 |
| B3 | ASBR sem default-information originate | EDGE |
| B4 | reference-bandwidth padrão (100 Mbps): 1G e 100M com o mesmo custo | todos |

Índice: [Com problemas](#1-configurações-com-problemas) · [Solução](#2-configurações-da-solução)

## 1. Configurações com problemas

Cada defeito está marcado com `! >>>>> DEFEITO`.

### ABR1 (com problemas)

```
! ==========================================================
! Lab OSPF-B - ABR1 - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### ABR2 (com problemas)

```
! ==========================================================
! Lab OSPF-B - ABR2 - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### BR1 (com problemas)

```
! ==========================================================
! Lab OSPF-B - BR1 - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
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

### DC1 (com problemas)

```
! ==========================================================
! Lab OSPF-B - DC1 - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### EDGE (com problemas)

```
! ==========================================================
! Lab OSPF-B - EDGE - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
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

### ISP (com problemas)

```
! ==========================================================
! Lab OSPF-B - ISP - COM OS DEFEITOS PLANTADOS (linhas '! >>>>> DEFEITO' indicam cada um)
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

### ABR1 (solução)

```
! ==========================================================
! Lab OSPF-B - ABR1 - SOLUCAO - todos os tickets corrigidos
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### ABR2 (solução)

```
! ==========================================================
! Lab OSPF-B - ABR2 - SOLUCAO - todos os tickets corrigidos
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### BR1 (solução)

```
! ==========================================================
! Lab OSPF-B - BR1 - SOLUCAO - todos os tickets corrigidos
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

### DC1 (solução)

```
! ==========================================================
! Lab OSPF-B - DC1 - SOLUCAO - todos os tickets corrigidos
! Plataforma testada: Cisco IOL-XE 17.18 (CML2). Interfaces Ethernet0/x.
! Uso: entre em 'configure terminal' e cole o bloco abaixo.
! ==========================================================
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

### EDGE (solução)

```
! ==========================================================
! Lab OSPF-B - EDGE - SOLUCAO - todos os tickets corrigidos
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

### ISP (solução)

```
! ==========================================================
! Lab OSPF-B - ISP - SOLUCAO - todos os tickets corrigidos
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
