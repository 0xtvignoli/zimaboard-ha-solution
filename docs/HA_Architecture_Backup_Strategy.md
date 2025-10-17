# Architettura HA e Strategia di Backup per Zimaboard

## 1. Introduzione

Questo documento delinea l'architettura per una soluzione self-hosted ad alta disponibilità (HA) e la strategia di backup per i servizi eseguiti su due Zimaboard. L'obiettivo è garantire la continuità operativa dei servizi multimediali e di storage, minimizzando i tempi di inattività in caso di guasto hardware su una delle due unità.

## 2. Configurazione Hardware

Si utilizzeranno due Zimaboard con le seguenti specifiche:

*   **Zimaboard Principale (Node 1):** Intel N150, 16GB RAM
*   **Zimaboard Secondaria (Node 2):** Intel N3450, 8GB RAM

Entrambe le unità eseguiranno ZimaOS, che offre un ambiente Dockerizzato nativo, semplificando la gestione dei container e la coerenza tra i nodi.

## 3. Architettura ad Alta Disponibilità (HA)

Per raggiungere l'HA, adotteremo un modello **attivo/passivo** per i servizi stateful e un modello **attivo/attivo** con bilanciamento del carico per i servizi stateless (come il reverse proxy). La Zimaboard Principale (Node 1) ospiterà i servizi primari, mentre la Zimaboard Secondaria (Node 2) sarà configurata per subentrare in caso di fallimento o per ospitare servizi meno critici/di supporto.

### 3.1. Servizi Stateful (Nextcloud, PostgreSQL, Navidrome, Jellyfin, Audiobookshelf)

I servizi stateful richiedono che i loro dati persistano e siano accessibili in modo coerente da entrambi i nodi. La strategia prevede:

*   **Archiviazione Dati Condivisa/Replicata:**
    *   **Opzione 1 (Consigliata - Semplice):** Utilizzo di **NFS (Network File System)** o **SMB/CIFS** per condividere i volumi di dati persistenti da un NAS esterno o da un disco USB/SATA collegato alla Zimaboard Principale e condiviso con la Secondaria. Questo semplifica la gestione dei dati, ma introduce un singolo punto di fallimento per lo storage se il NAS/disco esterno non è HA.
    *   **Opzione 2 (Avanzata - Replicata):** Implementazione di una soluzione di replica a livello di blocco come **DRBD (Distributed Replicated Block Device)** o un filesystem distribuito come **GlusterFS/Ceph**. Queste soluzioni offrono una vera HA per lo storage, ma sono significativamente più complesse da configurare e gestire su hardware limitato come le Zimaboard. Richiedono anche dischi dedicati su entrambi i nodi per la replica.
    *   **Database (PostgreSQL):** Per PostgreSQL, si può configurare la **replica streaming** (master-replica) tra i due nodi. Il Node 1 sarà il master e il Node 2 sarà la replica. In caso di fallimento del master, la replica può essere promossa a nuovo master. Questo richiede una gestione attenta della promozione e del failover (es. con `Patroni` o script personalizzati).

*   **Failover dei Servizi:**
    *   Utilizzo di **Keepalived** o **HAProxy** (in modalità passiva) per gestire un **IP virtuale (VIP)**. Il VIP punterà sempre al nodo attivo che esegue i servizi stateful. In caso di fallimento del Node 1, il VIP si sposterà automaticamente al Node 2, che avrà avviato i servizi come primario.
    *   I container Docker Compose per i servizi stateful saranno configurati per essere avviati solo sul nodo designato come primario in quel momento (gestito da script di failover).

### 3.2. Servizi Stateless (Traefik, DDNS Client, WireGuard VPN)

I servizi stateless possono essere eseguiti contemporaneamente su entrambi i nodi, con il traffico bilanciato tra di essi. Questo offre una maggiore resilienza e può distribuire il carico.

*   **Traefik:** Configurato in modalità **attivo/attivo** su entrambi i nodi. Un bilanciatore di carico esterno (es. router con supporto a round-robin DNS o un servizio cloud) o un VIP gestito da Keepalived/HAProxy indirizzerà il traffico a entrambi i Traefik. Per la configurazione e i certificati, Traefik può utilizzare un backend distribuito come `Consul` o `Etcd`, oppure condividere un volume di configurazione persistente (vedi sezione storage condiviso).
*   **DDNS Client:** Eseguito su entrambi i nodi, ma configurato per aggiornare il record DDNS solo se il nodo è il primario o se rileva che il record non è corretto (per evitare conflitti).
*   **WireGuard VPN:** Il server WireGuard può essere configurato su entrambi i nodi, ma solo uno sarà attivo alla volta, oppure si può configurare un VIP per il server VPN. Alternativamente, si possono avere due configurazioni VPN separate (una per ogni Zimaboard) e connettersi a quella disponibile.

## 4. Strategia di Backup e Ripristino

La strategia di backup è fondamentale per la protezione dei dati e il ripristino in caso di disastro.

### 4.1. Backup dei Dati Persistenti

*   **Dati Utente (Nextcloud, Media):** Utilizzo di **Duplicati** per eseguire backup incrementali e crittografati dei volumi di dati persistenti. I backup saranno inviati a una destinazione remota e sicura (es. storage S3-compatibile, server SFTP esterno, o un altro disco/NAS non collegato direttamente alle Zimaboard).
*   **Dati Database (PostgreSQL):** Oltre alla replica streaming, si eseguiranno backup regolari del database (es. `pg_dump`) e si salveranno con Duplicati.
*   **Configurazioni:** Backup dei file di configurazione di Docker Compose, Traefik, Authelia, WireGuard, ecc., anch'essi con Duplicati.

### 4.2. Ripristino (Disaster Recovery)

*   **Ripristino di un Nodo:** In caso di fallimento di un nodo, il nodo rimanente può assumere il ruolo primario (se configurato per HA). Il nodo guasto può essere sostituito e i dati ripristinati dai backup, o sincronizzati dalla replica (per PostgreSQL).
*   **Ripristino Completo:** In caso di fallimento di entrambi i nodi o di corruzione dei dati, si procederà con l'installazione di ZimaOS su una nuova Zimaboard, la configurazione di base, il ripristino dei volumi di dati e delle configurazioni dai backup di Duplicati, e il riavvio dei servizi.

## 5. Implementazione

L'implementazione richiederà i seguenti passaggi chiave:

1.  **Configurazione di Rete:** Assegnazione di IP statici ai due nodi e configurazione di un IP virtuale (VIP) per il failover.
2.  **Preparazione dello Storage:** Configurazione dello storage condiviso/replicato (NFS/SMB o DRBD/GlusterFS).
3.  **Docker Compose:** Creazione di file `docker-compose.yml` per ogni servizio, con configurazioni specifiche per HA (es. volumi, dipendenze, riavvio).
4.  **Traefik HA:** Configurazione di Traefik su entrambi i nodi con gestione dei certificati e routing.
5.  **DDNS e VPN:** Implementazione del client DDNS e del server WireGuard VPN.
6.  **Authelia:** Integrazione di Authelia per SSO.
7.  **Monitoraggio:** Configurazione di Netdata su entrambi i nodi per monitorare lo stato HA.
8.  **Backup:** Configurazione di Duplicati per i backup automatici.
9.  **Script di Failover:** Sviluppo e test di script per gestire il failover automatico dei servizi e del VIP.

Questo approccio fornirà una base robusta per un'infrastruttura self-hosted resiliente e ad alta disponibilità sulla Zimaboard.
