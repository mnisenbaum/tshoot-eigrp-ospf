# Lab EIGRP-A — Vizinhança (adjacência)

> Lab no CML2: **ENARSI Tshoot EIGRP-A Vizinhanca** · Plataforma IOL-XE 17.18 · Console sem senha
> Os nós sobem **já com os defeitos** (startup-config). Para recomeçar do zero: *Stop → Wipe → Start* no CML2, ou aplique os blocos **Quebrar** de cada ticket.

## 1. Cenário

Empresa com **matriz** (CORE1/CORE2), **três filiais** (BR1, BR2, BR3) e um **datacenter** (DC1).
EIGRP *named mode* `EMPRESA`, AS 100, **pilha dupla** (AF IPv4 + AF IPv6).
Todos os defeitos estão **só no AF IPv4**: o IPv6 continua funcionando, o que ajuda a mostrar que o problema está no processo/AF e não no meio físico.

```
                 CORE1 e0/0 ───────── e0/0 CORE2
                /  e0/1      \ e0/2      e0/1 /     \ e0/2
          e0/0 /         e0/0 \       e0/0 /        \ e0/0
             BR1              BR3          BR2        DC1
```

## 2. Endereçamento

| Enlace | | IPv4 | IPv6 |
|---|---|---|---|
| CORE1 e0/0 | CORE2 e0/0 | 10.0.12.0/30 | 2001:db8:acad:12::/64 |
| CORE1 e0/1 | BR1 e0/0 | 10.0.11.0/30 | 2001:db8:acad:11::/64 |
| CORE1 e0/2 | BR3 e0/0 | 10.0.13.0/30 | 2001:db8:acad:13::/64 |
| CORE2 e0/1 | BR2 e0/0 | 10.0.14.0/30 | 2001:db8:acad:14::/64 |
| CORE2 e0/2 | DC1 e0/0 | 10.0.15.0/30 | 2001:db8:acad:15::/64 |

| Roteador | LANs (loopbacks) | Router-ID |
|---|---|---|
| CORE1 | Lo1 10.1.1.1/24 | RID 1.1.1.1 |
| CORE2 | Lo1 10.2.1.1/24 | RID 2.2.2.2 |
| BR1 | Lo1 10.11.1.1/24 | RID 11.11.11.11 |
| BR2 | Lo1 10.12.1.1/24 | RID 12.12.12.12 |
| BR3 | Lo1 10.13.1.1/24 | RID 13.13.13.13 |
| DC1 | Lo1 10.21.1.1/24, Lo2 10.21.2.1/24 | RID 21.21.21.21 |

Loopback0 de cada nó: `10.255.255.x/32` e `2001:db8:acad:ffff::x/128`.

## 3. Resumo dos tickets

| # | Problema | Categoria ENARSI | Onde |
|---|---|---|---|
| A1 | Autenticação: key-string diferente | 3.9 Autenticação | BR1 |
| A2 | Passive-interface no uplink | 3.4 Vizinhança | BR2 |
| A3 | Número de AS diferente | 3.4 Vizinhança | BR3 |
| A4 | K-values diferentes | 3.4 Vizinhança | DC1 |
| A5 ★ | BÔNUS IPv6 — af-interface shutdown | 3.8 EIGRP IPv6 | CORE2 |

★ = bônus. Começa **corrigido**; aplique o bloco *Quebrar* se houver tempo.

## 4. Roteiro dos tickets (sintoma → isolar → causa → correção → verificação)

### Ticket A1 — Autenticação: key-string diferente

**Chamado do usuário:** “A Filial 1 está sem acesso aos sistemas da matriz desde a troca de senhas da rede ontem à noite.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `ping 10.11.1.1 source 10.1.1.1` | Falha (.....) |
| CORE1 | `ping 2001:db8:acad:1101::1 source loopback1` | **Sucesso** — o IPv6 funciona! |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ip eigrp neighbors` | Não aparece 10.0.11.2 (BR1) em Et0/1 |
| CORE1 | `show ipv6 eigrp neighbors` | BR1 aparece (link-local em Et0/1) |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| BR1 | `show ip eigrp interfaces detail Ethernet0/0` | `Authentication mode is md5, key-chain is "EIGRP-KEYS"` |
| BR1 / CORE1 | `show key chain` | BR1: `key 1 -- text "Cisco123"` × CORE1: `"CISCO123"` (maiúsc./minúsc.) |
| BR1 | `debug eigrp packets hello  (depois: undebug all)` | Mensagens de *authentication failure / bad digest* |

**4) Correção** (em `configure terminal`)

```
! BR1 — corrige o ticket A1
key chain EIGRP-KEYS
 key 1
  key-string CISCO123
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ip eigrp neighbors` | 10.0.11.2 em Et0/1 |
| CORE1 | `ping 10.11.1.1 source 10.1.1.1` | !!!!! |

**💡 Lição:** Autenticação errada = vizinho nunca sobe. Compare **key chain** dos dois lados (é *case-sensitive*). No named mode a autenticação é **por AF**: por isso o IPv6 continuou de pé.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! BR1 — reaplica o defeito A1
key chain EIGRP-KEYS
 key 1
  key-string Cisco123
end
```
</details>

