# Configurações IOS por nó

Configurações completas de cada roteador dos 4 labs, para montar a topologia em **EVE-NG, PNETLab, GNS3** ou equipamento real.

Um arquivo por topologia, com todos os nós: [EIGRP-A](EIGRP-A.md) · [EIGRP-B](EIGRP-B.md) · [OSPF-A](OSPF-A.md) · [OSPF-B](OSPF-B.md)

Ou um arquivo por nó, para copiar e colar:

```
configs/
  <LAB>/problemas/<NÓ>.txt   ← sobe com os defeitos (marcados com '! >>>>> DEFEITO')
  <LAB>/solucao/<NÓ>.txt     ← todos os tickets corrigidos
```

As duas versões foram testadas no CML2 com Cisco IOL-XE 17.18.2.

## Como aplicar

1. Monte a topologia com os cabos da tabela do lab (abaixo).
2. Em cada roteador: `enable` → `configure terminal` → cole o arquivo inteiro. As linhas com `!` são comentários e o IOS as ignora.
3. Opcional: `copy running-config startup-config`.
4. Espere convergir (EIGRP ~30 s; OSPF ~40 s).

## Imagem e nomes de interface

| Plataforma | Imagem sugerida | Interfaces |
|---|---|---|
| EVE-NG / PNETLab | IOL (i86bi-linux-l3) ou IOL-XE | `Ethernet0/x`: usar os arquivos como estão |
| GNS3 | IOU/IOL L3 | `Ethernet0/x`: usar os arquivos como estão |
| GNS3 / EVE com IOSv ou CSR1000v / Cat8000v | IOSv 15.9 ou IOS-XE 17 | trocar `Ethernet0/0..3` por `GigabitEthernet0/0..3` (IOSv) ou `GigabitEthernet1..4` (CSR/C8000v) |

Recursos usados: EIGRP *named mode*, OSPFv3 com address-family, `ip mtu`, key chain MD5. Precisa de IOS 15.x ou IOS-XE recente; imagens antigas (12.x) não servem.

> No CSR1000v/Cat8000v a interface `GigabitEthernet1` costuma ser a de gerência. Ajuste o mapeamento se for o caso.

## Cabeamento por lab

### EIGRP-A

| Lado A | Lado B | IPv4 | IPv6 |
|---|---|---|---|
| CORE1 e0/0 | CORE2 e0/0 | 10.0.12.0/30 | 2001:db8:acad:12::/64 |
| CORE1 e0/1 | BR1 e0/0 | 10.0.11.0/30 | 2001:db8:acad:11::/64 |
| CORE1 e0/2 | BR3 e0/0 | 10.0.13.0/30 | 2001:db8:acad:13::/64 |
| CORE2 e0/1 | BR2 e0/0 | 10.0.14.0/30 | 2001:db8:acad:14::/64 |
| CORE2 e0/2 | DC1 e0/0 | 10.0.15.0/30 | 2001:db8:acad:15::/64 |

Arquivos: [`problemas/`](EIGRP-A/problemas/) · [`solucao/`](EIGRP-A/solucao/) · roteiro: [`Lab_EIGRP-A.md`](../Lab_EIGRP-A.md)

### EIGRP-B

| Lado A | Lado B | IPv4 | IPv6 |
|---|---|---|---|
| CORE1 e0/0 | CORE2 e0/0 | 10.0.12.0/30 | 2001:db8:acad:12::/64 |
| CORE1 e0/1 | EDGE e0/0 | 10.0.13.0/30 | 2001:db8:acad:13::/64 |
| CORE2 e0/1 | EDGE e0/1 | 10.0.23.0/30 | 2001:db8:acad:23::/64 |
| CORE1 e0/2 | BR1 e0/0 | 10.0.14.0/30 | 2001:db8:acad:14::/64 |
| CORE2 e0/2 | BR2 e0/0 | 10.0.15.0/30 | 2001:db8:acad:15::/64 |
| EDGE e0/2 | ISP e0/0 | 203.0.113.0/30 | 2001:db8:ffff::/64 |

Arquivos: [`problemas/`](EIGRP-B/problemas/) · [`solucao/`](EIGRP-B/solucao/) · roteiro: [`Lab_EIGRP-B.md`](../Lab_EIGRP-B.md)

### OSPF-A

| Lado A | Lado B | IPv4 | IPv6 |
|---|---|---|---|
| CORE e0/0 | ABR1 e0/0 | 10.0.12.0/30 (área 0) | 2001:db8:acad:12::/64 |
| CORE e0/1 | ABR2 e0/0 | 10.0.13.0/30 (área 0) | 2001:db8:acad:13::/64 |
| ABR1 e0/1 | BR1 e0/0 | 10.1.11.0/30 (área 1) | 2001:db8:acad:111::/64 |
| ABR1 e0/2 | BR2 e0/0 | 10.1.12.0/30 (área 1) | 2001:db8:acad:112::/64 |
| ABR2 e0/1 | DC1 e0/0 | 10.2.21.0/30 (área 2) | 2001:db8:acad:221::/64 |

Arquivos: [`problemas/`](OSPF-A/problemas/) · [`solucao/`](OSPF-A/solucao/) · roteiro: [`Lab_OSPF-A.md`](../Lab_OSPF-A.md)

### OSPF-B

| Lado A | Lado B | IPv4 | IPv6 |
|---|---|---|---|
| EDGE e0/0 | ABR1 e0/0 | 10.0.12.0/30 (área 0, 1G) | 2001:db8:acad:12::/64 |
| EDGE e0/1 | ABR2 e0/0 | 10.0.13.0/30 (área 0, 1G) | 2001:db8:acad:13::/64 |
| ABR1 e0/1 | ABR2 e0/1 | 10.0.23.0/30 (área 0, 1G) | 2001:db8:acad:23::/64 |
| ABR1 e0/2 | BR1 e0/0 | 10.1.11.0/30 (área 1, 1G) | 2001:db8:acad:111::/64 |
| ABR2 e0/2 | DC1 e0/0 | 10.2.21.0/30 (área 2, **1G principal**) | 2001:db8:acad:221::/64 |
| ABR1 e0/3 | DC1 e0/1 | 10.2.22.0/30 (área 2, **100M backup**) | 2001:db8:acad:222::/64 |
| EDGE e0/2 | ISP e0/0 | 203.0.113.0/30 (fora do OSPF) | 2001:db8:ffff::/64 |

Arquivos: [`problemas/`](OSPF-B/problemas/) · [`solucao/`](OSPF-B/solucao/) · roteiro: [`Lab_OSPF-B.md`](../Lab_OSPF-B.md)
