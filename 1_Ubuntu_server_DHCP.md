> Administrowanie sieciami komputerowymi

# Serwer *DHCP* w Ubuntu 24.04.2 LTS  

## Uzyskiwanie podstawowych informacji o sieci w serwerze *Ubuntu*  

czyli informacje podobne do tych, które można w systemie Windows uzyskać przy pomocy polecenia *ipconfig /all*.

<details>
<summary><b>Sprawdzenie interfejsów i adresów IPv4</b></summary>

<font size="2">

```bash
ip a
```

</font>

Przykład wyniku:

<font size="2">

```bash
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:90:20:52 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.28/24 metric 100 brd 192.168.0.255 scope global dynamic enp0s3
       valid_lft 593413sec preferred_lft 593413sec
    inet6 fe80::a00:27ff:fe90:2052/64 scope link
       valid_lft forever preferred_lft forever
```

</font>

Interpretacja wyniku:

<table>
<tr><td>Informacja</td><td>Gdzie występuje w wyniku</td></tr>
<tr><td>Nazwa interfejsu</td><td>enp0s3: (na początku wiersza)</td></tr>
<tr><td>Adres IP</td><td>inet 192.168.0.28/24 → adres IP = 192.168.0.28</td></tr>
<tr><td>Maska sieci</td><td>/24 → maska = 255.255.255.0</td></tr>
<tr><td>Adres rozgłoszeniowy (broadcast)</td><td>brd 192.168.0.255</td></tr>
<tr><td>MAC adres (adres fizyczny)</td><td>link/ether 08:00:27:90:20:52</td></tr>
</table>
</details>

<details>
<summary><b>Sprawdzenie adresu sieci i bramy domyślnej</b></summary>

<font size="2">

```bash
ip route
```

</font>

Przykład wyniku:

<font size="2">

```bash
default via 192.168.0.1 dev enp0s3 proto dhcp metric 100
192.168.0.0/24 dev enp0s3 proto kernel scope link src 192.168.0.28 
```

</font>

Interpretacja wyniku:

<table>
<tr><td>Informacja</td><td>Znaczenie</td></tr>
<tr><td>Bramka domyślna (default gateway)</td><td>default via 192.168.0.1 → 192.168.0.1</td></tr>
<tr><td>Adres sieci</td><td>192.168.0.0/24 → sieć: 192.168.0.0, maska /24 (czyli 255.255.255.0)</td></tr>
<tr><td>Adres IP urządzenia w sieci</td><td>src 192.168.0.28</td></tr>
</table>
</details>

<details>
<summary><b>Sprawdzenie serwerów DNS i domen</b></summary>  

<font size="2">

```bash
resolvectl status
```

</font>

Przykład wyniku:

<font size="2">

```bash
Global
         Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub

Link 2 (enp0s3)
    Current Scopes: DNS
         Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
       DNS Servers: 178.235.153.33 178.235.153.32
        DNS Domain: olsztyn.vectranet.pl
```

</font>

Interpretacja wyniku:

<table>
<tr><td>Informacja</td><td>Wartość</td></tr>
<tr><td>Serwery DNS</td><td>178.235.153.33, 178.235.153.32</td></tr>
<tr><td>Domena (search domain)</td><td>olsztyn.vectranet.pl</td></tr>
</table>
</details>

<details>
<summary><b>Potwierdzenie konfiguracji interfejsu</b></summary>  

<font size="2">

```bash
networkctl status enp0s3
```

</font>

Przykład wyniku:

<font size="2">

```bash
● 2: enp0s3
                   Link File: /usr/lib/systemd/network/99-default.link
                Network File: /run/systemd/network/10-netplan-enp0s3.network
                       State: routable (configured)
                Online state: online                                           >
                        Type: ether
                        Path: pci-0000:00:03.0
                      Driver: e1000
                      Vendor: Intel Corporation
                       Model: 82540EM Gigabit Ethernet Controller (PRO/1000 MT >
            Hardware Address: 08:00:27:90:20:52 (PCS Systemtechnik GmbH)
                         MTU: 1500 (min: 46, max: 16110)
                       QDisc: fq_codel
IPv6 Address Generation Mode: eui64
    Number of Queues (Tx/Rx): 1/1
            Auto negotiation: yes
                       Speed: 1Gbps
                      Duplex: full
                        Port: tp
                     Address: 192.168.0.28 (DHCP4 via 192.168.0.1)
                              fe80::a00:27ff:fe90:2052
                     Gateway: 192.168.0.1
                         DNS: 178.235.153.33
                              178.235.153.32
              Search Domains: olsztyn.vectranet.pl
           Activation Policy: up
         Required For Online: yes
             DHCP4 Client ID: IAID:0xe2343f3e/DUID
           DHCP6 Client DUID: DUID-EN/Vendor:0000ab11bc9be88266818d59

```

