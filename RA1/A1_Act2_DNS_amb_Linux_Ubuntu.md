![Portada](img/portada.jpg)

**Serveis de xarxa i Internet**
**Configuració DNS a Linux**

---

# Configuració d'un servidor DNS sota Linux Ubuntu

Aquesta activitat consisteix a crear un servidor DNS i clients en GNU/Linux. Per aconseguir-ho utilitzarem **BIND** (*Berkeley Internet Name Domain*), l'aplicació més coneguda i utilitzada a Internet per aquesta feina. En concret, **Bind9** és la versió actual de BIND.

Aquesta aplicació pot funcionar tant a **Linux/Unix** com a **Windows** (i també sobre altres sistemes, com FreeBSD), tot i que en aquest cas la implementem sobre Linux.

Al servidor DNS, la configuració del servei es fa editant i modificant els fitxers de configuració, sense utilitzar cap eina gràfica. A Ubuntu podem utilitzar l'eina gràfica *Synaptic* per instal·lar els paquets; la configuració del DNS es fa exactament de la mateixa manera.

---

## Requisits previs: xarxa a ISARD

Aquesta activitat es fa sobre màquines virtuals d'**ISARD**. Cal tenir en compte:

- Cada màquina (**servidor** i **client**) ha de tenir connectada la xarxa **`personal1`** d'ISARD, de manera que totes dues quedin a la mateixa xarxa privada.
- Totes les màquines han d'anar amb **IP fixa** (no DHCP) dins del rang `192.168.100.0/24`, segons la taula del punt 2:
  - Servidor (DNSserver): `192.168.100.100`
  - Client Ubuntu (ClientUbuntu): `192.168.100.201`
  - Client Windows (ClientWin): `192.168.100.202`
- Cal que el servidor tingui IP fixa perquè els registres de zona i el `dns-nameservers` dels clients apunten a la seva adreça. Als clients també s'hi assigna IP fixa perquè coincideixi amb els registres A i PTR de les zones.
- La xarxa `personal1` no dóna sortida a Internet. Si necessiteu instal·lar paquets (`apt`), feu-ho **abans** de canviar la xarxa o utilitzeu també la xarxa per defecte d'ISARD. Recordeu que, si hi ha més d'una interfície, la IP fixa de `192.168.100.x` s'ha de posar a la interfície connectada a `personal1`.

Al client Windows, assigneu la IP `192.168.100.202`, màscara `255.255.255.0`, porta d'enllaç `192.168.100.1` i DNS preferent `192.168.100.100`.

---

## 1. Instal·lació de Bind9 a la màquina servidor DNS

Instal·lem **Bind9** a la màquina dedicada a fer de servidor DNS. Des de la línia d'ordres:

```bash
# apt-get install bind9 bind9-doc dnsutils
```

Una prova que Bind9 ja està instal·lat és que el directori `/etc/bind` existeix.

---

## 2. Configuració del DNS

Configureu el servei DNS basant-vos en l'exemple que es detalla al punt següent.

> ⚠️ **Atenció!** Aquesta configuració del servei DNS (servidor i clients) és un exemple de com s'han d'utilitzar les dades de la taula següent; les dades mostrades a les captures de pantalla no tenen perquè coincidir amb les dades que heu d'emprar.

**Domini `classe.local` a la xarxa `192.168.100.0/24`**

| Noms          | Màquina           |
|---------------|-------------------|
| DNSserver     | `192.168.100.100` |
| ClientUbuntu  | `192.168.100.201` |
| ClientWin     | `192.168.100.202` |
| Linux (àlies) | ClientLinux       |
| Gateway       | `192.168.100.1`   |
| Reenviador    | `8.8.8.8`         |

- El servidor DNS és la màquina amb **IP `192.168.100.100`**.
- L'arxiu de zona directa del domini `classe.local` s'ha de dir **`db.classe.local`**.
- L'arxiu de zona inversa del domini `classe.local` s'ha de dir **`db.192.168.100`**.

> ⚠️ **Atenció!** Recordeu que per realitzar les proves sobre aquest cas concret haureu de canviar la configuració de xarxa del servidor i dels clients per ajustar-les a la xarxa concreta (`192.168.100.0`).

---

## 3. Exemple de configuració del servidor DNS

En aquest exemple el domini es dirà **`classe.local`** i estem a la xarxa **`192.168.100.0/24`**. El servidor s'anomenarà **DNSserver** i la seva IP és `192.168.100.100`. Els clients Ubuntu i Windows s'anomenaran respectivament **ClientUbuntu** i **ClientWin**, i les seves IP seran `192.168.100.201` i `192.168.100.202`. A més, crearem un àlies per a la màquina ClientUbuntu amb nom **ClientLinux**.

### 3.1. Zona directa

El primer pas és crear els arxius de zona directa i de zona inversa. Al fitxer de zona directa li direm `db.classe.local`.

> Normalment a aquest arxiu se l'acostuma a anomenar amb la paraula `db` seguida d'un punt i el nom del domini. Això és un conveni i no és necessari seguir-lo.

