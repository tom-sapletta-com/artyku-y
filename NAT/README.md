# Dokumentacja konfiguracji routingu i NAT na serwerze AlmaLinux

## Problem początkowy

Serwer z systemem AlmaLinux posiadał dwie karty sieciowe:
- **eno1**: konfiguracja przez DHCP, posiada dostęp do internetu (X.X.X.X/24)
- **eno2**: statyczna konfiguracja IP (192.168.1.1/24), działa jako serwer DHCP dla sieci lokalnej

Celem było skonfigurowanie serwera jako routera/bramy NAT, aby urządzenia w sieci lokalnej (192.168.1.0/24) mogły korzystać z internetu przez interfejs eno1.

## Zidentyfikowane problemy i rozwiązania

### 1. Dwie bramy domyślne
Na początku serwer miał skonfigurowane dwie bramy domyślne - jedną przez eno1 i drugą przez eno2. Brama domyślna na interfejsie eno2 była ustawiona na ten sam adres IP, co adres samego interfejsu (192.168.1.1), co powodowało zapętlenie routingu.

**Rozwiązanie**: Usunięto bramę domyślną z interfejsu eno2, pozostawiając tylko jedną trasę domyślną przez eno1.
```bash
sudo ip route del default via 192.168.1.1 dev eno2
sudo nmcli connection modify eno2 ipv4.gateway ""
```

### 2. Przekazywanie pakietów IP (IP Forwarding)
Domyślnie system Linux ma wyłączone przekazywanie pakietów IP między interfejsami.

**Rozwiązanie**: Włączono przekazywanie pakietów IP:
```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```

### 3. Filtrowanie odwrotnej ścieżki (Reverse Path Filtering)
Linux domyślnie włącza filtrowanie odwrotnej ścieżki (rp_filter), które blokuje pakiety, gdy trasa powrotna nie prowadzi przez ten sam interfejs.

**Rozwiązanie**: Wyłączono rp_filter dla interfejsów:
```bash
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.default.rp_filter=0
sudo sysctl -w net.ipv4.conf.eno1.rp_filter=0
sudo sysctl -w net.ipv4.conf.eno2.rp_filter=0
```

Te zmiany dodano również do pliku `/etc/sysctl.conf` dla trwałości.

### 4. Konfiguracja NAT (Network Address Translation)
Konieczne było skonfigurowanie NAT, aby pakiety z sieci lokalnej mogły być tłumaczone na adres publiczny.

**Rozwiązanie**: Skonfigurowano reguły iptables dla MASQUERADE i przekazywania:
```bash
sudo iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
sudo iptables -A FORWARD -i eno2 -o eno1 -j ACCEPT
sudo iptables -A FORWARD -i eno1 -o eno2 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### 5. Firewall (firewalld)
Firewalld aktywny na serwerze mógł blokować przekazywanie pakietów.

**Rozwiązanie**: Tymczasowo wyłączono firewalld lub skonfigurowano go, aby umożliwiał masquerade i przekazywanie pakietów:
```bash
sudo systemctl stop firewalld
# lub
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --permanent --zone=public --add-interface=eno1
sudo firewall-cmd --permanent --zone=internal --add-interface=eno2
sudo firewall-cmd --permanent --zone=internal --set-target=ACCEPT
sudo firewall-cmd --reload
```

## Skrypt konfiguracyjny

Poniżej znajduje się skrypt, który automatycznie skonfiguruje routing i NAT na serwerze AlmaLinux z dwoma interfejsami sieciowymi:

```bash
#!/bin/bash

# Skrypt do konfiguracji routingu i NAT na serwerze AlmaLinux
# Autor: Administrator
# Data: 18.03.2025

# Zmienne konfiguracyjne - dostosuj do własnych potrzeb
EXTERNAL_IF="eno1"      # Interfejs z dostępem do internetu
INTERNAL_IF="eno2"      # Interfejs do sieci lokalnej
INTERNAL_NET="192.168.1.0/24"  # Adres sieci lokalnej

