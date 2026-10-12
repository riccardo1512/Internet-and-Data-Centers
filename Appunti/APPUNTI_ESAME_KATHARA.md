# APPUNTI E COMANDI ESAME - KATHARÁ & FRROUTING

Guida rapida ai comandi di terminale, debugging e verifica estratti direttamente dalle slide di laboratorio (Computer Networks Research Group - Roma Tre University).

---

## 1. COMANDI KATHARÁ (CLI)

Da lanciare nel terminale Linux all'interno della cartella del lab:

```bash
# Avviare tutti i nodi del lab
kathara lstart

# Fermare e ripulire il lab (rimuove container e collision domain)
kathara lclean

# Riavviare il lab (lclean + lstart)
kathara lrestart

# Entrare nella shell di una macchina specifica
kathara connect r1

# Elencare i lab Kathará attualmente in esecuzione
kathara list

# SALVA-VITA: Rimuove forzatamente TUTTI i container e le reti Docker/Kathara rimaste bloccate
kathara wipe -a
```

---

## 2. COMANDI DI SISTEMA (LINUX SHELL DEI NODI)

Comandi da inserire negli script `.startup` o da digitare direttamente nella shell di un router/nodo:

### 2.1 Indirizzi IP e Interfacce
```bash
# Assegnare un indirizzo IP a una scheda di rete (usato nei .startup)
ip address add 100.1.0.1/30 dev eth0
ip address add 10.0.0.1/24 dev eth1

# Mostrare gli indirizzi IP e lo stato delle interfacce
ip a
# oppure:
ip address show
```

### 2.2 Routing del Kernel Linux
```bash
# Mostrare la tabella di routing del kernel
ip route

# Aggiungere una rotta statica (es. su router esterno verso la rete interna)
ip route add 100.1.0.0/16 via 100.2.0.1

# Aggiungere la default route statica (es. su border router verso Internet)
ip route add default via 100.2.0.2

# Eliminare una rotta statica
ip route del 100.1.0.0/16 via 100.2.0.1
```

### 2.3 Simulazione Guasti Link (Slide Style)
```bash
# Spegnere un'interfaccia (simula guasto del link o caduta di un DR)
ip link set eth0 down
# (oppure forma classica):
ifconfig eth0 down

# Riaccendere l'interfaccia
ip link set eth0 up
# (oppure forma classica):
ifconfig eth0 up
```

### 2.4 Test di Connettività e Traceroute
```bash
# Ping (test connettività di base con 3 pacchetti)
ping -c 3 100.1.2.1

# Traceroute numerico (senza risoluzione DNS per massima velocità)
traceroute -n 100.1.2.1
```

### 2.5 Packet Sniffing con `tcpdump` (Slide Style)
```bash
# Sniffing standard leggibile su un'interfaccia
tcpdump -tenni eth0

# Sniffing dettagliato con decodifica dei protocolli
tcpdump -tenni eth0 -v

# Filtrare solo pacchetti OSPF (protocollo IP 89)
tcpdump -tenni eth0 proto 89

# Filtrare solo pacchetti RIP (porta UDP 520)
tcpdump -tenni eth0 udp port 520
```

### 2.6 Demone di Routing FRR
```bash
# Avviare il demone FRRouting (obbligatorio in ogni .startup dei router FRR)
systemctl start frr

# Verificare che il demone sia attivo e funzionante
systemctl status frr
```

---

## 3. MODALITÀ DI UTILIZZO DI `vtysh`

`vtysh` è la shell unificata di FRRouting (Cisco-like):

```bash
# Modalità interattiva (entra nella shell di FRR)
vtysh

# Eseguire un singolo comando da terminale Linux senza entrare (VELOCISSIMO)
vtysh -c "show ip route"
vtysh -c "show ip ospf neighbor"

# Eseguire comando con filtraggio grep da bash (Slide Style)
vtysh -e "show ip ospf interface" | egrep "eth|Cost"
```

---

## 4. DEBUGGING RIP (DENTRO `vtysh`)

Da eseguire dentro `vtysh` (o con `vtysh -c "<comando>"`):

