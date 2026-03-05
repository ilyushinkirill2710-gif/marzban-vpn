Marzban VPN - Anti-Censorship Installer
Автоматическая установка VPN-панели Marzban с протоколами VLESS TCP Reality и VLESS WebSocket TLS.
Обходит блокировки РКН и DPI. Работает в России.

Быстрый старт - одна команда
bash <(curl -fsSL https://raw.githubusercontent.com/ilyushinkirill2710-gif/marzban-vpn/main/marzban-vpn.sh)
Требования
Параметр	Значение
ОС	Ubuntu 20.04 / 22.04 / 24.04, Debian 11 / 12
RAM	от 512 MB
Доступ	root или sudo
Домен	DNS A-запись указывает на IP сервера
Порты	443, 8080, 7979 открыты
Протоколы
VLESS TCP Reality (порт 443)
Маскируется под TLS-трафик к настоящему сайту (Google, Microsoft)
Не определяется DPI-анализом РКН
Не требует собственного сертификата
Максимальная скорость и стабильность
VLESS WebSocket TLS (порт 8080)
Работает через CDN (Cloudflare и другие)
Скрывает реальный IP сервера за CDN
Запасной вариант при блокировке Reality
Что устанавливается
[+] Docker + Docker Compose
[+] Marzban (веб-панель + REST API)
[+] Xray-core (последняя версия)
[+] SSL сертификат Let's Encrypt (acme.sh)
[+] UFW Firewall (только нужные порты)
[+] BBR + оптимизация TCP буферов
[+] Авто-обновление SSL каждую неделю
[+] Утилита marzban-ctl для управления
Управление сервером
marzban-ctl start       # Запустить
marzban-ctl stop        # Остановить
marzban-ctl restart     # Перезапустить
marzban-ctl update      # Обновить до последней версии
marzban-ctl logs        # Логи в реальном времени
marzban-ctl status      # Статус контейнеров
marzban-ctl info        # Адрес панели и данные доступа
marzban-ctl renew-ssl   # Принудительно обновить SSL
marzban-ctl backup      # Резервная копия
marzban-ctl uninstall   # Удалить Marzban
Клиентские приложения
Платформа	Приложение
Android	v2rayNG / Hiddify
iOS	Streisand / FoXray
Windows	Hiddify Next / v2rayN
macOS	Hiddify Next
Linux	Hiddify Next
Тихая установка (без вопросов)
export DOMAIN="vpn.example.com"
export ADMIN_PASS="СильныйПароль123"
export SNI_SITE="www.microsoft.com"
bash <(curl -fsSL https://raw.githubusercontent.com/ilyushinkirill2710-gif/marzban-vpn/main/marzban-vpn.sh)
Дисклеймер
Проект создан в образовательных целях. Используйте только в соответствии с законодательством вашей страны.

Основан на Marzban и Xray-core
