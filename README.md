#!/bin/bash

# ╔══════════════════════════════════════════════════════════════════════════════╗
# ║          MARZBAN VPN — ОДНИМ ФАЙЛОМ                                        ║
# ║          VLESS TCP Reality + VLESS WebSocket TLS                           ║
# ║          Ubuntu 20.04 / 22.04 / 24.04 | Debian 11 / 12                    ║
# ╚══════════════════════════════════════════════════════════════════════════════╝
#
#  Установка одной командой:
#  bash <(curl -fsSL https://github.com/ilyushinkirill2710-gif/marzban-vpn/releases/download/marzban-vpn/marzban-vpn.sh)
#
#  Или скачай файл и запусти:
#  wget https://github.com/ilyushinkirill2710-gif/marzban-vpn/releases/download/marzban-vpn/marzban-vpn.sh
#  bash marzban-vpn.sh

set -eo pipefail


# ──────────────────────────────────────────────────────────────────────────────
#  ЦВЕТА
# ──────────────────────────────────────────────────────────────────────────────
R='\033[0;31m' G='\033[0;32m' Y='\033[1;33m'
B='\033[0;34m' C='\033[0;36m' W='\033[1m' N='\033[0m'

# ──────────────────────────────────────────────────────────────────────────────
#  ПЕРЕМЕННЫЕ (можно задать через env перед запуском)
# ──────────────────────────────────────────────────────────────────────────────
MARZBAN_DIR="${MARZBAN_DIR:-/opt/marzban}"
DOMAIN="${DOMAIN:-}"
ADMIN_USER="${ADMIN_USER:-admin}"
ADMIN_PASS="${ADMIN_PASS:-}"
PORT_REALITY="${PORT_REALITY:-443}"
PORT_WS="${PORT_WS:-8080}"
PORT_PANEL="${PORT_PANEL:-7979}"
SNI="${SNI:-www.google.com}"
WS_PATH=""   # генерируется ниже
LOG="/var/log/marzban-install.log"
REALITY_PRIVATE="" REALITY_PUBLIC="" REALITY_SHORT_ID=""
CERT_FILE="" KEY_FILE=""


# ──────────────────────────────────────────────────────────────────────────────
#  СОВМЕСТИМОСТЬ DOCKER COMPOSE
# ──────────────────────────────────────────────────────────────────────────────
dc() {
    if docker compose version &>/dev/null 2>&1; then
        docker compose "$@"
    elif command -v docker-compose &>/dev/null; then
        docker-compose "$@"
    else
        err "docker compose не найден! Установите Docker 20.10+"
    fi
}

# ──────────────────────────────────────────────────────────────────────────────
#  ХЕЛПЕРЫ
# ──────────────────────────────────────────────────────────────────────────────
ok()  { echo -e "${G}[OK]${N} $*" | tee -a "$LOG"; }
inf() { echo -e "${C}[..]${N} $*" | tee -a "$LOG"; }
wrn() { echo -e "${Y}[!]${N} $*"  | tee -a "$LOG"; }
err() { echo -e "${R}[!!]${N} $*" | tee -a "$LOG"; exit 1; }
hdr() { echo -e "\n${W}${B}───────────────────────────────────────────────────${N}"; \
        echo -e " ${W}${C}$*${N}"; \
        echo -e "${W}${B}───────────────────────────────────────────────────${N}\n"; }
sep() { echo -e "${B}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${N}"; }

spin() {
    local pid=$! chars='⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏' i=0
    while kill -0 "$pid" 2>/dev/null; do
        printf "\r  ${C}%s${N} %s" "${chars:$((i%10)):1}" "$1"
        i=$((i+1)); sleep 0.1
    done
    printf "\r  ${G}✓${N} %-50s\n" "$1"
}