---

### Ticket A2 — Passive-interface no uplink

**Chamado do usuário:** “A Filial 2 foi ‘endurecida’ pela equipe de segurança e desde então não fala com ninguém.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `ping 10.12.1.1 source 10.2.1.1` | Falha |
| CORE2 | `ping 2001:db8:acad:1201::1 source loopback1` | Sucesso (IPv6 OK) |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `show ip eigrp neighbors` | Sem 10.0.14.2 (BR2) |
| BR2 | `show ip eigrp interfaces` | Et0/0 **não aparece** na lista (interface passiva não é listada) |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| BR2 | `show ip protocols | section eigrp` | `Passive Interface(s): Ethernet0/0` |
| BR2 | `show running-config | section router eigrp` | `af-interface Ethernet0/0` / `passive-interface` |

**4) Correção** (em `configure terminal`)

```
! BR2 — corrige o ticket A2
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/0
   no passive-interface
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `show ip eigrp neighbors` | 10.0.14.2 em Et0/1 |
| CORE2 | `ping 10.12.1.1 source 10.2.1.1` | !!!!! |

**💡 Lição:** Interface passiva **não envia hellos** → sem vizinho. Passive é para LANs de usuários, nunca para uplinks.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! BR2 — reaplica o defeito A2
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/0
   passive-interface
end
```
</details>

---

### Ticket A3 — Número de AS diferente

**Chamado do usuário:** “Inauguramos a Filial 3 hoje. O técnico de campo configurou o roteador pelo modelo, mas nada funciona.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `ping 10.13.1.1 source 10.1.1.1` | Falha |
| CORE1 | `ping 2001:db8:acad:1301::1 source loopback1` | Sucesso (IPv6 OK) |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ip eigrp neighbors` | Sem 10.0.13.2 (BR3) |
| BR3 | `show ip eigrp neighbors` | Cabeçalho mostra `AS(1000)` e nenhum vizinho |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| BR3 | `show ip protocols | include eigrp|Routing Protocol` | `Routing Protocol is "eigrp 1000"` |
| CORE1 | `show ip protocols | include eigrp|Routing Protocol` | `eigrp 100` → AS diferentes |

**4) Correção** (em `configure terminal`)

```
! BR3 — corrige o ticket A3
router eigrp EMPRESA
 no address-family ipv4 unicast autonomous-system 1000
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0
  eigrp router-id 13.13.13.13
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| CORE1 | `show ip eigrp neighbors` | 10.0.13.2 em Et0/2 |
| CORE1 | `ping 10.13.1.1 source 10.1.1.1` | !!!!! |

**💡 Lição:** O AS vai no hello: AS diferente = pacotes ignorados **sem log**. No named mode o AS fica no `address-family`, não no nome do processo.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! BR3 — reaplica o defeito A3
router eigrp EMPRESA
 no address-family ipv4 unicast autonomous-system 100
 address-family ipv4 unicast autonomous-system 1000
  network 10.0.0.0
  eigrp router-id 13.13.13.13
end
```
</details>

---

### Ticket A4 — K-values diferentes

**Chamado do usuário:** “O datacenter caiu depois que um analista ‘otimizou a métrica para considerar carga’.”

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `ping 10.21.1.1 source 10.2.1.1` | Falha |
| CORE2 | `ping 2001:db8:acad:2101::1 source loopback1` | Sucesso (IPv6 OK) |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `show ip eigrp neighbors` | Sem 10.0.15.2 (DC1) |
| CORE2 | `show logging | include K-value` | `Neighbor 10.0.15.2 (Ethernet0/2) is down: K-value mismatch` (repetido) |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| DC1 | `show ip protocols | include K|Metric` | `Metric weight K1=1, K2=1, K3=1 ...` × CORE2 `K1=1, K2=0, K3=1, K4=0, K5=0` |
| DC1 | `show running-config | section router eigrp` | `metric weights 0 1 1 1 0 0 0` |

**4) Correção** (em `configure terminal`)

```
! DC1 — corrige o ticket A4
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  no metric weights 0 1 1 1 0 0 0
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `show ip eigrp neighbors` | 10.0.15.2 em Et0/2 |
| CORE2 | `ping 10.21.1.1 source 10.2.1.1` | !!!!! |

