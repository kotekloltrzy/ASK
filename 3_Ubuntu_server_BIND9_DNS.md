<style>
code {
  color: #228B22;  /* zielony, jak zmienne bash */
  font-family: monospace;
  font-size: 0.9em;
  padding: 0 3px;            /* mały odstęp */
  border-radius: 4px;
}
</style>

# Instalacja i konfiguracja serwera DNS w Ubuntu Server 24.04.2 LTS

> <b>UWAGA</b>  
Niezbędna jest kontrola i ewentualna poprawa ustawiń konfiguracyjnych serwera DHCP na maszynie SerwerDHCP. Należy otworzyć do edycji plik <code>/etc/dhcp/dhcpd.conf</code>
> ```bash
> sudo nano /etc/dhcp/dhcpd.conf
> ```
> 
> W pliku tym należy odnaleźć wskazane tu linie i - jeżeli trzeba - zmienić ich zawartość na poniższą:
> ```bash 
> option domain-name "a175189.local"; # 175189 - numer albumu
> option domain-name-servers 192.168.10.1;
> ```
> Następnie należy odnaleźć linie konfigurujące podsieć 192.168.10.0 i również - jeżeli to niezbędne - dostosować konfigurację podsieci do poniższych wartości:
> ```bash
> subnet 192.168.10.0 netmask 255.255.255.0 {
>   range 192.168.10.100 192.168.10.200;
>   option domain-name "a175189.local";
>   option domain-name-servers 192.168.10.1;
>   option routers 192.168.10.1;
>   option broadcast-address 192.168.10.255;
> }
> ```
> Następnie restartujemy usługę serwer DHCP:
> ```bash
> sudo systemctl restart isc-dhcp-server
> ```

## 1. Wprowadzenie do DNS

### Co to jest DNS

