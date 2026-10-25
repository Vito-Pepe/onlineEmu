# onlineEmu

Libreria di giochi retro sul Raspberry Pi, da giocare dal browser (anche da iPhone).

Usa **[RomM](https://github.com/rommapp/romm)** con l'emulatore **EmulatorJS** integrato:
- il Pi **conserva** i giochi, la pagina con le copertine e i **salvataggi**;
- il gioco **gira sul dispositivo** che lo apre (iPhone, PC...), dentro la pagina web: niente da installare, nessun ritardo.

Per ora: **Game Boy Advance**. La PS1 si aggiunge più avanti (serve il BIOS, vedi in fondo).

| | |
|---|---|
| LAN | `http://192.168.1.22:8088` |
| Tailnet (HTTPS) | `https://retro.<tailnet>.ts.net` (repo `ingressi-https`) |
| Cartella sul Pi | `~/onlineEmu` (progetto compose `onlineemu`) |
| Container | `onlineemu-romm-1` (RomM), `onlineemu-db-1` (MariaDB) |
| Volumi | `onlineemu_db-data` (database, da non perdere), `onlineemu_cache` |
| Dati | `~/onlineEmu/dati/` (giochi, BIOS, salvataggi, copertine; fuori da git) |
| Cookie | `romm_session`, `romm_csrftoken` |

Solo tailnet e LAN, **mai su Internet**.

## Installazione sul Raspberry Pi

Un blocco alla volta.

**1. Controlla che porta e nomi siano liberi**

```bash
sudo ss -ltnp | grep -q ':8088 ' && echo "8088 OCCUPATA" || echo "8088 libera"
docker ps -a --format '{{.Names}}' | grep -iE 'onlineemu|romm' || echo "nomi liberi"
free -h
```

Atteso: `8088 libera` e `nomi liberi`. RomM e il suo database usano circa 0,5–1 GB di RAM.

**2. Scarica la repo e crea le cartelle dei giochi**

```bash
cd ~
git clone https://github.com/Vito-Pepe/onlineEmu.git
cd onlineEmu
mkdir -p dati/library/roms/gba dati/library/bios
```

Le cartelle si creano a mano, così sono tue (utente `pepe`) e puoi copiarci i giochi senza `sudo`.

**3. Crea i segreti** (generati sul Pi, non passano da nessuna parte)

```bash
{ echo "DB_PASSWD=$(openssl rand -hex 24)"; echo "DB_ROOT_PASSWD=$(openssl rand -hex 24)"; echo "ROMM_AUTH_SECRET_KEY=$(openssl rand -hex 32)"; } >> .env
chmod 600 .env
grep -c '=' .env
```

Atteso: `3`.

**4. Avvia**

```bash
docker compose up -d
```

Il primo avvio scarica le immagini (qualche minuto). Poi, dopo circa un minuto:

```bash
docker compose ps
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8088/api/heartbeat
```

Atteso: due container `Up` (il database `healthy`) e `200`. Se esce `000` aspetta ancora un po': al primo avvio RomM prepara il database.

**5. Aggiornamento automatico** (senza sudo)

```bash
(crontab -l 2>/dev/null; echo "*/5 * * * * cd $HOME/onlineEmu && ./update.sh >> $HOME/onlineEmu/update.log 2>&1") | crontab -
crontab -l | grep onlineEmu
```

**6. Primo accesso**

Apri `http://192.168.1.22:8088` (o `https://retro.<tailnet>.ts.net`): RomM chiede di creare l'**account amministratore** (nome utente e password a tua scelta). Poi entri.

## Aggiungere i giochi GBA

Formati: `.gba` (anche dentro `.zip`).

- **Dalla pagina** (anche da iPhone): pulsante di caricamento (Upload), scegli la piattaforma Game Boy Advance e i file.
- **Dal computer**, copiandoli sul Pi:

  ```bash
  scp *.gba pepe@192.168.1.22:~/onlineEmu/dati/library/roms/gba/
  ```

  poi nella pagina: **Scan** (scansione della libreria) per farli comparire.

Copertine e informazioni arrivano in automatico da Hasheous (nessun account necessario).

Solo giochi di cui possiedi la cartuccia, o homebrew/giochi liberi.

## Su iPhone

- In Safari apri `https://retro.<tailnet>.ts.net`, poi **Condividi → Aggiungi alla schermata Home**: si apre a schermo intero come un'app e iOS non cancella i giochi già scaricati.
- L'audio parte dopo il primo tocco (regola di iOS).
- I controller Bluetooth (PS4/PS5, Xbox, Backbone) funzionano.
- Con il **Relay privato di iCloud** gli indirizzi `*.ts.net` possono non aprirsi: usa `http://vito:8088`.

## Aggiornare RomM

La versione è fissata in `docker-compose.yml` (`rommapp/romm:5.4.1`). Per aggiornare: cambiare il numero con una PR; dopo il merge `update.sh` scarica la nuova immagine e RomM aggiorna il database da solo all'avvio.

## Comandi utili

```bash
cd ~/onlineEmu
docker compose ps
docker compose logs --tail 50 romm
docker compose restart romm
```

Mai `docker compose down -v`: cancellerebbe il database.

## Più avanti: PS1

1. Estrarre il BIOS dalla propria PS1 (es. con [PS1 BIOS dumper](https://github.com/ShendoXT/biosdumper)).
2. Copiarlo in `dati/library/bios/psx/` (cartella da creare) e i giochi in `dati/library/roms/psx/` (meglio in formato `.chd`, più piccolo).
3. **Scan** nella pagina.