**💡 Lição:** K-values precisam ser **iguais** em todo o domínio. É um dos poucos casos com log explícito: sempre olhe o `show logging`.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! DC1 — reaplica o defeito A4
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  metric weights 0 1 1 1 0 0 0
end
```
</details>

---

### Ticket A5 — BÔNUS IPv6 — af-interface shutdown (BÔNUS)

**Chamado do usuário:** “Agora é o contrário: o IPv4 do datacenter voltou, mas o IPv6 parou.” (aplique o *break* antes)

**1) Sintoma**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `ping 2001:db8:acad:2101::1 source loopback1` | Falha |
| CORE2 | `ping 10.21.1.1 source 10.2.1.1` | Sucesso |

**2) Isolar**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `show ipv6 eigrp neighbors` | Sem vizinho em Et0/2 |
| CORE2 | `show ipv6 eigrp interfaces` | Et0/2 não aparece |

**3) Causa**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `show running-config | section router eigrp` | No AF IPv6: `af-interface Ethernet0/2` / `shutdown` |

**4) Correção** (em `configure terminal`)

```
! CORE2 — corrige o ticket A5
router eigrp EMPRESA
 address-family ipv6 unicast autonomous-system 100
  af-interface Ethernet0/2
   no shutdown
end
```

**5) Verificação**

| Onde | Comando | O que observar |
|---|---|---|
| CORE2 | `show ipv6 eigrp neighbors` | Vizinho em Et0/2 |

**💡 Lição:** No AF IPv6 todas as interfaces com IPv6 entram automaticamente; para tirar uma, usa-se `af-interface ... shutdown`.

<details><summary>Quebrar de novo (reaplicar o defeito)</summary>

```
! CORE2 — reaplica o defeito A5
router eigrp EMPRESA
 address-family ipv6 unicast autonomous-system 100
  af-interface Ethernet0/2
   shutdown
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
key chain EIGRP-KEYS
 key 1
! >>>>> DEFEITO A1: key-string diferente do CORE1 (CISCO123)
  key-string Cisco123
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip address 10.255.255.3 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::3/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.11.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:1101::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE1
 ip address 10.0.11.2 255.255.255.252
 ipv6 address 2001:db8:acad:011::2/64
 no shutdown
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/0
   authentication mode md5
   authentication key-chain EIGRP-KEYS
  exit-af-interface
  network 10.0.0.0
  eigrp router-id 11.11.11.11
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
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
 ip address 10.255.255.5 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::5/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.12.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:1201::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE2
 ip address 10.0.14.2 255.255.255.252
 ipv6 address 2001:db8:acad:014::2/64
 no shutdown
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/0
! >>>>> DEFEITO A2: uplink passivo -> nao envia hellos IPv4
   passive-interface
  exit-af-interface
  network 10.0.0.0
  eigrp router-id 12.12.12.12
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 12.12.12.12
 exit-address-family
end
```

### BR3

```
! --- Nome do equipamento
hostname BR3
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
 ip address 10.13.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:1301::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE1
 ip address 10.0.13.2 255.255.255.252
 ipv6 address 2001:db8:acad:013::2/64
 no shutdown
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
! >>>>> DEFEITO A3: AS 1000 no AF IPv4 (dominio usa AS 100)
 address-family ipv4 unicast autonomous-system 1000
  network 10.0.0.0
  eigrp router-id 13.13.13.13
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 13.13.13.13
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
key chain EIGRP-KEYS
 key 1
  key-string CISCO123
! --- Loopback0: endereco de gerencia / identificacao
interface Loopback0
 ip address 10.255.255.1 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::1/128
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
 description Link para BR1
 ip address 10.0.11.1 255.255.255.252
 ipv6 address 2001:db8:acad:011::1/64
 no shutdown
