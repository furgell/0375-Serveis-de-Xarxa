# Activitat: Instal·lació de Pi-hole en un Ubuntu client

## Context i objectius

En aquesta activitat instal·lareu Pi-hole en un Ubuntu client i comprovareu que bloqueja anuncis i rastrejadors a nivell de DNS. Pi-hole és un servidor DNS que filtra: si un nom és a les llistes de bloqueig, respon amb una adreça nul·la (0.0.0.0) i el navegador no arriba a descarregar l'anunci. La resta de noms els reenvia a un DNS extern (*forwarder*), com ja vam veure a la teoria del servei DNS.

**Objectius d'aprenentatge**

- Instal·lar Pi-hole en un Ubuntu amb IP estàtica i accedir al seu panell web.
- Configurar el client perquè faci servir Pi-hole com a servidor DNS.
- Comprovar el bloqueig amb `dig` i amb el navegador, i interpretar les respostes (NOERROR, adreça nul·la, caché).
- Crear exclusions (*allowlist*) per a dominis legítims bloquejats per error.
- Extreure el llistat d'exclusions i les evidències de la pràctica.

**Escenari i materials**

| Element | Valor |
| --- | --- |
| Equip | `clientubuntu`, escriptori virtual a Isard VDI |
| Sistema operatiu | Ubuntu 22.04 o 24.04 LTS (escriptori o servidor) |
| Recursos mínims | 2 GB de RAM, 20 GB de disc, accés a Internet |
| Xarxa | Xarxa *default* d'Isard VDI (una sola interfície), adreça IP rebuda per DHCP |
| DNS del client | 127.0.0.1 (el propi Pi-hole), forçat amb Netplan |
| DNS extern (*upstream*) | 1.1.1.1 o 8.8.8.8 |
| Durada estimada | 2 hores |

> L'adreça IP, la porta d'enllaç i el nom de la interfície depenen de l'escriptori d'Isard: no copieu valors d'aquest enunciat, comproveu-los sempre amb `ip a` i `ip r`.

## Part 1. Preparació del sistema

Abans d'instal·lar, cal comprovar que l'equip té xarxa i Internet, identificar la interfície i veure qui fa servir el port 53. La IP no cal fixar-la: es manté per DHCP i només canviarem el DNS (Part 3).

**1.1. Actualitzeu el sistema i instal·leu les eines**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git dnsutils sqlite3
```

`dnsutils` aporta `dig` i `nslookup`; `sqlite3` el farem servir al final per extreure el llistat d'exclusions.

**1.2. Comproveu la xarxa i identifiqueu la interfície**

```bash
ip a
ip r
ping -c 3 8.8.8.8
```

Anoteu el **nom de la interfície** (per exemple `enp1s0` o `ens3`) i l'adreça IP que ha rebut per DHCP. El nom el necessitareu a la Part 3. La línia `default via ...` de `ip r` és la porta d'enllaç.

**1.3. Comproveu qui fa servir el port 53**

```bash
sudo ss -tulpn | grep :53
```

En Ubuntu, `systemd-resolved` escolta a 127.0.0.53:53 i pot entrar en conflicte amb Pi-hole. Deseu la sortida de la comanda: la comparareu després de la instal·lació.

- [ ] Sistema actualitzat
- [ ] Interfície i adreça IP identificades, i `ping` a Internet correcte
- [ ] Sortida de `ss` inicial desada

> **Captura 1.** Sortida de `ip a` (amb la IP rebuda per DHCP) i de `ss -tulpn | grep :53`.

## Part 2. Instal·lació de Pi-hole

L'instal·lador oficial és un script interactiu d'uns minuts que deixa Pi-hole funcionant com a servei (`pihole-FTL`) amb el panell web inclòs.

**2.1. Executeu l'instal·lador**

```bash
curl -sSL https://install.pi-hole.net | bash
```

**2.2. Respongueu l'assistent**

| Pantalla | Què heu de triar |
| --- | --- |
| Avís de IP estàtica | Continueu: l'equip rep la IP per DHCP i només fixarem el DNS a la Part 3 |
| Interfície | Trieu la que heu anotat a la Part 1 |
| Upstream DNS Provider | Cloudflare o Google |
| Blocklists | Accepteu la llista per defecte (StevenBlack Unified Hosts) |
| Query logging | Sí, activat; privacitat: mostra-ho tot |
| Resum final | Anoteu la contrasenya del panell que us mostra |

> Si no heu pogut anotar la contrasenya, definiu-ne una de nova amb `sudo pihole setpassword`.

**2.3. Comproveu que el servei funciona**

```bash
pihole status
pihole -v
sudo ss -tulpn | grep :53
```

La darrera comanda ha de mostrar `pihole-FTL` escoltant al port 53 (UDP i TCP). Compareu-ho amb la captura de la Part 1.

**2.4. Entreu al panell web**

Obriu el navegador i aneu a `http://localhost/admin` (o `http://pi.hole/admin` un cop el client usi Pi-hole com a DNS). Accediu amb la contrasenya i localitzeu aquestes pantalles: **Dashboard**, **Query Log**, **Adlists**, **Domains** i **Settings**.

