# SMTP Enumeration & Interaction

## 1. Cos'è SMTP

**SMTP (Simple Mail Transfer Protocol)** è il protocollo utilizzato per la trasmissione delle email.

La porta TCP più comune è:

```text
25/tcp
```

Altre porte comunemente associate alla posta:

```text
25    SMTP
465   SMTPS
587   SMTP Submission
```

Durante un penetration test, un server SMTP esposto può fornire informazioni utili sugli utenti e sulla configurazione del sistema.

Una delle attività più interessanti è la **user enumeration**, cioè verificare se determinati username esistono sul server.

---

# 2. Individuare SMTP con Nmap

Per verificare se la porta 25 è aperta:

```bash
nmap -p25 <IP>
```

Esempio:

```bash
nmap -p25 10.129.231.210
```

Output:

```text
PORT   STATE SERVICE
25/tcp open  smtp
```

Per ottenere maggiori informazioni:

```bash
sudo nmap -p25 -sC -sV 10.129.231.210
```

Possiamo quindi identificare:

* stato della porta
* servizio
* versione del server SMTP
* eventuali informazioni ottenibili tramite gli script NSE

---

# 3. Collegarsi manualmente con Telnet

Un modo molto utile per capire come funziona SMTP è collegarsi direttamente al server.

```bash
telnet <IP> 25
```

Esempio:

```bash
telnet 10.129.231.210 25
```

Se la connessione è riuscita, il server può rispondere con un banner simile a:

```text
220 mail.example.com ESMTP
```

Il codice `220` indica che il server è pronto a ricevere comandi.

---

# 4. EHLO / HELO

Dopo la connessione possiamo identificarci con:

```text
EHLO example.com
```

oppure:

```text
HELO example.com
```

È preferibile usare `EHLO`, perché permette di utilizzare le estensioni SMTP.

Esempio:

```text
EHLO example.com
```

Il server potrebbe rispondere:

```text
250-mail.example.com
250-SIZE 52428800
250-8BITMIME
250-PIPELINING
250-AUTH LOGIN
```

Queste informazioni possono essere utili per capire quali funzionalità supporta il server.

---

# 5. VRFY: verificare un utente

Il comando SMTP `VRFY` può essere utilizzato per chiedere al server se un determinato utente esiste.

Sintassi:

```text
VRFY username
```

Esempio:

```text
VRFY root
```

Una risposta positiva può indicare che l'utente esiste:

```text
252 2.0.0 root
```

Una risposta negativa può essere:

```text
550 5.1.1 User unknown
```

### Importante

Non tutti i server SMTP permettono `VRFY`.

Per esempio, il server potrebbe rispondere negativamente anche quando l'utente esiste, oppure potrebbe disabilitare completamente il comando.

Per questo motivo **una risposta negativa a VRFY non dimostra necessariamente che l'utente non esista**.

---

# 6. EXPN

Un altro comando utilizzabile per l'enumerazione è:

```text
EXPN username
```

`EXPN` nasce per espandere una mailing list o un alias.

Su alcuni server può però fornire informazioni sugli utenti.

Esempio:

```text
EXPN admin
```

Anche `EXPN`, come `VRFY`, può essere disabilitato dal server.

---

# 7. RCPT TO

Un'altra tecnica consiste nell'utilizzare:

```text
RCPT TO:<utente@dominio>
```

dopo aver iniziato una transazione SMTP.

Esempio:

```text
MAIL FROM:<test@example.com>
RCPT TO:<root@example.com>
```

Il server potrebbe rispondere diversamente a seconda che il destinatario esista oppure no.

Per esempio:

```text
250 2.1.5 OK
```

può indicare che il destinatario è accettato.

Una risposta come:

```text
550 5.1.1 User unknown
```

può invece indicare che il destinatario non è valido.

Anche questo metodo dipende dalla configurazione del server.

---

# 8. Creare una email manualmente da Telnet

Possiamo anche utilizzare Telnet per simulare una vera transazione SMTP.

Collegarsi:

```bash
telnet <IP> 25
```

Poi:

```text
EHLO example.com
```

Specificare il mittente:

```text
MAIL FROM:<attacker@example.com>
```

Specificare il destinatario:

```text
RCPT TO:<user@example.com>
```

Iniziare il contenuto del messaggio:

```text
DATA
```

