# Introducció al DNS

**UF1: El servei DNS · M8: Serveis · CFGS ASIX**

---

## Índex

1. [Introducció i història](#1-introducció-i-història)
2. [El sistema de noms DNS](#2-el-sistema-de-noms-dns)
3. [Usos del servei de resolució de noms](#3-usos-del-servei-de-resolució-de-noms)
4. [Arquitectura DNS](#4-arquitectura-dns)
5. [Servidors arrel (*root servers*)](#5-servidors-arrel-root-servers)
6. [Jerarquia i resolució de noms](#6-jerarquia-i-resolució-de-noms)
7. [Zones d'autoritat i tipus de servidors](#7-zones-dautoritat-i-tipus-de-servidors)
8. [Base de dades DNS i registres de recurs](#8-base-de-dades-dns-i-registres-de-recurs)
9. [Tipus de registres DNS](#9-tipus-de-registres-dns)
10. [Consultes DNS](#10-consultes-dns)
11. [Tipus de respostes](#11-tipus-de-respostes)
12. [El protocol DNS](#12-el-protocol-dns)
13. [Comandes útils: Linux i Windows](#13-comandes-útils-linux-i-windows)
14. [Seguretat i evolució](#14-seguretat-i-evolució)
15. [Resum ràpid](#15-resum-ràpid)

---

## 1. Introducció i història

**DNS** (*Domain Name System*, sistema de noms de domini) és un sistema de nomenclatura **jeràrquica** per a ordinadors, serveis o qualsevol recurs connectat a Internet o a una xarxa privada.

Funció principal: **traduir (resoldre)** els identificadors binaris (adreces IP) dels equips a **noms entenedors per als humans**, per poder localitzar i adreçar aquests equips arreu del món.

### Història

| Etapa | Descripció |
|---|---|
| **Inicis (ARPANET)** | El DNS consistia en un sol fitxer, la **taula de hosts** (`HOSTS.TXT`), mantinguda pel **SRI-NIC** (*Stanford Research Institute's Network Information Center*). |
| **Funcionament** | Quan algú registrava un domini s'afegia a la taula, i els administradors havien d'actualitzar la seva còpia per **FTP**. |
| **Problema** | En créixer la xarxa el sistema es va quedar obsolet: poc eficient de distribuir i amb un cost de manteniment molt alt. |
| **Solució** | **Paul Mockapetris** (amb col·legues) va dissenyar el sistema actual el **1983-1984** (RFC 882 i 883; revisat el 1987 als **RFC 1034 i 1035**, encara vigents). |

El nou DNS es va pensar segons la naturalesa d'Internet: **una base de dades distribuïda i interconnectada, de la qual cap organització és directament responsable**. Gràcies a aquesta arquitectura, el DNS està preparat per a un creixement pràcticament il·limitat.

---

## 2. El sistema de noms DNS

L'estructura del sistema descriu un **arbre de dominis**, que permet diferents subdominis dins d'un mateix domini o subdomini.

Un nom de domini consta de dues o més parts (*etiquetes*) separades per punts:

```text
www.exemple.org
 │     │     └── TLD (Top Level Domain) → domini de nivell superior
 │     └──────── domini de segon nivell
 └────────────── hostname (nom de la màquina)
```

- L'etiqueta situada **més a la dreta** és el **TLD** (*Top Level Domain*, domini de nivell superior).
- Cada etiqueta a l'esquerra especifica una **subdivisió o subdomini**.
- La part **més a l'esquerra** sol expressar el nom de la màquina (*hostname*). Pot no correspondre a una màquina física (p. ex. `es.wikipedia.org`).

**Límits tècnics:**

| Element | Màxim |
|---|---|
| Nivells (etiquetes) | 127 |
| Longitud d'una etiqueta | **63 caràcters** |
| Longitud total del nom | **253 caràcters** en text (255 bytes en format de xarxa) |

**Definicions:**

| Terme | Significat | Exemple |
|---|---|---|
| **FQHN / FQDN** (*Fully Qualified Host/Domain Name*) | Nom complet d'un *host* | `pc1.insjaumehuguet.cat` |
| **Domain Name** | Part d'un FQDN a la dreta del nom de l'*host* | `insjaumehuguet.cat` |

> ℹ️ Un FQDN en sentit estricte acaba amb un punt (`pc1.insjaumehuguet.cat.`), que representa l'arrel.

---

## 3. Usos del servei de resolució de noms

El DNS s'utilitza sobretot per decidir **quina IP correspon a un nom complet d'*host***. Els usos més comuns són:

| Ús | Descripció | Exemple |
|---|---|---|
| **Resolució de noms** (directa) | Nom → IP | `www.xtec.cat` → `213.176.161.13`\* |
| **Resolució inversa** | IP → nom | `213.176.161.13` → `www.xtec.cat`\* |
| **Resolució de servidors de correu** | Domini → servidor que rep el correu (registre MX) | `gmail.com` → `gmail-smtp-in.l.google.com` |

\* *Les IP d'exemple dels apunts originals poden haver canviat; comprova-ho amb les comandes de l'[apartat 13](#13-comandes-útils-linux-i-windows).*

---

## 4. Arquitectura DNS

Tres components principals:

| Component | Funció |
|---|---|
| **Clients DNS** | Programa (*resolver*) a l'ordinador de l'usuari que genera peticions de resolució de noms cap a un servidor DNS. |
| **Servidors DNS** | Contesten les peticions dels clients. Els servidors **recursius** poden reenviar la petició a un altre servidor si no tenen l'adreça demanada. |
| **Zones d'autoritat** | Porcions de l'espai de noms que emmagatzemen les dades. Una zona abasta com a mínim un domini i, possiblement, els seus subdominis (si no han estat delegats). |

El sistema s'estructura en **arbre**. Cada node de l'arbre és un grup de servidors que resolen un conjunt de dominis (una **zona d'autoritat**).

**Delegació:** un servidor pot delegar en un altre (o altres) l'autoritat sobre alguna de les seves subzones (un subdomini de la zona).

```text
                    . (arrel)
          ┌─────────┼─────────┐
         .com      .org      .cat          ← nivell TLD (topall)
                              │
                        xtec.cat           ← nivell secundari
                         │
                      www.xtec.cat         ← host
```

---

## 5. Servidors arrel (*root servers*)

- Són els servidors amb autoritat sobre el nivell **arrel**, que coneixen els **TLD**.
- Són **fixos** (canvien molt poc) i n'hi ha **13 identitats**, de la **A a la M**: `a.root-servers.net` … `m.root-servers.net`.
- Realment les màquines físiques són moltes més (centenars d'instàncies repartides per tot el món amb **anycast**).
- Proporcionen accés al **fitxer de zona arrel** (*root zone file*), que és com un "directori de directoris": conté la informació dels TLD genèrics (`.com`, `.org`…) i dels de país (`.es`, `.it`, `.uk`…).

Més informació: <https://root-servers.org/> · <https://www.iana.org/domains/root/servers>

**Consultar-los des del terminal:**

```bash
# Linux
dig . NS
dig @a.root-servers.net cat. NS
```

```powershell
# Windows PowerShell
Resolve-DnsName -Type NS -Name .
nslookup -type=NS . 
```

---

## 6. Jerarquia i resolució de noms

L'espai de noms d'Internet es divideix bàsicament en **tres nivells**:

1. **Nivell arrel** (`.`)
2. **Nivell topall** (TLD: `.com`, `.cat`, `.es`…)
3. **Nivell secundari** (dominis assignats a organitzacions: `xtec.cat`, `wikipedia.org`…)

Els servidors d'arrel **deleguen** la resolució als servidors de nivell topall (TLD) i aquests als de nivell secundari, on hi ha els noms dels equips que ofereixen serveis a Internet.

### Resolució inversa: `in-addr.arpa`

Dins l'espai de noms hi ha el domini especial **`in-addr.arpa`** (per a IPv4; `ip6.arpa` per a IPv6), que associa una adreça IP amb un nom de domini. L'adreça s'escriu **al revés**:

```text
213.176.161.13  →  13.161.176.213.in-addr.arpa.   →   www.xtec.cat
```

> ⚠️ Els apunts originals escrivien `in-arpa.addr`; el nom correcte és **`in-addr.arpa`**.

```bash
# Linux
dig -x 213.176.161.13
host 213.176.161.13
```

```powershell
# Windows
nslookup 213.176.161.13
Resolve-DnsName 213.176.161.13
```

---

## 7. Zones d'autoritat i tipus de servidors

Cada zona té un **servidor de noms autoritari**, que conté tots els registres de la zona. Es defineix amb els registres **NS** i **SOA**.

Per tolerar fallades, es recomana **dos o més servidors autoritaris per zona, amb almenys un màster**.

| Tipus de servidor | Descripció |
|---|---|
| **Primari o màster** | Conté les dades de la zona en el seu sistema de fitxers (els **fitxers de zona**). |
| **Secundari o esclau** | Carrega el contingut de la zona d'un altre servidor (normalment el primari) mitjançant una **transferència de zona** (AXFR/IXFR). |
| **Recursiu o caché** | No és autoritari. Fa cerques recursives per trobar la IP d'un nom i **emmagatzema els resultats en caché** per accelerar futures consultes. |

> Un mateix programa (p. ex. BIND) pot fer de servidor autoritari i recursiu alhora, però és una bona pràctica separar-los.

**Comprovar-ho amb comandes:**

```bash
# Linux: quin servidor és primari i quin és el correu admin?
dig classe.local SOA +short
dig classe.local NS +short

# Transferència de zona (si el servidor ho permet)
dig @192.168.100.100 classe.local AXFR
```

```powershell
# Windows
nslookup -type=SOA classe.local 192.168.100.100
nslookup -type=NS  classe.local 192.168.100.100
```

---

## 8. Base de dades DNS i registres de recurs

- Cada servidor manté una base de dades que associa **noms de domini ↔ IP**: els **fitxers de zona** (resolució directa).
- També manté una base de dades de **resolució inversa**: els **fitxers de zona inversa**.
- Ambdues són gestionades pel servidor de noms, que respon les sol·licituds dels clients.
- El format són **fitxers de text** on es defineixen els **registres de recurs** (*Resource Records*, **RR**).

### Estructura d'un registre DNS

| Camp | Descripció |
|---|---|
| **Nom del registre** | Un FQDN o un nom de domini, segons el tipus de registre. |
| **TTL** | Temps de vida en **segons**: quant de temps es pot guardar el registre a la caché d'un servidor no autoritari o d'un client. Si s'omet, s'usa el TTL per defecte de la zona (`$TTL` / camp *minimum* del SOA). |
| **Classe** | Sempre **`IN`** (Internet). |
| **Tipus** | Valor de 16 bits que defineix el tipus de recurs (A, NS, MX…). |
| **RDLENGTH** | Longitud del camp RDATA. |
| **RDATA** | Les dades pròpiament dites, segons el tipus. |

**Exemple (format de fitxer de zona):**

```dns
;  nom        TTL   classe  tipus   RDATA
www.xtec.cat. 3600  IN      A       213.176.161.13
```

Contingut de RDATA segons el tipus:

| Tipus | RDATA |
|---|---|
| **A** | Adreça IP de 32 bits |
| **AAAA** | Adreça IP de 128 bits |
| **CNAME** | Nom de domini canònic |
| **HINFO** | Diversos camps (CPU, SO) |
| **MX** | Prioritat de 16 bits + FQDN del servidor de correu |
| **NS** | FQDN d'un servidor de noms autoritari |
| **PTR** | Nom de domini |
| **SOA** | Diversos camps |

---

## 9. Tipus de registres DNS

| Registre | Nom complet | Funció |
|---|---|---|
| **A** | Address | Fa coincidir un nom amb una **IPv4**. Pot haver-n'hi molts, per a diferents equips. |
| **AAAA** | IPv6 Address | El mateix, per a **IPv6**. |
| **CNAME** | Canonical Name | Defineix un **àlies** per a un nom canònic. Útil per donar noms alternatius a diferents serveis del mateix equip. |
| **HINFO** | Host Info | Descripció del maquinari (CPU) i SO. **No es recomana omplir-lo**: dona informació útil a atacants. |
| **MX** | Mail eXchange | **Servidor de correu** del domini. Pot haver-n'hi diversos amb **prioritat** (0 a 65535; **com més baix, més prioritat**) per tenir redundància. |
| **NS** | Name Server | Un registre per cada **servidor de noms autoritari** de la zona. |
| **PTR** | Pointer | Associa una **IP amb un nom** (resolució inversa). |
| **SOA** | Start Of Authority | Descriu el servidor autoritari de la zona i el **correu de l'administrador** (la `@` es substitueix per un punt). |
| **TXT** | Text | Text lliure (SPF, DKIM, verificacions de domini). |
| **SRV** | Service | Localitza serveis (servidor + port), p. ex. Active Directory. |

**Exemple complet de fitxer de zona:**

```dns
$TTL 3600
$ORIGIN exemple.cat.

@      IN SOA ns1.exemple.cat. admin.exemple.cat. (
              2026100201 ; serial
              8H         ; refresh
              2H         ; retry
              4W         ; expire
              1D )       ; minimum TTL (negative caching)

       IN NS    ns1.exemple.cat.
       IN NS    ns2.exemple.cat.
       IN MX 10 correu.exemple.cat.
       IN MX 20 correu2.exemple.cat.

ns1    IN A     192.168.0.5
ns2    IN A     192.168.0.7
correu IN A     192.168.0.6
correu2 IN A    192.168.0.8
web    IN A     192.168.0.10
web    IN AAAA  2001:db8::10
www    IN CNAME web
```

**Consultar cada tipus de registre:**

| Registre | Linux | Windows |
|---|---|---|
| A | `dig www.xtec.cat A` | `nslookup -type=A www.xtec.cat` |
| AAAA | `dig www.xtec.cat AAAA` | `nslookup -type=AAAA www.xtec.cat` |
| CNAME | `dig www.exemple.cat CNAME` | `nslookup -type=CNAME www.exemple.cat` |
| MX | `dig gmail.com MX` | `nslookup -type=MX gmail.com` |
| NS | `dig xtec.cat NS` | `nslookup -type=NS xtec.cat` |
| PTR | `dig -x 192.168.100.100` | `nslookup 192.168.100.100` |
| SOA | `dig xtec.cat SOA` | `nslookup -type=SOA xtec.cat` |
| TXT | `dig gmail.com TXT` | `nslookup -type=TXT gmail.com` |
| HINFO | `dig host HINFO` | `nslookup -type=HINFO host` |
| Tots | `dig xtec.cat ANY` | `nslookup -type=ANY xtec.cat` |

> A PowerShell: `Resolve-DnsName -Name gmail.com -Type MX` (canvia `MX` pel tipus que vulguis).

---

## 10. Consultes DNS

Una **consulta** és una sol·licitud de resolució de nom enviada a un servidor DNS. Hi ha dos tipus: **recursives** i **iteratives**.

### 10.1 Procés al client

1. Una aplicació utilitza un nom de domini → genera una sol·licitud al servei **Client DNS**.
2. El client mira primer la **caché DNS local** (memòria on es guarden temporalment els registres de les consultes anteriors). Si el nom hi és, **es respon i s'acaba**.
3. Si no hi és (ni al fitxer `hosts`), el client consulta el **servidor DNS** configurat.

**Veure i buidar la caché DNS del client:**

| Acció | Linux (Ubuntu) | Windows |
|---|---|---|
| Veure la caché | `resolvectl statistics` | `ipconfig /displaydns` |
| Buidar la caché | `sudo resolvectl flush-caches` | `ipconfig /flushdns` |
| Veure servidors DNS en ús | `resolvectl status` | `ipconfig /all` |
| Veure fitxer `hosts` | `cat /etc/hosts` | `type C:\Windows\System32\drivers\etc\hosts` |

### 10.2 Consulta recursiva

> Les consultes **client → servidor DNS** són sempre **recursives**.

El client demana al servidor una **resposta completa**. El servidor comprova la seva zona i la seva caché; si no hi troba la resposta, ha de **recórrer a altres servidors** fins a obtenir-la i la retorna al client.

```text
Computer1 ──(recursiva: www.xtec.cat?)──► Servidor DNS local
Computer1 ◄──────── 213.176.161.13 ─────── Servidor DNS local
```

### 10.3 Consulta iterativa

> Les consultes **entre servidors DNS** són **iteratives** per defecte (excepte si s'utilitzen reenviadors).

El servidor consultat dona la **millor resposta que té**, sense buscar ajuda d'altres servidors. Normalment és una **referència** a un servidor de nivell inferior a l'arbre. La resolució comença sempre als servidors arrel (*root hints*) i baixa fins al servidor que té la informació.

```text
1. Servidor local → Arrel (.)          "Pregunta a .cat"
2. Servidor local → Servidor .cat      "Pregunta a xtec.cat"
3. Servidor local → Servidor xtec.cat  "www.xtec.cat = 213.176.161.13"  (autoritativa)
```

**Veure tot el procés amb una sola comanda:**

```bash
# Linux
dig +trace www.xtec.cat

# Forçar una consulta no recursiva (iterativa) a un servidor
dig +norecurse @a.root-servers.net www.xtec.cat
```

```powershell
# Windows: no hi ha +trace; es pot fer pas a pas
nslookup -type=NS .
nslookup -norecurse www.xtec.cat a.root-servers.net
```

### 10.4 Reenviadors (*forwarders*)

Un **reenviador** és un servidor DNS designat pels servidors DNS interns per **reenviar les consultes** de noms externs (fora del lloc) en lloc de resoldre-les ells mateixos des de l'arrel.

```text
Computer1 → Servidor DNS local → Reenviador (p. ex. 8.8.8.8) → Internet
```

Configuració a **Bind9** (`/etc/bind/named.conf.options`):

```text
options {
    directory "/var/cache/bind";
    forwarders { 8.8.8.8; 1.1.1.1; };
    forward first;      // "only" = no resol per ell mateix si el reenviador falla
};
```

Configuració a **Windows Server**:

```powershell
Add-DnsServerForwarder -IPAddress 8.8.8.8
Get-DnsServerForwarder
```

---

## 11. Tipus de respostes

Les respostes possibles d'un servidor són quatre:

| Resposta | Descripció |
|---|---|
| **Amb autoritat** | Resposta positiva amb el **bit AA** activat: ve d'un servidor autoritari directe per al nom consultat. |
| **Positiva** | Conté el registre (o un conjunt de registres, **RRset**) que coincideix amb el nom i el tipus demanats. |
| **De referència** | Conté registres addicionals no demanats exactament (p. ex. NS cap a un servidor de nivell inferior). Es retorna quan el servidor no admet recursivitat; el client pot continuar la consulta per iteració. També, per exemple, si es demana l'`A` de `www` i hi ha un `CNAME`, el servidor l'inclou. |
| **Negativa** | El servidor autoritari informa que: (1) el **nom no existeix** (`NXDOMAIN`), o (2) el nom existeix però **no hi ha registres d'aquest tipus** (`NODATA`). |

El client retorna el resultat (positiu o negatiu) a l'aplicació i el guarda a la **caché**.

**Com reconèixer-les a la sortida de les comandes:**

```bash
dig www.xtec.cat
#   flags: qr rd ra          → resposta de caché (no autoritativa)
#   flags: qr aa rd          → resposta AUTORITATIVA (bit aa)
#   status: NOERROR          → positiva
#   status: NXDOMAIN         → negativa (el nom no existeix)
#   status: NOERROR + ANSWER: 0 → NODATA (existeix però no té aquest tipus)
#   status: SERVFAIL         → fallada del servidor
#   status: REFUSED          → rebutjat per política
```

```text
C:\> nslookup www.xtec.cat
Non-authoritative answer:        ← resposta no autoritativa (de caché)
Name:    www.xtec.cat
Address: ...

C:\> nslookup inexistent.xtec.cat
*** ... can't find inexistent.xtec.cat: Non-existent domain   ← NXDOMAIN
```

---

## 12. El protocol DNS

- Treballa a la **capa d'aplicació** i utilitza el **port 53**.
- **UDP** per defecte; **TCP** quan la resposta és massa gran o en transferències de zona.
  - Als apunts originals: UDP si el missatge és < 512 bytes, TCP en cas contrari. Avui, amb **EDNS(0)**, UDP pot portar missatges més grans (típicament fins a ~1232 bytes), i s'usa TCP quan la resposta es **trunca** (bit TC) i en AXFR/IXFR.
- **Cal obrir UDP 53 i TCP 53** al tallafocs d'un servidor DNS.

### Format del missatge

```text
 0                                16                               31
 ┌────────────────────────────────┬────────────────────────────────┐
 │ Identification (ID)            │ Parameters (flags)             │
 ├────────────────────────────────┼────────────────────────────────┤
 │ QDCOUNT                        │ ANCOUNT                        │
 ├────────────────────────────────┼────────────────────────────────┤
 │ NSCOUNT                        │ ARCOUNT                        │
 ├────────────────────────────────┴────────────────────────────────┤
 │ Question      (consultes)                                       │
 │ Answer        (RR de resposta)                                  │
 │ Authority     (RR dels servidors autoritaris)                   │
 │ Additional    (RR addicionals)                                  │
 └─────────────────────────────────────────────────────────────────┘
```

| Camp | Descripció |
|---|---|
| **Identification** | Identificador de **16 bits** assignat pel programa. Es copia a la resposta; permet distingir respostes quan hi ha múltiples consultes concurrents. |
| **Parameters** | 16 bits amb els *flags* de sota. |
| **QDCOUNT** | Nombre d'entrades de la secció *Question*. |
| **ANCOUNT** | Nombre de RR de la secció *Answer*. |
| **NSCOUNT** | Nombre de RR de la secció *Authority* (servidors autoritaris). |
| **ARCOUNT** | Nombre de RR de la secció *Additional*. |

### Camps de *Parameters*

| Flag | Mida | Significat |
|---|---|---|
| **QR** | 1 bit | `0` = consulta, `1` = resposta. |
| **Opcode** | 4 bits | `0` = consulta estàndard · `1` = consulta inversa (*obsoleta*) · `2` = estat del servidor. Altres valors, reservats (avui també `4` = NOTIFY, `5` = UPDATE). |
| **AA** | 1 bit | *Authoritative Answer*: el servidor que respon té autoritat sobre el domini consultat. |
| **TC** | 1 bit | *Truncated*: el missatge és més llarg del que permet la transmissió (cal reintentar per TCP). |
| **RD** | 1 bit | *Recursion Desired*: el client demana resolució recursiva (es copia a la resposta). |
| **RA** | 1 bit | *Recursion Available*: el servidor suporta recursivitat. |
| **Z** | 3 bits | Reservats (originalment, a zero; avui s'usen per a **AD** i **CD** de DNSSEC). |
| **RCODE** | 4 bits | Codi de resposta (vegeu sota). |

### Codis RCODE

| Codi | Nom | Significat |
|---|---|---|
| 0 | `NOERROR` | Cap error. |
| 1 | `FORMERR` | Error de format: el servidor no ha pogut interpretar el missatge. |
| 2 | `SERVFAIL` | Fallada al servidor: no s'ha pogut processar la consulta. |
| 3 | `NXDOMAIN` | El nom de domini consultat no existeix (només vàlid si AA actiu). |
| 4 | `NOTIMP` | Tipus de consulta no implementat al servidor. |
| 5 | `REFUSED` | El servidor rebutja respondre per raons de política. |

### Seccions del missatge

| Secció | Contingut |
|---|---|
| **Question** | Les consultes al servei DNS. |
| **Answer** | Els RR de la resposta. |
| **Authority** | Els servidors DNS autoritzats per respondre. |
| **Additional** | Informació addicional per completar la resposta. |

### Veure el protocol en acció

```bash
# Linux: capturar tràfic DNS
sudo tcpdump -i any -n port 53

# Veure la sortida completa amb totes les seccions
dig www.xtec.cat +noall +question +answer +authority +additional

# Forçar TCP
dig +tcp www.xtec.cat

# Comprovar que el servidor escolta al 53
sudo ss -tulpn | grep :53
```

```powershell
# Windows
Test-NetConnection 192.168.100.100 -Port 53     # prova TCP 53
nslookup -vc www.xtec.cat                       # força TCP
nslookup -debug www.xtec.cat                    # mostra el missatge complet
# Captura de tràfic: Wireshark amb filtre "dns"
```

---

## 13. Comandes útils: Linux i Windows

### 13.1 Consultes bàsiques

| Acció | Linux | Windows (CMD) | Windows (PowerShell) |
|---|---|---|---|
| Resoldre un nom | `host www.xtec.cat`<br>`dig www.xtec.cat` | `nslookup www.xtec.cat` | `Resolve-DnsName www.xtec.cat` |
| Només la IP | `dig +short www.xtec.cat` | — | `(Resolve-DnsName www.xtec.cat).IPAddress` |
| Resolució inversa | `dig -x 213.176.161.13`<br>`host 213.176.161.13` | `nslookup 213.176.161.13` | `Resolve-DnsName 213.176.161.13` |
| Servidor DNS concret | `dig @8.8.8.8 www.xtec.cat` | `nslookup www.xtec.cat 8.8.8.8` | `Resolve-DnsName www.xtec.cat -Server 8.8.8.8` |
| Tipus de registre | `dig gmail.com MX` | `nslookup -type=MX gmail.com` | `Resolve-DnsName gmail.com -Type MX` |
| Servidors de correu | `host -t MX gmail.com` | `nslookup -type=MX gmail.com` | `Resolve-DnsName gmail.com -Type MX` |
| Seguir la jerarquia | `dig +trace www.xtec.cat` | — | — |
| Comprovar connectivitat | `ping www.xtec.cat` | `ping www.xtec.cat` | `Test-Connection www.xtec.cat` |

### 13.2 `nslookup` interactiu (Linux i Windows)

```text
nslookup
> server 8.8.8.8
> set type=MX
> gmail.com
> set type=PTR
> 213.176.161.13
> set debug
> www.xtec.cat
> exit
```

### 13.3 Configuració del client

**Linux (Netplan):**

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [192.168.100.201/24]
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        search: [classe.local]
        addresses: [192.168.100.100, 8.8.8.8]
```

```bash
sudo netplan try && sudo netplan apply
resolvectl status
```

**Windows (PowerShell com a administrador):**

```powershell
Get-NetAdapter
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.100.100,8.8.8.8
Get-DnsClientServerAddress
```

**Windows (CMD):**

```cmd
netsh interface ip set dns name="Ethernet" static 192.168.100.100
netsh interface ip add dns name="Ethernet" 8.8.8.8 index=2
ipconfig /all
```

### 13.4 Servidor DNS: Linux (Bind9) i Windows Server

| Acció | Linux (Bind9) | Windows Server |
|---|---|---|
| Instal·lar | `sudo apt install bind9 bind9-doc dnsutils` | `Install-WindowsFeature DNS -IncludeManagementTools` |
| Estat del servei | `sudo systemctl status bind9` | `Get-Service DNS` |
| Reiniciar | `sudo systemctl restart bind9` | `Restart-Service DNS` |
| Validar configuració | `sudo named-checkconf` | — |
| Validar una zona | `sudo named-checkzone classe.local /etc/bind/db.classe.local` | — |
| Recarregar zones | `sudo rndc reload` | `Get-DnsServerZone` |
| Buidar caché del servidor | `sudo rndc flush` | `Clear-DnsServerCache` |
| Crear zona | Editar `named.conf.local` + fitxer de zona | `Add-DnsServerPrimaryZone -Name "classe.local" -ZoneFile "classe.local.dns"` |
| Afegir registre A | Editar fitxer de zona (i pujar el `serial`) | `Add-DnsServerResourceRecordA -ZoneName classe.local -Name pc1 -IPv4Address 192.168.100.50` |
| Afegir CNAME | Editar fitxer de zona | `Add-DnsServerResourceRecordCName -ZoneName classe.local -Name web -HostNameAlias srv.classe.local` |
| Afegir reenviador | `forwarders { 8.8.8.8; };` | `Add-DnsServerForwarder -IPAddress 8.8.8.8` |
| Veure registres | `cat /etc/bind/db.classe.local` | `Get-DnsServerResourceRecord -ZoneName classe.local` |
| Logs | `sudo journalctl -u bind9 -f` | Visor d'esdeveniments → DNS Server |
| Port 53 obert | `sudo ufw allow 53` | `New-NetFirewallRule -DisplayName "DNS" -Protocol UDP -LocalPort 53 -Action Allow` |

### 13.5 Diagnòstic ràpid

```bash
# Linux
ping -c 4 192.168.100.100              # arribem al servidor?
dig @192.168.100.100 classe.local SOA  # el servidor respon?
sudo ss -tulpn | grep :53              # escolta al port 53?
sudo tail -f /var/log/syslog | grep named
```

```powershell
# Windows
ping 192.168.100.100
Test-NetConnection 192.168.100.100 -Port 53
nslookup classe.local 192.168.100.100
ipconfig /flushdns
```

---

## 14. Seguretat i evolució

- **HINFO:** no s'hauria d'omplir, perquè revela informació útil per a atacants.
- **DNSSEC:** signa les respostes (registres `RRSIG`, `DNSKEY`, `DS`) per garantir autenticitat i integritat. Es comprova amb `dig +dnssec`.
- **DoT / DoH:** xifren la comunicació client ↔ resolutor (ports 853 i 443).
- **Bones pràctiques en un servidor propi:**
  - Limitar la recursió només als clients de la xarxa (`allow-recursion`).
  - Restringir les transferències de zona al secundari (`allow-transfer`).
  - Mantenir el programari actualitzat.
- **Atacs típics:** enverinament de caché (*cache poisoning*), amplificació DDoS amb servidors recursius oberts, segrest de domini.

---

## 15. Resum ràpid

- El **DNS** és una base de dades **jeràrquica i distribuïda** que tradueix noms ↔ IP. Va ser dissenyat per **Paul Mockapetris** (1983-84) per substituir el fitxer `HOSTS.TXT`.
- **Components:** clients (resolvers), servidors i zones d'autoritat.
- **Nivells:** arrel → TLD → domini secundari. Hi ha **13 identitats de servidors arrel** (A–M, `root-servers.net`).
- **Servidors:** primari/màster, secundari/esclau i recursiu/caché.
- **Consultes:** client → servidor = **recursiva**; servidor → servidor = **iterativa** (excepte amb reenviadors).
- **Respostes:** amb autoritat, positiva, de referència i negativa.
- **Registres:** `A`, `AAAA`, `CNAME`, `MX`, `NS`, `PTR`, `SOA`, `HINFO`, `TXT`, `SRV`; classe sempre `IN`.
- **Protocol:** port **53**, UDP per defecte i TCP per a respostes grans i transferències de zona.
- **Resolució inversa:** domini `in-addr.arpa` amb la IP escrita al revés.
- Per veure la caché del client: `ipconfig /displaydns` (Windows) · `resolvectl statistics` (Linux).