- [ ] Instal·lador finalitzat sense errors
- [ ] `pihole status` indica que el bloqueig està actiu
- [ ] Accés al panell web amb la contrasenya

> **Captura 2.** Sortida de `pihole status` i de `ss -tulpn | grep :53`.
>
> **Captura 3.** Dashboard del panell web amb el nom de l'equip a la barra del navegador.

## Part 3. El client fa servir Pi-hole com a DNS

Perquè el bloqueig funcioni, el propi Ubuntu ha de preguntar a Pi-hole, és a dir, a ell mateix (127.0.0.1).

**3.1. Forceu el DNS a Netplan (mantenint el DHCP)**

La IP i la ruta continuaran venint del DHCP d'Isard, però ignorarem els servidors DNS que ens ofereix i posarem el Pi-hole local. Mireu quins fitxers hi ha i creeu-ne un de nou que s'apliqui l'últim:

```bash
ls /etc/netplan/
sudo nano /etc/netplan/99-pihole.yaml
```

Contingut (canvieu `enp1s0` pel nom de la vostra interfície):

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false
      nameservers:
        addresses: [127.0.0.1]
```

Què fa cada línia:

| Línia | Efecte |
| --- | --- |
| `dhcp4: true` | L'equip continua rebent IP i porta d'enllaç per DHCP |
| `dhcp4-overrides: use-dns: false` | Ignora els servidors DNS que envia el DHCP |
| `nameservers: addresses: [127.0.0.1]` | Fixa com a únic DNS el Pi-hole local |
| `renderer: networkd` | Fa que Netplan gestioni la interfície directament (a l'escriptori, per defecte la gestiona NetworkManager) |

Apliqueu-ho i comproveu-ho:

```bash
sudo chmod 600 /etc/netplan/99-pihole.yaml
sudo netplan try
sudo netplan apply
ip a
resolvectl status
```

`netplan try` desfà els canvis si no els confirmeu en 120 segons: és la xarxa de seguretat si us equivoqueu. Després d'aplicar, `ip a` ha de mostrar la mateixa IP que abans.

> Si l'equip ja té un altre fitxer a `/etc/netplan/` per a la mateixa interfície, no l'esborreu: el `99-pihole.yaml` s'aplica l'últim i té prioritat.

**3.2. Comproveu la configuració**

```bash
resolvectl status
cat /etc/resolv.conf
```

`resolvectl status` ha de mostrar 127.0.0.1 com a servidor DNS de la interfície. El fitxer `/etc/resolv.conf` apunta a 127.0.0.53 (el *stub* de `systemd-resolved`), que reenvia a Pi-hole: és normal.

**3.3. Proveu que resol**

```bash
dig @127.0.0.1 wikipedia.org
ping -c 2 wikipedia.org
```

Si no resol, passeu a l'apartat de resolució de problemes abans de continuar.

- [ ] `resolvectl status` mostra 127.0.0.1
- [ ] Un nom públic es resol correctament

> **Captura 4.** Sortida de `resolvectl status` i de `dig @127.0.0.1 wikipedia.org`.

## Part 4. Comprovació del bloqueig d'anuncis

Heu de demostrar el bloqueig de tres maneres: amb `dig`, amb el navegador i amb el registre de consultes de Pi-hole.

**4.1. Prova amb `dig`**

Compareu la resposta de Pi-hole amb la d'un DNS extern per a un domini publicitari:

```bash
dig @127.0.0.1 doubleclick.net +short
dig @1.1.1.1 doubleclick.net +short
dig @127.0.0.1 pagead2.googlesyndication.com +short
dig @127.0.0.1 wikipedia.org +short
```

Si el domini és a les llistes, Pi-hole respon 0.0.0.0 mentre que el DNS extern retorna l'adreça real. Wikipedia s'ha de resoldre amb normalitat. Per veure els detalls (estat, secció *ANSWER*, temps de consulta) executeu també la versió completa de la primera comanda sense `+short`.

Per saber quina llista bloqueja un domini:

```bash
pihole -q doubleclick.net
```

També ho podeu fer des del panell web, a **Tools → Search Adlists**.

**4.2. Prova amb el navegador**

1. Obriu una pàgina amb molta publicitat, per exemple un diari digital o el comprovador d'anuncis `https://d3ward.github.io/toolz/adblock.html` (pàgina de tercers, pot canviar).
2. Anoteu quants anuncis i bàners apareixen, o la puntuació de la pàgina de proves.
3. Obriu el **Query Log** del panell de Pi-hole i localitzeu les consultes bloquejades mentre carregueu la pàgina.
4. Desactiveu el bloqueig 30 segons i torneu a carregar la pàgina:

