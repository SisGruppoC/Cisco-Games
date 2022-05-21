# Cisco Game

## Lo scenario

Lo scenario rappresentato è quello di una piccola media azienda che ha una propria rete interna e necessita di esporre all'esterno servizi web e mail.

### ISP

Per completezza, è stata poi riportata anche la sezione relativa all'ISP e ad alcuni ipotetici utenti generici che possono quindi accedere ai servizi esposti dall'azienda.

Per semplificare, le architetture relative all'ISP sono state rappresentate con un router, a cui sono connesse l'azienda e le abitazioni domestiche e da un server DNS che sovraintende quindi le richieste di traduzione di domini in indirizzi IPv4.

### Reti client

Fatta eccezione per la rete del Client3, tutte le reti delle abitazioni domestiche hanno un router che:

- espone un solo indirizzo IP pubblico (tramite NAT);
- assegna gli indirizzi IP per gli host tramite **DHCP** in un unico pool (`192.168.1.0 /24`);
- redirige tutto il traffico verso reti ad esso non dirittamente collegate al router dell'ISP tramite un a rotta statica.

La rete del Client3 fa eccezione solamente per il fatto che al suo interno vi sono 2 reti locali:

- `192.168.1.0 /24`
- `192.168.2.0 /24`

e gli host di queste reti comunicano fra di loro grazie alle rotte generate dinamicamente grazie al protocollo **RIP**.

Gli indirizzi pubblici delle varie reti fanno parte del blocco di indirizzi `80.0.0.0` e hanno come maschera di sotto-rete `255.255.255.252`.