A questo punto inseriamo gli header:

```text
From: attacker@example.com
To: user@example.com
Subject: Test SMTP
```

Lasciamo una riga vuota e scriviamo il corpo:

```text
This is a test email.
```

Per terminare il messaggio bisogna inserire un punto `.` su una riga da solo:

```text
.
```

La sequenza completa è:

```text
EHLO example.com
MAIL FROM:<attacker@example.com>
RCPT TO:<user@example.com>
DATA
From: attacker@example.com
To: user@example.com
Subject: Test SMTP

This is a test email.
.
```

Il server potrebbe rispondere:

```text
250 2.0.0 OK
```

A quel punto la transazione è terminata.

Per chiudere la connessione:

```text
QUIT
```

---

# 9. Enumerazione con Nmap

Nmap dispone dello script NSE:

```text
smtp-enum-users
```

Possiamo eseguirlo con:

```bash
sudo nmap -p25 --script smtp-enum-users <IP>
```

Esempio:

```bash
sudo nmap -p25 --script smtp-enum-users 10.129.231.210
```

Lo script prova a enumerare gli utenti attraverso:

```text
VRFY
EXPN
RCPT TO
```

Nella versione di Nmap utilizzata durante il laboratorio, l'ordine predefinito è:

```text
RCPT
VRFY
EXPN
```

Possiamo modificare l'ordine:

```bash
sudo nmap -p25 \
--script smtp-enum-users \
--script-args 'smtp-enum-users.methods={VRFY,EXPN,RCPT}' \
10.129.231.210
```

Possiamo anche utilizzare soltanto VRFY:

```bash
sudo nmap -p25 \
--script smtp-enum-users \
--script-args 'smtp-enum-users.methods={VRFY}' \
10.129.231.210
```

---

# 10. Attenzione alla wordlist con Nmap

Una cosa importante emersa durante il laboratorio è che:

```text
smtp-enum-users
```

è uno **script NSE di Nmap**, ma nella versione utilizzata non espone un'opzione `userdb` per specificare direttamente una wordlist.

Quindi questo:

```bash
--script-args smtp-enum-users.userdb=wordlist.txt
```

non è il metodo corretto per fornire una wordlist a questo script.

Lo script Nmap può quindi restituire una lista di username predefinita, ad esempio:

```text
smtp-enum-users:
  root
  admin
  administrator
  webadmin
  sysadmin
  netadmin
  guest
  user
  web
  test
```

---

# 11. smtp-user-enum

Per utilizzare una **wordlist personalizzata** possiamo invece utilizzare un programma separato:

```text
smtp-user-enum
```

È importante distinguerlo dallo script Nmap:

```text
smtp-user-enum
    ↓
programma standalone

smtp-enum-users
    ↓
script NSE di Nmap
```

Quindi non sono la stessa cosa.

---

# 12. Enumerazione con smtp-user-enum

La sintassi generale è:

```bash
smtp-user-enum -M <METHOD> -U <USERLIST> -t <TARGET>
```

Per utilizzare `VRFY`:

```bash
smtp-user-enum -M VRFY \
-U userlist.txt \
-t 10.129.231.210
```

Dove:

```text
-M VRFY
```

specifica il metodo SMTP.

```text
-U userlist.txt
```

specifica la wordlist degli username.

```text
-t 10.129.231.210
```

specifica il server SMTP target.

---

# 13. Utilizzare EXPN

Possiamo utilizzare:

```bash
smtp-user-enum -M EXPN \
-U userlist.txt \
-t 10.129.231.210
```

---

# 14. Utilizzare RCPT

Oppure:

```bash
smtp-user-enum -M RCPT \
-U userlist.txt \
-t 10.129.231.210
```

Quindi possiamo provare separatamente:

```text
VRFY
EXPN
RCPT
```

perché un metodo potrebbe essere disabilitato mentre un altro potrebbe funzionare.

---

# 15. Utilizzare una wordlist di SecLists

Prima possiamo cercare la wordlist:

```bash
find /usr/share/wordlists /usr/share/seclists \
-type f 2>/dev/null | grep -Ei 'user|username|footprinting'
```

Oppure verificare un file specifico:

```bash
ls -lh /usr/share/wordlists/seclists/Usernames/
```

Una volta individuata la wordlist:

