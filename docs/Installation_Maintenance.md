# Documentazione di Installazione e Manutenzione per Zimaboard HA

## 1. Introduzione

Questa documentazione fornisce una guida dettagliata per l'installazione e la manutenzione dell'infrastruttura self-hosted ad alta disponibilità (HA) su due Zimaboard. L'obiettivo è coprire tutti i passaggi necessari per configurare i servizi principali, il DDNS, la VPN, l'SSO e la strategia di backup.

## 2. Prerequisiti

### 2.1. Hardware

*   **Zimaboard Principale (Node 1):** Intel N150, 16GB RAM (o superiore)
*   **Zimaboard Secondaria (Node 2):** Intel N3450, 8GB RAM (o superiore)
*   **Storage Esterno (NAS o USB/SATA):** Un dispositivo di archiviazione esterno affidabile per ospitare i dati persistenti dei servizi stateful (es. Nextcloud data, database PostgreSQL, librerie multimediali). Questo storage deve essere accessibile via NFS o SMB da entrambi i nodi.
*   **Router:** Con funzionalità di port forwarding e, idealmente, supporto per IP statici locali.

### 2.2. Software

*   **ZimaOS:** Installato su entrambe le Zimaboard (ZimaOS è nativamente Dockerizzato).
*   **Git:** Per clonare il repository del progetto.
*   **Docker Compose:** Già presente su ZimaOS.
*   **Editor di Testo:** Per modificare i file di configurazione.

### 2.3. Rete

*   **Indirizzi IP Statici:** Assegnare IP statici locali a entrambe le Zimaboard (es. `192.168.1.10` per Node 1, `192.168.1.11` per Node 2).
*   **IP Virtuale (VIP):** Decidere un IP virtuale per il failover dei servizi (es. `192.168.1.12`). Questo VIP sarà gestito da Keepalived o script di failover.
*   **Dominio DDNS:** Un nome di dominio registrato e configurato con un provider DDNS (es. `your-domain.com`).
*   **Port Forwarding:** Configurare il router per inoltrare le porte 80, 443 (per Traefik) e 51820 UDP (per WireGuard) verso l'IP virtuale (VIP) o, se non si usa un VIP per Traefik, verso l'IP del Node 1 inizialmente.

## 3. Setup Iniziale (Entrambe le Zimaboard)

1.  **Installazione ZimaOS:** Assicurarsi che ZimaOS sia installato e configurato su entrambe le Zimaboard.
2.  **Accesso SSH:** Abilitare e configurare l'accesso SSH per entrambe le unità.
3.  **Aggiornamento del Sistema:** Eseguire `sudo apt update && sudo apt upgrade -y`.
4.  **Installazione NFS Client:** Installare il client NFS per montare lo storage esterno condiviso:
    ```bash
    sudo apt install nfs-common -y
    ```
5.  **Creazione Punti di Mount:** Creare le directory per i mount NFS (es. `/mnt/nfs_share`).
    ```bash
    sudo mkdir -p /mnt/nfs_share/postgres_data
    sudo mkdir -p /mnt/nfs_share/nextcloud_data
    sudo mkdir -p /mnt/nfs_share/navidrome_data
    sudo mkdir -p /mnt/nfs_share/jellyfin_config
    sudo mkdir -p /mnt/nfs_share/jellyfin_cache
    sudo mkdir -p /mnt/nfs_share/media
    sudo mkdir -p /mnt/nfs_share/audiobookshelf_config
    sudo mkdir -p /mnt/nfs_share/audiobooks
    sudo mkdir -p /mnt/nfs_share/podcasts
    sudo mkdir -p /mnt/nfs_share/traefik_acme
    sudo mkdir -p /mnt/nfs_share/backup_destination
    sudo mkdir -p /mnt/nfs_share/data_to_backup # Questo sarà il mount root per duplicati
    ```
