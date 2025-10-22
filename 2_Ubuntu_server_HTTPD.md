# Instalacja i konfiguracja serwera <i>Apache2 (httpd)</i><br> w systemie <i>Ubuntu server 24.04.2 LTS</i>

## Część 1: protokół <i>http</i>

### 1. Sprawdzenie, czy serwer jest zainstalowany  

<font size = "2">

```bash
systemctl status apache2

# lub

apache2 -v

# lub

curl -I http://127.0.0.1

# lub

which apache2
```

</font>  

### 2. Instalacja serwera WWW  

<font size = "2">

```bash
sudo apt update
sudo apt install apache2
```

</font>  


### 3. Uruchomienie i sprawdzenie statusu serwera  

<font size = "2">

```bash
sudo systemctl start apache2
sudo systemctl enable apache2  # aby uruchamiał się automatycznie przy starcie systemu
sudo systemctl status apache2

```

</font>

### 4. Sprawdzenie nasłuchiwanych portów  
(proszę sprawdzić poleceniem <i>man ...</i> do czego służy polecenie <i>ss</i>)

<font size = "2">

```bash
# Standardowo Apache2 nasłuchuje na porcie 80 (http) i 443(https):

sudo ss -tulnp | grep :80
```

</font>

### 5. Próba połączenia lokalnego (tylko nagłówki strony www)  

<font size = "2">

```bash
curl -I http://127.0.0.1
```

</font>

### 6. Próba połączenia lokalnego (pełna strona www)

> Państwa zadanie polega na wyświetleniu na komputerze - gospodarzu strony www serwera <i>Apache2</i>, którą serwer wyświetla po zainstalowaniu. Początek (rozwiązania i polecenia) jest poniżej ...

<font size = "2">

```bash
curl http://127.0.0.1
```

</font>

### 7. Konfiguracja serwera <i>Apache</i>

Konfiguracja Apache2 polega na ustawieniu sposobu działania serwera WWW, czyli:  
1. jakie strony (witryny) ma obsługiwać,  
2. na jakich portach (HTTP/HTTPS) ma nasłuchiwać,  
3. gdzie znajdują się pliki stron (DocumentRoot),  
4. jakie moduły i funkcje mają być aktywne (np. PHP, SSL, proxy, rewrite),  
5. jak mają wyglądać uprawnienia i logi.  


### 8. Struktura konfiguracji w Ubuntu

Po instalacji Apache2 konfiguracja tego serwera znajduje się w:  

<font size = "2">

```bash
/etc/apache2/
```

</font>

Najważniejsze elementy:  

<table>

<tr><td>Katalog / plik</td><td>Zawartość</td></tr>
<tr><td>/etc/apache2/apache2.conf</td><td>Główny plik konfiguracyjny serwera</td></tr>
<tr><td>/etc/apache2/ports.conf</td><td>Ustawienia portów (np. 80 i 443)</td></tr>
<tr><td>/etc/apache2/sites-available/</td><td>Konfiguracje poszczególnych witryn (vhostów)</td></tr>
<tr><td>/etc/apache2/sites-enabled/</td><td>Aktywne witryny (symlinki do sites-available)</td></tr>
<tr><td>/etc/apache2/mods-available/</td><td>Moduły, które można włączyć</td></tr>
<tr><td>/etc/apache2/mods-enabled/</td><td>Moduły aktualnie włączone</td></tr>
<tr><td>/var/www/html/</td><td>Domyślny katalog z plikami strony WWW</td></tr>
</table>

### 9. Wirtualne hosty: Konfiguracja na SerwerDHCP (Apache2)  

SerwerDHCP ma adres IP:  

<font size = "2">

```bash
192.168.10.1
```

</font>

#### Katalogi dla każdej ze stron www  

Pierwsza strona www ma nazwę <i>abc.local</i>  
Druga strona www ma nazwę <i>xyz.local</i>  

<font size = "2">

```bash
sudo mkdir -p /var/www/abc.local
sudo mkdir -p /var/www/xyz.local
```

</font>

#### Nadanie utworzonym katalogom właściciela <i>www-data</i>  
(Apache2 działa jako <i>www-data</i>)  

<font size = "2">

```bash
sudo chown -R www-data:www-data /var/www/abc.local /var/www/xyz.local
```

</font>

#### Stworzenie testowych stron www  

<font size = "2">

```bash
echo "<h1>Witamy na stronie ABC.LOCAL</h1>" | sudo tee /var/www/abc.local/index.html
echo "<h1>To jest witryna XYZ.LOCAL</h1>" | sudo tee /var/www/xyz.local/index.html
```

</font>