```bash
smtp-user-enum -M VRFY \
-U /usr/share/wordlists/seclists/Usernames/<wordlist> \
-t 10.129.231.210
```

---

# 16. Timeout e numero di connessioni

` smtp-user-enum` permette di modificare alcuni parametri per server che rispondono lentamente.

Esempio:

```bash
smtp-user-enum -M VRFY \
-U userlist.txt \
-t 10.129.231.210 \
-m 60 \
-w 20
```

In questo esempio:

```text
-m 60
```

imposta il numero massimo di processi/thread concorrenti.

```text
-w 20
```

imposta il timeout.

Questi valori possono essere utili quando il server SMTP risponde lentamente.

---

# 17. Approccio consigliato durante un laboratorio HTB

Quando trovi:

```text
25/tcp open smtp
```

puoi procedere gradualmente.

### Step 1 — identificare il servizio

```bash
nmap -p25 -sC -sV <IP>
```

### Step 2 — collegarsi manualmente

```bash
telnet <IP> 25
```

### Step 3 — identificare le capacità

```text
EHLO example.com
```

### Step 4 — provare manualmente VRFY

```text
VRFY root
VRFY admin
VRFY test
```

### Step 5 — provare EXPN

```text
EXPN root
```

### Step 6 — verificare RCPT

```text
MAIL FROM:<test@example.com>
RCPT TO:<root@example.com>
```

### Step 7 — automatizzare con Nmap

```bash
sudo nmap -p25 --script smtp-enum-users <IP>
```

### Step 8 — utilizzare una wordlist personalizzata

Se il laboratorio fornisce una wordlist specifica:

```bash
smtp-user-enum -M VRFY \
-U <FOOTPRINTING-WORDLIST> \
-t <IP>
```

Eventualmente:

```bash
smtp-user-enum -M EXPN \
-U <FOOTPRINTING-WORDLIST> \
-t <IP>
```

e:

```bash
smtp-user-enum -M RCPT \
-U <FOOTPRINTING-WORDLIST> \
-t <IP>
```

---

# 18. Interpretare correttamente i risultati

Non bisogna considerare automaticamente ogni risposta come una conferma assoluta.

Per esempio:

```text
VRFY root
```

potrebbe dare una risposta positiva mentre:

```text
VRFY admin
```

dà una risposta negativa.

Questo può indicare che `root` è riconosciuto dal server, ma bisogna considerare anche la configurazione SMTP e il comportamento del metodo utilizzato.

È quindi utile confrontare:

```text
VRFY
EXPN
RCPT
```

e verificare i risultati con più di un metodo quando possibile.

---

# 19. Comandi essenziali da ricordare

### Nmap

```bash
sudo nmap -p25 -sC -sV <IP>
```

### Nmap SMTP enumeration

```bash
sudo nmap -p25 --script smtp-enum-users <IP>
```

### Nmap solo VRFY

```bash
sudo nmap -p25 \
--script smtp-enum-users \
--script-args 'smtp-enum-users.methods={VRFY}' \
<IP>
```

### Telnet

```bash
telnet <IP> 25
```

### SMTP greeting

```text
EHLO example.com
```

### User enumeration manuale

```text
VRFY username
```

```text
EXPN username
```

### SMTP transaction

```text
MAIL FROM:<sender@example.com>
RCPT TO:<user@example.com>
DATA
```

### smtp-user-enum

```bash
smtp-user-enum -M VRFY -U userlist.txt -t <IP>
```

```bash
smtp-user-enum -M EXPN -U userlist.txt -t <IP>
```

```bash
smtp-user-enum -M RCPT -U userlist.txt -t <IP>
```

### Con timeout/concorrenza

```bash
smtp-user-enum -M VRFY \
-U userlist.txt \
-t <IP> \
-m 60 \
-w 20
```

---

# 20. Concetto fondamentale

Il punto fondamentale da ricordare è:

```text
                 SMTP server
                     │
          ┌──────────┼──────────┐
          │          │          │
        VRFY        EXPN       RCPT
          │          │          │
          └──────────┼──────────┘
                     │
              User Enumeration
```

Puoi interagire manualmente con questi meccanismi tramite **Telnet**, automatizzarli con lo **script NSE di Nmap `smtp-enum-users`**, oppure utilizzare **`smtp-user-enum`** quando vuoi fornire una wordlist personalizzata.