6.  **Mount NFS:** Aggiungere le voci a `/etc/fstab` per montare automaticamente le condivisioni NFS. Sostituire `your_nfs_server_ip` con l'IP del server NFS e `/path/on/nfs` con i percorsi corretti:
    ```bash
    your_nfs_server_ip:/path/on/nfs/postgres_data /mnt/nfs_share/postgres_data nfs defaults,nofail 0 0
    your_nfs_server_ip:/path/on/nfs/nextcloud_data /mnt/nfs_share/nextcloud_data nfs defaults,nofail 0 0
    # ... ripetere per tutti i percorsi sopra
    ```
    Eseguire `sudo mount -a` per montare i volumi.
7.  **Clonare il Repository:** Su entrambe le Zimaboard, clonare il repository del progetto:
    ```bash
    git clone https://github.com/liv4-labs/reverse-ws-tunnel.git
    cd reverse-ws-tunnel
    git checkout develop
    ```

## 4. Configurazione dei Servizi

### 4.1. Configurazione Iniziale (Node 1)

Sul Node 1 (Zimaboard 16GB RAM):

1.  **Modificare i file Docker Compose:**
    *   Aprire `docker/docker-compose-node1.yml`.
    *   Sostituire `your-email@example.com` con la tua email per Let's Encrypt.
    *   Sostituire `your-domain.com`, `traefik.your-domain.com`, `your-nextcloud-domain.com`, `navidrome.your-domain.com`, `jellyfin.your-domain.com`, `audiobookshelf.your-domain.com` con i tuoi domini/sottodomini.
    *   Impostare `POSTGRES_PASSWORD` per PostgreSQL e `DB_PASSWORD` per Nextcloud.
    *   Generare un hash per la password del dashboard di Traefik: `echo $(htpasswd -nb user your_password) | sed -e s/\$/\$\$/g` e sostituire `user:$$apr1$$HASHED_PASSWORD`.
    *   Aprire `docker/authelia/config/configuration.yml`.
    *   Sostituire `your-domain.com` e `auth.your-domain.com` con i tuoi domini.
    *   Generare un secret per la sessione Authelia: `authelia-gen-secret` e sostituire `$(authelia-gen-secret)`.
    *   Configurare i dettagli SMTP per il notifier di Authelia.
    *   Aprire `docker/authelia/config/users.yml`.
    *   Generare un hash per la password dell'utente Authelia: `authelia hash-password your_password` e sostituire `hashed_password_placeholder`.
    *   Sostituire `user1@your-domain.com`.
    *   Aprire `docker/ddclient/config/ddclient.conf`.
    *   Sostituire `your_ddns_username`, `your_ddns_password`, `your-domain.com` con i dettagli del tuo provider DDNS.

2.  **Avviare i Servizi Principali sul Node 1:**
    ```bash
    cd ~/reverse-ws-tunnel/docker
    docker compose -f docker-compose-node1.yml up -d
    ```

### 4.2. Configurazione Iniziale (Node 2)

Sul Node 2 (Zimaboard 8GB RAM):

1.  **Modificare i file Docker Compose:**
    *   Aprire `docker/docker-compose-node2.yml`.
    *   Sostituire i domini e le password come fatto per il Node 1.
    *   Assicurarsi che `POSTGRES_PRIMARY_HOST` punti all'IP o al nome host del Node 1 (`postgres_node1` nel compose, che dovrà essere risolvibile, es. tramite `/etc/hosts` o DNS interno).
    *   Impostare `POSTGRES_PRIMARY_USER` e `POSTGRES_PRIMARY_PASSWORD` per la replica.
    *   Aprire `docker/authelia/config/configuration.yml` e `users.yml` e configurarli in modo identico al Node 1 (o replicare i file).
    *   Aprire `docker/ddclient/config/ddclient.conf` e configurarlo in modo identico al Node 1. Considerare l'aggiunta di logica per evitare aggiornamenti DDNS concorrenti (es. solo il nodo primario aggiorna).