</font>

</details>


## Podstawowe informacje o *DHCP*  

> Protokół *DHCP (Dynamic Host Configuration Protocol)* działa w warstwie aplikacji (TCP/IP, porty 67/68 UDP) i służy do automatycznego przydzielania adresów IP oraz innych parametrów sieciowych (DNS, brama, maska itd.).  
Sekwencja czterech podstawowych komunikatów DHCP:  
> <table>  
> <tr><td>Faza</td><td>Nazwa wiadomości</td><td>Kierunek</td><td>Opis</td></tr>  
> <tr><td>1</td><td>DHCPDISCOVER</td><td>Klient → Broadcast</td><td>Klient wysyła zapytanie: „Czy jest tu jakiś serwer DHCP?”</td></tr>  
> <tr><td>2</td><td>DHCPOFFER</td><td>Serwer → Broadcast (lub unicast)</td><td>Serwer odpowiada ofertą: „Mogę Ci dać adres X.X.X.X”</td></tr>  
> <tr><td>3</td><td>DHCPREQUEST</td><td>Klient → Broadcast</td><td>Klient mówi: „Wybieram ofertę od tego serwera, proszę o ten adres”</td></tr>  
> <tr><td>4</td><td>DHCPACK</td><td>Serwer → Broadcast (lub unicast)<td>Serwer potwierdza: „Adres został Ci przydzielony”</td></tr>  
> </table>  
> Cały ten proces określany jest mianem *DHCP handshake*  

> 1. DHCPDISCOVER  
Wysłany przez klienta, który jeszcze nie ma IP (wysyła z adresem źródłowym 0.0.0.0 → 255.255.255.255). Ramka zawiera m.in. MAC klienta (identyfikator).  

> 2. DHCPOFFER  
Serwer odpowiada ofertą: adres IP, czas dzierżawy, DNS, brama itd. Jeśli w sieci jest więcej niż jeden serwer DHCP, klient może dostać więcej niż jedną ofertę.  

> 3. DHCPREQUEST  
Klient wybiera jedną z ofert i publicznie ogłasza swój wybór. Pozostałe serwery wycofują swoje oferty.  

> 4. DHCPACK  
Serwer potwierdza i zapisuje dzierżawę w swojej bazie. Klient konfiguruje interfejs z przydzielonym adresem.

> **Uwaga**  
>Podczas konfiguracji trzeba wskazać interfejs sieciowy, aby serwer DHCP wiedział, na której karcie sieciowej ma nasłuchiwać i rozdawać adresy IP. Jeżeli w sytemie istnieje więcej niż jeden interfejs (np. jeden do Internetu, drugi do sieci lokalnej), to trzeba wskazać ręcznie serwerowi DHCP, w której sieci ma działać. Serwer DHCP nasłuchuje zapytań od klientów (pakiety DHCPDISCOVER) przez UDP na porcie 67. Te zapytania są rozgłaszane (broadcast) w sieci lokalnej — czyli trafiają tylko do komputerów w tej samej podsieci.  
Przykład:  
Jeśli w systemie występują następujące interfejsy sieciowe:  
> <table>
> <tr><td>Interfejs</td><td>Typ sieci</td><td>Adres IP</td><td>Rola</td></tr>  
> <tr><td>ens33</td><td>NAT</td><td>10.0.2.15</td><td>Internet</td></tr>  
> <tr><td>ens37</td><td>Host-only / LAN</td><td>192.168.56.1</td><td>sieć lokalna (DHCP)</><td></tr>  
> </table>  
> To serwer DHCP powinien działać tylko na ens37, bo tam są lokalne maszyny-klienci, którym mają być przydzielone adresy IP.

## Uwagi dotyczące wykonania ćwiczenia

Do wykonania poniższych zadań niezbędne jest posiadanie dwóch maszyn wirtualnych *Oracle VirtualBox* z obrazem *Ubuntu server* w wersji *Ubuntu 24.04.2 LTS*. Pierwsza z tych maszyn (nazwa np. ServerDHCP) będzie serwerem DHCP i powinna być wyposażona w dwie karty sieciowe: pierwsza to NAT, zapewniająca łączność z Internetem, druga karta to sieć wewnętrzna, pozwalająca tylko na komunikację wewnątrz tej wewnętrznej sieci. Druga maszyna (nazwa np. KlientDHCP) będzie klientem. Jej rola ograniczy się do uzyskania od serwera DHCP adresu IP.  