# Funkcja do sprawdzania błędów
check_error() {
    if [ $? -ne 0 ]; then
        echo "Błąd: $1"
        exit 1
    fi
}

# Sprawdź czy skrypt jest uruchomiony z uprawnieniami roota
if [ "$(id -u)" -ne 0 ]; then
    echo "Ten skrypt musi być uruchomiony jako root"
    exit 1
fi

echo "Rozpoczynam konfigurację routingu i NAT..."

# 1. Usuń bramę domyślną z interfejsu wewnętrznego (jeśli istnieje)
echo "Usuwanie bramy domyślnej z interfejsu $INTERNAL_IF..."
ip route show | grep -q "default via .* dev $INTERNAL_IF"
if [ $? -eq 0 ]; then
    ip route del default dev $INTERNAL_IF
    nmcli connection modify $INTERNAL_IF ipv4.gateway ""
    check_error "Nie można usunąć bramy domyślnej z $INTERNAL_IF"
fi

# 2. Włącz przekazywanie pakietów IP
echo "Włączanie przekazywania pakietów IP..."
echo 1 > /proc/sys/net/ipv4/ip_forward
check_error "Nie można włączyć przekazywania pakietów IP"

# Dodaj konfigurację do sysctl.conf, jeśli jeszcze nie istnieje
grep -q "net.ipv4.ip_forward = 1" /etc/sysctl.conf
if [ $? -ne 0 ]; then
    echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
    check_error "Nie można zapisać konfiguracji przekazywania IP do sysctl.conf"
fi

# 3. Wyłącz filtrowanie odwrotnej ścieżki (rp_filter)
echo "Wyłączanie filtrowania odwrotnej ścieżki..."
sysctl -w net.ipv4.conf.all.rp_filter=0
sysctl -w net.ipv4.conf.default.rp_filter=0
sysctl -w net.ipv4.conf.$EXTERNAL_IF.rp_filter=0
sysctl -w net.ipv4.conf.$INTERNAL_IF.rp_filter=0
check_error "Nie można wyłączyć filtrowania odwrotnej ścieżki"

# Dodaj konfigurację do sysctl.conf, jeśli jeszcze nie istnieje
grep -q "net.ipv4.conf.all.rp_filter = 0" /etc/sysctl.conf
if [ $? -ne 0 ]; then
    cat >> /etc/sysctl.conf << EOL
# Wyłączenie filtrowania odwrotnej ścieżki dla NAT/routingu
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.$EXTERNAL_IF.rp_filter = 0
net.ipv4.conf.$INTERNAL_IF.rp_filter = 0
EOL
    check_error "Nie można zapisać konfiguracji rp_filter do sysctl.conf"
fi

# 4. Skonfiguruj reguły iptables dla NAT
echo "Konfigurowanie reguł iptables dla NAT..."
# Wyczyść istniejące reguły
iptables -F
iptables -t nat -F
iptables -X
iptables -t nat -X

# Ustaw domyślne polityki
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# Dodaj reguły NAT i przekazywania
iptables -t nat -A POSTROUTING -s $INTERNAL_NET -o $EXTERNAL_IF -j MASQUERADE
iptables -A FORWARD -i $INTERNAL_IF -o $EXTERNAL_IF -j ACCEPT
iptables -A FORWARD -i $EXTERNAL_IF -o $INTERNAL_IF -m state --state RELATED,ESTABLISHED -j ACCEPT
check_error "Nie można skonfigurować reguł iptables"

# 5. Zapisz reguły iptables
echo "Zapisywanie reguł iptables..."
if ! command -v iptables-save &> /dev/null; then
    yum install -y iptables-services
    check_error "Nie można zainstalować iptables-services"
fi

iptables-save > /etc/sysconfig/iptables
systemctl enable iptables
check_error "Nie można zapisać reguł iptables"