2.  **Avviare i Servizi Replica/Standby sul Node 2:**
    ```bash
    cd ~/reverse-ws-tunnel/docker
    docker compose -f docker-compose-node2.yml up -d
    ```

## 5. Configurazione WireGuard VPN

1.  **Modificare `docker/docker-compose-vpn.yml`:**
    *   Sostituire `your-vpn-domain.com` con il tuo dominio DDNS.
    *   Definire i nomi dei client VPN in `PEERS` (es. `myphone,mylaptop`).
2.  **Avviare il Servizio WireGuard (su Node 1 o sul nodo designato come VPN server):**
    ```bash
    cd ~/reverse-ws-tunnel/docker
    docker compose -f docker-compose-vpn.yml up -d
    ```
3.  **Generare Configurazione Client:** Dopo l'avvio, la configurazione per i client WireGuard sarà disponibile in `docker/wireguard/config/peer_client_name/peer_client_name.conf`. Copiare questi file sui rispettivi dispositivi client e importarli nell'applicazione WireGuard.

## 6. Configurazione Duplicati per Backup

1.  **Modificare `docker/docker-compose-backup.yml`:**
    *   Sostituire `backup.your-domain.com` con il dominio desiderato per l'interfaccia web di Duplicati.
    *   Assicurarsi che `/mnt/nfs_share/data_to_backup` e `/mnt/nfs_share/backup_destination` siano correttamente montati e contengano i dati da salvare e la destinazione temporanea.
2.  **Avviare il Servizio Duplicati (su Node 1 o su un nodo dedicato al backup):**
    ```bash
    cd ~/reverse-ws-tunnel/docker
    docker compose -f docker-compose-backup.yml up -d
    ```
3.  **Configurare i Job di Backup:** Accedere all'interfaccia web di Duplicati (es. `https://backup.your-domain.com`) e configurare i job di backup, specificando la destinazione remota (S3, SFTP, ecc.) e i dati da includere (es. `/source` mount).

## 7. Configurazione Netdata per Monitoraggio

1.  **Modificare `docker/docker-compose-monitoring.yml`:**
    *   Sostituire `your_claim_token` con il token del tuo account Netdata Cloud.
    *   Impostare `NETDATA_HOSTNAME` con un nome univoco per ogni Zimaboard (es. `zimaboard-node1`, `zimaboard-node2`).
    *   Sostituire `netdata.your-domain.com` con il dominio desiderato per l'interfaccia web di Netdata.
2.  **Avviare il Servizio Netdata (su entrambe le Zimaboard):**
    ```bash
    cd ~/reverse-ws-tunnel/docker
    docker compose -f docker-compose-monitoring.yml up -d
    ```
3.  **Verifica:** Accedere al dashboard di Netdata Cloud per verificare che entrambi i nodi stiano inviando i dati.

## 8. Gestione HA e Failover

Questa sezione richiede la configurazione di strumenti esterni a Docker Compose per gestire il failover del VIP e l'orchestration dei container in caso di guasto.

### 8.1. Keepalived per IP Virtuale (VIP)

*   **Installazione:** Installare Keepalived su entrambe le Zimaboard.
    ```bash
    sudo apt install keepalived -y
    ```
