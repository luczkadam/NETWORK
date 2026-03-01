#!/bin/bash

# Sprawdzenie, czy skrypt został uruchomiony z uprawnieniami roota
if [ "$EUID" -ne 0 ]; then
  echo "Uruchom ten skrypt jako root (lub przez sudo)."
  exit 1
fi

# Sprawdzenie argumentów
if [ -z "$1" ] || [ -z "$2" ]; then
  echo "Użycie: $0 <adres_IP> <prefix>"
  echo "Przykład: $0 192.168.10.50 24"
  exit 1
fi

IP=$1
PREFIX=$2

# Przeliczanie prefixu (CIDR) na pełną maskę dziesiętną (np. 24 -> 255.255.255.0)
MASK=""
REMAINING=$PREFIX
for ((i=0; i<4; i++)); do
    if [ $REMAINING -ge 8 ]; then
        MASK+="255"
        let REMAINING-=8
    elif [ $REMAINING -gt 0 ]; then
        VAL=$(( 256 - (1 << (8 - REMAINING)) ))
        MASK+="$VAL"
        REMAINING=0
    else
        MASK+="0"
    fi
    [ $i -lt 3 ] && MASK+="."
done

# --- SEKCJA BACKUPU ---
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_DIR="/etc/sysconfig/network-scripts/OLD_INTERFACE/$TIMESTAMP"

echo "=> Tworzenie kopii zapasowej obecnej konfiguracji sieci w $BACKUP_DIR..."
mkdir -p "$BACKUP_DIR"

# Kopiowanie plików ifcfg-* oraz route-* (tylko pliki, pomija katalogi, przekierowanie błędów)
find /etc/sysconfig/network-scripts/ -maxdepth 1 -type f -name "ifcfg-*" -exec cp {} "$BACKUP_DIR/" \; 2>/dev/null
find /etc/sysconfig/network-scripts/ -maxdepth 1 -type f -name "route-*" -exec cp {} "$BACKUP_DIR/" \; 2>/dev/null

echo "   Kopia zapasowa gotowa."
# ----------------------

echo "=> Konfigurowanie adresu $IP maska $MASK (z prefixu /$PREFIX) w network-scripts..."

# Wyszukanie wszystkich interfejsów, pomijając loopback i wirtualne
INTERFACES=$(ls /sys/class/net | grep -v -E '^lo$|^virbr|^vnet|^docker|^veth')

for IFACE in $INTERFACES; do
    echo "-> Tworzenie konfiguracji dla interfejsu: $IFACE"

    # Pobranie adresu MAC (HWADDR) z systemu dla danego interfejsu (z zabezpieczeniem)
    MACADDR=""
    if [ -f "/sys/class/net/$IFACE/address" ]; then
        MACADDR=$(cat /sys/class/net/$IFACE/address)
    fi

    CONFIG_FILE="/etc/sysconfig/network-scripts/ifcfg-$IFACE"

    # Zapisanie nowej konfiguracji bezpośrednio do pliku (z tradycyjnym NETMASK)
    cat <<EOF > "$CONFIG_FILE"
TYPE=Ethernet
DEVICE=$IFACE
NAME=$IFACE
HWADDR=$MACADDR
BOOTPROTO=static
ONBOOT=yes
IPADDR=$IP
NETMASK=$MASK
NM_CONTROLLED=no
EOF

    echo "   Zapisano $CONFIG_FILE (MAC: $MACADDR)."
done

echo "=> Restartowanie usługi sieciowej..."

# Próba restartu sieci na różne sposoby zależnie od wersji RHEL
if command -v systemctl > /dev/null 2>&1; then
    systemctl restart network
else
    service network restart
fi

echo "=> Gotowe! Sprawdź dostęp do serwera."