```bash
pihole disable 30s
# recarregueu la pàgina al navegador
pihole enable
```

5. Compareu la pàgina amb el bloqueig activat i desactivat.

> Si veieu els mateixos anuncis amb el bloqueig activat, buideu la caché DNS del navegador i recarregueu amb Ctrl+Shift+R. Alguns navegadors tenen activat el DNS segur (DoH) i s'estan saltant Pi-hole: desactiveu-lo a la configuració del navegador.

**4.3. Taula de resultats**

Ompliu la taula amb el que observeu al vostre equip:

| Prova | Acció | Resultat esperat | Resultat obtingut |
| --- | --- | --- | --- |
| 1 | `dig @127.0.0.1 doubleclick.net` | Bloquejat (0.0.0.0) | |
| 2 | `dig @1.1.1.1 doubleclick.net` | IP real | |
| 3 | `dig @127.0.0.1 wikipedia.org` | IP real | |
| 4 | Pàgina amb anuncis, bloqueig actiu | Menys anuncis | |
| 5 | Pàgina amb anuncis, bloqueig desactivat | Anuncis visibles | |
| 6 | Consulta repetida de la prova 3 | Temps de resposta menor (caché) | |

- [ ] Les sis proves de la taula estan completades
- [ ] He trobat les consultes bloquejades al Query Log
- [ ] He identificat la llista que bloqueja `doubleclick.net`

> **Captura 5.** Resultat de `dig` amb domini bloquejat i amb domini permès.
>
> **Captura 6.** Query Log amb consultes bloquejades (en vermell o amb estat *Blocked*).
>
> **Captura 7.** La mateixa pàgina web amb el bloqueig activat i desactivat.

## Part 5. Llistes de bloqueig i llista d'exclusions

Un bloqueig massiu pot trencar pàgines legítimes (falsos positius); la solució és afegir exclusions, que a Pi-hole es diuen *allowlist*, i documentar-les.

**5.1. Reviseu les llistes de bloqueig (adlists)**

1. Al panell, aneu a **Adlists** i anoteu quantes llistes hi ha i quin és el seu origen.
2. Al **Dashboard**, anoteu el valor de *Domains on Lists* (dominis bloquejats en total).
3. Afegiu una segona llista de bloqueig, per exemple `https://adaway.org/hosts.txt`, amb un comentari que digui qui l'ha afegit.
4. Actualitzeu la base de dades de bloqueig (*gravity*) i comproveu que el comptador ha pujat:

```bash
pihole -g
```

**5.2. Creeu exclusions (allowlist)**

Cal afegir **com a mínim quatre exclusions**, i totes han de portar un comentari que expliqui per què s'han creat.

