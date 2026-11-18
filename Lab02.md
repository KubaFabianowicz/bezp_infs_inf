1. Skonfiguruj interfejs bridge1 na MikroTiku. Dodaj do niego porty ether1, ether2 i ether3. Sprawdź, czy wszystkie porty są aktywnymi portami mostu.

/interface bridge add name=bridge1

/interface bridge port add bridge=bridge1 interface=ether1

/interface bridge port add bridge=bridge1 interface=ether2

/interface bridge port add bridge=bridge1 interface=ether3

2. Nadajemy adres IP 192.168.254.X/25 na MikroTiku, gdzie X to końcówka adresu komputera z sali. Czy adres może być przydzielony do interfejsu etherX, jeżeli interfejsy są częścią bridge1?

TAK
/ip address add address=192.168.254.86/25 interface=bridge1

3. Jak pasywnie wykryć adresy MAC urządzeń MikroTik (z domyślną konfiguracją) podłączonych do lokalnej sieci? Podaj przykładowy adres.

Czekac na wiadomosc z mikrotika mndp na porcie 5678. W wiadomosci jest adres MAC i inne cechy danego mikrotika:

Przykladowy adres z ramki:
Routerboardc_91:03:47 (00:0c:42:91:03:47)

4. Jak sprawdzić producenta urządzenia na podstawie adresu MAC? Podaj dopasowanie do adresu MikroTika.

Pierwsze 24 bity adresu MAC to jest OUI. Mozna skorzystac z wyszukiwarki macvendors.com

5. Zmień adres MAC na losowy (bridge1).

/interface bridge set bridge1 auto-mac=no admin=mac=02:AA:BB:CC:DD:EE

6. Jak wyłączyć rozgłaszanie informacji o urządzeniach MikroTik w RouterOS?

/ip neighbor discovery-settings set discover-interface-list=none

7. Wyłączamy nieużywane serwery: bandwidth-test, mac server, mac winbox, mac ping?

/tool bandwidth-server set enabled=no
ip service disable mac-telnet
/ip service disable mac-winbox
/ip service disable mac-ping
/ip service print
/tool bandwidth-server print

8. Konfiguracja www-ssl (generowanie certyfikatu).

/certificate add name=self-signed-cert common-name=mikrotik-cert days-valid=365
/certificate sign self-signed-cert
/ip service set www-ssl certificate=self-signed-cert disabled=no


9. Wyłączenie wszystkich usług oprócz ssh lub www-ssl (do wyboru).

/ip service disable telnet
/ip service disable ftp
/ip service disable www
/ip service disable api
/ip service disable api-ssl
/ip service print

10. Konfiguracja algorytmu ed25519 w usłudze SSH.

/ip ssh set host-key-type=ed25519

11. Limitowanie dostępu do otwartych usług do zakresu adresów sieci lokalnej 192.168.254.X/25.

/ip service set ssh address=192.168.254.0/25
/ip service set www-ssl address=192.168.254.0/25
/ip service print

12. Co jeszcze można wyłączyć?

Przykładowo: 
/ip proxy set enabled=no
/ip cloud set ddns-enabled=no update-time=no

Mozna rowniez wylaczyc np adres IPv6 jezeli nieuzywany.

