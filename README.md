# 💣 Analisi della Fork Bomb: Un Denial-of-Service Locale

Questo documento fornisce un'analisi tecnica dettagliata di una "Fork Bomb" implementata come un semplice script di shell. L'obiettivo è esaminare il suo meccanismo d'azione, i componenti e l'impatto risultante su un sistema Unix-like.

**⚠️ ATTENZIONE: Eseguire questo codice sul proprio sistema lo renderà immediatamente inutilizzabile, richiedendo un riavvio forzato. Procedere con la massima cautela e solo in ambienti virtualizzati e isolati destinati a test distruttivi.**

## 🎯 Concetto (Proof of Concept)

Una Fork Bomb è un attacco di tipo Denial-of-Service (DoS) che agisce localmente. Il suo principio operativo si basa sulla creazione rapida e ricorsiva di nuovi processi. Ogni nuovo processo ("fork") consuma risorse di sistema, principalmente uno slot nella tabella dei processi (PID) e tempo di CPU. L'attacco ha successo quando il numero di processi creati esaurisce le risorse disponibili, portando il sistema a uno stato di stallo completo, incapace di eseguire ulteriori operazioni, incluse quelle amministrative necessarie per terminare l'attacco.

## 📜 Il Codice Sorgente

La forma più comune e concisa di una fork bomb in una shell Bourne-like è la seguente:
````
while true; do
    nohup yes > /dev/null 2>&1 &
done
````

A volte, viene rappresentata in una forma più oscura e compatta conosciuta come "bash fork bomb":

````
:(){ :|:& };:
````

Entrambe le versioni raggiungono lo stesso scopo attraverso meccanismi leggermente diversi. Per chiarezza, analizzeremo la prima versione, più leggibile.

## 🔬 Analisi Dettagliata dei Componenti

Ogni parte di questo script ha uno scopo preciso e contribuisce all'efficacia dell'attacco.
1. Il Loop Infinito: `while true; do ... done`

    Scopo: Creare un ciclo di esecuzione che non termina mai.

    Funzionamento: `true` è un comando che non fa nulla se non restituire uno stato di uscita pari a 0 (successo). Il costrutto `while` continua a eseguire il blocco di codice al suo interno finché la condizione (`true`) rimane tale, ovvero per sempre. Questo garantisce una generazione continua e ininterrotta di processi.

2. Esecuzione in Background: `&`

    Scopo: Eseguire un comando senza attendere il suo completamento.

    Funzionamento: L'operatore `&` alla fine di una riga di comando istruisce la shell a lanciare il processo in background. Questo è l'elemento chiave che permette al loop `while` di iterare istantaneamente. Senza di esso, il loop attenderebbe la fine del primo processo (che non avviene mai) e non potrebbe lanciare processi successivi.

3. Il Generatore di Processi: `yes`

    Scopo: Creare un processo semplice che consumi attivamente CPU.

    Funzionamento: Il comando `yes` è un'utility standard che stampa ripetutamente una stringa (di default, "y") sul suo standard output. È estremamente leggero da avviare ma impegna costantemente un thread di CPU per la sua esecuzione, rendendolo perfetto per saturare i processori.

4. Prevenzione dell'Interruzione: `nohup`

    Scopo: Rendere il processo immune alla disconnessione del terminale.

    Funzionamento: `nohup` (NO HangUP) intercetta il segnale `SIGHUP` che il sistema invia ai processi quando la loro shell di controllo viene chiusa. Questo assicura che la bomba continui a funzionare anche se l'utente che l'ha lanciata chiude la sessione SSH o il terminale.

5. Soppressione dell'Output: `> /dev/null 2>&1`

    Scopo: Eliminare tutto l'output generato per evitare di riempire il disco o il terminale.

    Funzionamento:

    - `> /dev/null`: Reindirizza lo standard output (stdout) del comando `yes` al file speciale `/dev/null`, che è un "buco nero" che scarta tutti i dati ricevuti.

    - `2>&1`: Reindirizza lo standard error (stderr, file descriptor 2) allo stesso posto dello standard output (stdout, file descriptor 1). Questo sopprime anche eventuali messaggi di errore, rendendo l'attacco completamente silente.

## 💥 Meccanismo d'Azione e Impatto Finale

1. Inizio: Il loop `while` inizia la sua prima iterazione.

2. Fork: Lancia un processo `yes` in background. Il processo viene immediatamente aggiunto alla tabella dei processi del kernel e lo scheduler della CPU gli assegna del tempo di esecuzione.

3. Iterazione Istantanea: Grazie all'operatore `&`, il controllo torna immediatamente al loop `while`, che inizia la sua seconda iterazione senza alcun ritardo.

4. Crescita Esponenziale (di fatto lineare molto rapida): I passaggi 2 e 3 si ripetono alla massima velocità consentita dalla CPU e dalla shell. In una frazione di secondo, vengono creati migliaia di processi.

5. Saturazione delle Risorse:

    - Esaurimento dei PID: Ogni sistema operativo ha un limite massimo di processi (configurabile tramite `kernel.pid_max`). La fork bomb consuma tutti i PID disponibili. Una volta esauriti, il sistema non può più creare alcun nuovo processo, bloccando l'avvio di programmi, l'apertura di nuove shell o persino l'esecuzione di comandi per la risoluzione dei problemi come `kill` o `pkill`.

    - Saturazione della CPU: Migliaia di processi `yes` competono per il tempo di esecuzione, portando l'utilizzo di tutti i core della CPU al 100%. Lo scheduler del kernel è sovraccarico nel tentativo di gestire questa enorme quantità di processi attivi.

6. Stallo del Sistema: Il sistema diventa estremamente lento o smette completamente di rispondere. L'input da tastiera e mouse viene ignorato. Le connessioni di rete vengono interrotte. L'unica soluzione praticabile è un riavvio hardware (hard reboot).
