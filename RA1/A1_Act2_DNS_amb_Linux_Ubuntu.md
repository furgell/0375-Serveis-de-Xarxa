# Configuració d'un servidor DNS sota Linux Ubuntu

**Serveis de xarxa i Internet · A1. Activitat 2 · Configuració DNS a Linux**

---

## Introducció

Aquesta activitat consisteix a crear un **servidor DNS** i els seus **clients** en GNU/Linux. Utilitzarem **BIND** (*Berkeley Internet Name Domain*), l'aplicació més coneguda i utilitzada a Internet per a aquesta feina. En concret, **Bind9** és la versió actual de BIND.

BIND pot funcionar tant en **Linux/Unix** com en **Windows** (i en altres sistemes, com FreeBSD). En aquesta activitat l'implementem sobre Linux.

La configuració del servei DNS es fa **editant els fitxers de configuració**, sense eines gràfiques. A Ubuntu es pot utilitzar Synaptic per instal·lar els paquets, però la configuració del DNS es fa exactament igual.

---

## 1. Instal·lació de Bind9 al servidor DNS

A la màquina dedicada a fer de servidor DNS, instal·la Bind9 des del terminal:

```bash
sudo apt-get install bind9 bind9-doc dnsutils
```

> **Comprovació:** Bind9 està instal·lat si existeix el directori `/etc/bind`.

---

## 2. Dades de la configuració

Configura el servei DNS a partir de les dades de la taula següent.

> ⚠️ **Atenció.** Les dades d'aquest enunciat són un **exemple**. Les dades que apareguin a les captures de pantalla no tenen per què coincidir amb les que has d'emprar. Per fer les proves, has d'adaptar la configuració de xarxa del servidor i dels clients a la xarxa concreta (`192.168.100.0/24`).

**Domini `classe.local` a la xarxa `192.168.100.0/24`**

| Nom | Màquina / IP |
|---|---|
| DNSserver | 192.168.100.100 |
| ClientUbuntu | 192.168.100.201 |
| ClientWin | 192.168.100.202 |
| Alies de ClientUbuntu | ClientLinux |
| Gateway | 192.168.100.1 |
| Reenviador (*forwarder*) | 8.8.8.8 |

Requisits:

- El servidor DNS és la màquina amb IP **192.168.100.100**.
- El fitxer de zona directa del domini `classe.local` s'ha d'anomenar **`db.classe.local`**.
- El fitxer de zona inversa s'ha d'anomenar **`db.192.168.100`**.

---

## 3. Exemple de configuració del servidor DNS

En aquest exemple el domini és `classe.local` i la xarxa és `192.168.100.0/24`. El servidor s'anomena `DNSserver` (IP `192.168.100.100`). Els clients `ClientUbuntu` i `ClientWin` tenen les IP `192.168.100.201` i `192.168.100.202`. A més, es crea l'àlies `ClientLinux` per a `ClientUbuntu`.

### 3.1 Fitxer de zona directa

Normalment aquest fitxer es nomena amb la paraula `db` seguida d'un punt i el nom del domini. És un conveni, no una obligació.

Crea'l amb l'editor `nano`:

```bash
sudo nano /etc/bind/db.classe.local
```

Contingut final:

```dns
$ORIGIN classe.local.
$TTL 604800
@   IN  SOA DNSserver.classe.local. root.classe.local. (
            2          ; Serial
            604800     ; Refresh
            86400      ; Retry
            2419200    ; Expire
            604800 )   ; Negative Cache TTL

@             IN  NS     DNSserver.classe.local.
DNSserver     IN  A      192.168.100.100
ClientUbuntu  IN  A      192.168.100.201
ClientWin     IN  A      192.168.100.202
ClientLinux   IN  CNAME  ClientUbuntu.classe.local.
```

Explicació dels elements:

- **`$ORIGIN`**: indica el nom del domini i permet escriure els noms de forma abreujada. Sense aquesta directiva caldria escriure `ClientUbuntu.classe.local.` en lloc de `ClientUbuntu`.
- **`SOA`**: indica el nom del servidor DNS; la resta de valors es deixen per defecte.
- **`NS`**: ha d'haver-hi almenys un registre que indiqui el servidor de noms. L'`@` equival al nom del domini.
- **`A`**: associen noms amb IP (servidor DNS, client Ubuntu i client Windows).
- **`CNAME`**: defineix l'àlies del client Ubuntu.

### 3.2 Fitxer de zona inversa

Per conveni, el nom és `db` seguit d'un punt i la part de xarxa de l'adreça. Aquí: `db.192.168.100`.

```bash
sudo nano /etc/bind/db.192.168.100
```

Contingut final:

```dns
$ORIGIN 100.168.192.in-addr.arpa.
$TTL 604800
@   IN  SOA DNSserver.classe.local. root.classe.local. (
            2          ; Serial
            604800     ; Refresh
            86400      ; Retry
            2419200    ; Expire
            604800 )   ; Negative Cache TTL

@     IN  NS   DNSserver.classe.local.
100   IN  PTR  DNSserver.classe.local.
201   IN  PTR  ClientUbuntu.classe.local.
202   IN  PTR  ClientWin.classe.local.
```