![Rete dei client](https://i.ibb.co/84ZPnFZ/Screenshot-2022-05-20-at-16-47-25.png)

### Configurazione rete aziendale

L'azienda ottiene 5 indirizzi IP pubblici dall'ISP:

- `80.0.0.18` per la traduzione tramite NAT degli indirizzi IP locali in indirizzi pubblici;
- `80.0.0.22`, `80.0.0.26`, `80.0.0.30`, per i web-server;
- `80.0.0.34`, per il mail-server.

Questi server, in quanto esposti alla rete esterna, fanno parte di una DMZ.

Nella rete locale, invece, si distinguono 4 sezioni rispettivamente dedicate a:

- **Amministrazione** (`192.168.1.64 /26`);
- **Utenti generici** (`192.168.1.0 /26`);
- **Sala server** (`192.168.1.128 /29`);
- **Sala server FTP** (`192.168.1.136 /29`), a cui si accede con appositi criteri;

![Rete aziendale](https://i.ibb.co/410gXZK/Screenshot-2022-05-20-at-19-10-11.png)


## Subnet

### Sotto-reti a maschera fissa

L'ISP effettua il subnetting a maschera fissa del blocco di indirizzi IP che ha a disposizione.

Per evitare lo spreco di indirizzi, l'ISP assegna ai client indirizzi IP con maschera `/30` così da avere per ogni sottorete solamente 4 indirizzi (di cui soltanto 2 sono utilizzabili).

### Sotto-reti a maschera variabile (sotto-reti aziendali)

All'interno della rete aziendale si ricorre al subnetting a maschera variabile per creare sottoreti di dimensione congrua a quella necessaria.

In particolare:

- Le reti relative alla **sala server** e alla **sala server FTP** hanno come subnet `255.255.255.248`;
- Le reti degli **utenti generici** e dell'**amministrazione** hanno, invece, come subnet `255.255.255.192`.

## Routing

### Rotte dinamiche interne alla rete aziendale - OSPF
Il routing della rete locale aziendale è stato configurato tramite l'uso del protocollo OSPF.

Seguono le configurazioni dei router:

- **RouterFTP**
	<br>
	`RouterFTP (config) # router ospf 1`
	<br>
	`RouterFTP (config-router) # network 192.168.1.136 0.0.0.7 area 0`
	<br>
	`RouterFTP (config-router) # network 12.0.0.0 0.0.0.3 area 0`


- **RouterServer**
	<br>
	`RouterServer (config) # router ospf 1`
	<br>
	`RouterServer (config-router) # network 192.168.1.128 0.0.0.7 area 0`
	<br>
	`RouterServer (config-router) # network 11.0.0.0 0.0.0.3 area 0`
	<br>
	`RouterServer (config-router) # network 12.0.0.0 0.0.0.3 area 0`
	
- **RouterCentrale**
	<br>
	`RouterCentrale (config) # router ospf 1`
	<br>
	`RouterCentrale (config-router) # network 11.0.0.0 0.0.0.3 area 0`
	<br>
	`RouterCentrale (config-router) # network 11.0.0.4 0.0.0.3 area 0`
	<br>
	`RouterCentrale (config-router) # network 11.0.0.8 0.0.0.3 area 0`
	<br>
	`RouterCentrale (config-router) # network 11.0.0.12 0.0.0.3 area 0`

- **RouterAmministrazione**
	<br>
	`RouterAmministrazione (config) # router ospf 1`
	<br>
	`RouterAmministrazione (config-router) # network 11.0.0.4 0.0.0.3 area 0`
	<br>
	`RouterAmministrazione (config-router) # network 192.168.1.0 0.0.0.63 area 0`
	
- **RouterUtenti**
	<br>
	`RouterUtenti (config) # router ospf 1`
	<br>
	`RouterUtenti (config-router) # network 11.0.0.8 0.0.0.3 area 0`
	<br>
	`RouterUtenti (config-router) # network 192.168.1.64 0.0.0.63 area 0`
	
- **RouterPrincipale**
	<br>
	`RouterPrincipale (config) # router ospf 1`
	<br>
	`RouterPrincipale (config-router) # network 192.168.1.136 0.0.0.7 area 0`
	<br>
	`RouterPrincipale (config-router) # network 11.0.0.12 0.0.0.3 area 0`
	<br>
	`RouterPrincipale (config-router) # network 80.0.0.0 0.255.255.255 area 0` &nbsp; (Supernetting)

### Rotte dinamiche nella rete Client 3 - RIP

La rotte delle reti locali del Client 3 sono state configurate tramite il protocollo RIP eseguendo i seguenti comandi:

- **Router principale**:
	<br>
	`Router (config) # router rip`
	<br>
	`Router (config-router) # network 10.0.0.0`
	<br>
	`Router (config-router) # network 192.168.1.0`

- **Router secondario**:
	<br>
	`Router2 (config) # router rip`
	<br>
	`Router2 (config-router) # network 10.0.0.0`
	<br>
	`Router2 (config-router) # network 192.168.2.0`


## Configurazione dei servizi

### DHCP aziendale

Il server DHCP interno all'azienda ha come indirizzo `192.168.1.131 /29`.

Al suo interno sono configurati 2 POOL nel seguente modo:

| Pool name | Default gateway | DNS server | Start IP address | Subnet Mask | Max user |
| --- | --- | --- | --- | --- | --- |
| ReteA | 192.168.1.1 | 192.168.1.134 | 192.168.1.2 | 255.255.255.192 | 61 |
| ReteB | 192.168.1.65 | 192.168.1.134 | 192.168.1.66 | 255.255.255.192 | 61 |


Non sono necessari utleriori POOL per i server perché questi hanno indirizzi IP statici.

Sui router relativi alle reti di **amministrazione** e **utenti generici** è stato eseguito il seguente comando sull'interfaccia GigabitEthernet 0/0:
`ip helper-address 192.168.1.131`



### DNS aziendale

Il server DNS interno all'azienda ha come indirizzo `192.168.1.134/29` e al suo interno sono stati configurati i seguenti record:

| Name | Type | Detail |
| --- | --- | --- |
| smtp.robertone.it | A Record | 80.0.0.34 |
| pop.robertone.it | A Record | 80.0.0.34 |
| robertone.it | A Record | 80.0.0.22 |
| robertone.it | A Record | 80.0.0.26 |
| robertone.it | A Record | 80.0.0.30 |


### Server FTP

Il server ftp ha indirizzo IP statico `192.168.1.138 /29`. L'accesso a quest'ultimo è consentito solo dalla rete dell'amministrazione tramite VPN e ACL come descritto nei punti seguenti.

| Username | Password | Permission |
| --- | --- | --- |
| cisco | cisco | RWDNL |
| leoncino | pippo | RWDNL |
| pizzamargherita | pippo | RWDNL |
| santapazienza | pippo | RWDNL |

### Servizi e-mail

Il server mail ha indirizzo IP statico `80.0.0.34 /30` e offre servizi SMTP e POP3.
Sono state configurati le seguenti mail:

| Username | Password | E-Mail |
| --- | --- | --- |
| santapazienza | pippo | santapazienza@robertone.it |
| leoncino | pippo | leoncino@robertone.it |
| pizzamargherita | pippo | pizzamargherita@robertone.it |
| lasagnaemiliana | pippo | lasagnaemiliana@robertone.it |
| francoforte | pippo | francoforte@robertone.it |
| remolabarca | pippo | remolabarca@robertone.it |

Per configurare una mail sul computer, è necessario aprire l'applicazione, cliccare su configure mail e successivamente inserire il proprio nome, il relativo indirizzo email. Nei campi incoming mail server `pop.robertone.it` e nell'outgoing `smtp.robertone.it`


## NAT

### Scopo e funzionamento del NAT

Il NAT è un protocollo che sta alla base del funzionamento di internet e permette di tradurre degli indirizzi locali in indirizzi globali e viceversa.

Ad esempio, ogni rete domestica dispone di più indirizzi locali (almeno uno per ogni host), ma un solo indirizzo pubblico.

L'uso del NAT inevitabilmente porta con sé il problema relativo a quale host (con indirizzo IP privato) recapitare un pacchetto indirizzato ad un determinato indirizzo pubblico.
Ciò viene gestito tramite delle politiche definite nel router di destinazione.

### Configurazione del NAT

Sui router dei client (quello aziendale non fa eccezzione) il NAT è stato impostato nel seguente modo: quando un host (interno) invia un pacchetto a degli host esterni, l'indirizzo IP sorgente verrà sostituito con l'indirizzo IP pubblico assegnato dall'ISP.
Così facendo, il router tiene traccia della trasformazione dell'IP ed è in grado di smistare i pacchetti di ritorno all'host interno che ha iniziato la comunicazione tramite la propria NAT Table.

Non è invece possibile per un host esterno alla rete contattare un host interno ad una rete che abbia configurato il NAT.

La configurazione del NAT sui router è la seguente:

`Router(config) # ip nat inside source list 1 interface ethernet 2/0 override`

`Router(config) # interface Ethernet 1/0`
<br>
`Router(config-if) # ip nat inside`
<br>
`Router(config-if) # exit`
<br>
`Router(config) # interface Ethernet 2/0`
<br>
`Router(config-if) # ip nat outside`
<br>
`Router(config-if) # exit`
<br>
`Router(config) # interface ethernet 0/0`
<br>
`Router(config-if) # ip nat inside`


## Server WEB distribuito

### Funzionamento tramite DNS

Il server Web distribuito è stato configurato impostando 3 record dns con il medesimo dominio, ma con traduzioni in ip differenti.
Così facendo il server DNS risponde alle query DNS alternando i record secondo la politica di round-robin.

### Funzionamento load balancer tramite NAT (non supportato in PKT)

Sebbene Cisco Packet Tracer, essendo un simulatore, non supporti questa funzionalità, disponendo dei router fisici sarebbe stato possibili utilizzare politiche di NAT dinamico per tradurre l'indirizzo IP pubblico in indirizzi IP privati differenti distribuendo quindi il carico sui vari cloni.

I comandi che permettono di effettuare l'operazione in questione sono i seguenti.

`Router(config) # ip nat pool POOL <primo-ip> <ultimo-ip> netmask <subnetmask> type rotary`
<br>
`ip nat outside destination <indirizzo-pubblico> pool POOL`

### Configurazione dei cloni

I 3 cloni permettono la fruizione di una pagina web statica, raffigurata nella schermata sottostante.

![](https://i.ibb.co/Y7vPY1y/Screenshot-2022-05-20-at-20-42-17.png)


## Firewall

### Configurazione ACL standard

È stata configurata una ACL sulla porta GigabitEthernet 0/1 del router RouterFTP per regolare l'accesso al server FTP.
In particolar modo, si stabilisce che vi possano accedere solamente gli host appartenenti alla rete relativa all'amministrazione.
`RouterFTP (config) # access-list 9 permit 192.168.1.0 0.0.0.63`
<br>
`RouterFTP (config) # interface GigabitEthernet0/1`
<br>
`RouterFTP (config-if) # ip access-group 9 in`


### Configurazione ACL estese

Per regolare poi l'accesso dall'esterno ai server mail e web è stata impostata un'ACL che accetta solamente il traffico (dell'appropriato protocollo) rivolto verso i server della DMZ.

`RouterPrincipale (config) # ip access-list extended SoloSRV`
<br>
`RouterPrincipale (config-ext-nacl) # permit tcp any host 80.0.0.34 eq pop3`
<br>
`RouterPrincipale (config-ext-nacl) # permit tcp any host 80.0.0.34 eq smtp`
<br>
`RouterPrincipale (config-ext-nacl) # permit tcp any host 80.0.0.24 eq www`
<br>
`RouterPrincipale (config-ext-nacl) # permit tcp any host 80.0.0.26 eq www`
<br>
`RouterPrincipale (config-ext-nacl) # permit tcp any host 80.0.0.26 eq www`
<br>
`RouterPrincipale (config-ext-nacl) # permit tcp any host 80.0.0.30 eq www`

## VPN (Virtual Private Network)

### Cos'è una VPN

Una VPN è una connessione privata tra dispositivi che viene creata avvalendosi di un canale di rete insicuro.

Il loro scopo è quello di offrire mantenere private le informazioni trasmesse e nascondere l'indirizzo IP.


### Come funziona il Protocollo IPSEC

> IPsec è una suite di protocolli che autentica e cripta i pacchetti per fornire una comunicazione crittografata sicura tra due computer su una rete di protocollo Internet.

Questo è composto da:

* protocolli che implementano lo scambio delle chiavi per la realizzazione del flusso crittografato: IKE
* protocolli che forniscono la cifratura del flusso di dati.<br>Questo è a sua volta composto da:
	* Authentication Header (AH)
	* Encapsulating Security Payload (ESP)

IPSec prevede due modalità di funzionamento:

* **Transport mode**;
* **Tunnel mode** (utilizzata nel progetto): per le connessioni gateway to gateway. I dati vengono cifrati dal gateway del mittente per poi essere de-cifrati dal gateway del destinatario.

### Impiego VPN all'interno del progetto

Ci si avvale di una VPN per connettersi al server FTP in quanto il protocollo FTP trasmette i dati in chiaro e si suppone che il server gestisca file sensibili.

### Comandi Configurazione

1. Configurazione della security license<br>`Router(config)#license boot module c1900 technology-package securityk9` dopo di che salvare la running config e riavviare<br>`Router#copy running-config startup-config`<br>`Router#reload`

- **Router FTP**

	`RouterFTP(config) # crypto isakmp policy 10`<br>
	`RouterFTP(config-isakmp) # encryption aes 256`<br>
	`RouterFTP(config-isakmp) # authentication pre-share`<br>
	`RouterFTP(config-isakmp) # group 5`<br>
	
	`RouterFTP(config-isakmp) # crypto isakmp key secretkey address 11.0.0.6` dove 11.0.0.6 è l'ip dell'interfaccia in uscita della rete dove sono presenti gli host che dovranno connettersi al server
	
	`RouterFTP(config) # crypto ipsec transform-set R1-R3 esp-aes 256 esp-sha-hmac`
	
	`RouterFTP(config) # crypto map IPSEC-MAP 10 ipsec-isakmpcrypto map IPSEC-MAP 10 ipsec-isakmp`&nbsp a seguito di questo comando potrebbe essere visualizzata una nota dove si indica che la criptazione sarà disabilitata fino alla configurazione del peer che verrà svolta in seguito<br>
	`RouterFTP(config-crypto-map) # set peer 11.0.0.6`<br> 
	`RouterFTP(config-crypto-map) # set pfs group5`<br>
	`RouterFTP(config-crypto-map) # set security-association lifetime seconds 86400`<br>
	`RouterFTP(config-crypto-map) # set transform-set R1-R3`<br>
	`RouterFTP(config-crypto-map) # match address 100`<br>
	
	`RouterFTP(config) # interface GigabitEthernet0/1` interfaccia di rete con cui è collegato il router al resto dell'infrastruttura di rete
	`RouterFTP(config-if) # crypto map IPSEC-MAP`
	
	`RouterFTP(config) # access-list 100 permit ip 192.168.1.136 0.0.0.3 192.168.1.0 0.0.0.63` ip della rete connessa al server con l'inverso della sua maschera di rete seguita dalla rete esterna con l'inverso della relativa maschera

Di seguito i comandi per la configurazione del Router 3 che differiscono dai precedenti solamente per gli indirizzi IP.


- **Router Amministrazione**

	`RouterAmministrazione(config) # crypto isakmp policy 10`<br>
	`RouterAmministrazione(config-isakmp) # encryption aes 256`<br>
	`RouterAmministrazione(config-isakmp) # authentication pre-share`<br>
	`RouterAmministrazione(config-isakmp) # group 5`<br>
	
	`RouterAmministrazione(config-isakmp) # crypto isakmp key secretkey address 12.0.0.1` dove 12.0.0.1 è l'ip dell'interfaccia in uscita della rete dove sono presenti gli host che dovranno connettersi al server
	
	`RouterAmministrazione(config) # crypto ipsec transform-set R1-R3 esp-aes 256 esp-sha-hmac`
	
	`RouterAmministrazione(config) # crypto map IPSEC-MAP 10 ipsec-isakmpcrypto map IPSEC-MAP 10 ipsec-isakmp`&nbsp a seguito di questo comando potrebbe essere visualizzata una nota dove si indica che la criptazione sarà disabilitata fino alla configurazione del peer che verrà svolta in seguito<br>
	`RouterAmministrazione(config-crypto-map) # set peer 11.0.0.6`<br> 
	`RouterAmministrazione(config-crypto-map) # set pfs group5`<br>
	`RouterAmministrazione(config-crypto-map) # set security-association lifetime seconds 86400`<br>
	`RouterAmministrazione(config-crypto-map) # set transform-set R1-R3`<br>
	`RouterAmministrazione(config-crypto-map) # match address 100`<br>
	
	`RouterAmministrazione(config) # interface GigabitEthernet0/1` interfaccia di rete con cui è collegato il router al resto dell'infrastruttura di rete
	`RouterAmministrazione(config-if) # crypto map IPSEC-MAP`
	
	`RouterAmministrazione(config) # access-list 100 permit ip 192.168.1.0 0.0.0.63 192.168.1.136 0.0.0.7` ip della rete connessa al router con l'inverso della sua maschera di rete seguita dalla rete esterna con l'inverso della relativa maschera