progress() {
    local total=$1 current=0
    shift
    for step in "$@"; do
        current=$((current+1))
        local pct=$(( current * 100 / total ))
        local filled=$(( pct / 5 ))
        local bar=""
        for j in $(seq 0 19); do
            [ "$j" -lt "$filled" ] && bar+="█" || bar+="░"
        done
        printf "\r  [${G}%s${N}] ${W}%3d%%${N} %s" "$bar" "$pct" "$step"
    done
    echo ""
}

# ──────────────────────────────────────────────────────────────────────────────
#  БАННЕР
# ──────────────────────────────────────────────────────────────────────────────
banner() {
    clear
    echo -e "${W}${B}"
    echo "  ╔══════════════════════════════════════════════════════╗"
    echo "  ║                                                      ║"
    echo "  ║   ███╗   ███╗ █████╗ ██████╗ ███████╗               ║"
    echo "  ║   ████╗ ████║██╔══██╗██╔══██╗╚══███╔╝               ║"
    echo "  ║   ██╔████╔██║███████║██████╔╝  ███╔╝                ║"
    echo "  ║   ██║╚██╔╝██║██╔══██║██╔══██╗ ███╔╝                 ║"
    echo "  ║   ██║ ╚═╝ ██║██║  ██║██║  ██║███████╗               ║"
    echo "  ║   ╚═╝     ╚═╝╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝  VPN         ║"
    echo "  ║                                                      ║"
    echo "  ║   Anti-Censorship One-File Installer v2.0            ║"
    echo "  ║   VLESS Reality + WebSocket  |  Обходит РКН/DPI     ║"
    echo "  ║                                                      ║"
    echo "  ╚══════════════════════════════════════════════════════╝"
    echo -e "${N}"
}

# ──────────────────────────────────────────────────────────────────────────────
#  ПРОВЕРКИ
# ──────────────────────────────────────────────────────────────────────────────
check_root() {
    [[ $EUID -ne 0 ]] && err "Нужны права root!\n  sudo bash marzban-vpn.sh"
}

check_os() {
    [[ ! -f /etc/os-release ]] && err "Не удалось определить ОС"
    source /etc/os-release
    case "$ID" in
        ubuntu|debian) ok "ОС: ${PRETTY_NAME:-$ID}" ;;
        *) wrn "ОС ${PRETTY_NAME:-$ID} — не тестировалась, продолжаем" ;;
    esac
}

check_ram() {
    local ram_mb
    ram_mb=$(awk '/MemTotal/{printf "%d", $2/1024}' /proc/meminfo)
    [[ $ram_mb -lt 400 ]] && wrn "Мало RAM: ${ram_mb}MB (рекомендуется 512MB+)"
    ok "RAM: ${ram_mb} MB"
}

check_internet() {
    inf "Проверка интернета..."
    curl -fsS --max-time 8 https://www.google.com &>/dev/null ||
    curl -fsS --max-time 8 https://github.com &>/dev/null ||
        err "Нет интернет-соединения!"
    ok "Интернет доступен"
}

free_port() {
    local port=$1
    if ss -tlnp 2>/dev/null | grep -q ":${port} " || \
       ss -tlnp 2>/dev/null | grep -q ":${port}\t"; then
        wrn "Порт $port занят, освобождаем..."
        fuser -k "${port}/tcp" 2>/dev/null || true
        sleep 1
    fi
}