## Kolejność czynności wykonywanych z maszyną *ServerDHCP*:  

<details>
<summary><b>Ustawienie w sekcji <i>Sieć</i> w <i>Ustawieniach</i> maszyny wirtualnej obu kart sieciowych</b></summary>  

1. Pierwszą kartę sieciową ustawiamy jako NAT  
2. Drugą kartę sieciową ustawiamy jako sieć wewnętrzną. Można zostawić jej domyślną nazwę, lub nadać jej własną. Ważne, żeby nazwa tej wewnętrznej sieci była identyczna w obu maszynach wirtualnych  

</details>

<details>
<summary><b>Ustawienie w sekcji <i>Sieć</i> w <i>Ustawieniach</i> maszyny wirtualnej przekierowania portów dla karty sieciowej NAT</b></summary>  

Na karcie Karty sieciowej 1 (NAT) rozwijamy opcję <i>Zaawansowane</i> i klikamy przycisk <i>Przekierowanie portów</i>. Dodajemy nową regułę:  
Nazwa: SSH  
Protokół: TCP  
Port hosta: 2222  
Port gościa: 22  
i zatwierdzamy ją.

</details>
<b>Uruchomienie maszyny wirtualnej <i>ServerDHCP</i></b>
<details>
<summary><b>Sprawdzenie wersji zainstalowanego serwera <i>Ubuntu</i></b></summary>

<font size="2">

```bash
cat /etc/os-release

# PRETTY_NAME="Ubuntu 24.04.2 LTS"
# NAME="Ubuntu"
# VERSION_ID="24.04"
# VERSION="24.04.2 LTS (Noble Numbat)"
# VERSION_CODENAME=noble
# ID=ubuntu
# ID_LIKE=debian
# HOME_URL="https://www.ubuntu.com/"
# SUPPORT_URL="https://help.ubuntu.com/"
# BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
# PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
# UBUNTU_CODENAME=noble
# LOGO=ubuntu-logo

```

</font>

Uzyskany wynik oznacza, że mamy zainstalowany w maszynie wirtualnej serwer *Ubuntu* w odpowiedniej wersji. 
</details>

<details>
<summary><b>Przypisanie statycznego adresu IP tylko dla sieci wewnętrznej (Netplan)</b></summary>

<font size="2">

```bash
sudo nano /etc/netplan/00-installer-config.yaml

# Wpisujemy

network:
  ethernets:
    enp0s3:
      dhcp4: true        # NAT - Internet
    enp0s8:
      addresses: [192.168.10.1/24]  # sieć wewnętrzna
      dhcp4: no
  version: 2

# Stosujemy nowe ustawienia

sudo netplan apply
```

</font>


</details>


<details>
<summary><b>Sprawdzenie obecności i ewentualna instalacja ssh</b></summary>  

<font size="2">

```bash
sudo systemctl status ssh

# jeżeli pakiet ten nie jest zainstalowany, otrzymamy odpowiedź podobną do poniższej:

# unit.sshservice could not be found.

# Instalujemy pakiet openssh-server:

sudo apt update
sudo apt install openssh-server

# Uruchamiamy ponownie nasz ServerDHCP

sudo reboot

# Ponownie sprawdzamy, czy w naszym serwerze Ubuntu jest zainstalowany pakiet openssh-server:

sudo systemctl status ssh
```

</font>

</details>

<details>
<summary><b>Sprawdzenie czy w naszym serwerze <i>Ubuntu</i> jest zainstalowany serwer <i>DHCP</i></b></summary>

<font size="2">

```bash
sudo systemctl status isc-dhcp-server

# Unit isc-dhcp-server.service could not be found.
```

</font>

Uzyskany wynik oznacza, że w chwili obecnej nie mamy zainstalowanego w systemie serwera *DHCP*.

</details>

<details>
<summary><b>Instalacja serwera <i>DHCP</i></b></summary>

W Ubuntu działanie serwera *DHCP* umożliwia instalacja pakietu **_isc-dhcp-server:_**

<font size="2">

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

</font>

**Wskazanie interfejsu sieciowego**  

<font size="2">

```bash
# Badamy adresy IPv4 istniejących w systemie Ubuntu interfejsów sieciowych
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:49:fa:9e brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 80417sec preferred_lft 80417sec
    inet6 fe80::a00:27ff:fe49:fa9e/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:43:ff:1f brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.1/24 brd 192.168.10.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe43:ff1f/64 scope link
       valid_lft forever preferred_lft forever


# Otwieramy do edycji plik /etc/default/isc-dhcp-server
sudo nano /etc/default/isc-dhcp-server

# Następnie w edytowanym pliku należy znaleźć linię: INTERFACESv4=""
# i wpisać nazwę interfejsu sieciowego: enp0s8
INTERFACESv4="enp0s8"

```

