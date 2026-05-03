# Traefik

Shared Traefik reverse proxy for BluesBraces projects.

Documentation: [Recron Docs -> External -> Traefik](https://docs.bluebraces.online/external/traefik/)

## Migracja z Hostinger na Contabo VPS

Ten dokument opisuje migracje projektow z Hostinger na Contabo VPS, gdzie Traefik dziala jako wspolny reverse proxy dla aplikacji Dockerowych.

Docelowy uklad:

- Traefik dziala raz na VPS w katalogu `/opt/traefik`.
- Wszystkie aplikacje Dockerowe sa podpiete do wspolnej sieci Docker `proxy`.
- Domeny i subdomeny wskazuja rekordami DNS na publiczne IP Contabo VPS.
- Traefik automatycznie obsluguje HTTPS przez Let's Encrypt.

## 1. Polacz sie z VPS

Zaloguj sie na serwer:

```bash
ssh root@IP_TWOJEGO_CONTABO
```

Przy pierwszym polaczeniu SSH moze pokazac komunikat podobny do:

```txt
The authenticity of host '109.199.115.83 (109.199.115.83)' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Jesli laczysz sie z poprawnym adresem IP swojego VPS, wpisz pelne:

```txt
yes
```

Samo `y` nie wystarczy.

## 2. Zaktualizuj system

Na VPS uruchom:

```bash
apt update
apt upgrade -y
```

Zainstaluj podstawowe narzedzia:

```bash
apt install -y ca-certificates curl gnupg git ufw
```

## 3. Zainstaluj Docker

Zainstaluj Docker oraz Docker Compose plugin:

```bash
curl -fsSL https://get.docker.com | sh
```

Sprawdz instalacje:

```bash
docker --version
docker compose version
```

## 4. Skonfiguruj firewall

Otworz porty potrzebne dla SSH, HTTP i HTTPS:

```bash
ufw allow OpenSSH
ufw allow 80
ufw allow 443
ufw enable
```

Sprawdz status:

```bash
ufw status
```

Powinny byc widoczne reguly dla:

- `22` / `OpenSSH`
- `80`
- `443`

W panelu Contabo sprawdz tez, czy dodatkowy firewall nie blokuje portow `80` i `443`.

## 5. Utworz wspolna siec Docker

Traefik i aplikacje musza byc w tej samej sieci Docker:

```bash
docker network create proxy
```

Jesli Docker zwroci komunikat, ze siec juz istnieje, mozna to zignorowac.

## 6. Przygotuj deploy przez GitHub Actions

W tym repozytorium Traefik powinien byc wdrazany przez GitHub Actions, a nie recznie kopiowany za kazdym razem.

Workflow znajduje sie w:

```txt
.github/workflows/deploy.yml
```

Pipeline uruchamia sie automatycznie po pushu do brancha `main` i robi nastepujace rzeczy:

- kopiuje `docker-compose.yml` na VPS do `/opt/traefik`,
- tworzy plik `.env` z secreta `ACME_EMAIL`,
- tworzy plik `acme.json`,
- ustawia uprawnienia `600` dla `acme.json`,
- tworzy siec Docker `proxy`, jesli jej jeszcze nie ma,
- wykonuje `docker compose pull`,
- wykonuje `docker compose up -d --remove-orphans`,
- robi prosty smoke test kontenera Traefika.

Na GitHubie ustaw sekrety repozytorium lub srodowiska `production`:

```txt
VPS_HOST=IP_TWOJEGO_CONTABO
VPS_USER=root
VPS_SSH_KEY=KLUCZ_PRYWATNY_SSH
ACME_EMAIL=jan.krajewski.it@gmail.com
```

Jesli nie chcesz uzywac `root`, utworz osobnego uzytkownika na VPS i upewnij sie, ze ma dostep do Dockera oraz katalogu `/opt/traefik`.

## 7. Przygotuj katalog /opt/traefik na VPS

Pipeline kopiuje pliki do `/opt/traefik`, wiec katalog powinien istniec i uzytkownik z `VPS_USER` powinien miec do niego dostep.

Utworz katalog:

```bash
mkdir -p /opt/traefik
cd /opt/traefik
```

Jesli deployujesz jako `root`, to wystarczy.

Jesli deployujesz jako inny uzytkownik, ustaw wlasciciela katalogu:

```bash
chown -R NAZWA_UZYTKOWNIKA:NAZWA_UZYTKOWNIKA /opt/traefik
```

## 8. Dodaj klucz SSH do GitHub Actions

Najlepiej uzyc osobnego klucza SSH tylko do deploymentu.

Na swoim komputerze albo na VPS wygeneruj klucz:

```bash
ssh-keygen -t ed25519 -C "github-actions-traefik-deploy"
```

Dodaj klucz publiczny do VPS:

```bash
ssh-copy-id -i SCIEZKA_DO_KLUCZA.pub root@IP_TWOJEGO_CONTABO
```

Jesli `ssh-copy-id` nie jest dostepne, wklej zawartosc pliku `.pub` do:

```txt
/root/.ssh/authorized_keys
```

Klucz prywatny wklej w GitHub jako secret:

```txt
VPS_SSH_KEY
```

## 9. Uruchom deploy z pipeline

Po ustawieniu sekretow zrob push do `main`:

```bash
git push origin main
```

Nastepnie wejdz w GitHub:

```txt
Repository -> Actions -> Deploy Traefik
```

Sprawdz, czy workflow przeszedl poprawnie.

Po udanym deployu na VPS mozesz sprawdzic:

```bash
docker ps
docker logs traefik
```

## 10. Reczny deploy awaryjny

Reczny deploy traktuj jako fallback, gdy GitHub Actions nie jest jeszcze skonfigurowany albo trzeba szybko sprawdzic serwer.

Skopiuj do `/opt/traefik` pliki z tego repozytorium:

- `docker-compose.yml`
- `.env.example`

Jesli repozytorium jest dostepne przez Git:

```bash
git clone ADRES_TEGO_REPO /opt/traefik
cd /opt/traefik
```

Jesli katalog `/opt/traefik` juz istnieje i nie jest pusty, skopiuj pliki recznie przez SCP/SFTP albo sklonuj repo do katalogu tymczasowego i przenies zawartosc.

## 11. Przygotuj plik .env przy recznym deployu

Utworz `.env` na bazie przykladu:

```bash
cp .env.example .env
nano .env
```

W pliku powinien znalezc sie email do Let's Encrypt:

```env
ACME_EMAIL=jan.krajewski.it@gmail.com
```

Mozesz zostawic obecny email albo ustawic inny adres do powiadomien o certyfikatach.

Przy deployu przez GitHub Actions ten plik jest tworzony automatycznie z secreta `ACME_EMAIL`.

## 12. Przygotuj plik na certyfikaty Let's Encrypt przy recznym deployu

Traefik zapisuje certyfikaty w pliku `acme.json`.

```bash
touch acme.json
chmod 600 acme.json
```

Uprawnienia `600` sa wazne. Traefik moze odrzucic plik zbyt szeroko dostepny.

Przy deployu przez GitHub Actions `acme.json` jest tworzony automatycznie.

## 13. Uruchom Traefik recznie

W katalogu `/opt/traefik`:

```bash
docker compose up -d
```

Sprawdz, czy kontener dziala:

```bash
docker ps
docker logs traefik
```

Na tym etapie Traefik moze jeszcze nie miec certyfikatow. Certyfikaty zostana pobrane dopiero wtedy, gdy domena bedzie wskazywala na VPS i przynajmniej jedna aplikacja bedzie miala poprawne labelki Traefika.

## 14. Przepnij DNS z Hostinger na Contabo

W panelu DNS domeny ustaw rekordy `A` na publiczne IP Contabo VPS.

Domena glowna:

```txt
Typ: A
Nazwa: @
Wartosc: IP_TWOJEGO_CONTABO
TTL: 300 albo domyslne
```

Subdomeny:

```txt
Typ: A
Nazwa: app
Wartosc: IP_TWOJEGO_CONTABO
```

```txt
Typ: A
Nazwa: api
Wartosc: IP_TWOJEGO_CONTABO
```

```txt
Typ: A
Nazwa: docs
Wartosc: IP_TWOJEGO_CONTABO
```

Jesli uzywasz `www`:

```txt
Typ: A
Nazwa: www
Wartosc: IP_TWOJEGO_CONTABO
```

Usun albo zmien stare rekordy wskazujace na Hostinger.

Po zmianie sprawdz propagacje:

```bash
nslookup twoja-domena.pl
nslookup app.twoja-domena.pl
```

Wyniki powinny zwracac IP Contabo.

## 15. Przenies aplikacje na VPS

Kazda aplikacja uruchamiana za Traefikiem musi:

- dzialac na tym samym VPS,
- byc podpieta do sieci Docker `proxy`,
- miec labelki Traefika,
- wskazywac poprawny wewnetrzny port kontenera.

Przykladowy `docker-compose.yml` aplikacji:

```yaml
services:
  app:
    image: twoja-aplikacja:latest
    restart: unless-stopped
    env_file:
      - .env
    networks:
      - proxy
    labels:
      - traefik.enable=true
      - traefik.http.routers.myapp.rule=Host(`app.twoja-domena.pl`)
      - traefik.http.routers.myapp.entrypoints=websecure
      - traefik.http.routers.myapp.tls.certresolver=letsencrypt
      - traefik.http.services.myapp.loadbalancer.server.port=3000

networks:
  proxy:
    external: true
```

Najwazniejsza linia do dopasowania:

```yaml
- traefik.http.services.myapp.loadbalancer.server.port=3000
```

To jest port, na ktorym aplikacja slucha wewnatrz kontenera. To nie musi byc publiczny port VPS.

Jesli aplikacja jest za Traefikiem, zwykle nie trzeba wystawiac jej przez `ports`. Traefik komunikuje sie z nia przez Docker network.

## 16. Przenies zmienne srodowiskowe aplikacji

W kazdej aplikacji sprawdz pliki `.env` i podmien stare wartosci z Hostinger.

Szczegolnie sprawdz:

```env
APP_URL=
API_URL=
DATABASE_URL=
MYSQL_HOST=
POSTGRES_HOST=
REDIS_HOST=
SMTP_HOST=
S3_ENDPOINT=
```

Przyklad:

```env
APP_URL=https://twoj-projekt.hostingerapp.com
```

zmien na:

```env
APP_URL=https://app.twoja-domena.pl
```

Jesli baza danych dziala jako kontener w tym samym `docker-compose.yml`, hostem bazy zwykle jest nazwa serwisu, np. `postgres`, `mysql` albo `mariadb`, a nie `localhost`.

Przyklad:

```env
DATABASE_URL=postgresql://user:password@postgres:5432/dbname
```

## 17. Przenies baze danych

### MySQL / MariaDB

Na starym hostingu wykonaj dump:

```bash
mysqldump -u USER -p NAZWA_BAZY > backup.sql
```

Przenies `backup.sql` na VPS i odtworz:

```bash
mysql -u USER -p NAZWA_BAZY < backup.sql
```

Jesli baza dziala w kontenerze:

```bash
docker exec -i NAZWA_KONTENERA mysql -u USER -p NAZWA_BAZY < backup.sql
```

### PostgreSQL

Na starym hostingu:

```bash
pg_dump DATABASE_URL > backup.sql
```

Na Contabo:

```bash
psql DATABASE_URL < backup.sql
```

Jesli baza dziala w kontenerze:

```bash
docker exec -i NAZWA_KONTENERA psql -U USER -d NAZWA_BAZY < backup.sql
```

Jesli Hostinger udostepnia phpMyAdmin albo panel baz danych, mozna wykonac eksport/import przez panel.

## 18. Przenies pliki aplikacji

Jesli aplikacja przechowuje uploady, zdjecia, dokumenty albo inne pliki uzytkownikow, przenies katalogi typu:

- `uploads/`
- `storage/`
- `public/uploads/`
- `media/`

Przyklad z `rsync`:

```bash
rsync -avz ./uploads/ root@IP_TWOJEGO_CONTABO:/opt/twoja-aplikacja/uploads/
```

Jesli dane sa w Docker volumes, sprawdz je:

```bash
docker volume ls
```

Volumes trzeba przeniesc osobno albo odtworzic z backupu.

## 19. Uruchom aplikacje

Dla kazdej aplikacji:

```bash
cd /opt/twoja-aplikacja
docker compose up -d
```

Sprawdz kontenery:

```bash
docker ps
```

Sprawdz logi aplikacji:

```bash
docker logs NAZWA_KONTENERA
```

Sprawdz logi Traefika:

```bash
docker logs traefik
```

## 20. Przetestuj domeny

W przegladarce sprawdz:

```txt
https://twoja-domena.pl
https://app.twoja-domena.pl
https://api.twoja-domena.pl
https://docs.twoja-domena.pl
```

Z poziomu VPS:

```bash
curl -I https://app.twoja-domena.pl
```

Poprawne odpowiedzi to np.:

- `HTTP/2 200`
- `HTTP/2 301`
- `HTTP/2 302`

Zalezy to od aplikacji. Problemem sa przede wszystkim:

- `404 page not found`
- `502 Bad Gateway`
- blad certyfikatu HTTPS

## 21. Najczestsze problemy

### `404 page not found` z Traefika

Traefik dziala, ale nie znalazl routera dla domeny.

Sprawdz:

- czy domena w labelce `Host(...)` jest poprawna,
- czy kontener ma `traefik.enable=true`,
- czy aplikacja jest w sieci `proxy`,
- czy DNS wskazuje na IP Contabo.

### `502 Bad Gateway`

Traefik widzi kontener, ale nie moze polaczyc sie z aplikacja.

Sprawdz:

- czy aplikacja faktycznie dziala,
- czy podany jest poprawny port w `loadbalancer.server.port`,
- czy kontener aplikacji jest w sieci `proxy`,
- czy aplikacja slucha na `0.0.0.0`, a nie tylko na `localhost` wewnatrz kontenera.

### Blad certyfikatu HTTPS

Let's Encrypt nie mogl wystawic certyfikatu.

Sprawdz:

- czy rekord DNS wskazuje na IP Contabo,
- czy port `80` jest otwarty,
- czy port `443` jest otwarty,
- czy Traefik dziala,
- czy plik `acme.json` istnieje i ma uprawnienia `600`.

### Aplikacja dziala po IP/porcie, ale nie przez domene

Najczesciej problemem sa labelki Traefika albo brak sieci `proxy`.

Sprawdz:

```bash
docker inspect NAZWA_KONTENERA
```

Kontener powinien byc podpiety do sieci `proxy`.

## 22. Zalecana kolejnosc migracji

1. Przygotuj Contabo VPS: Docker, firewall, siec `proxy`.
2. Przygotuj `/opt/traefik` i dostep SSH dla GitHub Actions.
3. Ustaw sekrety `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`, `ACME_EMAIL`.
4. Zrob push do `main` i uruchom deploy Traefika przez pipeline.
5. Przenies jedna najmniej ryzykowna aplikacje testowo.
6. Ustaw jej subdomene na IP Contabo.
7. Sprawdz HTTPS i certyfikat.
8. Przenies pozostale aplikacje.
9. Przenies bazy danych i pliki uzytkownikow.
10. Sprawdz logowanie, formularze, API, uploady i maile.
11. Dopiero po pelnym tescie wylacz stare rzeczy na Hostinger.