```text
! Mostrare la tabella di routing interna RIP (metriche, next-hop, codici)
show ip rip

! Mostrare lo stato del protocollo RIP (timer di update/timeout, interfacce attive, vicini)
show ip rip status

! Mostrare la tabella di routing di Zebra / Kernel (rotte RIP indicate con 'R')
show ip route
```

*Codici tipici in `show ip rip`:*
* `R(n)`: Rotta normale appresa via RIP.
* `C`: Rete direttamente connessa.
* `S`: Rotta statica.
* Metrica 16: Rete irraggiungibile (Infinity).

---

## 5. DEBUGGING OSPF (DENTRO `vtysh`)

### 5.1 Informazioni Generali e Router-ID
```text
! Mostrare Router-ID, stato generale del processo OSPF, aree a cui appartiene il router
show ip ospf
```

### 5.2 Interfacce e Costi
```text
! Dettagli completi di tutte le interfacce OSPF (Cost, MTU, stato DR/BDR, timer)
show ip ospf interface

! Dettagli di una specifica interfaccia (es. eth0)
show ip ospf interface eth0

! FILTRO RAPIDO (Slide Style): Mostra solo le interfacce e i rispettivi costi
! (da bash):
vtysh -e "show ip ospf interface" | egrep "eth|Cost"
```

### 5.3 Vicini e Stati di Adiacenza
```text
! Mostrare la lista dei router vicini (Neighbor ID, Priorità, Stato, Indirizzo, Interfaccia)
show ip ospf neighbor

! Dettagli estesi su ciascun vicino
show ip ospf neighbor detail
```

*Interpretazione degli stati:*
* `Full/DR`: Adiacenza completa con il Designated Router.
* `Full/Backup`: Adiacenza completa con il Backup Designated Router.
* `2-Way/DROther`: Relazione tra due router ordinari su rete broadcast (non scambiano LSDB completo tra loro, solo con DR/BDR).

### 5.4 Tabelle di Routing OSPF
```text
! Rotte interne calcolate dall'algoritmo SPF (intra-area, inter-area IA, costi totali)
show ip ospf route

! Tabella di routing di Zebra/Kernel (rotte installate nel sistema)
show ip route
```

