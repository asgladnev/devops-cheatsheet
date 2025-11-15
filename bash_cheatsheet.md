# Полная шпаргалка по Bash для DevOps

## 📋 Содержание
- [Основы синтаксиса](#основы-синтаксиса)
- [Переменные](#переменные)
- [Условия](#условия)
- [Циклы](#циклы)
- [Функции](#функции)
- [Массивы](#массивы)
- [Работа с файлами](#работа-с-файлами)
- [Работа со строками](#работа-со-строками)
- [Обработка ошибок](#обработка-ошибок)
- [Лучшие практики](#лучшие-практики)

---

## Основы синтаксиса

### Шебанг (Shebang)
```bash
#!/usr/bin/env bash
# Лучше чем #!/bin/bash - работает независимо от расположения bash
```

### Строгий режим (Fail Fast)
```bash
#!/usr/bin/env bash
set -euo pipefail
# -e: exit при ошибке команды
# -u: exit при использовании неопределенной переменной
# -o pipefail: exit если хоть одна команда в pipeline завершилась с ошибкой

# Опционально для debug:
set -x  # Выводит каждую команду перед выполнением
```

### Комментарии
```bash
# Однострочный комментарий

: '
Многострочный
комментарий
'
```

---

## Переменные

### Наименование переменных
```bash
# Константы - UPPERCASE с подчеркиванием
readonly DATABASE_HOST="localhost"
readonly MAX_RETRIES=3

# Глобальные переменные - lowercase с подчеркиванием
script_name="$(basename "$0")"
log_file="/var/log/app.log"

# Локальные переменные в функциях - тоже lowercase
function process_data() {
    local input_file="$1"
    local output_dir="$2"
}

# Environment переменные - UPPERCASE
export PATH="/usr/local/bin:$PATH"
export NODE_ENV="production"
```

### Объявление и использование
```bash
# Объявление
name="value"                    # БЕЗ пробелов вокруг =
readonly CONST="constant"       # Константа
declare -r CONST2="constant"    # Альтернативный способ

# Использование - ВСЕГДА в двойных кавычках
echo "$name"                    # ✓ Правильно
echo "${name}"                  # ✓ Правильно (явная граница)
echo $name                      # ✗ Неправильно (word splitting)

# Значение по умолчанию
echo "${var:-default}"          # Если var пустая, вернет "default"
echo "${var:=default}"          # Если var пустая, присвоит и вернет "default"
echo "${var:?error message}"    # Exit с ошибкой если var пустая
echo "${var:+alternate}"        # Если var НЕ пустая, вернет "alternate"

# Длина строки
echo "${#name}"

# Удаление префикса/суффикса
file="script.sh.bak"
echo "${file%.bak}"             # script.sh (удалить .bak)
echo "${file#script.}"          # sh.bak (удалить script.)
```

### Экспорт переменных
```bash
# Экспорт для дочерних процессов
export DATABASE_URL="postgres://localhost/db"

# Экспорт и объявление одной строкой
export LOG_LEVEL="debug"

# Проверка переменных окружения
if [[ -z "${API_KEY:-}" ]]; then
    echo "ERROR: API_KEY not set" >&2
    exit 1
fi
```

### Командная подстановка
```bash
# Современный синтаксис (предпочтительный)
current_date=$(date +%Y-%m-%d)
file_count=$(find . -type f | wc -l)

# Старый синтаксис (избегать)
old_style=`date`
```

---

## Условия

### Структура if-else
```bash
# Базовый синтаксис
if [[ условие ]]; then
    команды
elif [[ другое_условие ]]; then
    команды
else
    команды
fi

# Использовать [[ ]] вместо [ ] (более безопасно и функционально)
```

### Проверка файлов
```bash
if [[ -f "$file" ]]; then          # файл существует и это обычный файл
    echo "File exists"
fi

if [[ -d "$dir" ]]; then           # директория существует
    echo "Directory exists"
fi

if [[ -e "$path" ]]; then          # путь существует (файл или директория)
    echo "Path exists"
fi

if [[ -r "$file" ]]; then          # файл читаемый
if [[ -w "$file" ]]; then          # файл записываемый
if [[ -x "$file" ]]; then          # файл исполняемый
if [[ -s "$file" ]]; then          # файл не пустой
if [[ -L "$link" ]]; then          # это символическая ссылка

if [[ "$file1" -nt "$file2" ]]; then  # file1 новее file2
if [[ "$file1" -ot "$file2" ]]; then  # file1 старше file2
```

### Проверка строк
```bash
if [[ -z "$string" ]]; then        # строка пустая
if [[ -n "$string" ]]; then        # строка не пустая

if [[ "$str1" == "$str2" ]]; then  # строки равны
if [[ "$str1" != "$str2" ]]; then  # строки не равны

if [[ "$string" =~ ^[0-9]+$ ]]; then  # regex: только цифры
if [[ "$string" == prefix* ]]; then    # начинается с "prefix"
if [[ "$string" == *suffix ]]; then    # заканчивается на "suffix"
```

### Проверка чисел
```bash
if [[ "$num1" -eq "$num2" ]]; then # равно
if [[ "$num1" -ne "$num2" ]]; then # не равно
if [[ "$num1" -lt "$num2" ]]; then # меньше
if [[ "$num1" -le "$num2" ]]; then # меньше или равно
if [[ "$num1" -gt "$num2" ]]; then # больше
if [[ "$num1" -ge "$num2" ]]; then # больше или равно
```

### Логические операторы
```bash
# AND
if [[ -f "$file" && -r "$file" ]]; then
    echo "File exists and readable"
fi

# OR
if [[ "$var" == "yes" || "$var" == "y" ]]; then
    echo "Confirmed"
fi

# NOT
if [[ ! -f "$file" ]]; then
    echo "File does not exist"
fi

# Комбинации
if [[ -f "$file" && ( "$var" == "yes" || "$var" == "y" ) ]]; then
    echo "Complex condition"
fi
```

### Case statement
```bash
case "$variable" in
    pattern1)
        commands
        ;;
    pattern2|pattern3)  # Несколько паттернов
        commands
        ;;
    *)  # Default случай
        commands
        ;;
esac

# Пример с опциями
case "$1" in
    start|run)
        start_service
        ;;
    stop)
        stop_service
        ;;
    restart)
        stop_service
        start_service
        ;;
    *)
        echo "Usage: $0 {start|stop|restart}" >&2
        exit 1
        ;;
esac
```

---

## Циклы

### For loop
```bash
# По элементам списка
for item in item1 item2 item3; do
    echo "$item"
done

# По файлам (правильный способ)
for file in *.txt; do
    [[ -e "$file" ]] || continue  # Защита от несуществующих файлов
    echo "Processing: $file"
done

# По выводу команды (ПЛОХО - word splitting)
for file in $(ls *.txt); do  # ✗ Не делайте так!
    echo "$file"
done

# По диапазону чисел
for i in {1..10}; do
    echo "Number: $i"
done

# C-style for loop
for ((i=0; i<10; i++)); do
    echo "Index: $i"
done

# По массиву
array=("one" "two" "three")
for element in "${array[@]}"; do
    echo "$element"
done
```

### While loop
```bash
# Базовый while
counter=0
while [[ $counter -lt 10 ]]; do
    echo "$counter"
    ((counter++))
done

# Чтение файла построчно (ПРАВИЛЬНЫЙ способ)
while IFS= read -r line; do
    echo "Line: $line"
done < "$file"

# Бесконечный цикл
while true; do
    echo "Running..."
    sleep 1
done

# Until (противоположность while)
counter=0
until [[ $counter -ge 10 ]]; do
    echo "$counter"
    ((counter++))
done
```

### Break и Continue
```bash
for i in {1..10}; do
    if [[ $i -eq 5 ]]; then
        continue  # Пропустить итерацию
    fi
    if [[ $i -eq 8 ]]; then
        break     # Выйти из цикла
    fi
    echo "$i"
done
```

---

## Функции

### Объявление функций
```bash
# Предпочтительный синтаксис
function my_function() {
    local param1="$1"
    local param2="$2"
    
    echo "Processing $param1 and $param2"
}

# Альтернативный синтаксис (без function keyword)
my_function() {
    commands
}
```

### Параметры функции
```bash
function process_file() {
    local file="$1"
    local output_dir="${2:-/tmp}"  # Значение по умолчанию
    
    # Проверка обязательных параметров
    if [[ -z "$file" ]]; then
        echo "ERROR: file parameter required" >&2
        return 1
    fi
    
    # Проверка всех параметров
    echo "Positional params: $@"    # Все параметры
    echo "Number of params: $#"     # Количество параметров
    
    # Использование
    echo "Processing $file -> $output_dir"
}

# Вызов
process_file "/path/to/file" "/output"
```

### Возврат значений
```bash
# Через return code (0-255)
function check_status() {
    if [[ -f "$1" ]]; then
        return 0  # Успех
    else
        return 1  # Ошибка
    fi
}

# Проверка return code
if check_status "/tmp/file"; then
    echo "File exists"
fi

# Через echo (для возврата строк)
function get_timestamp() {
    echo "$(date +%Y%m%d_%H%M%S)"
}

# Захват вывода
timestamp=$(get_timestamp)
```

### Локальные и глобальные переменные
```bash
global_var="global"

function demo() {
    local local_var="local"       # Видна только в функции
    global_var="modified global"  # Изменяет глобальную
    
    echo "Local: $local_var"
    echo "Global: $global_var"
}

demo
echo "$global_var"  # "modified global"
echo "$local_var"   # Пусто (не определена)
```

### Лучшие практики функций
```bash
function deploy_application() {
    # 1. Объявить все локальные переменные в начале
    local app_name="$1"
    local environment="$2"
    local version="${3:-latest}"
    local result
    local exit_code
    
    # 2. Валидация входных данных
    if [[ -z "$app_name" || -z "$environment" ]]; then
        echo "ERROR: Missing required parameters" >&2
        echo "Usage: deploy_application <app_name> <environment> [version]" >&2
        return 1
    fi
    
    # 3. Основная логика
    echo "Deploying $app_name ($version) to $environment..."
    
    result=$(kubectl apply -f "deploy-${app_name}.yaml" 2>&1)
    exit_code=$?
    
    # 4. Обработка результата
    if [[ $exit_code -eq 0 ]]; then
        echo "SUCCESS: Deployment completed"
        return 0
    else
        echo "ERROR: Deployment failed: $result" >&2
        return 1
    fi
}
```

---

## Массивы

### Объявление и инициализация
```bash
# Индексированный массив
array=()                          # Пустой массив
array=(element1 element2 element3)
array=("one" "two" "three")

# Ассоциативный массив (hash/dictionary)
declare -A assoc_array
assoc_array=(
    [key1]="value1"
    [key2]="value2"
)
```

### Операции с массивами
```bash
# Добавление элементов
array+=("new_element")
array[3]="fourth element"

# Доступ к элементам
echo "${array[0]}"                # Первый элемент
echo "${array[-1]}"               # Последний элемент
echo "${array[@]}"                # Все элементы
echo "${array[*]}"                # Все элементы (одна строка)

# Количество элементов
echo "${#array[@]}"

# Получение индексов
echo "${!array[@]}"

# Слайс массива
echo "${array[@]:1:3}"            # Элементы с индекса 1, 3 штуки

# Итерация по массиву
for element in "${array[@]}"; do
    echo "$element"
done

# Итерация с индексами
for index in "${!array[@]}"; do
    echo "Index: $index, Value: ${array[$index]}"
done
```

### Ассоциативные массивы
```bash
declare -A config
config[host]="localhost"
config[port]="5432"
config[database]="mydb"

# Доступ
echo "${config[host]}"

# Все ключи
echo "${!config[@]}"

# Все значения
echo "${config[@]}"

# Проверка существования ключа
if [[ -v config[host] ]]; then
    echo "Key exists"
fi

# Итерация
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done
```

---

## Работа с файлами

### Чтение файлов
```bash
# Построчно (ПРАВИЛЬНЫЙ способ)
while IFS= read -r line; do
    echo "Line: $line"
done < "$filename"

# С обработкой последней строки без \n
while IFS= read -r line || [[ -n "$line" ]]; do
    echo "$line"
done < "$filename"

# Чтение в массив
mapfile -t lines < "$filename"
# или
readarray -t lines < "$filename"

# Чтение в переменную
content=$(<"$filename")

# Чтение первых N строк
head -n 10 "$filename"

# Чтение последних N строк
tail -n 10 "$filename"

# Следить за файлом (tail -f)
tail -f "$logfile"
```

### Запись в файлы
```bash
# Перезапись файла
echo "content" > "$filename"

# Добавление в конец
echo "content" >> "$filename"

# Запись нескольких строк (heredoc)
cat > "$filename" << 'EOF'
Line 1
Line 2
Line 3
EOF

# Heredoc с подстановкой переменных
cat > "$filename" << EOF
Current date: $(date)
User: $USER
EOF

# Запись в файл с проверкой
{
    echo "Line 1"
    echo "Line 2"
} > "$filename" || {
    echo "ERROR: Failed to write to $filename" >&2
    exit 1
}
```

### Операции с файлами
```bash
# Копирование
cp "$source" "$dest"
cp -r "$source_dir" "$dest_dir"   # Рекурсивно
cp -p "$source" "$dest"            # Сохранить атрибуты

# Перемещение/переименование
mv "$old_name" "$new_name"

# Удаление
rm "$file"
rm -rf "$directory"                # Рекурсивно, force

# Создание директории
mkdir -p "$directory"              # С родительскими директориями

# Проверка и создание
[[ -d "$directory" ]] || mkdir -p "$directory"

# Временные файлы (безопасно)
temp_file=$(mktemp)
temp_dir=$(mktemp -d)

# Очистка при выходе
trap 'rm -f "$temp_file"' EXIT
```

### Поиск файлов
```bash
# Find
find . -name "*.log"                    # По имени
find . -type f -mtime -7                # Файлы, изменённые за последние 7 дней
find . -type f -size +100M              # Файлы больше 100MB
find . -name "*.tmp" -delete            # Найти и удалить

# Find с exec
find . -name "*.log" -exec gzip {} \;   # Сжать каждый файл
find . -type f -exec chmod 644 {} +     # Изменить права (быстрее)

# Grep
grep -r "pattern" .                     # Рекурсивный поиск
grep -i "pattern" file                  # Игнорировать регистр
grep -v "pattern" file                  # Инвертировать (исключить)
grep -E "pattern1|pattern2" file        # Extended regex
```

---

## Работа со строками

### Манипуляции со строками
```bash
string="Hello, World!"

# Длина
echo "${#string}"                       # 13

# Извлечение подстроки
echo "${string:7:5}"                    # World (с позиции 7, длина 5)
echo "${string:7}"                      # World! (с позиции 7 до конца)
echo "${string: -6}"                    # World! (последние 6 символов)

# Замена (первое вхождение)
echo "${string/World/Universe}"         # Hello, Universe!

# Замена (все вхождения)
echo "${string//o/0}"                   # Hell0, W0rld!

# Удаление префикса
path="/usr/local/bin/script"
echo "${path#/usr/}"                    # local/bin/script
echo "${path##*/}"                      # script (имя файла)

# Удаление суффикса
file="script.sh.bak"
echo "${file%.bak}"                     # script.sh
echo "${file%%.*}"                      # script (удалить всё после первой точки)

# Изменение регистра (Bash 4+)
echo "${string^^}"                      # HELLO, WORLD!
echo "${string,,}"                      # hello, world!
echo "${string^}"                       # Hello, World! (первая буква)
```

### Проверка строк
```bash
# Содержит подстроку
if [[ "$string" == *"substring"* ]]; then
    echo "Contains substring"
fi

# Начинается с
if [[ "$string" == "prefix"* ]]; then
    echo "Starts with prefix"
fi

# Заканчивается на
if [[ "$string" == *"suffix" ]]; then
    echo "Ends with suffix"
fi

# Regex
if [[ "$string" =~ ^[A-Z][a-z]+$ ]]; then
    echo "Matches pattern"
fi
```

### Разбиение строк
```bash
# IFS (Internal Field Separator)
string="one,two,three"

# Разбить в массив
IFS=',' read -ra array <<< "$string"
for item in "${array[@]}"; do
    echo "$item"
done

# Или без изменения IFS
oldIFS="$IFS"
IFS=','
read -ra array <<< "$string"
IFS="$oldIFS"
```

---

## Обработка ошибок

### Exit codes
```bash
# Проверка кода возврата
command
if [[ $? -eq 0 ]]; then
    echo "Success"
else
    echo "Failed"
fi

# Лучше:
if command; then
    echo "Success"
else
    echo "Failed"
fi

# Или:
command && echo "Success" || echo "Failed"
```

### Trap для очистки
```bash
#!/usr/bin/env bash
set -euo pipefail

# Временные файлы
temp_file=$(mktemp)
temp_dir=$(mktemp -d)

# Функция очистки
cleanup() {
    local exit_code=$?
    echo "Cleaning up..." >&2
    rm -f "$temp_file"
    rm -rf "$temp_dir"
    exit "$exit_code"
}

# Установка trap
trap cleanup EXIT INT TERM

# Ваш код здесь
echo "Working..." > "$temp_file"
```

### Логирование
```bash
# Функции логирования
readonly LOG_FILE="/var/log/script.log"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

log_error() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] ERROR: $*" | tee -a "$LOG_FILE" >&2
}

log_debug() {
    if [[ "${DEBUG:-}" == "true" ]]; then
        echo "[$(date +'%Y-%m-%d %H:%M:%S')] DEBUG: $*" | tee -a "$LOG_FILE" >&2
    fi
}

# Использование
log "Starting process"
log_error "Something went wrong"
log_debug "Variable value: $var"
```

### Обработка ошибок в pipeline
```bash
# Без set -o pipefail
command1 | command2 | command3
# Вернёт код возврата только command3

# С set -o pipefail
set -o pipefail
command1 | command2 | command3
# Вернёт код первой неудачной команды

# Проверка каждой команды
if ! command1 | command2 | command3; then
    echo "Pipeline failed" >&2
    exit 1
fi
```

---

## Лучшие практики

### Шаблон DevOps скрипта
```bash
#!/usr/bin/env bash

#######################################
# Description: Краткое описание скрипта
# Globals:
#   LOG_FILE
#   DEBUG
# Arguments:
#   $1 - первый аргумент
#   $2 - второй аргумент
# Returns:
#   0 if success, non-zero on error
#######################################

set -euo pipefail

# Константы
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"

# Цвета для вывода (опционально)
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly NC='\033[0m' # No Color

# Функции логирования
log_info() {
    echo -e "${GREEN}[INFO]${NC} $*" | tee -a "$LOG_FILE"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $*" | tee -a "$LOG_FILE" >&2
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $*" | tee -a "$LOG_FILE" >&2
}

# Cleanup функция
cleanup() {
    local exit_code=$?
    log_info "Cleaning up..."
    # Ваш cleanup код
    exit "$exit_code"
}

trap cleanup EXIT INT TERM

# Функция помощи
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS] <required_arg>

Description of the script

OPTIONS:
    -h, --help      Show this help message
    -v, --verbose   Enable verbose output
    -d, --debug     Enable debug mode

EXAMPLES:
    $SCRIPT_NAME file.txt
    $SCRIPT_NAME --verbose file.txt

EOF
}

# Парсинг аргументов
parse_args() {
    local verbose=false
    local debug=false
    
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -h|--help)
                usage
                exit 0
                ;;
            -v|--verbose)
                verbose=true
                shift
                ;;
            -d|--debug)
                debug=true
                set -x
                shift
                ;;
            -*)
                log_error "Unknown option: $1"
                usage
                exit 1
                ;;
            *)
                # Позиционные аргументы
                break
                ;;
        esac
    done
    
    # Проверка обязательных аргументов
    if [[ $# -lt 1 ]]; then
        log_error "Missing required argument"
        usage
        exit 1
    fi
    
    readonly VERBOSE="$verbose"
    readonly DEBUG="$debug"
}

# Основная функция
main() {
    parse_args "$@"
    
    log_info "Starting $SCRIPT_NAME"
    
    # Ваша основная логика здесь
    
    log_info "Completed successfully"
}

# Запуск
main "$@"
```

### Проверки и валидация
```bash
# Проверка зависимостей
check_dependencies() {
    local missing_deps=()
    
    for cmd in jq curl kubectl; do
        if ! command -v "$cmd" &> /dev/null; then
            missing_deps+=("$cmd")
        fi
    done
    
    if [[ ${#missing_deps[@]} -gt 0 ]]; then
        log_error "Missing dependencies: ${missing_deps[*]}"
        exit 1
    fi
}

# Проверка прав доступа
check_permissions() {
    if [[ $EUID -ne 0 ]]; then
        log_error "This script must be run as root"
        exit 1
    fi
}

# Проверка переменных окружения
check_env() {
    local required_vars=(
        "DATABASE_URL"
        "API_KEY"
        "AWS_REGION"
    )
    
    local missing_vars=()
    for var in "${required_vars[@]}"; do
        if [[ -z "${!var:-}" ]]; then
            missing_vars+=("$var")
        fi
    done
    
    if [[ ${#missing_vars[@]} -gt 0 ]]; then
        log_error "Missing environment variables: ${missing_vars[*]}"
        exit 1
    fi
}
```

### Безопасность
```bash
# Не используйте eval
eval "$command"  # ✗ ОПАСНО!

# Используйте массивы для команд
command=("kubectl" "get" "pods" "-n" "$namespace")
"${command[@]}"

# Кавычки для всех переменных
rm "$file"                    # ✓ Безопасно
rm $file                      # ✗ Опасно (word splitting)

# Проверяйте пути
if [[ "$path" == /* ]]; then
    echo "Absolute path"
else
    echo "Relative path"
fi

# Избегайте небезопасных функций
# Вместо:
filename="user_input.txt"
cat $filename  # ✗ Опасно

# Используйте:
filename="user_input.txt"
if [[ -f "$filename" && "$filename" != */* ]]; then
    cat "$filename"
fi
```

### Производительность
```bash
# Избегайте лишних процессов
# Плохо:
cat file | grep pattern

# Хорошо:
grep pattern file

# Используйте встроенные возможности bash
# Вместо: echo "$var" | cut -d: -f1
echo "${var%%:*}"

# Группировка команд для редиректа
{
    echo "Line 1"
    echo "Line 2"
    echo "Line 3"
} > file

# Вместо:
echo "Line 1" > file
echo "Line 2" >> file
echo "Line 3" >> file
```

### Переносимость
```bash
# Используйте #!/usr/bin/env bash вместо #!/bin/bash

# Избегайте bashisms если нужен POSIX shell
# Bashism:
[[ $var == "value" ]]
# POSIX:
[ "$var" = "value" ]

# Проверяйте версию bash если используете новые функции
if ((BASH_VERSINFO[0] < 4)); then
    echo "Bash 4+ required" >&2
    exit 1
fi
```

### Отладка
```bash
# Включить трассировку
set -x

# Отключить трассировку
set +x

# Трассировка только части скрипта
(set -x; complex_command; another_command)

# Debug функция
debug() {
    if [[ "${DEBUG:-}" == "true" ]]; then
        echo "[DEBUG] $*" >&2
    fi
}

# PS4 для лучшего вывода при set -x
export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
```

---

## Полезные паттерны DevOps

### Retry механизм
```bash
# Функция с повторными попытками
retry() {
    local max_attempts="$1"
    local delay="$2"
    local command=("${@:3}")
    local attempt=1
    
    while [[ $attempt -le $max_attempts ]]; do
        log_info "Attempt $attempt/$max_attempts: ${command[*]}"
        
        if "${command[@]}"; then
            log_info "Success on attempt $attempt"
            return 0
        fi
        
        if [[ $attempt -lt $max_attempts ]]; then
            log_warn "Failed, retrying in ${delay}s..."
            sleep "$delay"
        fi
        
        ((attempt++))
    done
    
    log_error "All $max_attempts attempts failed"
    return 1
}

# Использование
retry 3 5 curl -f "https://api.example.com/health"
```

### Параллельное выполнение
```bash
# Запуск задач в фоне
process_file() {
    local file="$1"
    echo "Processing $file..."
    sleep 2
    echo "Done: $file"
}

# Экспорт функции для использования в subshell
export -f process_file

# Параллельная обработка
max_jobs=4
job_count=0

for file in *.txt; do
    # Ждать если достигнут лимит
    while [[ $(jobs -r | wc -l) -ge $max_jobs ]]; do
        sleep 0.1
    done
    
    # Запустить в фоне
    process_file "$file" &
done

# Ждать завершения всех задач
wait

echo "All files processed"
```

### Прогресс бар
```bash
show_progress() {
    local current="$1"
    local total="$2"
    local width=50
    
    local percentage=$((current * 100 / total))
    local completed=$((width * current / total))
    
    printf "\rProgress: ["
    printf "%${completed}s" | tr ' ' '='
    printf "%$((width - completed))s" | tr ' ' ' '
    printf "] %3d%% (%d/%d)" "$percentage" "$current" "$total"
    
    if [[ $current -eq $total ]]; then
        echo ""
    fi
}

# Использование
total=100
for i in $(seq 1 $total); do
    show_progress "$i" "$total"
    sleep 0.05
done
```

### Работа с JSON (jq)
```bash
# Проверка наличия jq
if ! command -v jq &> /dev/null; then
    log_error "jq is required but not installed"
    exit 1
fi

# Парсинг JSON
json='{"name":"John","age":30,"city":"New York"}'

# Извлечение значения
name=$(echo "$json" | jq -r '.name')
age=$(echo "$json" | jq -r '.age')

# Извлечение из файла
name=$(jq -r '.name' < config.json)

# Массивы в JSON
json='{"users":["alice","bob","charlie"]}'
mapfile -t users < <(echo "$json" | jq -r '.users[]')

# Фильтрация
echo "$json" | jq '.users[] | select(. == "alice")'

# Создание JSON
jq -n \
    --arg name "Alice" \
    --arg email "alice@example.com" \
    '{name: $name, email: $email}'

# Модификация JSON
jq '.version = "2.0"' config.json > config.tmp && mv config.tmp config.json
```

### Работа с YAML (yq)
```bash
# Чтение YAML
value=$(yq eval '.database.host' config.yaml)

# Изменение YAML
yq eval '.database.port = 5433' -i config.yaml

# Массивы
yq eval '.servers[0].name' config.yaml
```

### Docker операции
```bash
# Проверка Docker
check_docker() {
    if ! command -v docker &> /dev/null; then
        log_error "Docker is not installed"
        exit 1
    fi
    
    if ! docker info &> /dev/null; then
        log_error "Docker daemon is not running"
        exit 1
    fi
}

# Очистка старых контейнеров и образов
docker_cleanup() {
    log_info "Cleaning up Docker resources..."
    
    # Остановить все контейнеры
    docker ps -q | xargs -r docker stop
    
    # Удалить остановленные контейнеры
    docker ps -aq | xargs -r docker rm
    
    # Удалить неиспользуемые образы
    docker image prune -af
    
    # Удалить неиспользуемые volume
    docker volume prune -f
    
    log_info "Cleanup completed"
}

# Билд и пуш образа
build_and_push() {
    local image_name="$1"
    local tag="$2"
    local dockerfile="${3:-Dockerfile}"
    
    log_info "Building image: $image_name:$tag"
    docker build -t "$image_name:$tag" -f "$dockerfile" . || return 1
    
    log_info "Pushing image: $image_name:$tag"
    docker push "$image_name:$tag" || return 1
    
    log_info "Successfully built and pushed $image_name:$tag"
}
```

### Kubernetes операции
```bash
# Проверка kubectl
check_kubectl() {
    if ! command -v kubectl &> /dev/null; then
        log_error "kubectl is not installed"
        exit 1
    fi
    
    if ! kubectl cluster-info &> /dev/null; then
        log_error "Cannot connect to Kubernetes cluster"
        exit 1
    fi
}

# Ожидание готовности deployment
wait_for_deployment() {
    local namespace="$1"
    local deployment="$2"
    local timeout="${3:-300}"
    
    log_info "Waiting for deployment $deployment in namespace $namespace..."
    
    if kubectl rollout status deployment/"$deployment" \
        -n "$namespace" \
        --timeout="${timeout}s"; then
        log_info "Deployment $deployment is ready"
        return 0
    else
        log_error "Deployment $deployment failed to become ready"
        return 1
    fi
}

# Получение логов с нескольких подов
get_pod_logs() {
    local namespace="$1"
    local label_selector="$2"
    local since="${3:-1h}"
    
    log_info "Fetching logs from pods with label $label_selector"
    
    local pods
    pods=$(kubectl get pods -n "$namespace" -l "$label_selector" -o name)
    
    for pod in $pods; do
        log_info "Logs from $pod:"
        kubectl logs -n "$namespace" "$pod" --since="$since" || true
        echo "---"
    done
}

# Scale deployment
scale_deployment() {
    local namespace="$1"
    local deployment="$2"
    local replicas="$3"
    
    log_info "Scaling $deployment to $replicas replicas..."
    kubectl scale deployment/"$deployment" \
        -n "$namespace" \
        --replicas="$replicas"
}
```

### AWS CLI операции
```bash
# Проверка AWS CLI
check_aws() {
    if ! command -v aws &> /dev/null; then
        log_error "AWS CLI is not installed"
        exit 1
    fi
    
    if ! aws sts get-caller-identity &> /dev/null; then
        log_error "AWS credentials not configured"
        exit 1
    fi
}

# Загрузка файла в S3
upload_to_s3() {
    local file="$1"
    local bucket="$2"
    local key="$3"
    
    log_info "Uploading $file to s3://$bucket/$key"
    
    if aws s3 cp "$file" "s3://$bucket/$key"; then
        log_info "Upload successful"
        return 0
    else
        log_error "Upload failed"
        return 1
    fi
}

# Получение секрета из AWS Secrets Manager
get_secret() {
    local secret_name="$1"
    local region="${2:-us-east-1}"
    
    aws secretsmanager get-secret-value \
        --secret-id "$secret_name" \
        --region "$region" \
        --query SecretString \
        --output text
}

# Список EC2 инстансов
list_ec2_instances() {
    local tag_key="$1"
    local tag_value="$2"
    
    aws ec2 describe-instances \
        --filters "Name=tag:$tag_key,Values=$tag_value" \
        --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PrivateIpAddress,Tags[?Key==`Name`].Value|[0]]' \
        --output table
}
```

### Git операции
```bash
# Проверка Git репозитория
check_git_repo() {
    if ! git rev-parse --git-dir &> /dev/null; then
        log_error "Not a git repository"
        exit 1
    fi
}

# Получение текущей ветки
get_current_branch() {
    git rev-parse --abbrev-ref HEAD
}

# Проверка чистоты рабочей директории
check_clean_working_tree() {
    if ! git diff-index --quiet HEAD --; then
        log_error "Working tree is not clean. Commit or stash changes."
        return 1
    fi
}

# Создание тега и пуш
create_and_push_tag() {
    local tag="$1"
    local message="$2"
    
    log_info "Creating tag $tag"
    git tag -a "$tag" -m "$message"
    
    log_info "Pushing tag $tag"
    git push origin "$tag"
}

# Получение последнего тега
get_latest_tag() {
    git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0"
}

# Инкремент версии (semver)
increment_version() {
    local version="$1"
    local position="${2:-patch}"  # major, minor, patch
    
    # Удалить 'v' если есть
    version="${version#v}"
    
    IFS='.' read -r major minor patch <<< "$version"
    
    case "$position" in
        major)
            ((major++))
            minor=0
            patch=0
            ;;
        minor)
            ((minor++))
            patch=0
            ;;
        patch)
            ((patch++))
            ;;
        *)
            log_error "Invalid version position: $position"
            return 1
            ;;
    esac
    
    echo "v$major.$minor.$patch"
}
```

### HTTP запросы
```bash
# Проверка доступности URL
check_url() {
    local url="$1"
    local max_attempts="${2:-3}"
    local timeout="${3:-10}"
    
    for attempt in $(seq 1 "$max_attempts"); do
        if curl -sSf -m "$timeout" "$url" &> /dev/null; then
            log_info "URL $url is accessible"
            return 0
        fi
        
        if [[ $attempt -lt $max_attempts ]]; then
            sleep 2
        fi
    done
    
    log_error "URL $url is not accessible after $max_attempts attempts"
    return 1
}

# POST запрос с JSON
api_post() {
    local url="$1"
    local data="$2"
    local token="${3:-}"
    
    local headers=(-H "Content-Type: application/json")
    
    if [[ -n "$token" ]]; then
        headers+=(-H "Authorization: Bearer $token")
    fi
    
    curl -sSf -X POST "$url" \
        "${headers[@]}" \
        -d "$data"
}

# GET запрос с обработкой ошибок
api_get() {
    local url="$1"
    local token="${2:-}"
    local response
    local http_code
    
    response=$(curl -sSf -w "\n%{http_code}" \
        -H "Authorization: Bearer $token" \
        "$url")
    
    http_code=$(echo "$response" | tail -n1)
    response=$(echo "$response" | sed '$d')
    
    if [[ "$http_code" -ge 200 && "$http_code" -lt 300 ]]; then
        echo "$response"
        return 0
    else
        log_error "HTTP $http_code: $response"
        return 1
    fi
}
```

### Работа с датами и временем
```bash
# Текущие дата и время
current_date=$(date +%Y-%m-%d)
current_time=$(date +%H:%M:%S)
current_timestamp=$(date +%s)
iso_timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)

# Дата N дней назад/вперёд
date_7_days_ago=$(date -d "7 days ago" +%Y-%m-%d)
date_in_30_days=$(date -d "30 days" +%Y-%m-%d)

# Конвертация timestamp
date -d @1234567890

# Разница между датами
start_time=$(date +%s)
# ... выполнение команд ...
end_time=$(date +%s)
duration=$((end_time - start_time))
echo "Execution time: ${duration}s"

# Форматирование времени выполнения
format_duration() {
    local duration="$1"
    local hours=$((duration / 3600))
    local minutes=$(((duration % 3600) / 60))
    local seconds=$((duration % 60))
    
    printf "%02d:%02d:%02d" "$hours" "$minutes" "$seconds"
}
```

### Работа с процессами
```bash
# Проверка запущенного процесса
is_process_running() {
    local process_name="$1"
    pgrep -f "$process_name" > /dev/null
}

# Ожидание завершения процесса
wait_for_process() {
    local pid="$1"
    local timeout="${2:-60}"
    local elapsed=0
    
    while kill -0 "$pid" 2>/dev/null; do
        if [[ $elapsed -ge $timeout ]]; then
            log_error "Process $pid did not finish within ${timeout}s"
            return 1
        fi
        sleep 1
        ((elapsed++))
    done
    
    log_info "Process $pid finished"
    return 0
}

# Убить процесс по имени
kill_process() {
    local process_name="$1"
    local signal="${2:-TERM}"
    
    local pids
    pids=$(pgrep -f "$process_name")
    
    if [[ -z "$pids" ]]; then
        log_warn "No processes found matching: $process_name"
        return 0
    fi
    
    for pid in $pids; do
        log_info "Killing process $pid with signal $signal"
        kill -"$signal" "$pid"
    done
}

# Получение использования ресурсов процессом
get_process_stats() {
    local pid="$1"
    
    if [[ ! -d "/proc/$pid" ]]; then
        log_error "Process $pid not found"
        return 1
    fi
    
    local cpu
    local mem
    cpu=$(ps -p "$pid" -o %cpu --no-headers)
    mem=$(ps -p "$pid" -o %mem --no-headers)
    
    echo "CPU: ${cpu}% | Memory: ${mem}%"
}
```

### Блокировки (lock files)
```bash
# Создание lock файла
acquire_lock() {
    local lock_file="$1"
    local timeout="${2:-0}"
    local elapsed=0
    
    while true; do
        if mkdir "$lock_file" 2>/dev/null; then
            # Lock acquired
            trap "rm -rf '$lock_file'" EXIT
            return 0
        fi
        
        if [[ $timeout -eq 0 || $elapsed -ge $timeout ]]; then
            log_error "Could not acquire lock: $lock_file"
            return 1
        fi
        
        sleep 1
        ((elapsed++))
    done
}

# Использование
readonly LOCK_FILE="/var/lock/myscript.lock"

if acquire_lock "$LOCK_FILE" 30; then
    log_info "Lock acquired, proceeding..."
    # Ваш код здесь
else
    log_error "Could not acquire lock, exiting"
    exit 1
fi
```

### Email уведомления
```bash
# Отправка email (требует mailutils или mailx)
send_email() {
    local to="$1"
    local subject="$2"
    local body="$3"
    local attachment="${4:-}"
    
    if [[ -n "$attachment" ]]; then
        echo "$body" | mail -s "$subject" -a "$attachment" "$to"
    else
        echo "$body" | mail -s "$subject" "$to"
    fi
}

# Отправка через SMTP (с curl)
send_email_smtp() {
    local to="$1"
    local subject="$2"
    local body="$3"
    local smtp_server="${SMTP_SERVER:-smtp.gmail.com:587}"
    local smtp_user="${SMTP_USER}"
    local smtp_pass="${SMTP_PASS}"
    
    local email_content
    email_content=$(cat <<EOF
From: $smtp_user
To: $to
Subject: $subject

$body
EOF
)
    
    echo "$email_content" | curl --ssl-reqd \
        --url "smtp://$smtp_server" \
        --user "$smtp_user:$smtp_pass" \
        --mail-from "$smtp_user" \
        --mail-rcpt "$to" \
        --upload-file -
}
```

### Генерация отчётов
```bash
# Генерация HTML отчёта
generate_html_report() {
    local title="$1"
    local output_file="$2"
    shift 2
    local content=("$@")
    
    cat > "$output_file" << EOF
<!DOCTYPE html>
<html>
<head>
    <title>$title</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        h1 { color: #333; }
        .info { background: #e7f3fe; padding: 10px; margin: 10px 0; }
        .error { background: #ffebee; padding: 10px; margin: 10px 0; }
        .success { background: #e8f5e9; padding: 10px; margin: 10px 0; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #4CAF50; color: white; }
    </style>
</head>
<body>
    <h1>$title</h1>
    <p>Generated: $(date)</p>
EOF
    
    for line in "${content[@]}"; do
        echo "    <div class=\"info\">$line</div>" >> "$output_file"
    done
    
    cat >> "$output_file" << 'EOF'
</body>
</html>
EOF
    
    log_info "Report generated: $output_file"
}

# Генерация CSV отчёта
generate_csv_report() {
    local output_file="$1"
    local headers="$2"
    shift 2
    local rows=("$@")
    
    echo "$headers" > "$output_file"
    
    for row in "${rows[@]}"; do
        echo "$row" >> "$output_file"
    done
    
    log_info "CSV report generated: $output_file"
}
```

---

## Полезные однострочные команды

### Системная информация
```bash
# Использование диска
df -h

# Использование памяти
free -h

# Топ процессов по CPU
ps aux --sort=-%cpu | head -10

# Топ процессов по памяти
ps aux --sort=-%mem | head -10

# Информация о системе
uname -a

# Uptime системы
uptime

# Количество CPU
nproc

# Открытые порты
ss -tulpn
# или
netstat -tulpn
```

### Работа с текстом
```bash
# Уникальные строки
sort file.txt | uniq

# Подсчёт уникальных строк
sort file.txt | uniq -c | sort -rn

# Топ 10 самых частых слов
cat file.txt | tr -s '[:space:]' '\n' | sort | uniq -c | sort -rn | head -10

# Замена строки во всех файлах
find . -type f -name "*.txt" -exec sed -i 's/old/new/g' {} +

# Удаление пустых строк
grep -v '^ file.txt

# Извлечение email адресов
grep -oE '\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b' file.txt

# Извлечение IP адресов
grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' file.txt
```

### Архивация и сжатие
```bash
# Создать tar.gz архив
tar -czf archive.tar.gz directory/

# Извлечь tar.gz
tar -xzf archive.tar.gz

# Создать zip
zip -r archive.zip directory/

# Извлечь zip
unzip archive.zip

# Сжатие с прогрессом
tar -czf - directory/ | pv > archive.tar.gz

# Размер директории
du -sh directory/

# Топ 10 самых больших файлов
find . -type f -exec du -h {} + | sort -rh | head -10
```

---

## Часто используемые сниппеты

### Проверка успешности команды
```bash
if command; then
    echo "Success"
else
    echo "Failed" >&2
    exit 1
fi

# Короткая форма
command || { echo "Failed" >&2; exit 1; }
command && echo "Success"
```

### Чтение конфигурационного файла
```bash
# config.conf:
# KEY1=value1
# KEY2=value2

while IFS='=' read -r key value; do
    # Пропустить комментарии и пустые строки
    [[ "$key" =~ ^#.*$ || -z "$key" ]] && continue
    
    # Удалить пробелы
    key=$(echo "$key" | tr -d ' ')
    value=$(echo "$value" | tr -d ' ')
    
    # Создать переменную
    declare "$key=$value"
done < config.conf
```

### Работа с CSV
```bash
# Чтение CSV с заголовком
{
    IFS=, read -r header
    while IFS=, read -r col1 col2 col3; do
        echo "Processing: $col1, $col2, $col3"
    done
} < data.csv
```

### Проверка интернет-соединения
```bash
check_internet() {
    if ping -c 1 8.8.8.8 &> /dev/null; then
        return 0
    else
        return 1
    fi
}

# Или через curl
check_internet() {
    curl -sSf https://www.google.com > /dev/null 2>&1
}
```

### Меню выбора
```bash
show_menu() {
    echo "Select an option:"
    echo "1) Option 1"
    echo "2) Option 2"
    echo "3) Option 3"
    echo "4) Exit"
    
    read -rp "Enter choice [1-4]: " choice
    
    case "$choice" in
        1)
            echo "You selected option 1"
            ;;
        2)
            echo "You selected option 2"
            ;;
        3)
            echo "You selected option 3"
            ;;
        4)
            echo "Exiting..."
            exit 0
            ;;
        *)
            echo "Invalid option" >&2
            return 1
            ;;
    esac
}
```

### Подтверждение действия
```bash
confirm() {
    local prompt="${1:-Are you sure?}"
    local default="${2:-N}"
    
    if [[ "$default" == "Y" ]]; then
        prompt="$prompt [Y/n]: "
    else
        prompt="$prompt [y/N]: "
    fi
    
    read -rp "$prompt" response
    response=${response:-$default}
    
    case "$response" in
        [yY][eE][sS]|[yY])
            return 0
            ;;
        *)
            return 1
            ;;
    esac
}

# Использование
if confirm "Delete all files?"; then
    rm -rf *
fi
```

---

## Чеклист для ревью скрипта

- [ ] Используется `#!/usr/bin/env bash` в начале
- [ ] Установлен `set -euo pipefail`
- [ ] Все переменные в двойных кавычках: `"$var"`
- [ ] Использован `readonly` для констант
- [ ] Все переменные в функциях объявлены как `local`
- [ ] Имена переменных: lowercase_with_underscores
- [ ] Имена констант: UPPERCASE_WITH_UNDERSCORES
- [ ] Имена функций: lowercase_with_underscores
- [ ] Используется `[[ ]]` вместо `[ ]`
- [ ] Проверка обязательных зависимостей
- [ ] Проверка обязательных аргументов
- [ ] Функция `usage()` для help
- [ ] Обработка ошибок с понятными сообщениями
- [ ] Логирование важных операций
- [ ] Cleanup функция с `trap`
- [ ] Избегается использование `eval`
- [ ] Нет hardcoded паролей/токенов
- [ ] Комментарии для сложной логики
- [ ] Проверен shellcheck (если доступен)

---

## Полезные ссылки

- [ShellCheck](https://www.shellcheck.net/) - Линтер для shell скриптов
- [Bash Hackers Wiki](https://wiki.bash-hackers.org/)
- [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)
- [Advanced Bash-Scripting Guide](https://tldp.org/LDP/abs/html/)
- [explainshell.com](https://explainshell.com/) - Объяснение команд bash

---

**Примечание**: Эта шпаргалка покрывает Bash 4+. Для старых версий некоторые возможности могут быть недоступны.