# 6. Konfiguracja firewalld (opcjonalnie)
echo "Sprawdzanie statusu firewalld..."
if systemctl is-active --quiet firewalld; then
    echo "Konfigurowanie firewalld..."
    firewall-cmd --permanent --zone=public --add-masquerade
    firewall-cmd --permanent --zone=public --add-interface=$EXTERNAL_IF
    firewall-cmd --permanent --zone=internal --add-interface=$INTERNAL_IF
    firewall-cmd --permanent --zone=internal --set-target=ACCEPT
    firewall-cmd --reload
    check_error "Nie można skonfigurować firewalld"
else
    echo "Firewalld nie jest aktywny, pomijam konfigurację."
fi

# 7. Załaduj zmiany sysctl
echo "Ładowanie zmian sysctl..."
sysctl -p
check_error "Nie można załadować zmian sysctl"

echo "Konfiguracja zakończona pomyślnie!"
echo "Urządzenia w sieci $INTERNAL_NET powinny teraz mieć dostęp do internetu przez interfejs $EXTERNAL_IF"

# Podstawowe testy
echo ""
echo "Wykonywanie podstawowych testów..."
echo "Stan przekazywania pakietów IP: $(cat /proc/sys/net/ipv4/ip_forward)"
echo "Konfiguracja rp_filter:"
sysctl -a | grep rp_filter | grep -E "all|default|$EXTERNAL_IF|$INTERNAL_IF"
echo "Reguły NAT:"
iptables -t nat -L -v
echo ""
echo "Aby przetestować połączenie, podłącz urządzenie do sieci $INTERNAL_NET i spróbuj uzyskać dostęp do internetu."
```

## Sposób użycia skryptu

1. Zapisz powyższy skrypt do pliku np. `configure-nat.sh`
2. Nadaj uprawnienia do wykonania:
   ```bash
   chmod +x configure-nat.sh
   ```
3. Uruchom skrypt z uprawnieniami roota:
   ```bash
   sudo ./configure-nat.sh
   ```

## Weryfikacja działania

Po uruchomieniu skryptu można zweryfikować poprawność konfiguracji:

1. Sprawdź tablicę routingu:
   ```bash
   ip route
   ```
   Powinna być tylko jedna brama domyślna przez interfejs zewnętrzny.

2. Sprawdź przekazywanie pakietów IP:
   ```bash
   cat /proc/sys/net/ipv4/ip_forward
   ```
   Powinno zwrócić `1`.

3. Sprawdź ustawienia rp_filter:
   ```bash
   sysctl -a | grep rp_filter
   ```
   Wartości dla interfejsów powinny być `0`.

4. Sprawdź reguły iptables:
   ```bash
   sudo iptables -t nat -L -v
   sudo iptables -L FORWARD -v
   ```
   Powinny być widoczne reguły MASQUERADE i FORWARD.

5. Przetestuj połączenie internetowe z urządzenia w sieci lokalnej.

## Rozwiązywanie problemów

Jeśli po uruchomieniu skryptu nadal występują problemy:

1. Sprawdź logi systemowe:
   ```bash
   journalctl -xe
   ```

2. Użyj tcpdump do analizy ruchu sieciowego:
   ```bash
   tcpdump -i eno1 -n
   tcpdump -i eno2 -n
   ```

3. Sprawdź, czy firewalld nie blokuje przekazywania pakietów:
   ```bash
   sudo systemctl status firewalld
   ```

## Uwagi

- Skrypt można dostosować do własnych potrzeb, zmieniając nazwy interfejsów i adres sieci wewnętrznej w sekcji zmiennych konfiguracyjnych.
- W przypadku zmiany adresacji sieci lub interfejsów, skrypt należy uruchomić ponownie z zaktualizowanymi parametrami.
- Dla zwiększenia bezpieczeństwa, warto rozważyć bardziej restrykcyjne reguły firewall, ograniczające dostęp do serwera i sieci lokalnej.