*Codici in `show ip route`:*
* `O`: Rotta intra-area (stessa area OSPF).
* `O IA`: Rotta inter-area (appresa da un'altra area tramite ABR).
* `O E1` / `O E2`: Rotta esterna (iniettata da un ASBR).

### 5.5 Link-State Database (LSDB)
```text
! Panoramica sintetica di tutti gli LSA memorizzati nell'LSDB per ciascuna area
show ip ospf database

! LSA TIPO 1 (Router Link States): Generati da OGNI router per descrivere i propri link nell'area
show ip ospf database router

! LSA TIPO 2 (Net Link States): Generati solo dai DR per descrivere i router connessi alla LAN broadcast
show ip ospf database network

! LSA TIPO 3 (Summary Link States): Generati dagli ABR per descrivere le rotte verso altre aree e la default route (0.0.0.0/0) nelle Stub Area
show ip ospf database summary

! LSA TIPO 4 (ASBR-Summary Link States): Generati dagli ABR per descrivere la rotta verso un ASBR
show ip ospf database asbr-summary

! LSA TIPO 5 (AS-External Link States): Generati dagli ASBR per rotte esterne (NON presenti nelle Stub Area)
show ip ospf database external
```

---

## 6. CHEAT SHEET DOMANDE D'ESAME (`answers.txt`)

### Domanda 1: *"Which is the router-id of router X?"*
```bash
# Da lanciare nella shell del router X:
vtysh -c "show ip ospf" | grep "Router ID"
```
*Regola teorica di elezione del Router-ID:*
1. Valore esplicito configurato (`router-id X.X.X.X`), se presente.
2. L'indirizzo IP più alto assegnato a un'interfaccia di loopback (`lo`).
3. L'indirizzo IP più alto assegnato a una qualunque interfaccia fisica attiva (`ethX`).

---

### Domanda 2: *"Which is the designated router of LAN X?"*
```bash
# Da lanciare su uno qualunque dei router connessi a quella LAN (es. su eth0):
vtysh -c "show ip ospf interface eth0" | grep -A 2 "Designated Router"
```
*Regola teorica di elezione del DR:*
1. Vince il router con la **priorità OSPF più alta** (default: 1; se impostata a 0 il router non può diventare DR).
2. A parità di priorità, vince il router con il **Router-ID più alto**.

---

### Domanda 3: *"How many 0/0 (default route) entries can be found in the zebra routing table of router X?"*
```bash
# Da lanciare nella shell del router X:
vtysh -c "show ip route" | grep -E "0\.0\.0\.0/0|default"
# oppure da bash:
ip route | grep default
```
*Regola teorica:*
* **Router interno a una Stub Area**: riceve la default route generata dall'ABR tramite LSA Tipo 3 ($\rightarrow$ conta quante righe `via ...` compaiono; tipicamente **1**, o di più se ci sono ABR multipli ridondati).
* **Router nel Backbone (Area 0.0.0.0)**: di norma ha **0** entry (il Backbone non riceve la default route generata per le stub aree).

---

## 7. SINTASSI RAPIDA DEI FILE DI CONFIGURAZIONE (`frr.conf`)

Promemoria rapido delle parole chiave da usare nei file `/etc/frr/frr.conf`:

### 7.1 Configurazione RIP
```text
! /etc/frr/frr.conf per RIP
router rip
    ! Annunciare tutte le interfacce che ricadono nel prefisso
    network 100.1.0.0/16
    ! (Opzionale sul border router): propagare la default route ai vicini
    route 0.0.0.0/0
    ! (Opzionale sul border router): redistribuire le reti connesse
    redistribute connected
log file /var/log/frr/frr.log
```

### 7.2 Configurazione OSPF (Backbone & Costi)
```text
! /etc/frr/frr.conf per OSPF Single-Area con costi
interface eth0
    ospf cost 10
interface eth1
    ospf cost 50

router ospf
    ! Abilitare OSPF su tutte le interfacce con IP in questo prefisso
    network 10.0.0.0/16 area 0.0.0.0
log file /var/log/frr/frr.log
```

### 7.3 Configurazione OSPF ABR (Backbone + Stub Area)
```text
! /etc/frr/frr.conf per router ABR
router ospf
    ! Interfacce nel Backbone
    network 10.0.0.0/16 area 0.0.0.0
    ! Interfacce nella Stub Area 1.1.1.1
    network 100.0.0.0/30 area 1.1.1.1
    ! Dichiarare l'area come Stub
    area 1.1.1.1 stub
log file /var/log/frr/frr.log
```

### 7.4 Configurazione OSPF Router Interno Stub
```text
! /etc/frr/frr.conf per router interno alla Stub
router ospf
    network 100.0.0.0/30 area 1.1.1.1
    network 200.0.0.0/24 area 1.1.1.1
    ! Fondamentale: deve essere dichiarato stub su TUTTI i router dell'area!
    area 1.1.1.1 stub
log file /var/log/frr/frr.log
```

---

## 8. CONFIGURAZIONE AL VOLO DA `vtysh` (SENZA RIAVVIARE)

Se durante l'esame vuoi modificare una configurazione senza riavviare il lab:

```text
vtysh
# configure terminal
(config)# interface eth1
(config-if)# ip ospf cost 45
(config-if)# exit
(config)# router ospf
(config-router)# area 2.2.2.2 stub
(config-router)# exit
(config)# exit
# write
```
*(Il comando `write` salva la configurazione corrente direttamente nel file `frr.conf`).*

---

## 9. CONSEGNA DEL LAB

Da lanciare dal terminale nella cartella superiore (fuori dalla cartella del compito d'esame):

```bash
# Archivio compresso tar.gz (standard di consegna)
tar -czvf cognome_matricola.tar.gz cognome_matricola

# Oppure formato zip (se richiesto)
zip -r cognome_matricola.zip cognome_matricola
```

