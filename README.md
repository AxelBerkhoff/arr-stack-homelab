# ARR Stack Homelab Setup met VPN

> **Gebaseerd op het werk van [automation-avenue](https://github.com/automation-avenue/arr-new) — New ARR Stack 2026**
> Met dank aan hun docker-compose configuratie en uitgebreide documentatie.

---

Deze handleiding beschrijft hoe je een volledige ARR media-automatiseringsstack opzet met Docker.
De instructies zijn geschreven voor Debian/Ubuntu, maar Docker draait op elke Linux-distributie.
Gebruik je Windows of macOS? Dan kun je [Docker Desktop](https://docs.docker.com/desktop/) gebruiken.

---

## Docker installeren en omgeving voorbereiden

Installeer Docker en Docker Compose via de [officiële documentatie](https://docs.docker.com/compose/).
Ga naar **Install → Plugin → Install using the Repository → Ubuntu** (of jouw OS), of ga direct naar [deze pagina](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).

Voer de commando's uit die daar staan, vergelijkbaar met:

```bash
# Voeg Docker's officiële GPG-sleutel toe:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Voeg de repository toe aan de apt-bronnen:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Installeer daarna de Docker-pakketten en controleer of alles werkt:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl status docker
sudo docker run hello-world
```

Controleer of Docker Compose correct is geïnstalleerd:

```bash
docker compose version
```

### Mappenstructuur aanmaken

Maak de aanbevolen mappenstructuur aan volgens de [TRaSH Guide](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Docker/):

```bash
sudo mkdir -p /data/{torrents/{tv,movies,music},media/{tv,movies,music}}
sudo apt install tree
tree /data
sudo chown -R 1000:1000 /data
sudo chmod -R a=,a+rX,u+w,g+w /data
ls -ln /data
```

> Gebruik je naast torrents ook een Usenet-client (zoals NZBGet of SABnzbd)? Vervang dan de eerste regel door:
> ```bash
> mkdir -p /data/{usenet/{incomplete,complete}/{tv,movies,music},media/{tv,movies,music}}
> ```

### Docker Compose bestand

De `docker-compose.yml` van dit project is beschikbaar op [GitHub](https://github.com/automation-avenue/arr-new/blob/main/docker-compose.yml).
Kloon de repository of kopieer het bestand handmatig:

```bash
git clone https://github.com/automation-avenue/arr-new.git
# of maak het bestand handmatig aan:
sudo nano docker-compose.yml
```

> Hostnamen zijn niet nodig in dit configuratiebestand, omdat alle containers op hetzelfde Docker-netwerk draaien.

---

## Eerste keer opstarten

Start alle services met één commando:

```bash
sudo docker compose up -d
```

---

## Services configureren

Na het opstarten moeten de interne instellingen van elke applicatie op elkaar worden afgestemd:

- **Radarr**: Rootmap instellen op `/data/media/movies`
- **Sonarr**: Rootmap instellen op `/data/media/tv`
- **Lidarr**: Rootmap instellen op `/data/media/music`
- **qBittorrent**: Downloadpad instellen op `/data/torrents`

Omdat alle paden onder dezelfde mount (`/data`) vallen, behandelt het OS ze als één bestandssysteem. Dit maakt instant hardlinks (ook wel atomic moves) mogelijk.

---

### qBittorrent

Controleer de tijdelijke inloggegevens in de container-logs:

```bash
sudo docker logs qbittorrent
```

Je ziet iets als:
```
The WebUI administrator username is: admin
The WebUI administrator password was not set. A temporary password is provided for this session: <jouw-wachtwoord>
```

Open de webinterface via:
- Op de host zelf: `http://localhost:8080`
- Vanaf een ander apparaat: `http://<host-ip>:8080`

Wijzig het wachtwoord via **Tools → Options → WebUI**.

#### Categorieën aanmaken

Ga naar het linkerpaneel → **Categories → All → rechtermuisknop → Add category**:

| Service | Categorie | Pad      |
|---------|-----------|----------|
| Radarr  | `movies`  | `movies` |
| Sonarr  | `tv`      | `tv`     |
| Lidarr  | `music`   | `music`  |

> Maak de categorieën aan **vóórdat** je de overige instellingen configureert, anders kunnen ze verdwijnen.

#### Downloadinstellingen

Ga naar **Tools → Options → Downloads → Saving Management** en stel het volgende in (zie ook de [TRaSH Guide](https://trash-guides.info/Downloaders/qBittorrent/How-to-add-categories/)):

- **Default Torrent Management Mode**: `Automatic`
- **When Torrent Category changed**: `Relocate torrent`
- **When Default Save Path Changed**: `Switch affected torrents to Manual Mode`
- **When Category Save Path Changed**: `Switch affected torrents to Manual Mode`
- Vink **beide** vakjes aan: `Use Subcategories` en `Use Category paths in Manual Mode`
- **Default Save Path**: `/data/torrents`

Sla op met de **Save**-knop onderaan.

> Heb je problemen met categorieën? Probeer dan een alternatieve qBittorrent-image:
> ```yaml
>   qbittorrent:
>     <<: *common-keys
>     container_name: qbittorrent
>     image: ghcr.io/qbittorrent/docker-qbittorrent-nox:latest
>     ports:
>       - 8080:8080
>       - 6881:6881
>       - 6881:6881/udp
>     environment:
>       - QBT_LEGAL_NOTICE=confirm
>       - WEBUI_PORT=8080
>       - TORRENTING_PORT=6881
>     volumes:
>       - /etc/localtime:/etc/localtime:ro
>       - /docker/appdata/qbittorrent:/config
>       - /data:/data
> ```

---

### Prowlarr

Open: `http://<host-ip>:9696`

Stel bij het eerste gebruik een gebruikersnaam en wachtwoord in via **Form (login page) authentication**.

Ga naar **Settings → Download Clients → +** en voeg qBittorrent toe:
- **Use SSL**: uitvinken
- **Host**: `qbittorrent`
- **Port**: `8080`
- **Username/Password**: de inloggegevens die je in qBittorrent hebt ingesteld

Test de verbinding en sla op.

---

### Radarr

Open: `http://<host-ip>:7878`

1. **Settings → Media Management → Add Root Folder**: stel in op `/data/media/movies`
2. **Settings → Media Management → Show Advanced → Importing**: zet **Use Hardlinks instead of Copy** aan

Optioneel: zet **Rename Movies**, **Delete empty movie folders during disk scan** en **Import Extra Files** (`srt,sub,nfo`) aan.

3. **Settings → Download Clients → +**: voeg qBittorrent toe (zelfde stappen als bij Prowlarr), maar stel de categorie in op `movies`
4. **Settings → General**: kopieer de API-sleutel
5. Ga naar **Prowlarr → Settings → Apps → + → Radarr**: plak de API-sleutel
   - **Prowlarr Server**: `http://prowlarr:9696`
   - **Radarr Server**: `http://radarr:7878`

Test en sla op.

---

### Sonarr

Open: `http://<host-ip>:8989`

1. **Settings → Media Management → Add Root Folder**: stel in op `/data/media/tv`
2. Zet **Use Hardlinks instead of Copy** aan
3. Optioneel: **Rename Episodes**, **Delete empty Folders**, **Import Extra Files** (`srt,sub,nfo`)
4. Voeg qBittorrent toe als download client, categorie: `tv` (standaard staat dit op `tv-sonarr`, pas dit aan)
5. Koppel aan Prowlarr via **Settings → Apps → + → Sonarr**:
   - **Prowlarr Server**: `http://prowlarr:9696`
   - **Sonarr Server**: `http://sonarr:8989`

---

### Lidarr

Open: `http://<host-ip>:8686`

1. **Settings → Media Management → Add Root Folder**: stel in op `/data/media/music`
2. Voeg qBittorrent toe als download client, categorie: `music` (standaard: `lidarr`)
3. Koppel aan Prowlarr:
   - **Prowlarr Server**: `http://prowlarr:9696`
   - **Lidarr Server**: `http://lidarr:8686`

---

### Bazarr

Open: `http://<host-ip>:6767`

- **Settings → Languages**: maak een taalprofiel aan (bijv. "English" of "Any")
- **Settings → Providers**: voeg ondertitelbronnen toe zoals OpenSubtitles.org (gratis account vereist)
- Ga na het koppelen van Radarr/Sonarr naar het **Series** of **Movies** tabblad en klik **Update** om je bibliotheek te synchroniseren

---

## Services herstarten

Start alle services opnieuw om te controleren of alles correct opstart:

```bash
sudo docker compose down
sudo docker compose up -d
```

> De melding `WARN[0000] No services to build` is normaal en kan worden genegeerd.

---

## Indexers toevoegen

Nu kun je indexers toevoegen via Prowlarr. Zoek online naar legale indexers zoals:

- **The Internet Archive (archive.org)** — duizenden publieksdomeinefilms en -series
- Radarr ondersteunt films in het publiek domein of onder Creative Commons-licentie, zoals *Night of the Living Dead (1968)*, *His Girl Friday (1940)* en *Charade (1963)*

> De ARR-stack is niet uitsluitend bedoeld voor auteursrechtelijk beschermd materiaal. Het zijn krachtige automatiseringstools die ook uitstekend werken met legale, vrij beschikbare content.

---

## Jellyfin

Open: `http://<host-ip>:8096`

Maak een gebruikersaccount aan en voeg mediabibliotheken toe:

- **Movies**: koppel aan `/data/media/movies`
- **TV Shows**: koppel aan `/data/media/tv`
- **Music**: koppel aan `/data/media/music`

---

## Probleemoplossing

### DNS controleren

Controleer of containers de Cloudflare DNS gebruiken (geconfigureerd in `docker-compose.yml`):

```bash
sudo docker exec -it radarr cat /etc/resolv.conf
```

### Hardlinks controleren

Navigeer naar `/data` en controleer de mappenstructuur:

```bash
tree /data
du -sch *
```

Vergelijk de inode-nummers van hetzelfde bestand in zowel de torrent- als mediamappen:

```bash
ls -i /data/media/movies/<bestandsnaam>
ls -i /data/torrents/movies/<bestandsnaam>
```

Als de inode-nummers overeenkomen, werken de hardlinks correct. Werkt het niet? Controleer dan de logbestanden (**System → Log Files** in Radarr/Sonarr) — rechten op bron of bestemming zijn de meest voorkomende oorzaak.

### Bestanden verplaatsen niet automatisch

Als bestanden niet automatisch van de torrentmap naar de mediamap worden verplaatst, ga dan naar **Activity → Queue**. Zie je de melding *Downloaded - Unable to Import Automatically*? Klik dan op het handmatig importeer-icoon (het hoofd-icoontje rechts in de rij) en bevestig de juiste film of serie.

---

## Optionele uitbreidingen

### FlareSolverr (Cloudflare bypass)

Voeg toe aan je `docker-compose.yml` als Prowlarr bepaalde sites niet kan indexeren door Cloudflare-blokkades:

```yaml
  flaresolverr:
    <<: *common-keys
    container_name: flaresolverr
    image: ghcr.io/flaresolverr/flaresolverr:latest
    ports:
      - 8191:8191
    environment:
      - LOG_LEVEL=info
```

Configureer daarna in Prowlarr via **Settings → Indexers → + → FlareSolverr**:
- **Host**: `http://flaresolverr:8191`
- **Tags**: `cloudflare`

### Jellyfin hardwareversnelling

Voeg de volgende regels toe aan de Jellyfin-service in `docker-compose.yml` om GPU-versnelling in te schakelen (vereist aanvullende configuratie buiten Docker):

```yaml
jellyfin:
    <<: *common-keys
    devices:
      - /dev/dri:/dev/dri
```

### SABnzbd (Usenet-client)

Voeg toe aan `docker-compose.yml` als je SABnzbd gebruikt in plaats van of naast qBittorrent:

```yaml
  sabnzbd:
    container_name: sabnzbd
    image: ghcr.io/hotio/sabnzbd:latest
    ports:
      - 8080:8080
      - 9090:9090
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/sabnzbd:/config
      - /data:/data
```

> Draai je zowel qBittorrent als SABnzbd? Dan botsen ze op poort `8080`. Pas de externe poort van één van de services aan, bijvoorbeeld:
> ```yaml
>     ports:
>       - 8081:8080
> ```

Voor de SABnzbd-configuratie, zie de [TRaSH Guide voor mappenstructuur](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Docker/) en de [SABnzbd basisinstellingen](https://trash-guides.info/Downloaders/SABnzbd/Basic-Setup/).

---

## Credits

Deze handleiding is gebaseerd op en geïnspireerd door:

- **[automation-avenue/arr-new](https://github.com/automation-avenue/arr-new)** — New ARR Stack 2026
- **[TRaSH Guides](https://trash-guides.info/)** — voor mappenstructuur, hardlink-configuratie en downloadclientinstellingen
- **[Servarr Wiki](https://wiki.servarr.com/)** — voor Radarr, Sonarr en aanverwante documentatie