| Exclusió | Com es crea | Què heu de comprovar |
| --- | --- | --- |
| 1. `doubleclick.net` | Panell: **Domains → Add a new domain**, tipus *Allow*, comentari obligatori | Que `dig @127.0.0.1 doubleclick.net +short` ara retorna una IP real |
| 2. `pagead2.googlesyndication.com` | Terminal: `pihole allow pagead2.googlesyndication.com` | El mateix `dig` per a aquest nom |
| 3. Un domini de la vostra navegació | Cerqueu al **Query Log** un domini bloquejat que trenqui una pàgina legítima i permeteu-lo | Que la pàgina funciona després de l'exclusió |
| 4. Una expressió regular (*regex*) | Panell: **Domains → Add a new domain → Regex filter**, tipus *Allow* | Que els noms que encaixen amb l'expressió es resolen |

```bash
pihole allow pagead2.googlesyndication.com
dig @127.0.0.1 pagead2.googlesyndication.com +short
```

Un exemple de regex d'exclusió per permetre un domini i tots els seus subdominis és `(\.|^)exemple\.org$`. Adapteu-lo a un domini real que hàgiu trobat bloquejat.

> Cada exclusió redueix la protecció. Afegiu només els dominis que necessiteu, i no copieu llistes d'exclusions d'Internet sense revisar-les.

- [ ] Segona llista de bloqueig afegida i *gravity* actualitzat
- [ ] Quatre exclusions creades, cadascuna amb comentari
- [ ] Comprovat amb `dig` que els dominis exclosos ja es resolen

> **Captura 8.** Pantalla **Domains** amb les exclusions i els seus comentaris.
>
> **Captura 9.** Sortida de `dig` per a dos dominis exclosos.

## Part 6. Extracció del llistat d'exclusions

Heu d'obtenir el llistat d'exclusions de tres maneres: des del panell, des de la base de dades i amb una còpia de seguretat.

**6.1. Des del panell web**

A **Domains**, filtreu per la pestanya *Allow* i feu una captura on es vegin el domini, el tipus (exacte o regex), l'estat i el comentari.

**6.2. Des del terminal, amb la base de dades de Pi-hole**

Pi-hole desa els dominis permesos i bloquejats a la taula `domainlist` de la base de dades `gravity.db`. El camp `type` indica què és cada entrada:

| `type` | Significat |
| --- | --- |
| 0 | Allowlist, domini exacte |
| 1 | Denylist, domini exacte |
| 2 | Allowlist, expressió regular |
| 3 | Denylist, expressió regular |

Mostreu per pantalla només les exclusions (tipus 0 i 2):

```bash
sudo sqlite3 -header -column /etc/pihole/gravity.db \
  "SELECT id, domain, type, enabled, comment FROM domainlist WHERE type IN (0,2) ORDER BY domain;"
```

Deseu-les en un fitxer CSV amb la data de creació llegible. Substituïu `cognom` pel vostre cognom:

```bash
sudo sqlite3 -header -csv /etc/pihole/gravity.db \
  "SELECT domain, type, enabled, datetime(date_added,'unixepoch','localtime') AS afegit, comment FROM domainlist WHERE type IN (0,2) ORDER BY domain;" \
  > ~/exclusions_cognom.csv

cat ~/exclusions_cognom.csv
wc -l ~/exclusions_cognom.csv
```

El CSV ha de tenir una fila de capçalera i una fila per cada exclusió que heu creat.

**6.3. Còpia de seguretat de la configuració**

Al panell, aneu a **Settings → Teleporter** i exporteu la configuració. Obtindreu un fitxer comprimit que inclou les llistes de bloqueig i les exclusions i que serveix per restaurar Pi-hole en un altre equip.

**6.4. Estadístiques per a l'informe**

Al **Dashboard**, anoteu el percentatge de consultes bloquejades, el total de consultes i els tres dominis més bloquejats (*Top Blocked Domains*).

- [ ] Captura del panell amb les exclusions
- [ ] Fitxer `exclusions_cognom.csv` generat i revisat
- [ ] Còpia de Teleporter descarregada
- [ ] Estadístiques del Dashboard anotades

> **Captura 10.** Sortida de la consulta SQL a la terminal.
>
> **Captura 11.** Dashboard amb el percentatge de consultes bloquejades i els dominis més bloquejats.

## Resolució de problemes

La majoria d'errors venen del port 53, de la IP o d'un navegador que no fa servir el DNS del sistema.