Disposem de l'editor **nano** per crear i editar fitxers de text. Creem l'arxiu `db.classe.local`:

```bash
# nano /etc/bind/db.classe.local
```

i el modifiquem de manera que al final tingui la informació següent:

```dns
$ORIGIN classe.local.
$TTL 604800
@   IN  SOA DNSserver.classe.local. root.classe.local. (
            2           ; Serial
            604800      ; Refresh
            86400       ; Retry
            2419200     ; Expire
            604800 )    ; Negative Cache TTL

@             IN  NS     DNSserver.classe.local.
DNSserver     IN  A      192.168.100.100
ClientUbuntu  IN  A      192.168.100.201
ClientWin     IN  A      192.168.100.202
ClientLinux   IN  CNAME  ClientUbuntu.classe.local.
```

**Explicació:**

- La directiva `$ORIGIN` del començament especifica el nom del domini. Això ens permet definir posteriorment els noms de manera abreujada i no totalment qualificada (si no tinguéssim aquesta directiva, als registres de tipus A hauríem de posar `ClientUbuntu.classe.local.` i no simplement `ClientUbuntu`, tal com es mostra).
- El registre **SOA** especifica el nom del servidor DNS; la resta de valors els deixem per defecte.
- També ha d'estar present un registre **NS** (com a mínim) que indiqui quin és el servidor DNS. L'`@` equival al nom del domini.
- A continuació hi ha 3 registres **A** que associen els noms amb les IP del servidor DNS, el client Ubuntu i el client Windows.
- Finalment, un registre **CNAME** defineix l'àlies del client Ubuntu.

### 3.2. Zona inversa

El pas següent és crear l'arxiu de zona inversa corresponent a la xarxa `192.168.100.0/24`. Per conveni, utilitzem un nom format per la paraula `db` seguida d'un punt i la part de xarxa de l'adreça. Així, el nostre arxiu es dirà `db.192.168.100`.

Creem l'arxiu amb l'editor nano:

```bash
# nano /etc/bind/db.192.168.100
```

i el modifiquem perquè finalment sigui així:

```dns
$ORIGIN 100.168.192.in-addr.arpa.
$TTL 604800
@   IN  SOA DNSserver.classe.local. root.classe.local. (
            2           ; Serial
            604800      ; Refresh
            86400       ; Retry
            2419200     ; Expire
            604800 )    ; Negative Cache TTL

@     IN  NS   DNSserver.classe.local.
100   IN  PTR  DNSserver.classe.local.
201   IN  PTR  ClientUbuntu.classe.local.
202   IN  PTR  ClientWin.classe.local.
```

Fixem-nos que en aquest cas la directiva `$ORIGIN` ha canviat per ajustar-se a la nomenclatura estàndard de les zones inverses. Així, als registres **PTR** (que serveixen per obtenir el nom DNS a partir de la IP) només cal especificar la part de l'adreça IP que identifica el host dins la xarxa.

### 3.3. Declaració de les zones (`named.conf.local`)

El pas següent consisteix a editar el fitxer `named.conf.local` i afegir-hi:

```
// Zona directa
zone "classe.local" {
    type master;
    file "/etc/bind/db.classe.local";
};

// Zona inversa
zone "100.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.100";
};
```

En aquest fitxer especifiquem les dues zones que acabem de crear:

- La **zona directa** es diu `classe.local`; amb el paràmetre `file` indiquem la ruta del fitxer de zona directa creat en el pas anterior.
- La **zona inversa** es diu `100.168.192.in-addr.arpa` (el nom que ha de tenir la zona inversa corresponent a la xarxa `192.168.100.0/24`); amb el paràmetre `file` indiquem la ruta del fitxer de zona inversa.
- El paràmetre `type master` indica que són zones d'un servidor primari (en el nostre cas no hi ha servidors secundaris o esclaus).

Tot seguit comprovem que el fitxer `named.conf` inclou (no està comentada) la línia següent:

```
include "/etc/bind/named.conf.local";
```

### 3.4. Reenviadors (`named.conf.options`)

Finalment, configurarem els *forwarders* o reenviadors perquè, en cas que el servidor DNS no pugui resoldre un nom directament (no és a la memòria cau), reenviï la petició als reenviadors abans d'iniciar el procés de resolució de noms. Per fer-ho, editem el fitxer `named.conf.options` i posem a la secció `forwarders` les adreces IP dels servidors als quals volem reenviar les peticions. En aquest exemple posarem com a reenviadors els equips amb IP `62.81.23.15` i `62.42.201.12`. El fitxer modificat tindrà aquest aspecte:

```
options {
    directory "/var/cache/bind";
    forwarders {
        62.81.23.15;
        62.42.201.12;
    };
    auth-nxdomain no;    # conform to RFC1035
    listen-on-v6 { any; };
};
```

### 3.5. Comprovació i reinici del servei

Ja està tot configurat. Podem comprovar que els fitxers de configuració són sintàcticament correctes amb les ordres següents:

```bash
named-checkconf
# valida el fitxer named.conf

named-checkzone classe.local /etc/bind/db.classe.local
# valida el fitxer de la zona classe.local
```

