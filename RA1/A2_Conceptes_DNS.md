# Servei DNS

**Serveis de xarxa · A2. Conceptes DNS**

---

## Índex

1. [Resolució de noms](#1-resolució-de-noms)
2. [Sistemes de noms plans i jeràrquics](#2-sistemes-de-noms-plans-i-jeràrquics)
3. [Elements del sistema de noms de domini](#3-elements-del-sistema-de-noms-de-domini)
4. [Els noms de domini d'Internet (TLD)](#4-els-noms-de-domini-dinternet-tld)
5. [Zones i delegació](#5-zones-i-delegació)
6. [Mecanisme de resolució](#6-mecanisme-de-resolució)
7. [Resolució inversa](#7-resolució-inversa)
8. [El protocol DNS](#8-el-protocol-dns)
9. [Base de dades de zona i registres de recurs](#9-base-de-dades-de-zona-i-registres-de-recurs)
10. [Exemples de fitxers de zona](#10-exemples-de-fitxers-de-zona)
11. [Comandes útils: Linux i Windows](#11-comandes-útils-linux-i-windows)
12. [Seguretat i evolució del DNS](#12-seguretat-i-evolució-del-dns)
13. [Resum ràpid](#13-resum-ràpid)

---

## 1. Resolució de noms

- El **DNS** (*Domain Name System*, sistema de noms de domini) proporciona un mecanisme eficaç per fer la **resolució de noms de domini a adreces IP**.
- Als humans ens és més fàcil adreçar-nos a un servei amb un **text identificatiu** (per exemple, `www.institutjaumehuguet.cat`) que no pas amb l'adreça IP corresponent (per exemple, `213.73.40.230`).
- El DNS no només resol noms a IP (**resolució directa**), sinó també al revés: a partir d'una IP esbrina el nom de domini (**resolució inversa**).

---

## 2. Sistemes de noms plans i jeràrquics

### Sistema pla: el fitxer `hosts`

- Cal un mecanisme en "llenguatge humà" per identificar els equips de la xarxa, en especial els que ofereixen serveis.
- A la xarxa inicial **ARPANET**, els noms es feien públics mitjançant un **fitxer centralitzat** (`hosts.txt`) que contenia els noms i la identificació de tots els equips.
- Un **sistema de noms pla** es basa en un fitxer de text que descriu cada *host* amb la seva adreça IP. Encara s'usa per definir àlies d'equips locals, però **no és escalable**.

**Problemes del fitxer `hosts`:**

- Si la xarxa creix, és impossible de mantenir.
- Caldria un equip que centralitzés els noms de tots els *hosts* d'Internet en un sol fitxer.
- Aquest fitxer s'hauria de repartir entre tots els equips cada vegada que hi hagués una actualització.

**On és el fitxer `hosts` avui dia?**

| Sistema | Ruta |
|---|---|
| Linux / macOS | `/etc/hosts` |
| Windows | `C:\Windows\System32\drivers\etc\hosts` |

Exemple de línia (igual a tots dos sistemes):

```text
192.168.100.201   clientubuntu.classe.local   clientubuntu
```

### Sistema jeràrquic: el DNS

El DNS es basa en una **base de dades de noms de domini**:

- **Jeràrquica:** s'organitza en dominis que es divideixen en subdominis, i aquests en altres subdominis. Un nom complet (FQDN) pot tenir com a màxim **127 nivells** i **253 caràcters**; cada etiqueta (part entre punts) té com a màxim **63 caràcters**.
- **Distribuïda:** la informació no és en un sol repositori, sinó repartida entre els servidors DNS d'Internet. Cada servidor DNS **autoritari** conté la base de dades de la seva zona.

---

## 3. Elements del sistema de noms de domini

El DNS es basa en una estructura jeràrquica **en forma d'arbre**. L'arrel és el **domini arrel**, del qual deriven tots els altres nodes (`.com`, `.edu`, `.org`, `.cat`, …). Cada domini es pot dividir en subdominis, i així successivament.

> Un **domini** és el node indicat i tota la resta de l'arbre que en penja (com un directori i tots els seus subdirectoris).

| Terme | Significat |
|---|---|
| **Espai de noms** | Conjunt de tots els dominis (l'arbre DNS). |
| **Domini** | Text identificatiu d'un domini. |
| **FQDN** (*Fully Qualified Domain Name*) | Nom de domini absolut: va del node fins a l'arrel. |
| **Domini absolut** | És un FQDN i **acaba en punt** (`.`). Ex.: `www.classe.local.` |
| **Domini relatiu** | Nom de domini sense qualificar. Ex.: `www`. |
| **Domini arrel** | Domini del qual deriven tots els altres. S'indica amb un punt (`.`) o amb la cadena buida. |

---

## 4. Els noms de domini d'Internet (TLD)

- El node arrel es va dividir en subdominis anomenats **TLD** (*Top Level Domains*, dominis d'alt nivell).
- Els **TLD originals** eren: `com`, `edu`, `gov`, `mil`, `org`, `net` i `int`.
- Més endavant se'n van afegir d'altres, com `cat`, `name`, `biz`, `info`, `pro`, `aero`, `coop` i `museum`. La idea era organitzar els dominis per funcionalitat (empreses a `.com`, organitzacions a `.org`, etc.).
- Es va veure la necessitat d'agrupar-los **geogràficament** i van sorgir els identificadors de país: **ccTLD** de dos caràcters (`es`, `fr`, `uk`, …).

**Tipus de TLD actuals:**

| Tipus | Descripció | Exemples |
|---|---|---|
| **gTLD** | Genèrics | `.com`, `.org`, `.net`, `.info` |
| **sTLD** | Patrocinats / restringits | `.edu`, `.gov`, `.mil`, `.aero` |
| **ccTLD** | Codis de país | `.es`, `.fr`, `.de` |
| **TLD de comunitat** | Cultural o lingüístic | `.cat`, `.eus`, `.gal` |
| **Nous gTLD** | Des de 2012 n'hi ha centenars | `.app`, `.dev`, `.online`, `.barcelona` |
| **Especial** | Ús intern / infraestructura | `.arpa` (resolució inversa), `.local` (mDNS) |

> ℹ️ **Nota sobre `.local`:** el sufix `.local` està reservat per a **mDNS** (RFC 6762). En pràctiques de laboratori s'utilitza sovint (`classe.local`), però en entorns reals es recomana usar `.lan`, `.home.arpa` o un subdomini d'un domini propi. Per a proves, també són segurs `.test` i `.example` (RFC 2606).

---

## 5. Zones i delegació

### Arquitectura client-servidor

El DNS és una arquitectura **client-servidor**: els clients fan preguntes del tipus *"quina IP té aquest domini?"* i els servidors procuren contestar-les.

Els **servidors de noms DNS** són els programes que emmagatzemen i gestionen la informació d'una part de l'espai de noms anomenada **zona**.

### Domini ≠ zona

De bon principi podríem pensar que un servidor DNS gestiona un domini i que zona = domini, però no té per què ser així. Un domini es divideix en subdominis per facilitar-ne l'administració, i **cada part administrada per un (o més) servidors DNS és una zona**.

```text
                  A
        ┌─────┬───┴───┬─────┐
        B     C       D     E
      ┌─┴─┐ ┌─┼─┐     │     │
      F   G H I  J    K     L
              ┌┴┐
              M N
```

Per exemple, es poden definir zones com `A`, `B.A`, `C.A` i `I.C.A`, cadascuna amb un o més servidors DNS que la gestionen.

| Concepte | Definició |
|---|---|
| **Domini** | L'arbre de l'espai de noms. |
| **Zona** | La part de l'arbre administrada per un servidor DNS concret. |
| **Base de dades de zona** | Els fitxers que emmagatzemen la descripció dels equips de la zona. |
| **Autoritat** | Els servidors que gestionen una zona tenen informació completa sobre ella i es diu que tenen **autoritat**. |
| **Delegació** | Passar l'autoritat de la gestió d'un subdomini a una altra entitat. |

### La delegació

**Delegar** és passar l'autoritat sobre un subdomini a una altra entitat (uns altres servidors DNS). Aquesta entitat n'és la responsable i té tota l'autoritat per fer i desfer al seu criteri. La zona pare perd el control administratiu de la zona delegada i només **apunta als servidors de noms** de la zona delegada (mitjançant registres `NS`).

L'estàndard exigeix **dos o més servidors autoritaris** per zona:

- **Servidor primari** (*master*)
- **Servidor secundari** (*slave*)

Motiu: **redundància, robustesa, rendiment i còpia de seguretat**. Si l'únic servidor de noms falla, la xarxa quedarà inoperativa.

---

## 6. Mecanisme de resolució

El mecanisme consta d'un **client o *resolver*** que fa **consultes** (*queries*) a servidors DNS.

| Situació del servidor | Tipus de resposta |
|---|---|
| Té la informació perquè forma part de la base de dades de la seva zona | **Autoritativa** |
| Té la resposta emmagatzemada temporalment (**caché**) | **No autoritativa** |
| No té la informació | Consulta altres servidors (procés **recursiu** o **iteratiu**) |

Sempre existeix un camí per trobar el domini buscat: preguntar als **servidors arrel** (*root servers*) i recórrer l'arbre cap avall.

### Exemple: quina IP té `info.institutjaumehuguet.cat`?

Un estudiant d'Austràlia vol resoldre aquest nom des del seu servidor de Sydney:

1. Pregunta a un **servidor arrel**. Aquest no coneix el *host* `info`, però coneix tots els TLD, així que dona la llista de servidors de noms de `.cat`.
2. Pregunta a un servidor de `.cat`, que dona la llista de servidors DNS de `institutjaumehuguet.cat`.
3. Pregunta a un servidor de `institutjaumehuguet.cat`, que **és autoritari** per a aquest domini i retorna l'**adreça IP** del *host* `info`.

```text
Client → Servidor local ──► Arrel (.)             "Pregunta a .cat"
                        ──► Servidor .cat          "Pregunta a institutjaumehuguet.cat"
                        ──► institutjaumehuguet.cat "info = 213.73.40.xxx"  ✔
```

### Recursió i iteració

| Qui consulta | Mode | Funcionament |
|---|---|---|
| **Client → seu servidor DNS** | **Recursiu** | El servidor ha de lliurar la resposta final (o un error). Si no la té, pregunta a altres servidors fins a obtenir-la. |
| **Servidor → servidor** | **Iteratiu** | Si el servidor no té la resposta, retorna una llista dels servidors més propers al domini buscat. El que ha fet la consulta decideix a qui preguntar després. |

### Servidors arrel actuals

Hi ha **13 identitats** de servidors arrel, de `a.root-servers.net` a `m.root-servers.net`. Cadascuna es reparteix per tot el món en centenars d'instàncies mitjançant **anycast**.

---

## 7. Resolució inversa

El DNS permet obtenir el nom de domini d'una IP mitjançant un domini especial: **`in-addr.arpa`** (per a IPv4; `ip6.arpa` per a IPv6).

- Hi ha protocols i serveis que requereixen una resolució inversa correcta (p. ex. servidors de correu) i sovint s'utilitza com a mesura de seguretat per verificar l'existència de l'adreça IP en un domini.
- L'adreça s'escriu **al revés**:

> Un *host* amb IP `192.168.1.24` correspon al domini **`24.1.168.192.in-addr.arpa`**.

---

## 8. El protocol DNS

- Protocol de **capa d'aplicació**. Utilitza el **port 53**, normalment per **UDP**, però també per **TCP**.
- **TCP** s'usa quan la resposta és massa gran (o es talla) i per a les **transferències de zona** (AXFR/IXFR) entre servidor primari i secundari.
- Per aquest motiu cal obrir **UDP 53 i TCP 53** al tallafocs d'un servidor DNS.

**Seccions d'un missatge DNS:**

| Secció | Contingut |
|---|---|
| **HEADER** | Capçalera: indica si és consulta o resposta. Conté l'`id` del missatge, els *flags* i un resum de les seccions que porten informació i quanta. |
| **QUESTION** | La consulta efectuada: quina dada es demana al servidor (resolució d'una IP, llista de servidors de correu…). |
| **ANSWER** | La resposta obtinguda. Si ve d'una caché apareix com a *non-authoritative answer*. |
| **AUTHORITY** | Les respostes autoritatives per a la consulta. Pot ser buida. |
| **ADDITIONAL** | Informació addicional per completar la resposta (p. ex. la IP dels servidors de noms ja citats). |

**Altres ports i variants moderns:**

| Protocol | Port | Descripció |
|---|---|---|
| DNS clàssic | 53 UDP/TCP | Sense xifrar. |
| **DoT** (DNS over TLS) | 853 TCP | DNS xifrat. |
| **DoH** (DNS over HTTPS) | 443 TCP | DNS xifrat dins HTTPS. |

---

## 9. Base de dades de zona i registres de recurs

La informació d'una zona s'emmagatzema en **registres de recurs** (*Resource Records*, **RR**) dins de **fitxers de zona**.

- **Forward mapping** (resolució directa): nom → IP.
- **Reverse mapping** (resolució inversa): IP → nom.

**Fitxers que hi ha en una zona:**

1. Un fitxer amb les associacions **nom → IP** (resolució directa).
2. Un fitxer **per a cada subxarxa** amb l'associació **IP → nom canònic** (resolució inversa).
3. Un fitxer amb la resolució inversa del ***loopback*** (`127.0.0.1`).
4. Un fitxer amb la descripció dels **servidors arrel** d'Internet (*root hints*).

> 📁 A Bind9 d'Ubuntu aquests fitxers són, per exemple: `db.local`, `db.127`, `db.0`, `db.255` i `db.root` (o `named.conf.default-zones`) a `/etc/bind/`.

### 9.1 Registre SOA (*Start Of Authority*)

Indica que el fitxer de zona és la millor font de dades per a la zona i que el servidor és **autoritari**. Normalment és el primer RR del fitxer (no és obligatori) i **només n'hi pot haver un per fitxer de zona**.

**Format:**

```dns
nomDomini.  IN  SOA  nsPrimari.  admin.nsPrimari.  (
            serial
            refresh
            retry
            expire
            minimum )
```

**Exemple:**

```dns
ioc.cat.  IN  SOA  ns1.ioc.cat.  admin.ioc.cat.  (
          23      ; serial
          8H      ; refresh
          2H      ; retry
          4W      ; expire
          1D )    ; minimum TTL
```

| Camp | Descripció |
|---|---|
| `nomDomini.` | Domini que es defineix i pel qual el servidor és autoritari. **El punt final és important.** |
| `IN` | Classe Internet. |
| `SOA` | Tipus de registre. |
| `nsPrimari.` | Nom del *host* servidor de noms primari de la zona. |
| `admin.nsPrimari.` | Correu de l'administrador amb format `usuari.servidor` (el primer punt s'interpreta com una `@`: `admin@ns1.ioc.cat`). |

**Paràmetres entre parèntesis** (comunicació primari ↔ secundaris):

| Paràmetre | Significat |
|---|---|
| **Serial** | Número de sèrie de la versió de les dades. Cal **incrementar-lo** cada cop que es modifica la zona (convenció: `AAAAMMDDNN`). |
| **Refresh** | Temps entre refrescos de dades del secundari. |
| **Retry** | Temps d'espera per reintentar un refresc que ha fallat. |
| **Expire** | Temps a partir del qual les dades del secundari es consideren sense autoritat si no s'han refrescat. |
| **Minimum** | TTL de les respostes negatives (*negative caching*, RFC 2308). El TTL per defecte dels registres es defineix amb la directiva `$TTL`. |

**Unitats de temps:** `S` segons, `M` minuts, `H` hores, `D` dies, `W` setmanes (ex.: `8H`, `4W`).

### 9.2 Registre NS (*Name Server*)

Defineix un **servidor de noms autoritatiu** per a la zona. Hi ha tantes entrades NS com servidors autoritaris. L'estàndard en recomana **almenys dos** (primari i secundari).

```dns
nomDomini.  IN  NS  nameServer.
```

```dns
inf.cdm.cat.  IN  NS  ns1.inf.cdm.cat.
```

> Tant `nomDomini.` com `nameServer.` acaben en punt perquè són FQDN. Un NS ha d'apuntar a un **nom** que tingui registre A/AAAA, mai a una IP ni a un CNAME.

### 9.3 Registre A (*Address*)

Associa un nom de *host* a una adreça **IPv4** (resolució directa). Cal una entrada per a cada *host*.

```dns
nomHost.  IN  A  IP
```

```dns
mahatma.inf.cdm.cat.  IN  A  192.168.0.2
```

> Per a adreces **IPv6** s'utilitza el registre **AAAA**: `mahatma  IN  AAAA  2001:db8::2`.

### 9.4 Registre CNAME (*Canonical Name*)

Associa un **àlies** a un **nom canònic**.

```dns
alies.  IN  CNAME  hostCanonicalName.
```

```dns
ftp.inf.cdm.cat.  IN  CNAME  mahatma.inf.cdm.cat.
```

> ⚠️ **Un CNAME ha d'apuntar sempre a un nom, mai a una IP.** (Els apunts originals mostraven `tftp … CNAME 192.168.0.2`, que és incorrecte.) Tampoc es pot posar un CNAME al mateix nom que té altres registres (p. ex. a l'`@` de l'apex).

### 9.5 Registre MX (*Mail Exchanger*)

Defineix un **servidor de correu** del domini. No és obligatori.

```dns
nomDomini.  IN  MX  num  mailServer.
```

```dns
inf.cdm.cat.  IN  MX  10  correu.inf.cdm.cat.
inf.cdm.cat.  IN  MX  20  correu2.inf.cdm.cat.
```

| Camp | Descripció |
|---|---|
| `num` | Preferència. **Com més baix, més prioritari.** Valors arbitraris definits per l'administrador. |
| `mailServer.` | FQDN del servidor de correu. |

> ⚠️ El destí d'un MX ha de ser un nom amb registre **A/AAAA**, **no un CNAME** (RFC 2181).

### 9.6 Registre PTR (*Pointer*)

Associa una **IP a un nom** (resolució inversa). Cal una entrada PTR per a cada interfície de xarxa de la zona.

```dns
ipInversa.in-addr.arpa.  IN  PTR  hostCanonicalName.
```

```dns
2.20.168.192.in-addr.arpa.  IN  PTR  mahatma.inf.cdm.cat.
```

> La IP `192.168.20.2` s'escriu `2.20.168.192.in-addr.arpa.` El nom del *host* ha de ser el **nom canònic** (no un àlies); no hi pot haver dues definicions de la mateixa IP amb noms diferents.

### 9.7 Altres registres freqüents

| Registre | Funció | Exemple |
|---|---|---|
| **AAAA** | Nom → IPv6 | `host IN AAAA 2001:db8::1` |
| **TXT** | Text lliure (SPF, DKIM, verificacions) | `@ IN TXT "v=spf1 mx -all"` |
| **SRV** | Localitza serveis (host + port) | `_ldap._tcp IN SRV 0 5 389 dc1` |
| **CAA** | Quines CA poden emetre certificats | `@ IN CAA 0 issue "letsencrypt.org"` |

### 9.8 Resum de registres

| Tipus | Funció | Direcció |
|---|---|---|
| SOA | Inici d'autoritat de la zona | — |
| NS | Servidor de noms de la zona | — |
| A / AAAA | Nom → IP | Directa |
| CNAME | Àlies → nom canònic | Directa |
| MX | Servidor de correu | Directa |
| PTR | IP → nom | Inversa |

---

## 10. Exemples de fitxers de zona

> Els exemples dels apunts originals tenien algunes errades (claus `{` en lloc de parèntesis, un NS que apuntava a un altre domini, `CNAME` cap a una IP, etc.). Aquí es presenten **corregits**.

### 10.1 Zona directa `cdm.cat`

```dns
; Fitxer de zona directa cdm.cat
$TTL 3D
$ORIGIN cdm.cat.

@        IN  SOA   ns1.cdm.cat. admin.cdm.cat. (
                   23      ; serial
                   8H      ; refresh
                   2H      ; retry
                   4W      ; expire
                   1D )    ; minimum TTL

         IN  NS    ns1.cdm.cat.
         IN  NS    ns2.cdm.cat.
         IN  MX    10 correu.cdm.cat.

ns1      IN  A     192.168.0.5    ; servidor DNS primari
ns2      IN  A     192.168.0.7    ; servidor DNS secundari
correu   IN  A     192.168.0.6    ; servidor de correu
router   IN  A     192.168.0.1    ; router (nom relatiu)
hp-7200c IN  A     192.168.0.2    ; impressora
pc01     IN  A     192.168.0.50
pc02     IN  A     192.168.0.51

www      IN  CNAME ns1            ; àlies web
ftp      IN  CNAME ns1            ; àlies ftp
```

### 10.2 Zona inversa `0.168.192.in-addr.arpa`

```dns
; Fitxer de zona inversa de la xarxa 192.168.0.0/24
$TTL 3D
$ORIGIN 0.168.192.in-addr.arpa.

@   IN  SOA  ns1.cdm.cat. admin.cdm.cat. (
             23      ; serial
             8H      ; refresh
             2H      ; retry
             4W      ; expire
             1D )    ; minimum TTL

    IN  NS   ns1.cdm.cat.
    IN  NS   ns2.cdm.cat.

5   IN  PTR  ns1.cdm.cat.
7   IN  PTR  ns2.cdm.cat.
6   IN  PTR  correu.cdm.cat.
1   IN  PTR  router.cdm.cat.
2   IN  PTR  hp-7200c.cdm.cat.
50  IN  PTR  pc01.cdm.cat.
51  IN  PTR  pc02.cdm.cat.
```

### 10.3 Declaració a Bind9 (`named.conf.local`)

```text
zone "cdm.cat" {
    type master;
    file "/etc/bind/db.cdm.cat";
    allow-transfer { 192.168.0.7; };   // només el secundari
};

zone "0.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.0";
    allow-transfer { 192.168.0.7; };
};
```

Configuració del **secundari** (`ns2`):

```text
zone "cdm.cat" {
    type slave;
    file "/var/cache/bind/db.cdm.cat";
    masters { 192.168.0.5; };
};
```

---

## 11. Comandes útils: Linux i Windows

### 11.1 Consultes DNS (client)

| Acció | Linux (Ubuntu) | Windows (CMD / PowerShell) |
|---|---|---|
| Resolució directa bàsica | `host classe.local` | `nslookup classe.local` |
| Resolució amb `dig` | `dig clientubuntu.classe.local` | `Resolve-DnsName clientubuntu.classe.local` *(PowerShell)* |
| Resum curt (només la IP) | `dig +short clientubuntu.classe.local` | `(Resolve-DnsName clientubuntu.classe.local).IPAddress` |
| Resolució inversa | `dig -x 192.168.100.100`<br>`host 192.168.100.100` | `nslookup 192.168.100.100`<br>`Resolve-DnsName 192.168.100.100` |
| Usar un servidor DNS concret | `dig @192.168.100.100 clientwin.classe.local` | `nslookup clientwin.classe.local 192.168.100.100` |
| Consultar un tipus de registre | `dig classe.local NS`<br>`dig classe.local SOA`<br>`dig gmail.com MX` | `nslookup -type=NS classe.local`<br>`nslookup -type=SOA classe.local`<br>`nslookup -type=MX gmail.com` |
| Tots els registres | `dig classe.local ANY` | `nslookup -type=ANY classe.local` |
| Transferència de zona | `dig @192.168.100.100 classe.local AXFR` | `nslookup`<br>`> server 192.168.100.100`<br>`> ls -d classe.local` |
| Seguiment de la resolució des de l'arrel | `dig +trace www.institutjaumehuguet.cat` | — *(no disponible; usar `Resolve-DnsName -Type NS .`)* |
| Fer ping per nom | `ping clientlinux` | `ping clientlinux` |

### 11.2 `nslookup` en mode interactiu

Funciona igual a Linux i Windows:

```text
nslookup
> server 192.168.100.100
> set type=A
> clientubuntu.classe.local
> set type=PTR
> 192.168.100.100
> set type=MX
> classe.local
> exit
```

> En una resposta de `nslookup`, "**Non-authoritative answer**" indica que la resposta ve de la **caché** d'un altre servidor i no del servidor autoritari.

### 11.3 Caché DNS del client

| Acció | Linux (Ubuntu) | Windows |
|---|---|---|
| Veure la caché | `resolvectl statistics` | `ipconfig /displaydns` |
| Buidar la caché | `sudo resolvectl flush-caches` | `ipconfig /flushdns` |
| Registrar de nou el nom a DNS | — | `ipconfig /registerdns` |
| Veure el servidor DNS en ús | `resolvectl status` | `ipconfig /all` |
| Veure DNS configurats (PowerShell) | `cat /etc/resolv.conf` | `Get-DnsClientServerAddress` |

### 11.4 Configurar el servidor DNS al client

**Linux (Ubuntu 18.04 o superior — Netplan)**, fitxer `/etc/netplan/01-netcfg.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [192.168.100.201/24]
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        search: [classe.local]
        addresses: [192.168.100.100]
```

```bash
sudo netplan try        # prova amb reversió automàtica
sudo netplan apply      # aplica els canvis
```

**Linux (sistemes antics — `/etc/network/interfaces`)**:

```text
dns-search classe.local
dns-nameservers 192.168.100.100
```

**Windows (interfície gràfica):** *Configuració → Xarxa i Internet → Canvia les opcions de l'adaptador → Propietats → Protocol d'Internet versió 4 (TCP/IPv4) → Utilitza les següents adreces de servidor DNS.*

**Windows (PowerShell, com a administrador):**

```powershell
Get-NetAdapter                      # veure el nom de la interfície
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.100.100
Set-DnsClient -InterfaceAlias "Ethernet" -ConnectionSpecificSuffix "classe.local"
Get-DnsClientServerAddress          # comprovar
```

**Windows (CMD, com a administrador):**

```cmd
netsh interface ip set dns name="Ethernet" static 192.168.100.100
netsh interface ip show dns
```

### 11.5 Administració del servidor Bind9 (Ubuntu)

```bash
# Instal·lació
sudo apt update
sudo apt install bind9 bind9-doc dnsutils

# Servei
sudo systemctl status bind9      # a Ubuntu el servei també es diu "named"
sudo systemctl restart bind9
sudo systemctl enable bind9      # arrencada automàtica
sudo rndc reload                 # recarrega la configuració sense reiniciar
sudo rndc reload classe.local    # recarrega només una zona
sudo rndc flush                  # buida la caché del servidor

# Validació de sintaxi
sudo named-checkconf
sudo named-checkzone classe.local /etc/bind/db.classe.local
sudo named-checkzone 100.168.192.in-addr.arpa /etc/bind/db.192.168.100

# Registres i errors
sudo journalctl -u bind9 -f
sudo tail -f /var/log/syslog | grep named

# Comprovar que escolta al port 53
sudo ss -tulpn | grep :53

# Tallafoc (UFW)
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow Bind9             # perfil predefinit
```

### 11.6 Administració d'un servidor DNS a Windows Server *(referència)*

```powershell
Install-WindowsFeature DNS -IncludeManagementTools
Add-DnsServerPrimaryZone -Name "classe.local" -ZoneFile "classe.local.dns"
Add-DnsServerResourceRecordA -ZoneName "classe.local" -Name "clientwin" -IPv4Address 192.168.100.202
Add-DnsServerResourceRecordCName -ZoneName "classe.local" -Name "web" -HostNameAlias "dnsserver.classe.local"
Add-DnsServerPrimaryZone -NetworkId "192.168.100.0/24" -ZoneFile "100.168.192.in-addr.arpa.dns"
Add-DnsServerResourceRecordPtr -ZoneName "100.168.192.in-addr.arpa" -Name "202" -PtrDomainName "clientwin.classe.local"
Add-DnsServerForwarder -IPAddress 8.8.8.8
Get-DnsServerZone
Clear-DnsServerCache
```

### 11.7 Fitxer `hosts` (resolució local manual)

```bash
# Linux
sudo nano /etc/hosts
```

```powershell
# Windows (CMD o PowerShell com a administrador)
notepad C:\Windows\System32\drivers\etc\hosts
```

### 11.8 Diagnòstic

| Acció | Linux | Windows |
|---|---|---|
| Comprovar connexió amb el servidor DNS | `ping 192.168.100.100` | `ping 192.168.100.100` |
| Provar el port 53 | `nc -zvu 192.168.100.100 53` | `Test-NetConnection 192.168.100.100 -Port 53` *(TCP)* |
| Veure l'adreça IP | `ip a` | `ipconfig /all` |
| Veure la porta d'enllaç | `ip route` | `route print` |
| Capturar trànsit DNS | `sudo tcpdump -i any port 53` | Wireshark, filtre `dns` |

---

## 12. Seguretat i evolució del DNS

- **DNSSEC:** afegeix signatures digitals (registres `RRSIG`, `DNSKEY`, `DS`) per garantir que les respostes són autèntiques i no s'han modificat. Es pot comprovar amb `dig +dnssec dominio.cat`.
- **DoT / DoH:** xifren les consultes entre client i resolutor (ports 853 i 443). Windows 11 i els navegadors moderns ho suporten.
- **Atacs habituals:** enverinament de caché (*cache poisoning*), amplificació DDoS amb servidors recursius oberts, segrest de domini.
- **Bones pràctiques en Bind9:**
  - Limitar la recursió només als clients de la xarxa pròpia (`allow-recursion { 192.168.100.0/24; };`).
  - Limitar les transferències de zona al servidor secundari (`allow-transfer`).
  - Ocultar la versió (`version "not disclosed";`).
  - Mantenir el programari actualitzat.

Exemple de `named.conf.options` més segur:

```text
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { 127.0.0.1; 192.168.100.0/24; };
    allow-query     { 127.0.0.1; 192.168.100.0/24; };
    allow-transfer  { none; };

    forwarders { 8.8.8.8; };

    dnssec-validation auto;
    listen-on-v6 { any; };
};
```

---

## 13. Resum ràpid

- **DNS** = base de dades **jeràrquica i distribuïda** que tradueix noms ↔ IP. Usa el **port 53** (UDP i TCP).
- L'**espai de noms** és un arbre; una **zona** és la part administrada per un servidor; la **delegació** passa l'autoritat d'un subdomini.
- Cada zona té un servidor **primari** i, com a mínim, un de **secundari** (redundància).
- **Client → servidor local:** consulta **recursiva**. **Servidor → servidors:** consulta **iterativa**.
- Resposta **autoritativa** (de la zona pròpia) vs. **no autoritativa** (de la caché).
- **Registres:** `SOA` (autoritat), `NS` (servidors), `A`/`AAAA` (nom→IP), `CNAME` (àlies), `MX` (correu), `PTR` (IP→nom).
- **Resolució inversa:** domini `in-addr.arpa` amb l'IP escrita al revés.
- Els noms absoluts (FQDN) **acaben en punt**. Oblidar-lo és l'error més comú en fitxers de zona.
- Cada vegada que es modifica una zona, cal **incrementar el `serial`** i recarregar (`rndc reload`).