</font>

**Konfiguracja zakresu DHCP**  

<font size="2">

```bash
# Otwieramy do edycji główny plik konfiguracyjny:

sudo nano /etc/dhcp/dhcpd.conf

# Przykładowa konfiguracja dla sieci 192.168.10.0/24:

# Nazwa domeny i serwery DNS
option domain-name "a3.17.local";
option domain-name-servers 192.168.10.1;

# Domyślny czas dzierżawy (w sekundach)
default-lease-time 600;
max-lease-time 7200;

# Zakres adresów DHCP
subnet 192.168.10.0 netmask 255.255.255.0 {
  range 192.168.10.100 192.168.10.200;
  option routers 192.168.10.1;
  option broadcast-address 192.168.10.255;
}

# Czy ten serwer jest głównym DHCP
authoritative;

```

</font>

**Uruchomienie i autostart**  

<font size="2">

```bash
# Po zapisaniu zmian zrestartuj usługę

sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server

# Sprawdź status

sudo systemctl status isc-dhcp-server
```

</font>

</details>

## Kolejność czynności wykonywanych z maszyną *KlientDHCP*:  

<details>
<summary><b>Ustawienie w sekcji <i>Sieć</i> w <i>Ustawieniach</i> maszyny wirtualnej karty sieciowej</b></summary>  

Kartę sieciową ustawiamy jako NAT. Takie ustawienie karty sieciowej zapewnia łączność z Internetem, ale nie pozwala na łączność z gospodarzem. Łączność z Internetem będzie niezbędna do uruchomienia tej VM.   

</details>
<b>Uruchomienie maszyny <i>KlientDHCP</i></b>
<details>
<summary><b>Przypisanie dynamicznego adresu IP sieci wewnętrznej (Netplan)</b></summary>

<font size="2">

```bash

# Sprawdzamy interfejsy sieciowe

ip a

# Przypisujemy dynamiczny adres IP do właściwego interfejsu sieciowego

sudo nano /etc/netplan/00-installer-config.yaml

# Wpisujemy

network:
  ethernets:
    enp0s3:
      dhcp4: true
  version: 2

# Stosujemy nowe ustawienia

sudo netplan apply
```

</font>

</details>


<details>
<summary><b>Skrócenie startu systemu (wyłączenie cloud-init)</b></summary>

<font size="2">

```bash
sudo systemctl disable cloud-init
sudo systemctl mask cloud-init
sudo apt remove cloud-init -y

sudo systemctl disable systemd-networkd-wait-online.service
sudo systemctl mask systemd-networkd-wait-online.service

# Następnie wyłącz maszynę 

sudo shutdown -h now
```

</font>

</details>

<details>
<summary><b>Zmiana ustawienia w sekcji <i>Sieć</i> w <i>Ustawieniach</i> maszyny wirtualnej karty sieciowej</b></summary>  

Kartę sieciową ustawiamy jako sieć wewnętrzną. Ważne, żeby nazwa tej wewnętrznej sieci była identyczna z nazwą tej sieci w maszynie wirtualnej ServerDHCP.  

</details>


<details>
<summary><b>Restart maszyny</b></summary>  

<font size="2">

```bash
sudo reboot
```

</font>

</details>


## Przeglądanie logów <i>dhcpd</i> na SerwerDHCP  

<font size="2">

```bash
sudo grep dhcpd /var/log/syslog
```

</font>


## Sprawozdanie z ćwiczeń  

Przesłany jako praca domowa w zadaniu MsTeams plik o nazwie cwicz_2_XXXXXX.txt, gdzie XXXXXX to Państwa numer albumu. Zawartość pliku:  
1. Imię i nazwisko: ... 
2. Informacje o SerwerzeDHCP (interfejsy sieciowe (nazwa, adres IPv4, statyczny czy dynamiczny, rodzaj karty w VM, MAC adres))
3. Informacje o KliencieDHCP (interfejsy sieciowe (nazwa, adres IPv4, statyczny czy dynamiczny, rodzaj karty w VM, MAC adres))
4. Dowód, że serwer DHCP w SerwerDHCP działa
5. Konfiguracja DHCP
5. Logi dhcpd potwierdzające przydział i zwolnienie adresu ipv4 przez KlientDHCP