| Símptoma | Causa probable | Què fer |
| --- | --- | --- |
| `pihole status` diu que el servei no corre, o `ss` no mostra `pihole-FTL` al port 53 | `systemd-resolved` ocupa el port 53 | Editeu `/etc/systemd/resolved.conf`, poseu `DNSStubListener=no`, executeu `sudo systemctl restart systemd-resolved` i després `sudo systemctl restart pihole-FTL` |
| `dig` dona *connection timed out* | Pi-hole parat o tallafocs | Comproveu `pihole status` i `sudo ufw status`; si l'UFW està actiu, obriu el DNS i el panell: `sudo ufw allow 53` i `sudo ufw allow 80/tcp` |
| El client no resol res després de canviar el DNS | Pi-hole no pot sortir a Internet | `ping 8.8.8.8` i revisar el *upstream* a **Settings → DNS** |
| El navegador mostra anuncis tot i el bloqueig | Caché del navegador, DoH activat o anunci servit des del mateix domini | Buideu la caché, desactiveu el DNS segur del navegador i comproveu el Query Log |
| No puc entrar al panell | URL o contrasenya incorrectes | Feu servir `http://localhost/admin` i, si cal, `sudo pihole setpassword` |
| `resolvectl status` no mostra 127.0.0.1 com a DNS | Error al YAML, nom de la interfície incorrecte o fitxer sense permisos 600 | Reviseu `/etc/netplan/99-pihole.yaml` (espais, nom de la interfície amb `ip a`), feu `sudo chmod 600` i `sudo netplan apply` |
| Una pàgina legítima no carrega | Fals positiu de la llista de bloqueig | Mireu el Query Log, localitzeu el domini bloquejat i creeu una exclusió |
| `sqlite3: command not found` | Paquet no instal·lat | `sudo apt install -y sqlite3` |

Per buscar errors al registre del servei:

```bash
sudo journalctl -u pihole-FTL --no-pager | tail -n 30
```

Si la xarxa ha quedat inutilitzable, restabliu temporalment un DNS públic a Netplan (1.1.1.1) per poder seguir treballant i torneu-hi quan Pi-hole funcioni.

## Preguntes de reflexió, lliurament i avaluació

La nota es calcula sobre 10 punts a partir de les evidències i de les respostes a aquestes preguntes.

**Preguntes de reflexió** (responeu amb les vostres paraules, 3 o 4 línies cadascuna)

1. Quin paper fa Pi-hole en una consulta DNS? Explica la diferència entre consulta recursiva i iterativa i digues quin tipus fa el client a Pi-hole i quin fa Pi-hole als servidors externs.
2. Què és un *forwarder* i quin és el de la teva instal·lació?
3. Què retorna Pi-hole per a un domini bloquejat i per què no es veu l'anunci al navegador?
4. Per què la segona consulta al mateix domini és més ràpida? Relaciona-ho amb la caché i el TTL.
5. Per què un navegador amb DNS sobre HTTPS (DoH) pot saltar-se el bloqueig de Pi-hole?
6. Quins riscos té afegir exclusions sense criteri? Quin tipus de registre afegiries a una zona DNS pròpia perquè `pi.hole` o `pihole.classe.local` apuntés a aquest equip?
7. Si Pi-hole deixa de funcionar, què passa amb la navegació dels equips que només l'usen a ell com a DNS? Com es podria evitar?

**Què cal lliurar**

| Fitxer | Contingut |
| --- | --- |
| `informe_pihole_cognom.pdf` | Les 11 captures numerades, la taula de resultats de la Part 4 i les respostes a les preguntes de reflexió |
| `exclusions_cognom.csv` | Llistat d'exclusions extret amb `sqlite3` |
| `teleporter_cognom.zip` | Còpia de seguretat exportada des del panell |

**Rúbrica**

| Criteri | Punts |
| --- | --- |
| Sistema preparat: IP estàtica i comprovació del port 53 (Part 1) | 1 |
| Pi-hole instal·lat i funcionant, amb accés al panell (Part 2) | 2 |
| Client configurat per usar Pi-hole com a DNS (Part 3) | 1 |
| Comprovació del bloqueig amb `dig`, navegador i Query Log, i taula completada (Part 4) | 2 |
| Quatre exclusions amb comentari, i comprovades (Part 5) | 1 |
| Llistat d'exclusions extret en CSV i còpia de Teleporter (Part 6) | 1 |
| Resposta raonada a les preguntes de reflexió | 2 |
| **Total** | **10** |