*   **Configurazione:** Creare il file `/etc/keepalived/keepalived.conf` su entrambi i nodi. Il Node 1 sarà il `MASTER` e il Node 2 sarà il `BACKUP`. Entrambi monitoreranno lo stato dei servizi critici e, in caso di fallimento del MASTER, il BACKUP assumerà il VIP.
    ```nginx
    # Esempio di configurazione keepalived.conf per Node 1 (MASTER)
    vrrp_script chk_docker {
        script "/usr/bin/docker ps -q | grep -q \"traefik\" && /usr/bin/docker ps -q | grep -q \"postgres\""
        interval 2 # check every 2 seconds
        weight 2
    }

    vrrp_instance VI_1 {
        state MASTER
        interface eth0 # CHANGE THIS to your network interface
        virtual_router_id 51
        priority 101 # Higher priority for MASTER
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass mysecretpassword # CHANGE THIS
        }
        virtual_ipaddress {
            192.168.1.12/24 # CHANGE THIS to your VIP
        }
        track_script {
            chk_docker
        }
        notify_master "/path/to/master_script.sh"
        notify_backup "/path/to/backup_script.sh"
        notify_fault "/path/to/fault_script.sh"
    }
    ```
    *   Il `chk_docker` script verifica che i container critici (es. Traefik, PostgreSQL) siano in esecuzione. Se fallisce, il nodo degrada la sua priorità.
    *   Gli script `notify_master`, `notify_backup`, `notify_fault` possono essere usati per avviare/fermare i container Docker Compose specifici per il ruolo (es. avviare `nextcloud_standby` sul Node 2 quando diventa master).

### 8.2. Script di Failover e Promozione (Esempio Concettuale)

*   **master_script.sh (su Node 1):** Quando Node 1 diventa MASTER, assicura che i servizi primari siano attivi.
    ```bash
    #!/bin/bash
    docker compose -f ~/reverse-ws-tunnel/docker/docker-compose-node1.yml up -d
    # Potrebbe anche fermare i servizi di standby se fossero stati avviati per errore
    ```
*   **backup_script.sh (su Node 2):** Quando Node 2 diventa BACKUP, assicura che i servizi di replica siano attivi e i servizi primari siano fermi.
    ```bash
    #!/bin/bash
    docker compose -f ~/reverse-ws-tunnel/docker/docker-compose-node2.yml up -d
    # Logica per promuovere PostgreSQL replica a master se necessario
    # Logica per avviare nextcloud_standby, navidrome_standby, ecc.
    ```

## 9. Manutenzione e Aggiornamenti

*   **Aggiornamenti ZimaOS:** Seguire la procedura di aggiornamento di ZimaOS. È consigliabile aggiornare un nodo alla volta per mantenere l'HA.
*   **Aggiornamenti Docker Compose:** Per aggiornare i container, utilizzare `docker compose pull && docker compose up -d` dopo aver testato le nuove immagini in un ambiente di staging o su un nodo non critico.
*   **Monitoraggio:** Controllare regolarmente il dashboard di Netdata per anomalie.
*   **Backup:** Verificare l'integrità dei backup periodicamente e testare la procedura di ripristino.

## 10. Risoluzione dei Problemi

*   **Servizio non raggiungibile:**
    *   Verificare lo stato dei container con `docker ps`.
    *   Controllare i log dei container con `docker logs <container_name>`.
    *   Verificare lo stato di Traefik tramite il suo dashboard.
    *   Controllare le configurazioni di Traefik (`traefik.yml`, `dynamic.yml`).
    *   Verificare il funzionamento del DDNS.
    *   Controllare le regole del firewall e il port forwarding sul router.
*   **Failover non Funzionante:**
    *   Controllare i log di Keepalived (`sudo journalctl -u keepalived`).
    *   Verificare che gli script di `chk_docker` funzionino correttamente.
    *   Assicurarsi che le priorità e l'autenticazione di Keepalived siano corrette.

## 11. Riferimenti

[1] Audiobookshelf: [https://www.audiobookshelf.org/](https://www.audiobookshelf.org/)
[2] Keepalived: [https://keepalived.org/](https://keepalived.org/)
[3] Traefik: [https://traefik.io/](https://traefik.io/)
[4] Authelia: [https://www.authelia.com/](https://www.authelia.com/)
[5] WireGuard: [https://www.wireguard.com/](https://www.wireguard.com/)
[6] Duplicati: [https://www.duplicati.com/](https://www.duplicati.com/)
[7] Netdata: [https://www.netdata.cloud/](https://www.netdata.cloud/)

