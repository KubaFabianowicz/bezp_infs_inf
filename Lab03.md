0. Pełny reset ustawień routera:

/system reset-configuration no-defaults=yes skip-backup=yes

1. Utworzenie interfejsu bridge (LAN dla ehter1)

/interface bridge
add name=bridge-lan

/interface bridge port
add bridge=bridge-lan interface=ether1
add bridge=bridge-lan interface=ether2

/ip address
add address=192.168.88.254/24 interface=bridge-lan network=192.168.88.0


2. Klonowanie adresu MAC karty komputera na port WAN (ether3)

/interface ethernet
set [ find default-name=ether3 ] mac-address=00:e0:4c:68:08:58   


3. Konfiguracja WAN — klient DHCP na interfejsie ether3

/ip dhcp-client
add interface=ether3 disabled=no use-peer-dns=yes use-peer-ntp=yes


4. Uruchomienie serwera DNS oraz serwera DHCP dla sieci LAN (192.168.88.100-200)

/ip dns
set allow-remote-requests=yes servers=8.8.8.8,1.1.1.1

/ip pool
add name=dhcp-pool-lan ranges=192.168.88.100-192.168.88.200

/ip dhcp-server
add address-pool=dhcp-pool-lan interface=bridge-lan name=dhcp-lan disabled=no

/ip dhcp-server network
add address=192.168.88.0/24 dns-server=192.168.88.254 gateway=192.168.88.254

5. Maskarada (NAT) na interfejs wychodzący WAN (ether3)

/ip firewall nat
add chain=srcnat out-interface=ether3 action=masquerade comment="Maskarada LAN -> WAN"

6. Konfiguracja firewall:

/ip firewall filter

### Akceptacja połączeń established i related
add chain=input connection-state=established,related action=accept 

### Odrzucanie pakietów invalid
add chain=input connection-state=invalid action=drop

### Dopuszczenie ruchu niezbędnego do uzyskania dzierżawy DHCP na interfejsie WAN (UDP 68)
add chain=input in-interface=ether3 protocol=udp src-port=67 dst-port=68 action=accept

### Dopuszczenie zapytań DNS dla wszystkich klientów sieci LAN (UDP oraz TCP port 53)
add chain=input in-interface=bridge-lan protocol=udp dst-port=53 action=accept 
add chain=input in-interface=bridge-lan protocol=tcp dst-port=53 action=accept 

### Dopuszczenie ruchu DHCP (żądania klientów) oraz ICMP (ping) z sieci LAN
add chain=input in-interface=bridge-lan protocol=udp dst-port=67 action=accept 
add chain=input in-interface=bridge-lan protocol=icmp action=accept

### Zezwolenie na zarządzanie routerem tylko z zaufanego adresu IP w LAN
add chain=input in-interface=bridge-lan src-address=192.168.88.100 protocol=tcp dst-port=8291,22,80 action=accept

### Blokowanie pozostałego ruchu przychodzącego od strony WAN
add chain=input in-interface=ether3 action=drop \
    comment="Blokada pozostalego ruchu z WAN"

7/8. Końcowe odrzucenie pozostałego ruchu (blokuje m.in. próby zarządzania z innych IP w LAN)
add chain=input action=drop \
    comment="Domyslne odrzucenie pozostalego ruchu do routera"

9. Skonfiguruj przekierowanie z portu TCP 2137 na adres WAN routera

/ip firewall nat
add chain=dstnat in-interface=ether3 protocol=tcp dst-port=2137 \
    action=dst-nat to-addresses=192.168.88.100 to-ports=22 
    
/ip firewall filter
add chain=forward connection-state=established,related action=accept

add chain=forward connection-state=invalid action=drop

add chain=forward in-interface=bridge-lan out-interface=ether3 action=accept

add chain=forward in-interface=ether3 protocol=tcp dst-address=192.168.88.100 dst-port=22 \
    connection-nat-state=dstnat action=accept 
    
add chain=forward action=drop

10. Skonfiguruj mechanizm port-knocking

/ip firewall filter

# Połączenie TCP na port 31337
add chain=input in-interface=ether3 protocol=tcp dst-port=31337 \
    action=add-src-to-address-list address-list=knock_step1 address-list-timeout=5s

# Pakiet ICMP Echo Request (type 8:0) o rozmiarze 1306 B (lub 1334 B z nagłówkami)
add chain=input in-interface=ether3 protocol=icmp icmp-options=8:0 packet-size=1306 \
    src-address-list=knock_step1 \
    action=add-src-to-address-list address-list=knock_step2 address-list-timeout=5s

add chain=input in-interface=ether3 protocol=icmp icmp-options=8:0 packet-size=1334 \
    src-address-list=knock_step1 \
    action=add-src-to-address-list address-list=knock_step2 address-list-timeout=5s

# Krok 3: Połączenie TCP na port 1337 -> dodanie IP do listy knock_step3 na 5 sekund
add chain=input in-interface=ether3 protocol=tcp dst-port=1337 \
    src-address-list=knock_step2 \
    action=add-src-to-address-list address-list=knock_step3 address-list-timeout=5s

# Krok 4: Pakiet ICMP Echo Request o rozmiarze 26 B (54 B z nagłówkami IP+ICMP) -> dostęp na 1 godzinę
add chain=input in-interface=ether3 protocol=icmp icmp-options=8:0 packet-size=26 \
    src-address-list=knock_step3 \
    action=add-src-to-address-list address-list=winbox_allowed address-list-timeout=1h

add chain=input in-interface=ether3 protocol=icmp icmp-options=8:0 packet-size=54 \
    src-address-list=knock_step3 \
    action=add-src-to-address-list address-list=winbox_allowed address-list-timeout=1h

# Zezwolenie na dostęp do WinBox (TCP 8291) z WAN dla odblokowanych adresów IP
add chain=input in-interface=ether3 protocol=tcp dst-port=8291 \
    src-address-list=winbox_allowed action=accept
    
    


# Przesunięcie reguł blokujących input na sam koniec listy:
/ip firewall filter
move [find chain=input comment="Blokada pozostalego ruchu z WAN"]
move [find chain=input comment="Domyslne odrzucenie pozostalego ruchu do routera"]
