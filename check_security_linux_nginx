#!/bin/bash
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; NC='\033[0m'

echo -e "\n${YELLOW}=== ПОЛНАЯ ПРОВЕРКА БЕЗОПАСНОСТИ СЕРВЕРА ===${NC}\n"

# === 1. Проверка симлинков (не root, не apt) ===
echo -e "${BLUE}--- Симлинки, созданные не root и не apt (за 7 дней) ---${NC}"
find / -type l -mtime -7 ! -user root 2>/dev/null | grep -v "dpkg\|apt" | head -10
echo -e "\n${BLUE}--- Симлинки в /tmp, /var/tmp, /dev/shm ---${NC}"
find /tmp /var/tmp /dev/shm -type l 2>/dev/null | head -5
echo -e "\n${BLUE}--- Симлинки в /usr/local/bin (внешние) ---${NC}"
find /usr/local/bin -type l 2>/dev/null | while read link; do
    target=$(readlink "$link" 2>/dev/null)
    if [ -n "$target" ] && [ ! -f "$target" ] && [ ! -d "$target" ]; then
        echo "  $link → $target (не существует)"
    fi
done
echo -e "\n${BLUE}--- Симлинки в /etc/rc*.d (автозапуск) ---${NC}"
find /etc/rc*.d -type l 2>/dev/null | head -5
echo -e "\n${BLUE}--- Симлинки с правами 777 ---${NC}"
find / -type l -perm 0777 2>/dev/null | head -5

# === 2. Проверка подозрительных файлов ===
echo -e "\n${BLUE}--- Проверка /usr/local/bin, /tmp, /var/tmp ---${NC}"
UNKNOWN_FILES=$(find /usr/local/bin /tmp /var/tmp -type f -mtime -7 2>/dev/null | grep -v "apt\|dpkg")
if [ -n "$UNKNOWN_FILES" ]; then
    echo -e "${RED}⚠️ Найдены недавние файлы:${NC}"
    echo "$UNKNOWN_FILES" | head -5
else
    echo -e "${GREEN}✅ Подозрительных файлов не найдено${NC}"
fi

# === 3. Проверка автозапуска ===
echo -e "\n${BLUE}--- Проверка cron и systemd таймеров ---${NC}"
CRON_JOBS=$(crontab -l 2>/dev/null | grep -v '^#' | wc -l)
TIMERS=$(systemctl list-timers --all 2>/dev/null | grep -E "active|running" | wc -l)
if [ "$CRON_JOBS" -gt 0 ] || [ "$TIMERS" -gt 0 ]; then
    echo -e "${YELLOW}⚠️ Найдены активные cron и systemd таймеры${NC}"
else
    echo -e "${GREEN}✅ Cron и systemd таймеры отсутствуют${NC}"
fi

# === 4. Проверка SUID/SGID битов ===
echo -e "\n${BLUE}--- Проверка SUID/SGID ---${NC}"
SUID_FILES=$(find / -type f \( -perm -4000 -o -perm -2000 \) ! -path "/snap/*" 2>/dev/null | wc -l)
if [ "$SUID_FILES" -gt 10 ]; then
    echo -e "${YELLOW}⚠️ Обнаружено $SUID_FILES файлов с SUID/SGID (проверь вручную)${NC}"
else
    echo -e "${GREEN}✅ SUID/SGID файлов в пределах нормы${NC}"
fi

# === 5. Проверка истории команд других пользователей ===
echo -e "\n${BLUE}--- Проверка истории команд (не root) ---${NC}"
for user in $(getent passwd | grep -E '(/bin/bash|/bin/sh)' | cut -d: -f1); do
    if [ "$user" != "root" ] && [ -f "/home/$user/.bash_history" ]; then
        HISTORY_SIZE=$(wc -l < /home/$user/.bash_history 2>/dev/null)
        echo -e "  ${YELLOW}⚠️ $user: $HISTORY_SIZE команд${NC}"
    fi
done

# === 6. Проверка rkhunter и chkrootkit ===
echo -e "\n${BLUE}--- Проверка антируткитов ---${NC}"
if command -v rkhunter &>/dev/null; then
    echo -e "${GREEN}✅ rkhunter установлен${NC}"
    rkhunter --check --skip-keypress 2>/dev/null | grep -E "Warning|Warning|Alert" | head -3
