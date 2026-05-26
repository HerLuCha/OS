# Лабораторная 3a. Реализация скрипта bash

## Задание
Разработать исполняемый Bash-скрипт, моделирующий 5 типовых задач разработки и администрирования:
Сбор базовой информации о системе (ОС, ОЗУ, диск, сеть).
Симуляция Git-воркфлоу (создание репозитория, ветвление, коммиты, просмотр лога).
Генерация тестового лога, его парсинг и статистика по уровням (INFO, WARN, ERROR).
Создание резервной копии директории, поиск и удаление файлов старше 30 дней.
Мониторинг процессов по имени пользователя (PID, CPU, MEM).
## Требования:
Использовать только стандартные утилиты macOS (BSD-совместимые).
Вести лог выполнения в файл.
Корректно обрабатывать ввод пользователя и ошибки.
Автоматически очищать временные ресурсы.

## Исходный код скрипта
```
#!/usr/bin/env bash

 Строгий режим: незаполненные переменные → ошибка, пайпы → ошибка
set -uo pipefail
# Примечание: set -e не используется, так как в интерактивном меню 
# некоторые команды (grep, pgrep) возвращают 1 при отсутствии совпадений.

LOG_FILE="$HOME/devops_lab.log"

# Функция логирования (терминал + файл)
log() {
    echo -e "$1" | tee -a "$LOG_FILE"
}

# Информация о системе
sys_info() {
    log "\n${BLUE}=== 1. Информация о системе ===${NC}"
    log "OS: macOS $(sw_vers -productVersion 2>/dev/null || uname -r)"
    log "Ядро: $(uname -v)"
    log "Время работы: $(uptime | sed -E 's/.*up //')"
    
    log "${CYAN}Диск (/):${NC}"
    df -h / | awk 'NR==2 {printf "  Использовано: %s | Свободно: %s (%s)\n", $3, $4, $5}'
    
    local total_ram
    total_ram=$(sysctl -n hw.memsize 2>/dev/null || echo 0)
    if [[ "$total_ram" -gt 0 ]]; then
        log "${CYAN}ОЗУ: $(awk "BEGIN {printf \"%.2f\", $total_ram/1024/1024/1024}") ГБ${NC}"
    fi
    
    log "${CYAN}Сетевые интерфейсы (IPv4):${NC}"
    ifconfig | grep -E "^[a-z0-9]+:" | sed 's/://' | while read -r iface; do
        ip=$(ifconfig "$iface" 2>/dev/null | grep "inet " | awk '{print $2}')
        log "  ${iface}: ${ip:-нет IPv4}"
    done
}

# Симуляция Git-воркфлоу
git_sim() {
    log "\n${BLUE}=== 2. Симуляция Git-воркфлоу ===${NC}"
    local tmpdir
    tmpdir=$(mktemp -d) || { log "${RED}Ошибка создания временной директории${NC}"; return; }
    
    cd "$tmpdir" || exit 1
    git init -q
    echo "# Проект" > README.md
    echo "print('test')" > main.py
    
    git add -A
    git commit -m "Initial commit" -q 2>/dev/null
    git checkout -b feature/login -q
    echo "## Auth" >> README.md
    git add -A
    git commit -m "feat: add login docs" -q 2>/dev/null
    
    log "Репозиторий создан: ${tmpdir}"
    log "${CYAN}История коммитов:${NC}"
    git log --oneline --graph
    log "${CYAN}Активная ветка: $(git branch --show-current)${NC}"
    
    cd - > /dev/null || exit 1
    read -rp "Нажмите Enter для очистки временных файлов..."
    rm -rf "$tmpdir"
}

# Анализ логов
log_analysis() {
    log "\n${BLUE}=== 3. Анализ логов ===${NC}"
    local logfile="/tmp/devops_lab_app.log"
    cat > "$logfile" << 'EOF'
2026-05-26 10:00:01 INFO  Service started
2026-05-26 10:00:05 DEBUG Config loaded
2026-05-26 10:01:20 WARN  Memory > 80%
2026-05-26 10:02:10 ERROR DB connection failed
2026-05-26 10:02:45 ERROR Retry timeout
2026-05-26 10:03:00 INFO  Connection restored
EOF

    log "Сгенерирован тестовый лог: ${logfile}"
    log "${CYAN}Статистика по уровням:${NC}"
    for level in INFO DEBUG WARN ERROR; do
        count=$(grep -c "$level" "$logfile" 2>/dev/null || echo 0)
        log "  ${level}: ${count}"
    done
    log "${CYAN}Все ошибки:${NC}"
    grep "ERROR" "$logfile" | sed 's/^/  /'
    rm -f "$logfile"
}

# Бэкап и очистка
backup_cleanup() {
    log "\n${BLUE}=== 4. Резервное копирование и очистка ===${NC}"
    local srcdir="/tmp/devops_lab_data"
    mkdir -p "$srcdir"
    echo "data" > "$srcdir/new.txt"
    touch -t 202501010000 "$srcdir/old.dat"  # файл старше 30 дней
    
    local backup="$HOME/devops_lab_backup_$(date +%Y%m%d_%H%M).tar.gz"
    log "Архивация: ${srcdir} → ${backup}"
    tar -czf "$backup" -C "$(dirname "$srcdir")" "$(basename "$srcdir")"
    log "Размер архива: $(du -h "$backup" | cut -f1)"
    
    log "${CYAN}Удаление файлов старше 30 дней в ${srcdir}...${NC}"
    find "$srcdir" -type f -mtime +30 -print -delete 2>/dev/null || log "  Нечего удалять"
    rm -rf "$srcdir" "$backup"
}

# Мониторинг процессов
proc_monitor() {
    log "\n${BLUE}=== 5. Мониторинг процессов ===${NC}"
    read -rp "Введите имя процесса (например: zsh, node, python3): " proc_name
    [[ -z "$proc_name" ]] && { log "${RED}Имя не указано${NC}"; return; }
    
    local pids
    pids=$(pgrep -x "$proc_name" 2>/dev/null || pgrep -f "$proc_name" 2>/dev/null || true)
    if [[ -z "$pids" ]]; then
        log "${YELLOW}Процесс '${proc_name}' не запущен.${NC}"
        return
    fi
    
    log "${CYAN}Найдено PID: ${pids}${NC}"
    for pid in $pids; do
        log "--- PID $pid ---"
        ps -p "$pid" -o pid,pcpu,pmem,comm 2>/dev/null || log "  Нет доступа"
    done
}

# Главное меню
show_menu() {
    clear
    log "${GREEN}╔══════════════════════════════════════════╗${NC}"
    log "${GREEN}║   Лабораторная работа: DevOps Bash       ║${NC}"
    log "${GREEN}╚══════════════════════════════════════════╝${NC}"
    log "${YELLOW}Меню:${NC}"
    log "  1) Системная информация"
    log "  2) Git-воркфлоу"
    log "  3) Анализ логов"
    log "  4) Бэкап и очистка"
    log "  5) Мониторинг процессов"
    log "  6) Выход"
}

main() {
    log "${GREEN}Скрипт запущен. Лог: ${LOG_FILE}${NC}"
    while true; do
        show_menu
        read -rp "Выбор [1-6]: " choice
        case $choice in
            1) sys_info ;; 2) git_sim ;; 3) log_analysis ;;
            4) backup_cleanup ;; 5) proc_monitor ;;
            6) log "\n${GREEN}Завершение работы.${NC}"; exit 0 ;;
            *) log "${RED}Неверный ввод. Повторите.${NC}" ;;
        esac
        read -rp "Нажмите Enter для возврата в меню..."
    done
}

main
```
## Примечания
Скрипт не требует sudo, Homebrew или python/perl. Работает на чистой macOS.
Все временные файлы создаются в /tmp или через mktemp и удаляются сразу после использования.
Лог сохраняется в $HOME/devops_lab.log для удобства