<b>DNS ([Domain Name System](https://pl.wikipedia.org/wiki/Domain_Name_System))</b> to system, który tłumaczy nazwy domenowe (np. <code>a175189.local</code>) na adresy IP (np. <code>192.168.10.1</code>), umożliwiając komunikację w sieci bez konieczności zapamiętywania numerów IP.

### Jak działa DNS

- Klient (np. przeglądarka) wysyła zapytanie o nazwę domenową.
- Serwer DNS zwraca odpowiadający jej adres IP.
- Klient łączy się z serwerem pod wskazanym adresem IP.

### Typy serwerów DNS

- Rekurencyjny (resolver) – wysyła zapytania do innych serwerów, aż znajdzie odpowiedź.
- Autorytatywny – posiada oficjalne dane dla określonej domeny (np. <code>a175189.local</code>) i udziela ostatecznych odpowiedzi.
- Cache’ujący – przechowuje w pamięci wyniki zapytań dla szybszej odpowiedzi.

### Strefy DNS

<b>Strefa DNS (zone)</b> to wydzielony fragment przestrzeni nazw domenowych, za który odpowiada konkretny serwer DNS.
Każda strefa opisuje rekordy (nazwy hostów, adresy IP, aliasy, serwery poczty itp.) w swoim zakresie.

W systemie BIND (serwerze DNS używanym w Ubuntu) wyróżniamy kilka typów stref, które określają sposób zarządzania danymi:

<table>
<tr><td><b>Typ strefy</b></td><td><b>Opis</b></td><td><b>Typowe zastosowanie</b></td></tr>
<tr><td><b>master</b> (strefa główna / primary)</td><td>Serwer główny, który przechowuje oryginalne dane DNS w pliku strefy. Jest autorytatywny dla swojej domeny</td><td>Pliki w /etc/bind/db.*, np. db.a175189.local</td><tr>
<tr><td><b>slave</b> (strefa podrzędna / secondary)</td><td>Kopia danych z serwera głównego (master). Dane są synchronizowane automatycznie przez AXFR/IXFR.</td><td>Zapasowy serwer DNS w sieci</td></tr>
<tr><td><b>forward</b></td><td>Nie przechowuje danych — przekazuje zapytania do innego serwera DNS</td><td>Lokalny cache lub przekierowanie zapytań do serwera ISP</td></tr>
<tr><td><b>hint</b></td><td>Zawiera listę tzw. root serwerów, używaną przy starcie usługi</td><td>named.conf.default-zones</td></tr>
<tr><td><b>stub</b></td><td>Zawiera tylko rekordy NS i minimalne dane o innej strefie</td><td>	Rzadziej używane, głównie w złożonych sieciach</td></tr>
<tr><td><b>reverse zone</b> (strefa odwrotna)</td><td>Nie jest osobnym typem w BIND — to po prostu zwykła strefa type master lub type slave, której nazwa kończy się na in-addr.arpa (dla IPv4) lub ip6.arpa (dla IPv6). Służy do tłumaczenia adresów IP na nazwy domenowe</td><td>np. 10.168.192.in-addr.arpa</td></tr>
</table>


### DNS w Ubuntu przed instalacją BIND9

W systemie Ubuntu działa lokalny stub resolver (systemd-resolved), który nasłuchuje na adresie 127.0.0.53. Ten stub resolver nie jest pełnoprawnym serwerem DNS — nie przechowuje własnych stref ani rekordów, tylko pośredniczy w zapytaniach. Kiedy np. wpiszesz <code>nslookup www.onet.pl</code> lub otworzysz stronę w przeglądarce, stub resolver wysyła zapytanie do serwerów DNS dostawcy internetu lub tych ustawionych przez DHCP. Odpowiedzi otrzymujesz w trybie non-authoritative, bo nie pochodzą z autorytatywnego serwera domeny. Dzięki temu Ubuntu potrafi rozwiązywać nazwy publiczne (<code>www.onet.pl</code>, <code>www.google.com</code>) nawet jeśli nie masz własnego serwera DNS. 
Ograniczenia: nie możesz definiować własnych domen lokalnych (np. <code>a175189.local</code>) ani w pełni kontrolować odpowiedzi DNS bez własnego serwera.  
Informacje o konfiguracji stuba są zawarte w <code>/etc/resolv.conf</code>

```bash
cat /etc/resolv.conf

# Zawartość pliku
nameserver 127.0.0.53
options edns0 trust-ad
search olsztyn.vectranet.pl
```

### Przykład działania lokalnego stuba resolvera 

#### 1. Pytanie o adres IP witryny internetowej

```bash
nslookup www.onet.pl
Server:         127.0.0.53
Address:        127.0.0.53#53
```

Polecenie <code>nslookup www.onet.pl</code> działa. Odpowiedź jest nieautorytatywna, bo stub resolver nie ma tej informacji u siebie. Jest tylko przekaźnikiem zapytań do zewnętrznych serwerów DNS. W tym przypadku do <code>olsztyn.vectranet.pl</code>

```bash
Non-authoritative answer:
Name:   www.onet.pl
Address: 13.227.146.123
Name:   www.onet.pl
Address: 13.227.146.90
Name:   www.onet.pl
Address: 13.227.146.122
Name:   www.onet.pl
Address: 13.227.146.19
```

#### 2. Pytanie o adres IP domeny lokalnej

```bash
nslookup a175189.local
```

Wynik tego polecenia to:

```bash
;; Got SERVFAIL reply from 127.0.0.53
Server:         127.0.0.53
Address:        127.0.0.53#53

** server can't find a175189.local: SERVFAIL
```

Stub próbował zapytać o domenę a175189.local, ale żaden serwer DNS w Internecie jej nie zna. W domenie .local zapytania nie są przekazywane do Internetu. Stub nie ma autorytatywnych danych o abc.local, więc zwraca błąd SERVFAIL. Dopiero po zainstalowaniu BIND9 i zdefiniowaniu własnej strefy lokalne domeny takie jak np. a175189.local lub abc.local czy xyz.local system będzie umiał rozwiązywać takie nazwy.  
> <b>Wniosek</b>  
Stub resolver (127.0.0.53) to tylko pośrednik.
Działa poprawnie dla nazw znanych w Internecie,
ale dla nazw lokalnych potrzebny jest własny serwer DNS (BIND9).

## 2. Instalacja serwera DNS (BIND9)

### Krok 1: Aktualizacja systemu

```bash
sudo apt update && sudo apt upgrade -y
```

Powyższy zestaw poleceń odświeża listę pakietów z repozytoriów (<code>sudo apt update</code>) — czyli pobiera aktualny spis dostępnych wersji. Nic nie instaluje. Należy go wykonać zawsze przed instalacją czegokolwiek nowego. Natomiast <code>sudo apt upgrade</code> aktualizuje zainstalowane pakiety do najnowszych wersji dostępnych w repozytoriach (bez zmiany zależności). Wykonanie obu poleceń pozwala na utrzymanie systemu na bieżąco i bez błędów zależności.

### Krok 2: Instalacja pakietu BIND9

```bash
sudo apt install bind9 bind9utils bind9-doc -y
```

### Krok 3: Ograniczenie działalności DNS tylko do IPv4 (opcjonalny)

- Otwieramy do edycji plik <code>/etc/default/named</code>:

  ```bash
  sudo nano /etc/default/named
  ```

- znajdujemy w pliku następującą linię:

  ```bash
  OPTIONS="-u bind"
  ```

- i zamieniamy ją na:

  ```bash
  OPTIONS="-u bind -4"
  ```

- zapisujemy i zamykamy plik
- restartujemy serwer DNS

  ```bash
  sudo systemctl restart named
  ```

### Krok 4: Sprawdzenie statusu usługi

```bash
sudo systemctl status named

# Jeśli działa – powinieneś zobaczyć m. in. active (running)
```

## 3. Konfiguracja podstawowa BIND9

### Pliki konfiguracyjne

<table>
<tr><td>Główne pliki:</td><td></td></tr>  
<tr><td>/etc/bind/named.conf</td><td>główny plik konfiguracyjny</td></tr>
<tr><td>/etc/bind/named.conf.options</td><td>opcje globalne (np. forwarding)</td></tr>
<tr><td>/etc/bind/named.conf.local</td><td>definicje lokalnych stref</td></tr>
<tr><td>/var/cache/bind/</td><td>katalog dla plików stref</td></tr>
</table>

## 4. Przykład: Strefa dla domeny <i>a175189.local</i>

### Krok 1: Definicja strefy w <code>named.conf.local</code>

Otwieramy plik:

```bash
sudo nano /etc/bind/named.conf.local
```

W pliku dodaj następującą zawartość:
```bash
zone "a175189.local" {
    type master;
    file "/etc/bind/db.a175189.local";
};

zone "10.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.10";
};
```

### Krok 2: Utwórz plik strefy normalnej (forward zone)

```bash
sudo cp /etc/bind/db.local /etc/bind/db.a175189.local
sudo nano /etc/bind/db.a175189.local
```

Proszę zastąpić zawartość pliku <code>/etc/bind/db.a175189.local</code> poniższą treścią:

```bash
$TTL    604800
@       IN      SOA     ns1.a175189.local. admin.a175189.local. (
                        1         ; Serial
                        604800    ; Refresh
                        86400     ; Retry
                        2419200   ; Expire
                        604800 )  ; Negative Cache TTL
;
@       IN      NS      ns1.a175189.local.
@       IN      A       192.168.10.1
ns1     IN      A       192.168.10.1
server1 IN      A       192.168.10.1
abc     IN      A       192.168.10.1
xyz     IN      A       192.168.10.1
www     IN      CNAME   server1
```

### Krok 3: Utwórz plik strefy odwrotnej (reverse zone)

```bash
sudo cp /etc/bind/db.127 /etc/bind/db.192.168.10
sudo nano /etc/bind/db.192.168.10
```
Proszę zastąpić zawartość pliku <code>/etc/bind/db.192.168.10</code> poniższą treścią:

```bash
$TTL    604800
@       IN      SOA     ns1.a175189.local. admin.a175189.local. (
                        1
                        604800
                        86400
                        2419200
                        604800 )
;
@       IN      NS      ns1.a175189.local.
1       IN      PTR     ns1.a175189.local.
```

### Krok 4: Sprawdzenie konfiguracji

```bash
sudo named-checkconf
sudo named-checkzone a175189.local /etc/bind/db.a175189.local
sudo named-checkzone 10.168.192.in-addr.arpa /etc/bind/db.192.168.10
```

Wynik powinien zawierać komunikat:

```bash
OK
```

### Krok 5: Aliasy dla hostów wirtualnych

Aby można wygodnie odwoływać się do hostów wirtualnych należy w plikach konfiguracyjnych tych hostów dodać odpowiednie aliasy.  
Dla hosta `abc.local`.
Otwieramy do edycji plik `/etc/apache2/sites-available/abc.local.conf`:

```bash
sudo nano /etc/apache2/sites-available/abc.local.conf
```

i dopisujemy alias, uwzględniający domenę DNS

```bash
<VirtualHost *:80>
        ServerName abc.local
        ServerAlias abc.a175189.local # Dopisany alias dla hosta abc.local
        DocumentRoot /var/www/abc.local

        <Directory /var/www/abc.local>
                Options Indexes FollowSymLinks
                AllowOverride All
                Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/abc_error.log
        CustomLog ${APACHE_LOG_DIR}/abc_access.log combined
</VirtualHost>
```

Dla hosta `xyz.local`.
Otwieramy do edycji plik `/etc/apache2/sites-available/xyz.local.conf`:

```bash
sudo nano /etc/apache2/sites-available/xyz.local.conf
```

i dopisujemy alias, uwzględniający domenę DNS

```bash
<VirtualHost *:80>
        ServerName xyz.local
        ServerAlias xyz.a175189.local # Dopisany alias dla hosta abc.local
        DocumentRoot /var/www/xyz.local

        <Directory /var/www/xyz.local>
                Options Indexes FollowSymLinks
                AllowOverride All
                Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/xyz_error.log
        CustomLog ${APACHE_LOG_DIR}/xyz_access.log combined
</VirtualHost>
```

### Krok 6: Restart usługi

>  <b>UWAGA! Uruchamianie i kontrola usługi DNS (BIND9)</b>  
W nowszych wersjach Ubuntu (np. 24.04 LTS) serwer DNS BIND9 działa jako usługa systemowa named.service.
Dawna jednostka bind9.service została zachowana jedynie jako alias ze względów kompatybilności, dlatego nie należy jej używać w poleceniach <code>systemctl</code>.  
<b>Podstawowe polecenia</b>:  
> <table>
> <tr><td><b>Czynność</b></td><td><b>Polecenie</b></td></tr>
> <tr><td>Uruchomienie usługi</td><td><code>sudo systemctl start named</code></td></tr>
> <tr><td>Zatrzymanie usługi</td><td><code>sudo systemctl stop named</code></td></tr>
> <tr><td>Restart usługi (np. po zmianach w konfiguracji)</td><td><code>sudo systemctl restart named</code></td></tr>
> <tr><td>Sprawdzenie statusu usługi</td><td><code>sudo systemctl status named</code></td></tr>
> <tr><td>Włączenie usługi przy starcie systemu</td><td><code>sudo systemctl enable named</code></td></tr>
> <tr><td>Wyłączenie autostartu</td><td><code>sudo systemctl disable named</code></td></tr>
> </table>

A zatem restartujemy usługę oraz włączamy autostart serwera DNS

```bash
sudo systemctl restart named
sudo systemctl enable named
```

## 5. Testowanie serwera DNS

lokalnie, z SerwerDHCP:

```bash
dig @192.168.10.1 a175189.local
dig @192.168.10.1 www.a175189.local
dig -x 192.168.10.1
```
Te polecenia testują, czy BIND odpowiada prawidłowo i czy pliki stref są wczytane:

#### Test strefy głównej (czy SOA i NS są poprawne)
```bash
dig @192.168.10.1 a175189.local
```

#### Test konkretnego hosta (rekord A)
```bash
dig @192.168.10.1 www.a175189.local
dig @192.168.10.1 abc.a175189.local
dig @192.168.10.1 xyz.a175189.local
```

#### Test odwrotnego DNS (czy reverse zone działa)
```bash
dig -x 192.168.10.1
```

#### Weryfikacja składni strefy (opcjonalnie)
```bash
sudo named-checkzone a175189.local /etc/bind/db.a175189.local
```

jeśli w każdym przypadku otrzymasz odpowiedź status: NOERROR i rekordy się zgadzają, to BIND działa.

#### Sprawdzenie aktualnej konfiguracji DNS w systemie

Sprawdź, co teraz widzi <code>systemd-resolved</code>:

```bash
resolvectl status
```

Zwróć uwagę na:
```bash
DNS Servers: 192.168.10.1 # powinien być widoczny,
DNS Domain: a175189.local # twoja domena,
resolv.conf mode: # zapewne stub.
```

Sprawdź, czy `curl` i `nslookup` trafiają do twojego DNS:

```bash
nslookup abc.a175189.local
curl http://abc.a175189.local
```
#### Wyłączenie stub resolvera i przejście na klasyczny /etc/resolv.conf

1. Wyłącz usługę `systemd-resolved`:
   ```bash
   sudo systemctl disable systemd-resolved
   sudo systemctl stop systemd-resolved
   ```

2. Usuń link symboliczny `/etc/resolv.conf`:
   ```bash
   sudo rm /etc/resolv.conf
   ```

3. Utwórz nowy, klasyczny plik `/etc/resolv.conf`:
   ```bash
   sudo nano /etc/resolv.conf
   ```

   Wpisz do pliku następującą zawartość:
   ```bash
   nameserver 192.168.10.1
   search a175189.local
   ```

#### Weryfikacja po zmianie

Po restarcie maszyny `shutdown -h now`, sprawdź:

```bash
cat /etc/resolv.conf
nslookup abc.a175189.local
dig abc.a175189.local
curl http://abc.a175189.local
```

## Konfiguracja maszyny KlientDHCP jako klienta DNS

KlientDHCP to również serwer Ubuntu. Domyślnie jest na nim działający stub, a my chcemy aby KlientDHCP korzystał wyłącznie z serwera DNS, działającego na SerwerDHCP.

#### Sprawdzamy wyniki działania polecenia `nslookup` dla domen lokalnych:  

Stub ich nie rozwiąże, bo to pośrednik, który odpytuje publiczne serwery DNS, które nie znają domen lokalnych. Lokalne domeny może opisać tylko lokalny serwer DNS (u nas na maszynie SerwerDHCP). Należy się zatem spodziewać komunikatów o nierozpoznanych nazwach. 

```bash
nslookup a175189.local
nslookup abc.a175189.local
```

#### Rozwiązanie siłowe - wyłączamy stub i korzystamy bezpośrednio z BIND:

> <b>Uwaga</b>  
w przypadku zastosowania tego rozwiązania należy sprawdzić, czy nie spowoduje ono problemów z innymi usługami, które mogą wymagać `/etc/resolv.conf`.

```bash
sudo systemctl disable --now systemd-resolved
sudo rm /etc/resolv.conf
echo "nameserver 192.168.10.1" | sudo tee /etc/resolv.conf
```

<b>Restartujemy sieć</b>:

```bash
sudo networkctl reconfigure enp0s3 # Trzeba się upewnić co do nazwy interfejsu
```

<b>Sprawdzamy zawartość pliku `/etc/resolv.conf`</b>:

```bash
cat /etc/resolv.conf

# Spodziewany wynik
nameserver 192.168.10.1
```
<b>Ponowne użycie polecenia `nslookup`</b>:

```bash
nslookup abc.a175189.local

# Wynik
Server:         192.168.10.1  # Odpowiedź autorytatywna
Address:        192.168.10.1#53

Name:   abc.a175189.local # Rozwiązana nazwa domenowa
Address: 192.168.10.1     # Podany odpowiadający jej adres IPv4.
                          # Wynik uzyskany z odpytania naszego serwera DNS
```

## Dodawanie kolejnych wpisów do strefy DNS (ręczne)

- na serwerze DNS otwieramy plik strefy:
  ```bash
  sudo nano /etc/bind/db.a175189.local
  ```
- dodajemy rekord dla klienta w strefie normalnej:
  ```bash
  $TTL    604800
  @       IN      SOA     ns1.a175189.local. admin.a175189.local. (
                        2         ; Serial  # Przy zmianach w pliku zwiększamy numer seryjny
                        604800    ; Refresh
                        86400     ; Retry
                        2419200   ; Expire
                        604800 )  ; Negative Cache TTL
  ;
  @       IN      NS      ns1.a175189.local.
  @       IN      A       192.168.10.1
  ns1     IN      A       192.168.10.1
  server1 IN      A       192.168.10.1
  abc     IN      A       192.168.10.1
  xyz     IN      A       192.168.10.1
  klient1 IN      A       192.168.10.200 # nowy wpis
  www     IN      CNAME   server1

  ```
  > <b>Uwaga</b>  
  > Numer seryjny zwiększamy przy każdej zmianie w pliku strefy  
  > (najlepiej w formacie YYYYMMDDnn)
- na serwerze DNS otwieramy plik strefy odwrotnej:
  ```bash
  sudo nano /etc/bind/db.192.168.10
  ```

- dodajemy rekord dla klienta w strefie odwrotnej:
  ```bash
  $TTL    604800
  @       IN      SOA     ns1.a175189.local. admin.a175189.local. (
                        2 # Zwiększamy numer seryjny
                        604800
                        86400
                        2419200
                        604800 )
  ;
  @       IN      NS      ns1.a175189.local.
  1       IN      PTR     ns1.a175189.local.
  200     IN      PTR     klient1.a175189.local.  # Nowy wpis
  ```
- restartujemy usługę:
  ```bash
  sudo systemctl reload named
  ```
- testujemy:

  ```bash
  nslookup klient1.a175189.local
  # Wynik
  Server:         192.168.10.1
  Address:        192.168.10.1#53
  
  Name:   klient1.a175189.local
  Address: 192.168.10.200
  ```

## Sprawozdanie z ćwiczeń

Przesłany jako praca domowa w zadaniu MsTeams plik o nazwie cwicz_4_XXXXXX.txt, gdzie XXXXXX to Państwa numer albumu. Zawartość pliku:  
1. Imię i nazwisko: ...   
2. Wynik polecenia `nslookup www.onet.pl` uzyskany z SerwerDHCP
3. Wynik polecenia `nslookup xyz.aXXXXXX.local` uzyskany z SerwerDHCP
4. Wynik polecenia `curl http://abc.aXXXXXX.local` uzyskany z SerwerDHCP