else
    echo -e "${RED}⚠️ rkhunter не установлен (рекомендация: apt install rkhunter -y)${NC}"
fi
if command -v chkrootkit &>/dev/null; then
    echo -e "${GREEN}✅ chkrootkit установлен${NC}"
else
    echo -e "${RED}⚠️ chkrootkit не установлен (рекомендация: apt install chkrootkit -y)${NC}"
fi

# === 7. Проверка открытых портов ===
echo -e "\n${BLUE}--- Проверка открытых портов ---${NC}"
OPEN_PORTS=$(ss -tulnp | grep -E "0.0.0.0:|:::" | wc -l)
if [ "$OPEN_PORTS" -gt 0 ]; then
    echo -e "${YELLOW}⚠️ Открыто $OPEN_PORTS портов${NC}"
    ss -tulnp | grep -E "0.0.0.0:|:::" | awk '{print $5}' | cut -d: -f2 | sort -n | uniq | head -5
else
    echo -e "${GREEN}✅ Открытых портов не найдено${NC}"
fi

# === 8. Проверка модулей ядра ===
echo -e "\n${BLUE}--- Проверка модулей ядра ---${NC}"
MODULES=$(lsmod | grep -E "crypto|miner|hide|rootkit" | wc -l)
if [ "$MODULES" -gt 0 ]; then
    echo -e "${RED}⚠️ Обнаружены подозрительные модули:${NC}"
    lsmod | grep -E "crypto|miner|hide|rootkit"
else
    echo -e "${GREEN}✅ Подозрительных модулей не найдено${NC}"
fi

# === 9. Проверка файлов, созданных вне сессии (не root, не apt) ===
echo -e "\n${BLUE}--- Файлы в /etc, /bin, /usr/bin, изменённые за 7 дней (не apt, не root) ---${NC}"
find /etc /bin /usr/bin -type f -mtime -7 ! -user root 2>/dev/null | grep -v "dpkg\|apt" | head -10
echo -e "\n${BLUE}--- Новые файлы в /tmp, /var/tmp, /dev/shm (за 3 дня) ---${NC}"
find /tmp /var/tmp /dev/shm -type f -mtime -3 2>/dev/null | head -10

