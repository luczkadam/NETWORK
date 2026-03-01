#!/bin/bash

# Sprawdzenie, czy skrypt został uruchomiony z uprawnieniami roota
if [ "$EUID" -ne 0 ]; then
  echo "Uruchom ten skrypt jako root."
  exit 1
fi

# Funkcja konwertująca prefix na maskę dziesiętną
prefix_to_mask() {
    local PREFIX=$1
    local MASK=""
    local REMAINING=$PREFIX
    for ((i=0; i<4; i++)); do
        if [ $REMAINING -ge 8 ]; then
            MASK+="255"
            let REMAINING-=8
        elif [ $REMAINING -gt 0 ]; then
            local VAL=$(( 256 - (1 << (8 - REMAINING)) ))
            MASK+="$VAL"
            REMAINING=0
        else
            MASK+="0"
        fi
        [ $i -lt 3 ] && MASK+="."
    done
    echo "$MASK"
}

# Wyszukanie wszystkich interfejsów fizycznych
INTERFACES=$(ls /sys/class/net | grep -v -E '^lo$|^virbr|^vnet|^docker|^veth')

echo "=========================================================="
echo "    INTERAKTYWNY KONFIGURATOR INTERFEJSÓW SIECIOWYCH"
echo "=========================================================="

for IFACE in $INTERFACES; do
    echo ""
    echo "----------------------------------------------------------"
    
    # Pobranie MAC
    MACADDR=""
    if [ -f "/sys/class/net/$IFACE/address" ]; then
        MACADDR=$(cat /sys/class/net/$IFACE/address)
    fi

    # Wyświetlenie aktualnego IP dla orientacji
    CURRENT_IP=$(ip -4 addr show $IFACE 2>/dev/null | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
    if [ -z "$CURRENT_IP" ]; then
        CURRENT_IP="Brak IP"
    fi

    echo ">>> Znaleziono interfejs: $IFACE (MAC: $MACADDR)"
    echo ">>> Aktualny adres IP:  $CURRENT_IP"
    
    # Wybór roli
    echo "Wybierz rolę dla tego interfejsu:"
    echo "  [1] MGMT  (Tylko weryfikacja / pozostawienie aktualnego IP)"
    echo "  [2] PROD  (Konfiguracja IP, Maski, GW, DNS)"
    echo "  [3] NAS   (Konfiguracja IP, Maski)"
    echo "  [4] POMIŃ (Nie zmieniaj konfiguracji)"
    read -p "Twój wybór (1-4): " ROLE_CHOICE

    CONFIG_FILE="/etc/sysconfig/network-scripts/ifcfg-$IFACE"

    case $ROLE_CHOICE in
        1)
            echo "-> Interfejs $IFACE zostaje oznaczony jako MGMT."
            echo "-> Plik $CONFIG_FILE pozostaje bez zmian (adres z koła ratunkowego)."
            ;;
        2)
            echo "-> Konfiguracja PROD dla $IFACE"
            read -p "   Podaj adres IP: " IP
            read -p "   Podaj prefix maski (np. 24): " PREFIX
            read -p "   Podaj Gateway (lub wciśnij ENTER, aby pominąć): " GW
            
            MASK=$(prefix_to_mask $PREFIX)

            # Generowanie bazowego pliku
            cat <<EOF > "$CONFIG_FILE"
DEVICE=$IFACE
NAME=$IFACE
HWADDR=$MACADDR
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
NM_CONTROLLED=no
IPADDR=$IP
NETMASK=$MASK
EOF
            
            # Dodawanie Gateway, jeśli podano
            if [ -n "$GW" ]; then
                echo "GATEWAY=$GW" >> "$CONFIG_FILE"
                echo "DEFROUTE=yes" >> "$CONFIG_FILE"
            fi

            # Logika DNS w zależności od IP (POKRYWA 192.100, 192.168, 192.1 itd.)
            if [[ "$IP" == 192.1* ]]; then
                echo "DNS1=10.222.10.10" >> "$CONFIG_FILE"
                echo "DNS2=10.333.10.10" >> "$CONFIG_FILE"
            elif [[ "$IP" == 192.2* ]]; then
                echo "DNS1=20.222.10.10" >> "$CONFIG_FILE"
                echo "DNS2=20.333.10.10" >> "$CONFIG_FILE"
            fi

            # Sztywne dopisywanie domeny
            echo 'DOMAIN="aa.pl bb.pl cc.pl dd.pl"' >> "$CONFIG_FILE"
            
            echo "   Zapisano $CONFIG_FILE jako PROD."
            ;;
        3)
            echo "-> Konfiguracja NAS dla $IFACE"
            read -p "   Podaj adres IP: " IP
            read -p "   Podaj prefix maski (np. 24): " PREFIX
            
            MASK=$(prefix_to_mask $PREFIX)

            cat <<EOF > "$CONFIG_FILE"
DEVICE=$IFACE
NAME=$IFACE
HWADDR=$MACADDR
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
NM_CONTROLLED=no
IPADDR=$IP
NETMASK=$MASK
EOF
            echo "   Zapisano $CONFIG_FILE jako NAS."
            ;;
        *)
            echo "-> Pomijam interfejs $IFACE."
            ;;
    esac
done

echo ""
echo "=========================================================="
echo "Konfiguracja zakończona."
read -p "Czy chcesz teraz zrestartować usługę sieciową? (t/n): " RESTART_CHOICE

if [[ "$RESTART_CHOICE" == "t" || "$RESTART_CHOICE" == "T" ]]; then
    echo "Restartowanie sieci..."
    if command -v systemctl > /dev/null 2>&1; then
        systemctl restart network
    else
        service network restart
    fi
    echo "Gotowe!"
else
    echo "Pominięto restart sieci. Pamiętaj, by zrobić to ręcznie (systemctl restart network)."
fi