# ──────────────────────────────────────────────────────────────────────────────
#  ИНТЕРАКТИВНЫЙ ВВОД
# ──────────────────────────────────────────────────────────────────────────────
ask_config() {
    hdr "Настройка параметров"

    # Домен
    if [[ -z "$DOMAIN" ]]; then
        while true; do
            echo -ne "  ${C}Ваш домен${N} (пример: vpn.example.com): "
            read -r DOMAIN
            [[ -n "$DOMAIN" ]] && break
            wrn "Домен не может быть пустым!"
        done
    fi

    # Пароль
    if [[ -z "$ADMIN_PASS" ]]; then
        while true; do
            echo -ne "  ${C}Пароль администратора${N} (мин. 8 символов): "
            read -rs ADMIN_PASS; echo ""
            [[ ${#ADMIN_PASS} -ge 8 ]] && break
            wrn "Слишком короткий пароль!"
        done
    fi

    # SNI
    echo ""
    echo -e "  ${Y}SNI сайт для VLESS Reality${N} (должен поддерживать TLS 1.3):"
    echo -e "  Рекомендую: ${W}www.google.com${N} | ${W}www.microsoft.com${N} | ${W}addons.mozilla.org${N}"
    echo -ne "  SNI [${W}${SNI}${N}]: "
    read -r input_sni
    [[ -n "$input_sni" ]] && SNI="$input_sni"

    # Генерируем случайный WS path
    WS_PATH="/$(openssl rand -hex 8)"

    echo ""
    sep
    echo -e "  ${W}Домен:${N}          $DOMAIN"
    echo -e "  ${W}Панель:${N}         https://$DOMAIN:$PORT_PANEL/dashboard"
    echo -e "  ${W}Reality порт:${N}   $PORT_REALITY"
    echo -e "  ${W}WS порт:${N}        $PORT_WS / путь: $WS_PATH"
    echo -e "  ${W}SNI:${N}            $SNI"
    sep
    echo ""
    echo -ne "  ${Y}Начать установку? [y/N]:${N} "
    read -r confirm
    [[ "$confirm" != "y" && "$confirm" != "Y" ]] && err "Установка отменена."
}

# ──────────────────────────────────────────────────────────────────────────────
#  ЗАВИСИМОСТИ
# ──────────────────────────────────────────────────────────────────────────────
install_deps() {
    hdr "Установка системных пакетов"
    export DEBIAN_FRONTEND=noninteractive
    local _pkg_pid
    (apt-get update -qq 2>&1 &&
     apt-get install -y -qq \
        curl wget openssl ca-certificates \
        gnupg lsb-release apt-transport-https \
        jq unzip ufw socat cron python3 python3-pip \
        net-tools psmisc 2>&1) &
    _pkg_pid=$!
    spin "Установка пакетов..."
    wait "$_pkg_pid" || err "Ошибка установки пакетов! Проверьте: apt-get update"
    ok "Пакеты установлены"
}

# ──────────────────────────────────────────────────────────────────────────────
#  DOCKER
# ──────────────────────────────────────────────────────────────────────────────
install_docker() {
    hdr "Docker"
    if command -v docker &>/dev/null; then
        ok "Docker уже установлен: $(docker --version 2>/dev/null | cut -d' ' -f3 | tr -d ',')"
        return
    fi
    inf "Устанавливаем Docker..."
    local _dock_pid
    (curl -fsSL https://get.docker.com | bash -s 2>&1) &
    _dock_pid=$!
    spin "Загрузка и установка Docker..."
    wait "$_dock_pid" || err "Ошибка установки Docker!"
    systemctl enable docker --now 2>/dev/null || true
    ok "Docker установлен: $(docker --version 2>/dev/null | cut -d' ' -f3 | tr -d ',')"
}

# ──────────────────────────────────────────────────────────────────────────────
#  КЛЮЧИ REALITY (X25519)
# ──────────────────────────────────────────────────────────────────────────────
generate_reality_keys() {
    hdr "Генерация ключей VLESS Reality"

    # Пробуем получить ключи через xray-core в Docker
    local xray_output=""
    xray_output=$(docker run --rm --pull=always \
        ghcr.io/xtls/xray-core:latest x25519 2>/dev/null) || true

    if echo "$xray_output" | grep -q "Private key"; then
        REALITY_PRIVATE=$(echo "$xray_output" | grep "Private key" | awk '{print $NF}')
        REALITY_PUBLIC=$(echo "$xray_output"  | grep "Public key"  | awk '{print $NF}')
        ok "Ключи сгенерированы через Xray-core"
    else
        # Fallback: генерируем через Python cryptography
        inf "Xray недоступен, генерируем ключи через Python..."
        pip3 install cryptography -q 2>/dev/null || true
        local keys
        keys=$(python3 - << 'PYEOF'
try:
    from cryptography.hazmat.primitives.asymmetric.x25519 import X25519PrivateKey
    from cryptography.hazmat.primitives.serialization import Encoding, PublicFormat, PrivateFormat, NoEncryption
    import base64
    priv = X25519PrivateKey.generate()
    pub  = priv.public_key()
    priv_b = priv.private_bytes(Encoding.Raw, PrivateFormat.Raw, NoEncryption())
    pub_b  = pub.public_key_bytes_raw()   if hasattr(pub, 'public_key_bytes_raw') else \
             pub.public_bytes(Encoding.Raw, PublicFormat.Raw)
    print(base64.urlsafe_b64encode(priv_b).decode().rstrip('='))
    print(base64.urlsafe_b64encode(pub_b).decode().rstrip('='))
except Exception:
    import secrets, base64
    k1 = secrets.token_bytes(32)
    k2 = secrets.token_bytes(32)
    print(base64.urlsafe_b64encode(k1).decode().rstrip('='))
    print(base64.urlsafe_b64encode(k2).decode().rstrip('='))
PYEOF
)
        REALITY_PRIVATE=$(echo "$keys" | head -1)
        REALITY_PUBLIC=$(echo "$keys"  | tail -1)
        ok "Ключи сгенерированы через Python"
    fi

    REALITY_SHORT_ID=$(openssl rand -hex 8)

    ok "Private key: ${REALITY_PRIVATE:0:24}..."
    ok "Public key:  ${REALITY_PUBLIC:0:24}..."
    ok "Short ID:    $REALITY_SHORT_ID"
}

# ──────────────────────────────────────────────────────────────────────────────
#  SSL СЕРТИФИКАТ
# ──────────────────────────────────────────────────────────────────────────────
setup_ssl() {
    hdr "SSL сертификат"

    local acme_cert="/root/.acme.sh/${DOMAIN}_ecc/fullchain.cer"
    local acme_key="/root/.acme.sh/${DOMAIN}_ecc/${DOMAIN}.key"

    if [[ -f "$acme_cert" ]]; then
        ok "Сертификат уже есть, используем его"
        CERT_FILE="$acme_cert"
        KEY_FILE="$acme_key"
        return
    fi

    # Устанавливаем acme.sh
    if [[ ! -f /root/.acme.sh/acme.sh ]]; then
        inf "Устанавливаем acme.sh..."
        local _acme_pid
        (curl -fsSL https://get.acme.sh | bash -s email="ssl@${DOMAIN}" 2>&1) &
        _acme_pid=$!
        spin "Установка acme.sh..."
        wait "$_acme_pid" || err "Ошибка установки acme.sh!"
        source /root/.acme.sh/acme.sh.env 2>/dev/null || true
    fi

    ufw allow 80/tcp comment "acme" 2>/dev/null || true
    inf "Запрашиваем сертификат Let's Encrypt для $DOMAIN..."

    /root/.acme.sh/acme.sh --issue \
        --standalone -d "$DOMAIN" \
        --server letsencrypt \
        --key-file    "/root/.acme.sh/${DOMAIN}_ecc/${DOMAIN}.key" \
        --fullchain-file "$acme_cert" \
        --reloadcmd "" \
        --force 2>&1 | tee -a "$LOG" || {
        wrn "Let's Encrypt недоступен, создаём self-signed сертификат..."
        mkdir -p "${MARZBAN_DIR}/certs"
        openssl req -x509 -newkey rsa:4096 -nodes \
            -keyout "${MARZBAN_DIR}/certs/${DOMAIN}.key" \
            -out    "${MARZBAN_DIR}/certs/${DOMAIN}.crt" \
            -days 3650 -subj "/CN=${DOMAIN}" 2>/dev/null
        CERT_FILE="${MARZBAN_DIR}/certs/${DOMAIN}.crt"
        KEY_FILE="${MARZBAN_DIR}/certs/${DOMAIN}.key"
        wrn "Self-signed сертификат создан (клиенты увидят предупреждение)"
        ufw delete allow 80/tcp 2>/dev/null || true
        return
    }

    ufw delete allow 80/tcp 2>/dev/null || true
    CERT_FILE="$acme_cert"
    KEY_FILE="$acme_key"
    ok "SSL сертификат получен"
}

# ──────────────────────────────────────────────────────────────────────────────
#  КОНФИГУРАЦИЯ XRAY
# ──────────────────────────────────────────────────────────────────────────────
create_xray_config() {
    hdr "Конфигурация Xray"

    local short2 short3
    short2=$(openssl rand -hex 4)
    short3=$(openssl rand -hex 6)

    mkdir -p "${MARZBAN_DIR}"
    cat > "${MARZBAN_DIR}/xray_config.json" << XRAYJSON
{
  "log": { "loglevel": "warning", "access": "/var/log/xray/access.log" },
  "api": {
    "tag": "API",
    "services": ["HandlerService", "StatsService", "LoggerService"]
  },
  "stats": {},
  "policy": {
    "system": {
      "statsInboundDownlink": true,
      "statsInboundUplink": true
    },
    "levels": {
      "0": {
        "statsUserDownlink": true,
        "statsUserUplink": true,
        "handshakeTimeout": 20,
        "connIdle": 300
      }
    }
  },
  "inbounds": [
    {
      "tag": "api",
      "listen": "127.0.0.1",
      "port": 62789,
      "protocol": "dokodemo-door",
      "settings": { "address": "127.0.0.1" }
    },
    {
      "tag": "VLESS_TCP_REALITY",
      "listen": "0.0.0.0",
      "port": ${PORT_REALITY},
      "protocol": "vless",
      "settings": {
        "clients": [],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "${SNI}:443",
          "xver": 0,
          "serverNames": ["${SNI}"],
          "privateKey": "${REALITY_PRIVATE}",
          "shortIds": [
            "${REALITY_SHORT_ID}",
            "${short2}",
            "${short3}"
          ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    },
    {
      "tag": "VLESS_WS_TLS",
      "listen": "0.0.0.0",
      "port": ${PORT_WS},
      "protocol": "vless",
      "settings": {
        "clients": [],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {
          "certificates": [
            {
              "certificateFile": "/var/lib/marzban/certs/fullchain.cer",
              "keyFile":         "/var/lib/marzban/certs/privkey.key"
            }
          ],
          "minVersion": "1.2",
          "alpn": ["http/1.1"]
        },
        "wsSettings": {
          "path": "${WS_PATH}",
          "headers": { "Host": "${DOMAIN}" }
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      }
    }
  ],
  "outbounds": [
    {
      "tag": "DIRECT",
      "protocol": "freedom",
      "settings": { "domainStrategy": "UseIPv4" }
    },
    {
      "tag": "BLOCK",
      "protocol": "blackhole",
      "settings": { "response": { "type": "http" } }
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      { "type": "field", "inboundTag": ["api"], "outboundTag": "api" },
      { "type": "field", "protocol": ["bittorrent"], "outboundTag": "BLOCK" },
      { "type": "field", "ip": ["geoip:private"], "outboundTag": "BLOCK" },
      { "type": "field", "outboundTag": "DIRECT", "network": "tcp,udp" }
    ]
  }
}
XRAYJSON

    ok "Xray конфиг создан"
}

# ──────────────────────────────────────────────────────────────────────────────
#  MARZBAN ENV + DOCKER COMPOSE
# ──────────────────────────────────────────────────────────────────────────────
setup_marzban() {
    hdr "Настройка Marzban"

    mkdir -p "${MARZBAN_DIR}/certs" /var/lib/marzban/certs /var/log/marzban

    # Копируем сертификаты
    cp "$CERT_FILE" "${MARZBAN_DIR}/certs/fullchain.cer"
    cp "$KEY_FILE"  "${MARZBAN_DIR}/certs/privkey.key"
    cp "${MARZBAN_DIR}/certs/fullchain.cer" /var/lib/marzban/certs/
    cp "${MARZBAN_DIR}/certs/privkey.key"   /var/lib/marzban/certs/
    cp "${MARZBAN_DIR}/xray_config.json"    /var/lib/marzban/

    # .env
    cat > "${MARZBAN_DIR}/.env" << ENVEOF
UVICORN_HOST = "0.0.0.0"
UVICORN_PORT = ${PORT_PANEL}
XRAY_JSON = "/var/lib/marzban/xray_config.json"
XRAY_SUBSCRIPTION_URL_PREFIX = "https://${DOMAIN}:${PORT_PANEL}"
UVICORN_SSL_CERTFILE = "/var/lib/marzban/certs/fullchain.cer"
UVICORN_SSL_KEYFILE  = "/var/lib/marzban/certs/privkey.key"
SUDO_USERNAME = "${ADMIN_USER}"
SUDO_PASSWORD = "${ADMIN_PASS}"
SQLALCHEMY_DATABASE_URL = "sqlite:////var/lib/marzban/db.sqlite3"
JWT_ACCESS_TOKEN_EXPIRE_MINUTES = 1440
ENVEOF

    # docker-compose.yml
    cat > "${MARZBAN_DIR}/docker-compose.yml" << 'COMPEOF'
version: "3.8"
services:
  marzban:
    image: gozargah/marzban:latest
    container_name: marzban
    restart: always
    env_file: .env
    network_mode: host
    volumes:
      - /var/lib/marzban:/var/lib/marzban
      - /var/log/marzban:/var/log/marzban
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"
COMPEOF

    ok "Marzban настроен"
}

# ──────────────────────────────────────────────────────────────────────────────
#  SYSTEMD СЕРВИС
# ──────────────────────────────────────────────────────────────────────────────
setup_service() {
    cat > /etc/systemd/system/marzban.service << SVCEOF
[Unit]
Description=Marzban VPN Panel
After=docker.service network-online.target
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=${MARZBAN_DIR}
ExecStart=/bin/bash -c "cd /opt/marzban && (docker compose up -d 2>/dev/null || docker-compose up -d)"
ExecStop=/bin/bash -c "cd /opt/marzban && (docker compose down 2>/dev/null || docker-compose down)"

[Install]
WantedBy=multi-user.target
SVCEOF
    systemctl daemon-reload
    systemctl enable marzban 2>/dev/null || true
    ok "Systemd сервис создан"
}

# ──────────────────────────────────────────────────────────────────────────────
#  FIREWALL
# ──────────────────────────────────────────────────────────────────────────────
setup_firewall() {
    hdr "Настройка Firewall"
    ufw --force reset    2>/dev/null
    ufw default deny incoming  2>/dev/null
    ufw default allow outgoing 2>/dev/null
    ufw allow 22/tcp   comment "SSH"              2>/dev/null
    ufw allow "${PORT_REALITY}/tcp" comment "VLESS Reality" 2>/dev/null
    ufw allow "${PORT_WS}/tcp"      comment "VLESS WS"      2>/dev/null
    ufw allow "${PORT_PANEL}/tcp"   comment "Marzban Panel" 2>/dev/null
    ufw --force enable 2>/dev/null
    ok "Firewall: 22, ${PORT_REALITY}, ${PORT_WS}, ${PORT_PANEL}"
}

# ──────────────────────────────────────────────────────────────────────────────
#  ОПТИМИЗАЦИЯ СЕТИ
# ──────────────────────────────────────────────────────────────────────────────
optimize_network() {
    hdr "Оптимизация сети (BBR)"

    # Проверяем, не добавлено ли уже
    if ! grep -q "tcp_congestion_control = bbr" /etc/sysctl.conf 2>/dev/null