interface Ethernet0/2
 description Link para BR3
 ip address 10.0.13.1 255.255.255.252
 ipv6 address 2001:db8:acad:013::1/64
 no shutdown
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/1
   authentication mode md5
   authentication key-chain EIGRP-KEYS
  exit-af-interface
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
 ip address 10.255.255.2 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::2/128
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
 description Link para BR2
 ip address 10.0.14.1 255.255.255.252
 ipv6 address 2001:db8:acad:014::1/64
 no shutdown
interface Ethernet0/2
 description Link para DC1
 ip address 10.0.15.1 255.255.255.252
 ipv6 address 2001:db8:acad:015::1/64
 no shutdown
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0
  eigrp router-id 2.2.2.2
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 2.2.2.2
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
 ip address 10.255.255.6 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::6/128
 no shutdown
! --- LAN simulada (loopback)
interface Loopback1
 ip address 10.21.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:2101::1/64
 no shutdown
! --- LAN simulada (loopback)
interface Loopback2
 ip address 10.21.2.1 255.255.255.0
 ipv6 address 2001:db8:acad:2102::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE2
 ip address 10.0.15.2 255.255.255.252
 ipv6 address 2001:db8:acad:015::2/64
 no shutdown
! --- EIGRP named mode - processo EMPRESA
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
! >>>>> DEFEITO A4: K2 e K3 alterados -> K-value mismatch
  metric weights 0 1 1 1 0 0 0
  network 10.0.0.0
  eigrp router-id 21.21.21.21
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 21.21.21.21
 exit-address-family
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
key chain EIGRP-KEYS
 key 1
  key-string CISCO123
interface Loopback0
 ip address 10.255.255.3 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::3/128
 no shutdown
interface Loopback1
 ip address 10.11.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:1101::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE1
 ip address 10.0.11.2 255.255.255.252
 ipv6 address 2001:db8:acad:011::2/64
 no shutdown
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/0
   authentication mode md5
   authentication key-chain EIGRP-KEYS
  exit-af-interface
  network 10.0.0.0
  eigrp router-id 11.11.11.11
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
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
 ip address 10.255.255.5 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::5/128
 no shutdown
interface Loopback1
 ip address 10.12.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:1201::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE2
 ip address 10.0.14.2 255.255.255.252
 ipv6 address 2001:db8:acad:014::2/64
 no shutdown
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0
  eigrp router-id 12.12.12.12
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 12.12.12.12
 exit-address-family
end
```
</details>

<details><summary>BR3</summary>

```
hostname BR3
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
 ip address 10.13.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:1301::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE1
 ip address 10.0.13.2 255.255.255.252
 ipv6 address 2001:db8:acad:013::2/64
 no shutdown
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0
  eigrp router-id 13.13.13.13
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 13.13.13.13
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
key chain EIGRP-KEYS
 key 1
  key-string CISCO123
interface Loopback0
 ip address 10.255.255.1 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::1/128
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
 description Link para BR1
 ip address 10.0.11.1 255.255.255.252
 ipv6 address 2001:db8:acad:011::1/64
 no shutdown
interface Ethernet0/2
 description Link para BR3
 ip address 10.0.13.1 255.255.255.252
 ipv6 address 2001:db8:acad:013::1/64
 no shutdown
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  af-interface Ethernet0/1
   authentication mode md5
   authentication key-chain EIGRP-KEYS
  exit-af-interface
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
 ip address 10.255.255.2 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::2/128
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
 description Link para BR2
 ip address 10.0.14.1 255.255.255.252
 ipv6 address 2001:db8:acad:014::1/64
 no shutdown
interface Ethernet0/2
 description Link para DC1
 ip address 10.0.15.1 255.255.255.252
 ipv6 address 2001:db8:acad:015::1/64
 no shutdown
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0
  eigrp router-id 2.2.2.2
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 2.2.2.2
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
 ip address 10.255.255.6 255.255.255.255
 ipv6 address 2001:db8:acad:ffff::6/128
 no shutdown
interface Loopback1
 ip address 10.21.1.1 255.255.255.0
 ipv6 address 2001:db8:acad:2101::1/64
 no shutdown
interface Loopback2
 ip address 10.21.2.1 255.255.255.0
 ipv6 address 2001:db8:acad:2102::1/64
 no shutdown
interface Ethernet0/0
 description Link para CORE2
 ip address 10.0.15.2 255.255.255.252
 ipv6 address 2001:db8:acad:015::2/64
 no shutdown
router eigrp EMPRESA
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0
  eigrp router-id 21.21.21.21
 exit-address-family
 address-family ipv6 unicast autonomous-system 100
  eigrp router-id 21.21.21.21
 exit-address-family
end
```
</details>