#### Plik konfiguracyjny pierwszego wirtualnego hosta (<i>abc.local</i>):<br>/etc/apache2/sites-available/abc.local.conf  

<font size = "2">

```bash
<VirtualHost *:80>
    ServerAdmin webmaster@abc.local
    ServerName abc.local
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

</font>

#### Plik konfiguracyjny drugiego wirtualnego hosta (<i>xyz.local</i>):<br>/etc/apache2/sites-available/xyz.local.conf  

<font size = "2">

```bash
<VirtualHost *:80>
    ServerAdmin webmaster@xyz.local
    ServerName xyz.local
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

</font>

#### Aktywacja obydwu witryn  

<font size = "2">

```bash
sudo a2ensite abc.local.conf
sudo a2ensite xyz.local.conf
sudo systemctl reload apache2
```

</font>

#### Testowanie konfiguracji

<font size = "2">

```bash
sudo apache2ctl configtest
```

</font>

### 10. Wirtualne hosty (konfiguracja na KlientDHCP (bez DNS))  

#### Edycja pliku /etc/hosts

<font size = "2">

```bash
sudo nano /etc/hosts

# Dodać na końcu pliku

192.168.10.1   abc.local
192.168.10.1   xyz.local
```

</font>

#### Sprawdzenie połączenia  

<font size = "2">

```bash
ping abc.local
ping xyz.local
```

</font>

#### Sprawdzenie stron www  

<font size = "2">

```bash
curl http://abc.local
curl http://xyz.local
```

</font>


## Część 2: Protokół <i>https (http + TLS)</i>  

Pokazane są rozwiązania dla domeny <i>abc.lokal</i>  
Rozwiązania dla <i>xyz.local</i> należy zaimplementować samodzielnie  

Włącz moduł SSL w Apache2 (SerwerDHCP)  

<font size = "2">

```bash
sudo a2enmod ssl
sudo systemctl restart apache2
```

</font>

Sprawdź, czy moduł jest aktywny

<font size = "2">

```bash
sudo a2query -m ssl
```

</font>



#### Utwórz certyfikat self-signed  

<font size = "2">

```bash
sudo mkdir -p /etc/ssl/certs /etc/ssl/private
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/abc.local.key \
    -out /etc/ssl/certs/abc.local.crt \
    -subj "/C=PL/ST=Pomorskie/L=Gdansk/O=Local/OU=IT/CN=abc.local"

# Ustaw odpowiednie prawa do klucza

sudo chmod 600 /etc/ssl/private/abc.local.key
sudo chown root:root /etc/ssl/private/abc.local.key
```
  
</font>

#### Utwórz vhost SSL dla <i>abc.local</i>  
Plik /etc/apache2/sites-available/abc.local-ssl.conf  

<font size = "2">

```bash
<VirtualHost *:443>
    ServerName abc.local
    DocumentRoot /var/www/abc.local

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/abc.local.crt
    SSLCertificateKeyFile /etc/ssl/private/abc.local.key

    <Directory /var/www/abc.local>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/abc.local-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/abc.local-ssl-access.log combined
</VirtualHost>
```

</font>

#### Włącz host SSL  

<font size = "2">

```bash
sudo a2ensite abc.local-ssl
sudo systemctl reload apache2
```

</font>

#### Sprawdź nasłuch portu 443  

<font size = "2">

```bash
sudo ss -tlnp | grep 443
```

</font>

#### Test połączenia (KlientDHCP)  

<font size = "2">

```bash
curl -k https://abc.local

# Flaga -k ignoruje brak zaufanego CA, bo certyfikat jest samopodpisany.
```

</font>

#### Przeglądanie logów dostępowych <i>apache2</i>

<font size = "2">

```bash
# HTTP
# ip a na KlientDHCP, żeby sprawdzić adres IP
sudo grep "192.168.10.100" /var/log/apache2/access.log  

# HTTPS (SSL)
# plik logu dla virtualnego hosta jest zdefiniowany w pliku konfiguracyjnym tego hosta
sudo grep "192.168.10.100" /var/log/apache2/abc.local-ssl-access.log    

```

</font>


## Sprawozdanie z ćwiczeń  

Przesłany jako praca domowa w zadaniu MsTeams plik o nazwie cwicz_3_XXXXXX.txt, gdzie XXXXXX to Państwa numer albumu. Zawartość pliku:  
1. Imię i nazwisko: ...   
2. Zawartość pliku konfiguracyjnego virtualnego hosta xyz.<i>local-ssl</i>
3. Wiersze logów dostępowych apache2 potwierdzające, że z maszyny KlientDHCP dokonywano połączeń https do domen <i>abc.local</i> i <i>xyz.local</i>