# === 10. Проверка нестандартных команд в /usr/local/bin ===
echo -e "\n${BLUE}--- Команды в /usr/local/bin (не из пакетов) ---${NC}"
for cmd in /usr/local/bin/*; do
    if [ -f "$cmd" ] && [ ! -L "$cmd" ]; then
        echo "  $cmd"
    fi
done | head -10

# === 11. Проверка логов на попытки файловых включений (RFI/LFI) ===
echo -e "\n${BLUE}--- Попытки удалённого/локального включения файлов (RFI/LFI) ---${NC}"
if [ -f /var/log/nginx/access.log ]; then
    tail -100 /var/log/nginx/access.log | grep -iE '(\?.*=.*(http|ftp)|\.\./|/etc/passwd|/etc/shadow|proc/self/environ|/bin/sh|base64_decode|system\(|eval\(|exec\(|passthru\()' | head -5 | sed 's/^/  /'
fi
if [ -f /var/log/nginx/error.log ]; then
    tail -50 /var/log/nginx/error.log | grep -iE '(open_basedir|failed to open stream|file not found|include|require|allow_url_include)' | head -5 | sed 's/^/  /'
fi

# === 12. Проверка логов на попытки загрузки веб-шеллов ===
echo -e "\n${BLUE}--- Попытки загрузки веб-шеллов (shell.php, cmd.php) ---${NC}"
if [ -f /var/log/nginx/access.log ]; then
    tail -200 /var/log/nginx/access.log | grep -iE '(shell|cmd|backdoor|webshell|mini.php|eval|wso|r57|c99|b374k)\.(php|phtml|shtml)' | head -5 | sed 's/^/  /'
fi

# === 13. Проверка логов на подозрительные POST-запросы ===
echo -e "\n${BLUE}--- Подозрительные POST-запросы (возможный LFI/RFI) ---${NC}"
if [ -f /var/log/nginx/access.log ]; then
    tail -100 /var/log/nginx/access.log | grep -iE 'POST.*(cgi-bin|wp-admin|xmlrpc|index.php\?.*=.*http)' | head -5 | sed 's/^/  /'
fi

# === 14. Проверка логов на атаки на установленные сервисы ===
echo -e "\n${BLUE}--- Попытки эксплуатации установленных сервисов ---${NC}"
SSH_BRUTE=0
if [ -f /var/log/auth.log ]; then
    SSH_BRUTE=$(grep -c "Failed password" /var/log/auth.log 2>/dev/null)
elif [ -f /var/log/secure ]; then
    SSH_BRUTE=$(grep -c "Failed password" /var/log/secure 2>/dev/null)
fi
if [ "$SSH_BRUTE" -gt 10 ]; then
    echo -e "${RED}⚠️ Обнаружено $SSH_BRUTE неудачных попыток SSH (брутфорс)${NC}"
else
    echo -e "${GREEN}✅ SSH атак не обнаружено${NC}"
fi

if [ -f /var/log/nginx/access.log ]; then
    SQLI=$(tail -100 /var/log/nginx/access.log | grep -ciE "(union.*select|select.*from|sleep\(|benchmark\(|' OR '1'='1|-- |; --|1=1)")
    XSS=$(tail -100 /var/log/nginx/access.log | grep -ciE "(<script|onerror=|onload=|alert\(|prompt\(|confirm\(|document\.cookie)")
    if [ "$SQLI" -gt 0 ] || [ "$XSS" -gt 0 ]; then
        echo -e "${RED}⚠️ Обнаружены попытки SQL-инъекций ($SQLI) и XSS ($XSS)${NC}"
    else
        echo -e "${GREEN}✅ Веб-атак не обнаружено${NC}"
    fi
fi

if [ -f /var/log/php8.4-fpm.log ]; then
    PHP_ERRORS=$(tail -50 /var/log/php8.4-fpm.log | grep -ciE "(error|warning|fatal)")
    if [ "$PHP_ERRORS" -gt 5 ]; then
        echo -e "${YELLOW}⚠️ Обнаружено $PHP_ERRORS ошибок PHP (возможны проблемы с кодом)${NC}"
    else
        echo -e "${GREEN}✅ PHP-ошибок не обнаружено${NC}"
    fi
fi

if [ -f /var/log/mail.log ]; then
    MAIL_ATTACKS=$(tail -100 /var/log/mail.log | grep -ciE "(authentication failed|relay denied|rejected|spam)")
    if [ "$MAIL_ATTACKS" -gt 5 ]; then
        echo -e "${YELLOW}⚠️ Обнаружено $MAIL_ATTACKS подозрительных почтовых событий${NC}"
    else
        echo -e "${GREEN}✅ Почтовых атак не обнаружено${NC}"
    fi
fi

if [ -f /var/log/nginx/access.log ]; then
    ADMIN_ACCESS=$(tail -200 /var/log/nginx/access.log | grep -ciE "(wp-admin|phpmyadmin|adminer|administrator|/admin/|/login)")
    if [ "$ADMIN_ACCESS" -gt 10 ]; then
        echo -e "${YELLOW}⚠️ Обнаружено $ADMIN_ACCESS попыток доступа к админкам${NC}"
    else
        echo -e "${GREEN}✅ Аномального доступа к админкам не обнаружено${NC}"
    fi
fi

# === 15. Проверка безопасности конфигурации Nginx ===
echo -e "\n${BLUE}--- Проверка безопасности Nginx ---${NC}"
NGINX_USER=$(ps aux | grep "nginx: master" | grep -v grep | awk '{print $1}')
if [ "$NGINX_USER" = "root" ]; then
    echo -e "${RED}⚠️ Nginx запущен от root (должен быть от www-data)${NC}"
else
    echo -e "${GREEN}✅ Nginx запущен от $NGINX_USER${NC}"
fi
if grep -q "server_tokens off" /etc/nginx/nginx.conf; then
    echo -e "${GREEN}✅ Версия Nginx скрыта (server_tokens off)${NC}"
else
    echo -e "${RED}⚠️ Версия Nginx не скрыта (рекомендуется server_tokens off)${NC}"
fi
if grep -q "location ~ /\." /etc/nginx/sites-available/default; then
    echo -e "${GREEN}✅ Доступ к скрытым файлам запрещён${NC}"
else
    echo -e "${YELLOW}⚠️ Нет явного запрета доступа к . файлам${NC}"
fi
if grep -q "client_max_body_size" /etc/nginx/nginx.conf; then
    echo -e "${GREEN}✅ Ограничен размер загружаемых файлов${NC}"
else
    echo -e "${YELLOW}⚠️ Не задан client_max_body_size (рекомендуется 10-50M)${NC}"
fi
if grep -q "client_body_timeout\|client_header_timeout\|send_timeout" /etc/nginx/nginx.conf; then
    echo -e "${GREEN}✅ Настроены таймауты соединений${NC}"
else
    echo -e "${YELLOW}⚠️ Не настроены таймауты (риск DoS)${NC}"
fi
if [ -f /etc/nginx/sites-available/default ] && grep -q "listen 443" /etc/nginx/sites-available/default; then
    echo -e "${GREEN}✅ HTTPS настроен${NC}"
else
    echo -e "${YELLOW}⚠️ HTTPS не настроен или не проверен${NC}"
fi

# === 16. Проверка ограничений HTTP-методов ===
echo -e "\n${BLUE}--- Проверка ограничений HTTP-методов ---${NC}"
if [ -f /etc/nginx/sites-available/default ]; then
    if grep -q "limit_except" /etc/nginx/sites-available/default; then
        echo -e "${GREEN}✅ Ограничения на методы заданы через limit_except${NC}"
        grep -A 5 "limit_except" /etc/nginx/sites-available/default | head -6 | sed 's/^/  /'
    else
        echo -e "${RED}⚠️ Нет ограничений на HTTP-методы${NC}"
    fi
    if grep -q "PUT\|DELETE\|PATCH" /etc/nginx/sites-available/default | grep -q "deny all"; then
        echo -e "${GREEN}✅ Методы PUT, DELETE, PATCH запрещены${NC}"
    else
        echo -e "${YELLOW}⚠️ Нет явного запрета PUT, DELETE, PATCH${NC}"
    fi
    if grep -q "TRACE\|TRACK" /etc/nginx/nginx.conf; then
        echo -e "${GREEN}✅ TRACE/TRACK методы запрещены${NC}"
    else
        echo -e "${YELLOW}⚠️ TRACE/TRACK методы не запрещены (рекомендуется)${NC}"
    fi
fi

# === 17. Проверка SSL-сертификата (срок действия) ===
echo -e "\n${BLUE}--- Проверка SSL-сертификата ---${NC}"
if [ -f /etc/nginx/ssl/whonpv.space.pem ]; then
    EXPIRY=$(openssl x509 -enddate -noout -in /etc/nginx/ssl/whonpv.space.pem | cut -d= -f2)
    EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
    NOW=$(date +%s)
    DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW) / 86400 ))
    if [ "$DAYS_LEFT" -lt 0 ]; then
        echo -e "${RED}⚠️ Сертификат истёк!${NC}"
    elif [ "$DAYS_LEFT" -lt 30 ]; then
        echo -e "${YELLOW}⚠️ Сертификат истекает через $DAYS_LEFT дней${NC}"
    else
        echo -e "${GREEN}✅ Сертификат действителен ещё $DAYS_LEFT дней${NC}"
    fi
else
    echo -e "${YELLOW}⚠️ SSL-сертификат не найден${NC}"
fi

# === 18. Проверка неудачных попыток входа в систему ===
echo -e "\n${BLUE}--- Неудачные попытки входа (логи) ---${NC}"
if [ -f /var/log/auth.log ]; then
    FAILED_LOGINS=$(grep "authentication failure" /var/log/auth.log | tail -10)
    if [ -n "$FAILED_LOGINS" ]; then
        echo -e "${RED}⚠️ Неудачные попытки входа (последние 10):${NC}"
        echo "$FAILED_LOGINS" | sed 's/^/  /'
    else
        echo -e "${GREEN}✅ Неудачных попыток входа не найдено${NC}"
    fi
fi

# === 19. Проверка подозрительных процессов ===
echo -e "\n${BLUE}--- Подозрительные процессы (по именам) ---${NC}"
SUSPICIOUS_PROCS=$(ps aux | grep -E 'crypto|miner|stratum|kworker|update|systemd' | grep -v grep | wc -l)
if [ "$SUSPICIOUS_PROCS" -gt 5 ]; then
    echo -e "${YELLOW}⚠️ Обнаружено $SUSPICIOUS_PROCS подозрительных процессов${NC}"
    ps aux | grep -E 'crypto|miner|stratum|kworker|update|systemd' | grep -v grep | head -5 | sed 's/^/  /'
else
    echo -e "${GREEN}✅ Подозрительных процессов не найдено${NC}"
fi

# === 20. Проверка файлов с правами 777 ===
echo -e "\n${BLUE}--- Проверка файлов с правами 777 ---${NC}"
W777=$(find / -type f -perm 0777 ! -path "/tmp/*" 2>/dev/null | head -3)
if [ -n "$W777" ]; then
    echo -e "${RED}⚠️ Найдены файлы с правами 777:${NC}"
    echo "$W777" | sed 's/^/  /'
else
    echo -e "${GREEN}✅ Файлов с правами 777 не найдено${NC}"
fi

# === 21. Проверка пользователей с UID=0 ===
echo -e "\n${BLUE}--- Проверка пользователей с UID=0 ---${NC}"
UID0=$(awk -F: '$3==0 {print $1}' /etc/passwd)
if [ "$UID0" != "root" ]; then
    echo -e "${RED}⚠️ Найдены пользователи с UID=0 (кроме root): $UID0${NC}"
else
    echo -e "${GREEN}✅ root единственный администратор${NC}"
fi

# === 22. Проверка miss configuration и стандартных паролей ===
echo -e "\n${BLUE}--- Проверка miss configuration и стандартных паролей ---${NC}"
if grep -q "PermitEmptyPasswords yes" /etc/ssh/sshd_config; then
    echo -e "${RED}⚠️ SSH разрешает пустые пароли (PermitEmptyPasswords yes)${NC}"
else
    echo -e "${GREEN}✅ SSH не разрешает пустые пароли${NC}"
fi
if grep -q "PermitRootLogin yes" /etc/ssh/sshd_config; then
    echo -e "${YELLOW}⚠️ SSH разрешает вход от root (рекомендуется запретить)${NC}"
else
    echo -e "${GREEN}✅ SSH запрещает вход от root${NC}"
fi
if [ -f /etc/shadow ]; then
    ROOT_HASH=$(grep "^root:" /etc/shadow | cut -d: -f2)
    if [ "$ROOT_HASH" = "!" ] || [ "$ROOT_HASH" = "*" ]; then
        echo -e "${RED}⚠️ Root-пароль заблокирован (возможно, не задан)${NC}"
    else
        echo -e "${GREEN}✅ Root-пароль установлен${NC}"
    fi
fi
if [ -f /etc/nginx/.htpasswd ]; then
    if grep -q "admin:admin" /etc/nginx/.htpasswd; then
        echo -e "${RED}⚠️ Найден стандартный пароль admin:admin в .htpasswd${NC}"
    else
        echo -e "${GREEN}✅ .htpasswd не содержит стандартных паролей${NC}"
    fi
fi
if [ -f /etc/nginx/sites-available/default ]; then
    if grep -q "autoindex on" /etc/nginx/sites-available/default; then
        echo -e "${YELLOW}⚠️ Включён autoindex (может быть уязвимостью)${NC}"
    else
        echo -e "${GREEN}✅ autoindex отключён${NC}"
    fi
fi
for user in test user admin guest; do
    if id "$user" &>/dev/null; then
        echo -e "${RED}⚠️ Найден стандартный пользователь: $user (рекомендуется удалить)${NC}"
    fi
done
if [ -f /etc/php/8.4/fpm/php.ini ]; then
    DISPLAY_ERRORS=$(grep -E "^display_errors\s*=" /etc/php/8.4/fpm/php.ini | cut -d= -f2 | xargs)
    EXPOSE_PHP=$(grep -E "^expose_php\s*=" /etc/php/8.4/fpm/php.ini | cut -d= -f2 | xargs)
    if [ "$DISPLAY_ERRORS" = "On" ] || [ "$DISPLAY_ERRORS" = "on" ]; then
        echo -e "${YELLOW}⚠️ display_errors включён (рекомендуется Off)${NC}"
    else
        echo -e "${GREEN}✅ display_errors отключён${NC}"
    fi
    if [ "$EXPOSE_PHP" = "On" ] || [ "$EXPOSE_PHP" = "on" ]; then
        echo -e "${YELLOW}⚠️ expose_php включён (рекомендуется Off)${NC}"
    else
        echo -e "${GREEN}✅ expose_php отключён${NC}"
    fi
fi

# === 23. Проверка uptime и нагрузка ===
echo -e "\n${BLUE}--- Нагрузка на сервер ---${NC}"
uptime | sed 's/^/  /'
echo -e "  ${YELLOW}Если нагрузка выше 2.0 на 1 ядро — возможна перегрузка или атака${NC}"

# === ИТОГОВЫЕ РЕКОМЕНДАЦИИ (на основе обнаруженного) ===
echo -e "\n${YELLOW}=== РЕКОМЕНДАЦИИ ===${NC}"
RECOMMENDATIONS=()

if grep -q "server_tokens" /etc/nginx/nginx.conf && ! grep -q "server_tokens off" /etc/nginx/nginx.conf; then
    RECOMMENDATIONS+=("Добавь server_tokens off; в /etc/nginx/nginx.conf")
fi
if [ ! -f /etc/nginx/ssl/whonpv.space.pem ]; then
    RECOMMENDATIONS+=("Настрой SSL-сертификат (Let's Encrypt или Cloudflare)")
fi
if [ "$SSH_BRUTE" -gt 20 ]; then
    RECOMMENDATIONS+=("Установи fail2ban: apt install fail2ban -y и смени SSH-порт (не 22)")
fi
if ! grep -q "limit_except" /etc/nginx/sites-available/default; then
    RECOMMENDATIONS+=("Добавь ограничение методов в Nginx: limit_except GET HEAD POST { deny all; }")
fi
if [ "$ADMIN_ACCESS" -gt 20 ]; then
    RECOMMENDATIONS+=("Заблокируй доступ к /admin, /wp-admin через Nginx или пароль")
fi
if [ ! -d /etc/fail2ban ]; then
    RECOMMENDATIONS+=("Установи fail2ban для защиты SSH и веб-сервера")
fi
if command -v rkhunter &>/dev/null && command -v chkrootkit &>/dev/null; then
    RECOMMENDATIONS+=("Запусти rkhunter --check и chkrootkit для глубокой проверки")
else
    RECOMMENDATIONS+=("Установи rkhunter и chkrootkit: apt install rkhunter chkrootkit -y")
fi
if [ -f /etc/php/8.4/fpm/php.ini ]; then
    if [ "$DISPLAY_ERRORS" = "On" ] || [ "$DISPLAY_ERRORS" = "on" ]; then
        RECOMMENDATIONS+=("Отключи display_errors в /etc/php/8.4/fpm/php.ini")
    fi
    if [ "$EXPOSE_PHP" = "On" ] || [ "$EXPOSE_PHP" = "on" ]; then
        RECOMMENDATIONS+=("Отключи expose_php в /etc/php/8.4/fpm/php.ini")
    fi
fi
if [ -f /etc/nginx/sites-available/default ] && grep -q "autoindex on" /etc/nginx/sites-available/default; then
    RECOMMENDATIONS+=("Отключи autoindex в Nginx")
fi
if grep -q "PermitRootLogin yes" /etc/ssh/sshd_config; then
    RECOMMENDATIONS+=("Запрети root-доступ по SSH: PermitRootLogin no")
fi
if grep -q "PermitEmptyPasswords yes" /etc/ssh/sshd_config; then
    RECOMMENDATIONS+=("Запрети пустые пароли в SSH: PermitEmptyPasswords no")
fi

if [ ${#RECOMMENDATIONS[@]} -eq 0 ]; then
    echo -e "${GREEN}✅ Критических проблем не обнаружено. Рекомендуется запускать проверку раз в неделю.${NC}"
else
    for rec in "${RECOMMENDATIONS[@]}"; do
        echo -e "  $rec"
    done
fi

echo -e "\n${GREEN}Проверка завершена. Скрипт создан для регулярного мониторинга.${NC}"