Fixa't que la directiva `$ORIGIN` canvia per adaptar-se a la nomenclatura estàndard de les zones inverses. Als registres `PTR` (que serveixen per obtenir el nom a partir de la IP) només cal indicar la part de l'adreça que identifica el *host* dins la xarxa.

### 3.3 Declaració de les zones (`named.conf.local`)

Edita `/etc/bind/named.conf.local` i afegeix:

```text
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

- La zona directa es diu `classe.local`; el paràmetre `file` indica la ruta del fitxer de zona directa.
- La zona inversa es diu `100.168.192.in-addr.arpa` (nom que ha de tenir la zona inversa de la xarxa `192.168.100.0/24`); `file` indica la ruta del fitxer de zona inversa.
- `type master` indica que són zones d'un servidor primari (en aquesta activitat no hi ha servidors secundaris).

Comprova que `/etc/bind/named.conf` inclou la línia següent (descomentada):

```text
include "/etc/bind/named.conf.local";
```

### 3.4 Reenviadors (`named.conf.options`)

Els *forwarders* fan que, si el servidor no pot resoldre un nom directament (no el té a la memòria cau), reenviï la petició a uns altres servidors DNS abans d'iniciar ell mateix la resolució.

Edita `/etc/bind/named.conf.options` i indica l'IP del reenviador a la secció `forwarders` (en aquesta activitat, `8.8.8.8`):

```text
options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
    };

    auth-nxdomain no;    # conform to RFC1035
    listen-on-v6 { any; };
};
```

### 3.5 Validació i reinici del servei

Comprova que la sintaxi dels fitxers és correcta:

```bash
named-checkconf                                          # valida named.conf
named-checkzone classe.local /etc/bind/db.classe.local   # valida la zona directa
named-checkzone 100.168.192.in-addr.arpa /etc/bind/db.192.168.100   # valida la zona inversa
```

Reinicia el servei:

```bash
sudo systemctl restart bind9
```

> Per comprovar que el servidor ha arrencat bé (o si hi ha errors de sintaxi), consulta `/var/log/syslog`, per exemple amb `sudo tail -f /var/log/syslog` o `sudo journalctl -u bind9`.

### 3.6 Configuració dels clients

Els clients han d'utilitzar el nostre servidor de noms.

**Client Linux.** Afegeix aquestes línies a `/etc/network/interfaces` (o la configuració equivalent del teu sistema, per exemple Netplan):

```text
dns-search classe.local
dns-nameservers 192.168.100.100
```

**Client Windows.** A les propietats TCP/IPv4 de l'adaptador de xarxa, indica com a servidor DNS preferit `192.168.100.100`.

---

## 4. Comprovació del funcionament del servidor DNS

Un cop configurat el servidor DNS amb les dades de l'apartat 2, respon les preguntes següents:

1. Assenyala les línies de `syslog` que indiquen que el servidor DNS **ha arrencat correctament**.
2. Assenyala unes línies de `syslog` que indiquin que hi ha **una errada als fitxers** (provoca-la tu mateix de manera controlada).
3. Fes que un client Windows faci resolució mitjançant el servidor DNS muntat amb Bind9. Pot resoldre `ClientLinux` (o `ClientUbuntu`) només amb el nom curt, o només pot resoldre el nom complet `ClientLinux.classe.local`? Per què?
4. Prova de posar el client Windows al domini `classe.local`. Què passa?

Per comprovar que la resolució funciona, fes `ping` d'una màquina a una altra. Haurien de tornar els paquets de resposta. Per exemple, des de `DNSserver` a `ClientLinux`:

```bash
ping ClientLinux
```

---

## 5. Comandes `host`, `dig` i `nslookup`

1. Fes servir la comanda **`host`** al context que has creat i indica com resoldre **una adreça** i **el nom d'un domini**.
2. Fes servir la comanda **`dig`** des del client perquè retorni l'adreça de `ClientLinux.classe.local`.
3. Fes servir la comanda **`nslookup`** des del client Ubuntu per obtenir l'adreça de l'equip Windows del domini `classe.local`. Funciona la comanda? Quin port utilitza el servidor?
4. Fes servir la comanda **`nslookup`** des del client Windows per obtenir el nom del servidor DNS que acabes de configurar mitjançant **resolució inversa**.
5. Fes servir la comanda **`dig`** des del mateix servidor DNS per obtenir informació sobre el domini `classe.local`.

---

## Resolució de l'activitat i lliurament

Configura el servidor DNS amb les dades de la xarxa que has fet servir a l'activitat de Windows.

Has de:

- Fer **captura de pantalla de cada apartat** de la configuració.
- **Documentar** els processos realitzats.
- Indicar les **incidències** que et puguis trobar, a l'apartat corresponent.
- Incloure els **enllaços** que hagis consultat si has necessitat informació extra.

**Format de lliurament:** PDF.