Ja només cal reiniciar el servei DNS:

```bash
# /etc/init.d/bind9 restart
```

Podem comprovar si el servidor s'ha iniciat correctament i si hi ha errors sintàctics als fitxers de configuració consultant l'arxiu `/var/log/syslog`.

### 3.6. Configuració dels clients

Només manca configurar els clients perquè utilitzin el servidor de noms. En Linux, cal afegir les línies següents al fitxer `/etc/network/interfaces`:

```
dns-search classe.local
dns-nameservers 192.168.100.100
```

---

## 4. Comprovació del funcionament del servidor DNS

Una vegada configurat el servidor DNS amb les dades del punt 2, contesta les preguntes següents per comprovar-ne el funcionament:

1. Assenyala les línies de l'arxiu `syslog` que diuen que el servidor DNS ha arrencat correctament.
2. Assenyala unes línies de l'arxiu `syslog` que indiquin que hi ha una errada als arxius.
3. Prova de fer que un client Windows faci resolució mitjançant el servidor DNS muntat amb Bind9. Pot resoldre **linuxdns** o **Ubuntu**, o només pot resoldre **linuxdns.classe.local**? Per què?
4. Tracta de ficar el client Windows al domini `classe.local`. Què passa?

Per comprovar que la **resolució** funciona correctament, fem *ping* d'una màquina a una altra i hauríem de rebre els paquets de resposta. Per exemple, des de **DNSserver** a **ClientLinux**:

```bash
ping ClientLinux
```

---

## 5. Ordres `host`, `dig` i `nslookup`

1. Fes servir l'ordre **`host`** al context que has creat i indica com resoldre una adreça i el nom d'un domini.
2. Fes servir l'ordre **`dig`** des del client perquè retorni l'adreça de `Linux.classe.local`.
3. Fes servir l'ordre **`nslookup`** des del client Ubuntu per obtenir l'adreça de l'equip Windows del domini `classe.local`. Funciona l'ordre? Quin port utilitza el servidor?
4. Fes servir l'ordre **`nslookup`** des del client Windows per obtenir el nom del servidor DNS que acabem de configurar fent resolució inversa.
5. Fes servir l'ordre **`dig`** des del mateix servidor DNS per obtenir informació sobre el domini `classe.local`.

---

## 6. Resum de les comprovacions: què es fa a cada màquina

| Màquina | Comprovació | Ordre / acció | Resultat esperat |
|---|---|---|---|
| **Servidor** | Sintaxi de la configuració | `named-checkconf` | Cap error |
| **Servidor** | Sintaxi de la zona directa | `named-checkzone classe.local /etc/bind/db.classe.local` | `OK` |
| **Servidor** | Sintaxi de la zona inversa | `named-checkzone 100.168.192.in-addr.arpa /etc/bind/db.192.168.100` | `OK` |
| **Servidor** | Servei arrencat | `systemctl status bind9` i `/var/log/syslog` | Servei actiu, zones carregades |
| **Servidor** | Resolució local | `ping ClientLinux`, `host ClientUbuntu`, `dig classe.local` | Respon amb `192.168.100.201` |
| **Servidor** | Port del servei | `ss -tulpn \| grep named` | Escolta al port 53 (UDP/TCP) |
| **Client Ubuntu** | Connectivitat | `ping 192.168.100.100` | Respon |
| **Client Ubuntu** | Resolució directa | `ping DNSserver`, `host ClientWin`, `dig ClientLinux.classe.local`, `nslookup ClientWin.classe.local` | Retorna `192.168.100.202` |
| **Client Ubuntu** | Àlies (CNAME) | `dig ClientLinux.classe.local` | CNAME → `ClientUbuntu`, IP `192.168.100.201` |
| **Client Ubuntu** | Resolució inversa | `nslookup 192.168.100.100` o `dig -x 192.168.100.100` | `DNSserver.classe.local` |
| **Client Windows** | Connectivitat | `ping 192.168.100.100` | Respon |
| **Client Windows** | Resolució directa | `nslookup DNSserver.classe.local` i `ping ClientLinux.classe.local` | Retorna la IP correcta |
| **Client Windows** | Resolució inversa | `nslookup 192.168.100.100` | `DNSserver.classe.local` |
| **Client Windows** | Nom curt | `ping ClientLinux` (sense sufix) | Depèn del sufix DNS configurat; raoneu-ho a la pregunta 3 del punt 4 |

> 💡 Sempre que sigui possible, comproveu primer la connectivitat amb `ping` per IP (si no respon, el problema és de xarxa, no de DNS) i després la resolució per nom (si respon per IP però no per nom, el problema és del DNS).

---

## Resolució de l'activitat

Configura el servidor DNS amb les dades de la xarxa que tu has fet servir a l'activitat de Windows.

- **Fes captura de pantalla de cada apartat** de la configuració.
- És imprescindible que **documentis els processos realitzats**, que **indiquis les incidències** que et puguis trobar a l'apartat corresponent i **els enllaços** als quals t'has dirigit en cas de necessitar consultar informació extra.

**Format de lliurament: PDF**
