# HCL (HashiCorp Configuration Language) - Полная шпаргалка

## 📋 Содержание
1. [Базовый синтаксис](#базовый-синтаксис)
2. [Типы данных](#типы-данных)
3. [Переменные](#переменные)
4. [Блоки](#блоки)
5. [Выражения и операторы](#выражения-и-операторы)
6. [Функции](#функции)
7. [Условные выражения](#условные-выражения)
8. [Циклы и итерации](#циклы-и-итерации)
9. [Locals (локальные значения)](#locals-локальные-значения)
10. [Outputs](#outputs)
11. [Data Sources](#data-sources)
12. [Модули](#модули)
13. [Terraform-специфичные конструкции](#terraform-специфичные-конструкции)
14. [Best Practices](#best-practices)

---

## Базовый синтаксис

### Структура файла
```hcl
# Комментарий однострочный

/*
  Многострочный
  комментарий
*/

// Также однострочный комментарий (стиль C++)

# Основная структура: БЛОК "ТИП" "ИМЯ" { }
resource "aws_instance" "example" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

### Правила именования
```hcl
# Имена блоков, переменных, ресурсов:
# - Только буквы, цифры, дефис (-) и подчёркивание (_)
# - Должны начинаться с буквы или подчёркивания
# - Рекомендуется snake_case для переменных и ресурсов
# - Рекомендуется kebab-case для имён модулей

# ✅ Правильно
variable "instance_count" {}
resource "aws_s3_bucket" "my_bucket" {}
locals {
  environment_name = "production"
}

# ❌ Неправильно
variable "instance-count" {}  # Допустимо, но не рекомендуется
variable "2_instances" {}     # Начинается с цифры
variable "my bucket" {}       # Пробелы не допускаются
```

### Структура атрибутов
```hcl
# Атрибуты записываются как ключ = значение
resource "aws_instance" "web" {
  # Простое значение
  ami = "ami-12345678"
  
  # Число
  count = 3
  
  # Булево значение
  monitoring = true
  
  # Список
  security_groups = ["sg-12345", "sg-67890"]
  
  # Map (словарь)
  tags = {
    Name        = "WebServer"
    Environment = "Production"
  }
  
  # Вложенный блок
  ebs_block_device {
    device_name = "/dev/sda1"
    volume_size = 20
  }
}
```

---

## Типы данных

### Примитивные типы
```hcl
# String (строка)
variable "region" {
  type    = string
  default = "us-east-1"
}

# Number (число - целое или с плавающей точкой)
variable "instance_count" {
  type    = number
  default = 5
}

variable "cpu_credits" {
  type    = number
  default = 0.5
}

# Bool (булево значение)
variable "enable_monitoring" {
  type    = bool
  default = true
}

# Null
variable "optional_value" {
  type    = string
  default = null
}
```

### Коллекции
```hcl
# List (список) - упорядоченная последовательность значений одного типа
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "port_numbers" {
  type    = list(number)
  default = [80, 443, 8080]
}

# Set (множество) - неупорядоченная коллекция уникальных значений
variable "unique_tags" {
  type    = set(string)
  default = ["production", "web", "critical"]
}

# Map (словарь) - коллекция пар ключ-значение
variable "instance_tags" {
  type = map(string)
  default = {
    Environment = "production"
    Application = "web"
    Owner       = "devops-team"
  }
}

variable "port_configs" {
  type = map(number)
  default = {
    http  = 80
    https = 443
    ssh   = 22
  }
}

# Tuple (кортеж) - упорядоченная последовательность значений разных типов
variable "mixed_list" {
  type    = tuple([string, number, bool])
  default = ["example", 42, true]
}

# Object (объект) - коллекция именованных атрибутов с разными типами
variable "server_config" {
  type = object({
    name          = string
    instance_type = string
    disk_size     = number
    enabled       = bool
  })
  default = {
    name          = "web-server"
    instance_type = "t2.micro"
    disk_size     = 20
    enabled       = true
  }
}
```

### Сложные типы
```hcl
# Вложенные коллекции
variable "vpc_config" {
  type = object({
    cidr_block = string
    subnets = list(object({
      cidr   = string
      zone   = string
      public = bool
    }))
    tags = map(string)
  })
  default = {
    cidr_block = "10.0.0.0/16"
    subnets = [
      {
        cidr   = "10.0.1.0/24"
        zone   = "us-east-1a"
        public = true
      },
      {
        cidr   = "10.0.2.0/24"
        zone   = "us-east-1b"
        public = false
      }
    ]
    tags = {
      Environment = "production"
    }
  }
}

# Any - принимает любой тип (не рекомендуется)
variable "flexible_value" {
  type    = any
  default = "can be anything"
}
```

---

## Переменные

### Объявление переменных
```hcl
# Базовое объявление
variable "instance_type" {
  description = "Тип EC2 инстанса"
  type        = string
  default     = "t2.micro"
}

# Переменная без значения по умолчанию (обязательная)
variable "vpc_id" {
  description = "ID VPC (обязательный параметр)"
  type        = string
}

# С валидацией
variable "environment" {
  description = "Название окружения"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment должен быть: dev, staging, или prod."
  }
}

# С несколькими правилами валидации
variable "instance_count" {
  description = "Количество инстансов"
  type        = number
  default     = 1
  
  validation {
    condition     = var.instance_count > 0
    error_message = "Количество инстансов должно быть больше 0."
  }
  
  validation {
    condition     = var.instance_count <= 10
    error_message = "Количество инстансов не может превышать 10."
  }
}

# С чувствительными данными
variable "database_password" {
  description = "Пароль для базы данных"
  type        = string
  sensitive   = true  # Скрывает значение в выводе
}

# С nullable (может быть null)
variable "optional_tag" {
  description = "Опциональный тег"
  type        = string
  default     = null
  nullable    = true
}
```

### Использование переменных
```hcl
# Обращение к переменной через var.
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = var.instance_type
  
  tags = {
    Name        = "web-${var.environment}"
    Environment = var.environment
  }
}

# В выражениях
locals {
  instance_name = "server-${var.environment}-${var.instance_count}"
  is_production = var.environment == "prod"
}
```

### Способы передачи значений переменных

#### 1. Через командную строку
```bash
terraform apply -var="instance_type=t2.small" -var="environment=prod"
```

#### 2. Через файлы переменных (.tfvars)
```hcl
# terraform.tfvars (загружается автоматически)
instance_type = "t2.small"
environment   = "prod"
instance_count = 3

vpc_config = {
  cidr_block = "10.0.0.0/16"
  subnets = [
    {
      cidr   = "10.0.1.0/24"
      zone   = "us-east-1a"
      public = true
    }
  ]
  tags = {
    Environment = "production"
  }
}

# prod.tfvars (загружается явно)
# terraform apply -var-file="prod.tfvars"
```

#### 3. Через переменные окружения
```bash
export TF_VAR_instance_type="t2.small"
export TF_VAR_environment="prod"
terraform apply
```

#### 4. Интерактивный ввод
```bash
# Terraform запросит значения для обязательных переменных
terraform apply
```

---

## Блоки

### Типы блоков
```hcl
# Resource - создаёт инфраструктурные ресурсы
resource "провайдер_тип_ресурса" "локальное_имя" {
  # конфигурация
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# Data - читает существующие ресурсы
data "провайдер_тип_данных" "локальное_имя" {
  # параметры поиска
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

# Provider - настройка провайдера
provider "провайдер" {
  # конфигурация провайдера
}

provider "aws" {
  region     = "us-east-1"
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
}

# Terraform - настройки самого Terraform
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
  
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}

# Module - использование модуля
module "имя_модуля" {
  source = "./путь/к/модулю"
  
  # входные переменные модуля
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# Output - экспорт значений
output "имя_вывода" {
  value       = выражение
  description = "описание"
  sensitive   = true/false
}

output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "Публичный IP веб-сервера"
}

# Variable - объявление переменной (см. раздел Переменные)

# Locals - локальные значения (см. раздел Locals)
```

### Meta-аргументы блоков resource

```hcl
# depends_on - явная зависимость
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  depends_on = [
    aws_security_group.web_sg,
    aws_subnet.public
  ]
}

# count - создание нескольких копий
resource "aws_instance" "web" {
  count = 3
  
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  tags = {
    Name = "web-${count.index}"  # web-0, web-1, web-2
  }
}

# Обращение к ресурсам с count
# aws_instance.web[0]
# aws_instance.web[1]
# aws_instance.web[*].id  # все ID

# for_each - создание ресурсов по словарю или множеству
resource "aws_instance" "web" {
  for_each = {
    prod    = "t2.large"
    staging = "t2.medium"
    dev     = "t2.micro"
  }
  
  ami           = "ami-12345678"
  instance_type = each.value
  
  tags = {
    Name        = "web-${each.key}"
    Environment = each.key
  }
}

# Обращение к ресурсам с for_each
# aws_instance.web["prod"]
# aws_instance.web["dev"].id

# for_each с set
resource "aws_iam_user" "developers" {
  for_each = toset(["alice", "bob", "charlie"])
  
  name = each.key  # each.value идентично each.key для set
}

# provider - выбор конкретного провайдера
resource "aws_instance" "west" {
  provider = aws.west
  
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# lifecycle - управление жизненным циклом ресурса
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  lifecycle {
    # Создать новый ресурс перед удалением старого
    create_before_destroy = true
    
    # Предотвратить удаление ресурса
    prevent_destroy = false
    
    # Игнорировать изменения определённых атрибутов
    ignore_changes = [
      tags,
      ami,
    ]
    
    # Игнорировать все изменения
    # ignore_changes = all
    
    # Заменить ресурс если условие истинно
    replace_triggered_by = [
      aws_security_group.web_sg.id
    ]
  }
}

# provisioner - выполнение действий при создании/удалении
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  # Выполняется при создании
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ip_addresses.txt"
  }
  
  # Выполняется при удалении
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Instance destroyed' >> log.txt"
  }
  
  # Продолжить даже если provisioner упал
  provisioner "remote-exec" {
    on_failure = continue  # или fail (по умолчанию)
    
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx"
    ]
  }
}
```

---

## Выражения и операторы

### Арифметические операторы
```hcl
locals {
  # Сложение
  sum = 5 + 3  # 8
  
  # Вычитание
  difference = 10 - 4  # 6
  
  # Умножение
  product = 7 * 6  # 42
  
  # Деление
  quotient = 20 / 4  # 5
  
  # Остаток от деления
  remainder = 17 % 5  # 2
  
  # Отрицание
  negative = -10
  
  # Комбинированные операции
  complex = (5 + 3) * 2 - 10 / 2  # 11
}
```

### Операторы сравнения
```hcl
locals {
  # Равенство
  is_equal = 5 == 5  # true
  
  # Неравенство
  not_equal = 5 != 3  # true
  
  # Больше
  greater = 10 > 5  # true
  
  # Больше или равно
  greater_or_equal = 10 >= 10  # true
  
  # Меньше
  less = 3 < 7  # true
  
  # Меньше или равно
  less_or_equal = 5 <= 5  # true
  
  # Сравнение строк
  string_equal = "hello" == "hello"  # true
  string_compare = "apple" < "banana"  # true (лексикографическое сравнение)
}
```

### Логические операторы
```hcl
locals {
  # AND (логическое И)
  and_result = true && false  # false
  
  # OR (логическое ИЛИ)
  or_result = true || false  # true
  
  # NOT (логическое НЕ)
  not_result = !true  # false
  
  # Комбинированные условия
  complex = (5 > 3) && (10 < 20)  # true
  
  # Приоритет операторов (NOT > AND > OR)
  priority = true || false && false  # true (как true || (false && false))
  with_parens = (true || false) && false  # false
}
```

### Строковые операции
```hcl
locals {
  # Интерполяция строк
  name = "John"
  greeting = "Hello, ${name}!"  # "Hello, John!"
  
  # Многострочные строки
  multiline = <<EOT
This is a
multiline string
with ${name}
EOT
  
  # Heredoc с отступами (удаляет общие отступы)
  indented = <<-EOT
    Line 1
      Line 2
    Line 3
  EOT
  
  # Шаблонные директивы
  # %{ if ... } - условие
  # %{ for ... } - цикл
  # %{ endif }, %{ endfor } - закрывающие теги
  # ~ - убирает пробелы/переносы строк
  
  template_example = <<-EOT
    %{ if var.environment == "prod" ~}
    Production Environment
    %{ else ~}
    Development Environment
    %{ endif ~}
    
    Servers:
    %{ for instance in aws_instance.web ~}
    - ${instance.id}: ${instance.public_ip}
    %{ endfor ~}
  EOT
  
  # Экранирование ${ и %{
  escaped = "Literal $${dollar} and %%{percent}"
}
```

### Операторы для коллекций
```hcl
locals {
  list1 = [1, 2, 3]
  list2 = [4, 5, 6]
  
  # Конкатенация списков
  combined_list = concat(list1, list2)  # [1, 2, 3, 4, 5, 6]
  
  # Доступ к элементу по индексу
  first_element = list1[0]  # 1
  
  # Отрицательные индексы (с конца)
  last_element = list1[-1]  # 3
  
  # Срезы [start:end] (end не включается)
  slice = list1[0:2]  # [1, 2]
  slice_from = list1[1:]  # [2, 3]
  slice_to = list1[:2]  # [1, 2]
  
  # Map (словарь)
  map1 = { a = 1, b = 2 }
  map2 = { c = 3, d = 4 }
  
  # Объединение map
  combined_map = merge(map1, map2)  # { a = 1, b = 2, c = 3, d = 4 }
  
  # Доступ к значению по ключу
  value_by_key = map1["a"]  # 1
  # или
  value_dot = map1.a  # 1
  
  # Проверка наличия элемента
  has_element = contains(list1, 2)  # true
  
  # Длина коллекции
  list_length = length(list1)  # 3
  map_length = length(map1)  # 2
}
```

### Splat-выражения
```hcl
# Получение атрибута от всех элементов
locals {
  # Для списка ресурсов с count
  instance_ids = aws_instance.web[*].id
  # Аналогично: [aws_instance.web[0].id, aws_instance.web[1].id, ...]
  
  # Для ресурсов с for_each (возвращает map)
  instance_ips = {
    for key, instance in aws_instance.web : key => instance.public_ip
  }
  
  # Атрибут с вложенными блоками
  # Если resource имеет несколько блоков ebs_block_device
  all_device_names = aws_instance.web[*].ebs_block_device[*].device_name
  # Результат: список списков [["/dev/sda1", "/dev/sdb1"], ["/dev/sda1"], ...]
  
  # Flatten для получения одноуровневого списка
  flat_device_names = flatten(aws_instance.web[*].ebs_block_device[*].device_name)
}
```

---

## Функции

### Числовые функции
```hcl
locals {
  # abs - абсолютное значение
  absolute = abs(-42)  # 42
  
  # ceil - округление вверх
  ceiling = ceil(4.3)  # 5
  
  # floor - округление вниз
  flooring = floor(4.8)  # 4
  
  # max - максимум
  maximum = max(5, 12, 9)  # 12
  
  # min - минимум
  minimum = min(5, 12, 9)  # 5
  
  # pow - возведение в степень
  power = pow(2, 8)  # 256
  
  # log - натуральный логарифм
  logarithm = log(100, 10)  # 2
  
  # signum - знак числа (-1, 0, 1)
  sign = signum(-15)  # -1
  
  # parseint - преобразование строки в число
  parsed = parseint("42", 10)  # 42
}
```

### Строковые функции
```hcl
locals {
  # format - форматирование строки (как printf)
  formatted = format("Hello, %s! You have %d messages.", "Alice", 5)
  # "Hello, Alice! You have 5 messages."
  
  # formatlist - применение format к спискам
  names = ["Alice", "Bob", "Charlie"]
  greetings = formatlist("Hello, %s!", names)
  # ["Hello, Alice!", "Hello, Bob!", "Hello, Charlie!"]
  
  # join - объединение списка в строку
  joined = join(", ", ["a", "b", "c"])  # "a, b, c"
  
  # split - разделение строки на список
  splitted = split(",", "a,b,c")  # ["a", "b", "c"]
  
  # lower - в нижний регистр
  lowercase = lower("HELLO")  # "hello"
  
  # upper - в верхний регистр
  uppercase = upper("hello")  # "HELLO"
  
  # title - первая буква каждого слова заглавная
  titled = title("hello world")  # "Hello World"
  
  # trim - удаление символов с концов
  trimmed = trim("  hello  ", " ")  # "hello"
  
  # trimprefix - удаление префикса
  no_prefix = trimprefix("prefix_name", "prefix_")  # "name"
  
  # trimsuffix - удаление суффикса
  no_suffix = trimsuffix("name_suffix", "_suffix")  # "name"
  
  # trimspace - удаление пробелов с концов
  no_spaces = trimspace("  hello  ")  # "hello"
  
  # substr - подстрока
  substring = substr("hello world", 0, 5)  # "hello"
  
  # replace - замена подстроки
  replaced = replace("hello world", "world", "terraform")  # "hello terraform"
  
  # regex - поиск по регулярному выражению
  matched = regex("[a-z]+", "hello123")  # "hello"
  
  # regexall - все совпадения
  all_matches = regexall("[0-9]+", "a1b2c3")  # ["1", "2", "3"]
  
  # strrev - реверс строки
  reversed = strrev("hello")  # "olleh"
  
  # chomp - удаление переноса строки в конце
  chomped = chomp("hello\n")  # "hello"
}
```

### Функции для коллекций
```hcl
locals {
  # concat - объединение списков
  list_concat = concat([1, 2], [3, 4], [5])  # [1, 2, 3, 4, 5]
  
  # contains - проверка наличия элемента
  has_item = contains([1, 2, 3], 2)  # true
  
  # distinct - уникальные элементы
  unique = distinct([1, 2, 2, 3, 1])  # [1, 2, 3]
  
  # element - получение элемента (с циклическим доступом)
  elem = element([1, 2, 3], 5)  # 3 (5 % 3 = 2, индекс 2)
  
  # flatten - сведение вложенных списков
  flattened = flatten([[1, 2], [3, 4]])  # [1, 2, 3, 4]
  
  # index - индекс элемента
  position = index(["a", "b", "c"], "b")  # 1
  
  # keys - ключи map
  map_keys = keys({ a = 1, b = 2 })  # ["a", "b"]
  
  # values - значения map
  map_values = values({ a = 1, b = 2 })  # [1, 2]
  
  # length - длина коллекции
  list_len = length([1, 2, 3])  # 3
  map_len = length({ a = 1, b = 2 })  # 2
  string_len = length("hello")  # 5
  
  # lookup - получение значения из map с дефолтом
  looked_up = lookup({ a = 1, b = 2 }, "c", 0)  # 0
  
  # merge - объединение map
  merged = merge({ a = 1 }, { b = 2 }, { c = 3 })  # { a = 1, b = 2, c = 3 }
  
  # reverse - реверс списка
  reversed_list = reverse([1, 2, 3])  # [3, 2, 1]
  
  # setintersection - пересечение множеств
  intersection = setintersection(["a", "b", "c"], ["b", "c", "d"])  # ["b", "c"]
  
  # setproduct - декартово произведение
  product = setproduct(["a", "b"], [1, 2])
  # [["a", 1], ["a", 2], ["b", 1], ["b", 2]]
  
  # setsubtract - разность множеств
  subtraction = setsubtract(["a", "b", "c"], ["b"])  # ["a", "c"]
  
  # setunion - объединение множеств
  union = setunion(["a", "b"], ["b", "c"])  # ["a", "b", "c"]
  
  # slice - срез списка
  sliced = slice([1, 2, 3, 4, 5], 1, 4)  # [2, 3, 4]
  
  # sort - сортировка списка
  sorted = sort(["c", "a", "b"])  # ["a", "b", "c"]
  
  # sum - сумма чисел в списке
  total = sum([1, 2, 3, 4])  # 10
  
  # transpose - транспонирование map списков
  transposed = transpose({
    a = ["1", "2"]
    b = ["1", "3"]
  })
  # { "1" = ["a", "b"], "2" = ["a"], "3" = ["b"] }
  
  # chunklist - разбиение списка на чанки
  chunks = chunklist([1, 2, 3, 4, 5], 2)  # [[1, 2], [3, 4], [5]]
  
  # compact - удаление пустых строк
  compacted = compact(["a", "", "b", "", "c"])  # ["a", "b", "c"]
  
  # coalesce - первое непустое значение
  first_non_empty = coalesce("", "", "hello", "world")  # "hello"
  
  # coalescelist - первый непустой список
  first_list = coalescelist([], [], [1, 2], [3, 4])  # [1, 2]
  
  # matchkeys - фильтрация по ключам
  filtered = matchkeys(
    ["a", "b", "c"],
    ["x", "y", "z"],
    ["x", "z"]
  )  # ["a", "c"]
  
  # range - создание последовательности чисел
  sequence = range(5)        # [0, 1, 2, 3, 4]
  sequence2 = range(2, 7)    # [2, 3, 4, 5, 6]
  sequence3 = range(0, 10, 2)  # [0, 2, 4, 6, 8]
  
  # zipmap - создание map из списков ключей и значений
  zipped = zipmap(["a", "b", "c"], [1, 2, 3])  # { a = 1, b = 2, c = 3 }
}
```

### Функции кодирования
```hcl
locals {
  # base64encode - кодирование в base64
  encoded = base64encode("hello")
  
  # base64decode - декодирование из base64
  decoded = base64decode("aGVsbG8=")  # "hello"
  
  # base64gzip - сжатие gzip и кодирование в base64
  compressed = base64gzip("hello world")
  
  # jsonencode - преобразование в JSON
  json = jsonencode({
    name = "John"
    age  = 30
  })
  # "{\"name\":\"John\",\"age\":30}"
  
  # jsondecode - преобразование из JSON
  parsed = jsondecode("{\"name\":\"John\"}")  # { name = "John" }
  
  # csvdecode - преобразование CSV в список объектов
  csv_data = csvdecode("a,b,c\n1,2,3\n4,5,6")
  # [
  #   { a = "1", b = "2", c = "3" },
  #   { a = "4", b = "5", c = "6" }
  # ]
  
  # urlencode - URL-кодирование
  url_encoded = urlencode("hello world")  # "hello+world"
  
  # textencodebase64 - кодирование текста с указанной кодировкой
  encoded_utf8 = textencodebase64("hello", "UTF-8")
  
  # textdecodebase64 - декодирование текста
  decoded_utf8 = textdecodebase64(encoded_utf8, "UTF-8")
  
  # yamlencode - преобразование в YAML
  yaml = yamlencode({
    name = "John"
    items = [1, 2, 3]
  })
  
  # yamldecode - преобразование из YAML
  from_yaml = yamldecode("name: John\nage: 30")
}
```

### Функции файловой системы
```hcl
locals {
  # file - чтение файла как строки
  config_content = file("${path.module}/config.txt")
  
  # fileexists - проверка существования файла
  file_exists = fileexists("${path.module}/config.txt")  # true/false
  
  # fileset - поиск файлов по паттерну
  yaml_files = fileset(path.module, "*.yaml")  # ["config.yaml", "vars.yaml"]
  nested_files = fileset(path.module, "**/*.tf")  # все .tf файлы рекурсивно
  
  # templatefile - рендеринг шаблона с переменными
  rendered = templatefile("${path.module}/template.tpl", {
    name        = "John"
    environment = var.environment
  })
  
  # filebase64 - чтение файла как base64
  file_b64 = filebase64("${path.module}/image.png")
  
  # basename - имя файла из пути
  filename = basename("/path/to/file.txt")  # "file.txt"
  
  # dirname - директория из пути
  directory = dirname("/path/to/file.txt")  # "/path/to"
  
  # pathexpand - раскрытие ~ в пути
  expanded = pathexpand("~/configs")  # "/home/user/configs"
  
  # abspath - абсолютный путь
  absolute = abspath("./configs")  # "/current/working/dir/configs"
}

# Специальные переменные path
locals {
  # path.module - путь к текущему модулю
  module_path = path.module
  
  # path.root - путь к корневому модулю
  root_path = path.root
  
  # path.cwd - текущая рабочая директория
  current_dir = path.cwd
}
```

### Функции даты и времени
```hcl
locals {
  # timestamp - текущее время в UTC (RFC3339)
  now = timestamp()  # "2025-10-30T12:34:56Z"
  
  # formatdate - форматирование даты
  formatted_date = formatdate("DD MMM YYYY hh:mm:ss", timestamp())
  # "30 Oct 2025 12:34:56"
  
  # timeadd - добавление времени
  future = timeadd(timestamp(), "24h")  # +24 часа
  future2 = timeadd(timestamp(), "30m")  # +30 минут
  
  # timecmp - сравнение времени (-1, 0, 1)
  comparison = timecmp(timestamp(), timeadd(timestamp(), "1h"))  # -1
  
  # plantimestamp - время запуска terraform plan
  # (постоянное в течение выполнения)
  plan_time = plantimestamp()
}
```

### Хеш и криптографические функции
```hcl
locals {
  # md5 - MD5 хеш
  hash_md5 = md5("hello")
  
  # sha1 - SHA1 хеш
  hash_sha1 = sha1("hello")
  
  # sha256 - SHA256 хеш
  hash_sha256 = sha256("hello")
  
  # sha512 - SHA512 хеш
  hash_sha512 = sha512("hello")
  
  # base64sha256 - SHA256 в base64
  b64_sha256 = base64sha256("hello")
  
  # base64sha512 - SHA512 в base64
  b64_sha512 = base64sha512("hello")
  
  # filemd5 - MD5 хеш файла
  file_hash = filemd5("${path.module}/file.txt")
  
  # filesha1 - SHA1 хеш файла
  file_sha1 = filesha1("${path.module}/file.txt")
  
  # filesha256 - SHA256 хеш файла
  file_sha256 = filesha256("${path.module}/file.txt")
  
  # filesha512 - SHA512 хеш файла
  file_sha512 = filesha512("${path.module}/file.txt")
  
  # filebase64sha256 - SHA256 файла в base64
  file_b64_sha256 = filebase64sha256("${path.module}/file.txt")
  
  # filebase64sha512 - SHA512 файла в base64
  file_b64_sha512 = filebase64sha512("${path.module}/file.txt")
  
  # bcrypt - bcrypt хеш (для паролей)
  password_hash = bcrypt("my-password")
  
  # uuid - генерация UUID v5
  unique_id = uuid()
  
  # uuidv5 - UUID v5 из namespace и имени
  namespace = uuidv5("dns", "terraform.io")
  
  # rsadecrypt - RSA дешифрование
  # decrypted = rsadecrypt(encrypted_data, private_key)
}
```

### Функции для IP-адресов и сетей
```hcl
locals {
  # cidrhost - получение IP-адреса хоста
  host_ip = cidrhost("10.0.0.0/24", 5)  # "10.0.0.5"
  
  # cidrnetmask - маска сети
  netmask = cidrnetmask("10.0.0.0/24")  # "255.255.255.0"
  
  # cidrsubnet - создание подсети
  subnet = cidrsubnet("10.0.0.0/16", 8, 1)  # "10.0.1.0/24"
  
  # cidrsubnets - создание нескольких подсетей
  subnets = cidrsubnets("10.0.0.0/16", 8, 8, 8)
  # ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
}
```

### Функции преобразования типов
```hcl
locals {
  # tobool - преобразование в bool
  as_bool = tobool("true")  # true
  
  # tolist - преобразование в список
  as_list = tolist(["a", "b"])
  
  # tomap - преобразование в map
  as_map = tomap({ a = 1, b = 2 })
  
  # tonumber - преобразование в число
  as_number = tonumber("42")  # 42
  
  # toset - преобразование в set
  as_set = toset(["a", "b", "a"])  # ["a", "b"]
  
  # tostring - преобразование в строку
  as_string = tostring(42)  # "42"
  
  # try - попытка выполнения выражений (возвращает первое успешное)
  safe_value = try(
    var.optional_value,
    local.default_value,
    "fallback"
  )
  
  # can - проверка возможности выполнения выражения
  can_convert = can(tonumber("not-a-number"))  # false
  
  # type - получение типа значения (для отладки)
  value_type = type([1, 2, 3])  # "list"
}
```

### Terraform-специфичные функции
```hcl
locals {
  # terraform.workspace - текущий workspace
  workspace = terraform.workspace
  
  # sensitive - пометка значения как чувствительного
  secret = sensitive("password123")
  
  # nonsensitive - снятие пометки чувствительности
  revealed = nonsensitive(var.sensitive_value)
}
```

---

## Условные выражения

### Тернарный оператор
```hcl
locals {
  # Синтаксис: условие ? значение_если_true : значение_если_false
  
  environment = var.is_production ? "prod" : "dev"
  
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
  
  # Вложенные условия
  size = (
    var.environment == "prod" ? "large" :
    var.environment == "staging" ? "medium" :
    "small"
  )
  
  # С булевыми значениями
  monitoring_enabled = var.environment == "prod" ? true : false
  
  # С null
  optional_value = var.enable_feature ? "enabled" : null
  
  # Сложные условия
  complex_condition = (
    var.environment == "prod" && var.region == "us-east-1"
    ? "primary"
    : "secondary"
  )
}

resource "aws_instance" "web" {
  ami           = var.environment == "prod" ? var.prod_ami : var.dev_ami
  instance_type = var.is_production ? "t2.large" : "t2.micro"
  
  # Условное создание ресурса через count
  count = var.create_instance ? 1 : 0
  
  tags = {
    Name = var.environment == "prod" ? "Production Server" : "Dev Server"
  }
}
```

### Условные блоки
```hcl
# Условное включение блоков через dynamic
resource "aws_security_group" "example" {
  name = "example"
  
  dynamic "ingress" {
    for_each = var.enable_https ? [1] : []
    
    content {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}

# Условие через count в ресурсах
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0
  
  alarm_name          = "high-cpu-usage"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  threshold           = 80
}

# Обращение к условным ресурсам
output "alarm_arn" {
  value = var.enable_monitoring ? aws_cloudwatch_metric_alarm.high_cpu[0].arn : null
}
```

---

## Циклы и итерации

### For expressions (для создания коллекций)
```hcl
locals {
  # Для списков: [for элемент in список : преобразование]
  
  # Простое преобразование
  numbers = [1, 2, 3, 4, 5]
  doubled = [for n in numbers : n * 2]  # [2, 4, 6, 8, 10]
  
  # С условием (фильтрацией)
  even_numbers = [for n in numbers : n if n % 2 == 0]  # [2, 4]
  
  # Преобразование строк
  names = ["alice", "bob", "charlie"]
  uppercase_names = [for name in names : upper(name)]
  # ["ALICE", "BOB", "CHARLIE"]
  
  # Для map: {for ключ, значение in map : новый_ключ => новое_значение}
  
  # Преобразование map
  original_map = {
    a = 1
    b = 2
    c = 3
  }
  doubled_map = {
    for k, v in original_map : k => v * 2
  }
  # { a = 2, b = 4, c = 6 }
  
  # Изменение ключей
  prefixed_map = {
    for k, v in original_map : "key_${k}" => v
  }
  # { key_a = 1, key_b = 2, key_c = 3 }
  
  # Map с фильтрацией
  filtered_map = {
    for k, v in original_map : k => v if v > 1
  }
  # { b = 2, c = 3 }
  
  # Список в map
  items = ["apple", "banana", "cherry"]
  indexed_map = {
    for idx, item in items : idx => item
  }
  # { 0 = "apple", 1 = "banana", 2 = "cherry" }
  
  # Map в список
  tags = { Name = "Server", Environment = "Prod" }
  tag_list = [
    for key, value in tags : "${key}=${value}"
  ]
  # ["Name=Server", "Environment=Prod"]
  
  # Группировка (создание map списков) - используйте ... после =>
  servers = [
    { name = "web-1", type = "web" },
    { name = "web-2", type = "web" },
    { name = "db-1", type = "database" }
  ]
  
  grouped = {
    for server in servers : server.type => server.name...
  }
  # {
  #   web      = ["web-1", "web-2"]
  #   database = ["db-1"]
  # }
  
  # Вложенные циклы (flatten нужен для одноуровневого списка)
  matrix = flatten([
    for i in [1, 2] : [
      for j in ["a", "b"] : "${i}-${j}"
    ]
  ])
  # ["1-a", "1-b", "2-a", "2-b"]
}
```

### Dynamic blocks (для создания повторяющихся вложенных блоков)
```hcl
# Базовый синтаксис
resource "aws_security_group" "example" {
  name = "example"
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}

# Входные данные
variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]
}

# С использованием map
locals {
  ports = {
    http  = 80
    https = 443
    ssh   = 22
  }
}

resource "aws_security_group" "web" {
  name = "web-sg"
  
  dynamic "ingress" {
    for_each = local.ports
    
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "Allow ${ingress.key}"
    }
  }
}

# Доступ к итератору:
# - ingress.key - ключ текущего элемента
# - ingress.value - значение текущего элемента

# Переименование итератора через iterator
resource "aws_instance" "example" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
  
  dynamic "ebs_block_device" {
    for_each = var.ebs_volumes
    iterator = volume  # Переименовываем итератор
    
    content {
      device_name = volume.value.device_name
      volume_size = volume.value.size
      volume_type = volume.value.type
    }
  }
}

# Вложенные dynamic блоки
resource "aws_elastic_beanstalk_environment" "example" {
  name = "my-env"
  
  dynamic "setting" {
    for_each = var.settings
    
    content {
      namespace = setting.value.namespace
      name      = setting.value.name
      value     = setting.value.value
      
      dynamic "resource" {
        for_each = setting.value.resources != null ? setting.value.resources : []
        
        content {
          name = resource.value
        }
      }
    }
  }
}

# Условный dynamic блок
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
  
  # Создаётся только если переменная true
  dynamic "credit_specification" {
    for_each = var.enable_t2_unlimited ? [1] : []
    
    content {
      cpu_credits = "unlimited"
    }
  }
}
```

### For_each в ресурсах и модулях
```hcl
# For_each с map
resource "aws_instance" "servers" {
  for_each = {
    web = {
      instance_type = "t2.medium"
      ami           = "ami-web"
    }
    db = {
      instance_type = "t2.large"
      ami           = "ami-db"
    }
  }
  
  ami           = each.value.ami
  instance_type = each.value.instance_type
  
  tags = {
    Name = "server-${each.key}"
    Type = each.key
  }
}

# Обращение к ресурсам:
# aws_instance.servers["web"].id
# aws_instance.servers["db"].private_ip

# For_each с set
resource "aws_iam_user" "developers" {
  for_each = toset(["alice", "bob", "charlie"])
  
  name = each.key  # each.value идентично each.key для set
}

# For_each с преобразованием списка в map
variable "users" {
  type = list(object({
    name  = string
    role  = string
    email = string
  }))
}

resource "aws_iam_user" "users" {
  for_each = {
    for user in var.users : user.name => user
  }
  
  name = each.value.name
  
  tags = {
    Role  = each.value.role
    Email = each.value.email
  }
}

# For_each в модулях
module "vpc" {
  source = "./modules/vpc"
  
  for_each = {
    prod    = "10.0.0.0/16"
    staging = "10.1.0.0/16"
    dev     = "10.2.0.0/16"
  }
  
  environment = each.key
  cidr_block  = each.value
}

# Обращение к выходам модулей:
# module.vpc["prod"].vpc_id
# module.vpc["staging"].vpc_id
```

### Count vs For_each
```hcl
# COUNT - использовать когда:
# - Нужно N идентичных копий ресурса
# - Количество определяется числом
# - Порядок важен (используются индексы)

resource "aws_instance" "web" {
  count = 3
  
  ami           = "ami-12345"
  instance_type = "t2.micro"
  
  tags = {
    Name = "web-${count.index}"  # web-0, web-1, web-2
  }
}

# Обращение через индекс:
output "first_ip" {
  value = aws_instance.web[0].private_ip
}

output "all_ips" {
  value = aws_instance.web[*].private_ip
}

# FOR_EACH - использовать когда:
# - Каждый ресурс уникален (разные параметры)
# - Нужно идентифицировать по ключу, а не индексу
# - Можно безопасно добавлять/удалять элементы

resource "aws_instance" "servers" {
  for_each = {
    web = "t2.micro"
    api = "t2.small"
    db  = "t2.large"
  }
  
  ami           = "ami-12345"
  instance_type = each.value
  
  tags = {
    Name = each.key
  }
}

# Обращение через ключ:
output "web_ip" {
  value = aws_instance.servers["web"].private_ip
}
```

---

## Locals (локальные значения)

```hcl
# Базовое объявление
locals {
  # Простые значения
  environment = "production"
  region      = "us-east-1"
  
  # Вычисляемые значения
  common_tags = {
    Environment = local.environment
    ManagedBy   = "Terraform"
    Region      = local.region
  }
  
  # С использованием переменных
  instance_name = "${var.project_name}-${var.environment}-instance"
  
  # Условные значения
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
  
  # Списки и map
  availability_zones = ["${local.region}a", "${local.region}b", "${local.region}c"]
  
  subnet_configs = {
    for idx, az in local.availability_zones : az => {
      cidr = cidrsubnet(var.vpc_cidr, 8, idx)
      zone = az
    }
  }
}

# Множественные блоки locals (можно объявлять несколько)
locals {
  vpc_config = {
    cidr = "10.0.0.0/16"
  }
}

locals {
  # Ссылка на другой locals
  subnet_cidr = cidrsubnet(local.vpc_config.cidr, 8, 0)
}

# Использование locals
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = local.instance_type
  
  tags = merge(
    local.common_tags,
    {
      Name = local.instance_name
    }
  )
}

# Сложные вычисления в locals
locals {
  # Парсинг и обработка данных
  config_file = jsondecode(file("${path.module}/config.json"))
  
  # Группировка данных
  servers_by_type = {
    for server in var.servers : server.type => server...
  }
  
  # Фильтрация
  production_servers = [
    for server in var.servers : server
    if server.environment == "production"
  ]
  
  # Преобразование
  server_ips = {
    for server in aws_instance.servers : server.tags["Name"] => server.private_ip
  }
  
  # Flatten вложенных структур
  all_subnets = flatten([
    for vpc_key, vpc in var.vpcs : [
      for subnet in vpc.subnets : {
        vpc_name    = vpc_key
        subnet_cidr = subnet.cidr
        subnet_zone = subnet.zone
      }
    ]
  ])
}

# Best practices для locals
locals {
  # ✅ Используйте locals для:
  # - Вычислений, используемых в нескольких местах
  # - Комплексных выражений для читаемости
  # - Стандартизированных значений (тегов, имён)
  # - Промежуточных вычислений
  
  # ❌ Не используйте locals для:
  # - Простых значений, используемых один раз
  # - Копирования переменных без преобразования
}
```

---

## Outputs

```hcl
# Базовый output
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "Публичный IP адрес веб-сервера"
}

# Output с чувствительными данными
output "database_password" {
  value       = aws_db_instance.main.password
  description = "Пароль базы данных"
  sensitive   = true  # Скрывает значение в выводе
}

# Output с условием
output "load_balancer_dns" {
  value       = var.create_load_balancer ? aws_lb.main[0].dns_name : null
  description = "DNS имя load balancer (если создан)"
}

# Output с зависимостями
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "ID VPC"
  depends_on  = [aws_vpc.main]
}

# Output списка
output "instance_ids" {
  value       = aws_instance.web[*].id
  description = "ID всех веб-инстансов"
}

# Output map
output "server_ips" {
  value = {
    for key, instance in aws_instance.servers : key => instance.private_ip
  }
  description = "Private IP адреса всех серверов"
}

# Output сложной структуры
output "vpc_config" {
  value = {
    vpc_id     = aws_vpc.main.id
    vpc_cidr   = aws_vpc.main.cidr_block
    subnet_ids = aws_subnet.private[*].id
    nat_gateway_ips = {
      for az, nat in aws_nat_gateway.main : az => nat.public_ip
    }
  }
  description = "Полная конфигурация VPC"
}

# Output из модуля
module "vpc" {
  source = "./modules/vpc"
  # ...
}

output "vpc_id_from_module" {
  value       = module.vpc.vpc_id
  description = "VPC ID из модуля"
}

# Output с преобразованием
output "instance_names" {
  value = [
    for instance in aws_instance.web : instance.tags["Name"]
  ]
  description = "Имена всех инстансов"
}

# Precondition для output (проверка перед выводом)
output "validated_output" {
  value       = aws_instance.web.public_ip
  description = "IP адрес"
  
  precondition {
    condition     = aws_instance.web.public_ip != ""
    error_message = "Instance должен иметь публичный IP"
  }
}

# Использование outputs:
# - terraform output - показать все outputs
# - terraform output instance_ip - показать конкретный output
# - terraform output -json - вывод в JSON формате
# - terraform output -raw instance_ip - вывод без кавычек (для скриптов)
```

---

## Data Sources

```hcl
# Базовый data source
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Использование data source
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
}

# Data source с одним результатом
data "aws_vpc" "main" {
  id = var.vpc_id
}

data "aws_subnet" "selected" {
  filter {
    name   = "tag:Name"
    values = ["private-subnet-1"]
  }
}

# Data source для получения текущей информации
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
data "aws_availability_zones" "available" {
  state = "available"
}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "current_region" {
  value = data.aws_region.current.name
}

output "available_zones" {
  value = data.aws_availability_zones.available.names
}

# Data source с зависимостями
data "aws_instance" "web" {
  instance_id = aws_instance.web.id
  
  depends_on = [aws_instance.web]
}

# Data source с count
data "aws_subnet" "private" {
  count = length(var.private_subnet_ids)
  id    = var.private_subnet_ids[count.index]
}

# Data source с for_each
data "aws_subnet" "selected" {
  for_each = toset(var.subnet_ids)
  id       = each.value
}

# Data source для чтения файлов
data "local_file" "config" {
  filename = "${path.module}/config.yaml"
}

locals {
  config = yamldecode(data.local_file.config.content)
}

# Data source для внешних данных
data "external" "git_info" {
  program = ["bash", "${path.module}/scripts/git-info.sh"]
  
  query = {
    path = path.module
  }
}

output "git_commit" {
  value = data.external.git_info.result["commit"]
}

# Data source для HTTP запросов
data "http" "myip" {
  url = "https://api.ipify.org?format=json"
  
  request_headers = {
    Accept = "application/json"
  }
}

locals {
  my_public_ip = jsondecode(data.http.myip.response_body)["ip"]
}

# Terraform data source (информация о состоянии)
data "terraform_remote_state" "network" {
  backend = "s3"
  
  config = {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

# Использование outputs из другого state
resource "aws_instance" "app" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
  subnet_id     = data.terraform_remote_state.network.outputs.private_subnet_id
}

# Data source с lifecycle
data "aws_ami" "ubuntu" {
  most_recent = true
  
  lifecycle {
    postcondition {
      condition     = self.architecture == "x86_64"
      error_message = "AMI должен быть x86_64 архитектуры"
    }
  }
}
```

---

## Модули

### Создание модуля

```hcl
# Структура директории модуля:
# modules/
# └── vpc/
#     ├── main.tf       # Основные ресурсы
#     ├── variables.tf  # Входные переменные
#     ├── outputs.tf    # Выходные значения
#     └── README.md     # Документация

# modules/vpc/variables.tf
variable "vpc_name" {
  description = "Имя VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR блок VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "azs" {
  description = "Список availability zones"
  type        = list(string)
}

variable "private_subnets" {
  description = "CIDR блоки для private подсетей"
  type        = list(string)
}

variable "public_subnets" {
  description = "CIDR блоки для public подсетей"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Создать NAT Gateway"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Теги для всех ресурсов"
  type        = map(string)
  default     = {}
}

# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(
    var.tags,
    {
      Name = var.vpc_name
    }
  )
}

resource "aws_subnet" "private" {
  count = length(var.private_subnets)
  
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.azs[count.index]
  
  tags = merge(
    var.tags,
    {
      Name = "${var.vpc_name}-private-${var.azs[count.index]}"
      Type = "private"
    }
  )
}

resource "aws_subnet" "public" {
  count = length(var.public_subnets)
  
  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnets[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(
    var.tags,
    {
      Name = "${var.vpc_name}-public-${var.azs[count.index]}"
      Type = "public"
    }
  )
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  
  tags = merge(
    var.tags,
    {
      Name = var.vpc_name
    }
  )
}

resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? length(var.public_subnets) : 0
  
  domain = "vpc"
  
  tags = merge(
    var.tags,
    {
      Name = "${var.vpc_name}-nat-${var.azs[count.index]}"
    }
  )
  
  depends_on = [aws_internet_gateway.this]
}

resource "aws_nat_gateway" "this" {
  count = var.enable_nat_gateway ? length(var.public_subnets) : 0
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.vpc_name}-${var.azs[count.index]}"
    }
  )
  
  depends_on = [aws_internet_gateway.this]
}

# modules/vpc/outputs.tf
output "vpc_id" {
  description = "ID VPC"
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "CIDR блок VPC"
  value       = aws_vpc.this.cidr_block
}

output "private_subnet_ids" {
  description = "ID private подсетей"
  value       = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  description = "ID public подсетей"
  value       = aws_subnet.public[*].id
}

output "nat_gateway_ids" {
  description = "ID NAT Gateways"
  value       = aws_nat_gateway.this[*].id
}

output "internet_gateway_id" {
  description = "ID Internet Gateway"
  value       = aws_internet_gateway.this.id
}
```

### Использование модуля

```hcl
# Локальный модуль
module "vpc" {
  source = "./modules/vpc"
  
  vpc_name        = "production-vpc"
  vpc_cidr        = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  
  tags = {
    Environment = "production"
    ManagedBy   = "Terraform"
  }
}

# Модуль из Terraform Registry
module "vpc_registry" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}

# Модуль из Git
module "vpc_git" {
  source = "git::https://github.com/user/terraform-modules.git//modules/vpc?ref=v1.0.0"
  
  vpc_name = "staging-vpc"
  vpc_cidr = "10.1.0.0/16"
}

# Модуль с count
module "vpc_multiple" {
  source = "./modules/vpc"
  count  = 3
  
  vpc_name = "vpc-${count.index}"
  vpc_cidr = cidrsubnet("10.0.0.0/8", 8, count.index)
  azs      = data.aws_availability_zones.available.names
  
  private_subnets = [
    cidrsubnet(cidrsubnet("10.0.0.0/8", 8, count.index), 8, 1),
    cidrsubnet(cidrsubnet("10.0.0.0/8", 8, count.index), 8, 2)
  ]
  
  public_subnets = [
    cidrsubnet(cidrsubnet("10.0.0.0/8", 8, count.index), 8, 101),
    cidrsubnet(cidrsubnet("10.0.0.0/8", 8, count.index), 8, 102)
  ]
}

# Модуль с for_each
module "vpc_environments" {
  source = "./modules/vpc"
  
  for_each = {
    prod    = "10.0.0.0/16"
    staging = "10.1.0.0/16"
    dev     = "10.2.0.0/16"
  }
  
  vpc_name = "${each.key}-vpc"
  vpc_cidr = each.value
  azs      = data.aws_availability_zones.available.names
  
  private_subnets = [
    cidrsubnet(each.value, 8, 1),
    cidrsubnet(each.value, 8, 2)
  ]
  
  public_subnets = [
    cidrsubnet(each.value, 8, 101),
    cidrsubnet(each.value, 8, 102)
  ]
  
  tags = {
    Environment = each.key
  }
}

# Использование outputs модуля
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  subnet_id     = module.vpc.private_subnet_ids[0]
  
  tags = {
    Name = "app-server"
  }
}

output "vpc_id" {
  value = module.vpc.vpc_id
}

# Outputs от модулей с count
output "all_vpc_ids" {
  value = module.vpc_multiple[*].vpc_id
}

# Outputs от модулей с for_each
output "environment_vpc_ids" {
  value = {
    for env, vpc_module in module.vpc_environments : env => vpc_module.vpc_id
  }
}

# Передача outputs между модулями
module "network" {
  source = "./modules/network"
  # ...
}

module "compute" {
  source = "./modules/compute"
  
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids
}
```

### Meta-аргументы модулей

```hcl
# providers - явное указание провайдеров
provider "aws" {
  region = "us-east-1"
  alias  = "east"
}

provider "aws" {
  region = "us-west-2"
  alias  = "west"
}

module "vpc_east" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.east
  }
  
  vpc_name = "east-vpc"
}

module "vpc_west" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.west
  }
  
  vpc_name = "west-vpc"
}

# depends_on для модулей
module "network" {
  source = "./modules/network"
  # ...
}

module "security" {
  source = "./modules/security"
  
  depends_on = [module.network]
}
```

---

## Terraform-специфичные конструкции

### Terraform блок

```hcl
terraform {
  # Требуемая версия Terraform
  required_version = ">= 1.0"
  
  # Требуемые провайдеры
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
    
    null = {
      source  = "hashicorp/null"
      version = "~> 3.0"
    }
  }
  
  # Backend для хранения state
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
  
  # Эксперименталь��ые фичи
  experiments = []
}

# Версионные ограничения:
# = 1.0.0      - точная версия
# != 1.0.0     - любая кроме указанной
# > 1.0.0      - больше
# >= 1.0.0     - больше или равно
# < 2.0.0      - меньше
# <= 2.0.0     - меньше или равно
# ~> 1.0.0     - любая 1.0.x (pessimistic constraint)
# ~> 1.0       - любая 1.x
# >= 1.0, < 2.0 - диапазон
```

### Provider блок

```hcl
# Базовый provider
provider "aws" {
  region     = var.aws_region
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
}

# Provider с alias (множественные провайдеры)
provider "aws" {
  alias  = "east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Использование конкретного провайдера в ресурсе
resource "aws_instance" "east_server" {
  provider = aws.east
  
  ami           = "ami-12345"
  instance_type = "t2.micro"
}

resource "aws_instance" "west_server" {
  provider = aws.west
  
  ami           = "ami-67890"
  instance_type = "t2.micro"
}

# Provider с assume_role
provider "aws" {
  region = "us-east-1"
  
  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformRole"
    session_name = "terraform-session"
  }
}

# Provider с default_tags
provider "aws" {
  region = "us-east-1"
  
  default_tags {
    tags = {
      Environment = "production"
      ManagedBy   = "Terraform"
      Team        = "DevOps"
    }
  }
}

# Provider с ignore_tags
provider "aws" {
  region = "us-east-1"
  
  ignore_tags {
    keys = ["AutoScaling"]
    key_prefixes = ["kubernetes.io/"]
  }
}
```

### Moved блок (рефакторинг без пересоздания)

```hcl
# Переименование ресурса
moved {
  from = aws_instance.web
  to   = aws_instance.webserver
}

# Перемещение в модуль
moved {
  from = aws_vpc.main
  to   = module.network.aws_vpc.main
}

# Изменение индекса count
moved {
  from = aws_instance.web[0]
  to   = aws_instance.web[2]
}

# Изменение ключа for_each
moved {
  from = aws_instance.servers["old_key"]
  to   = aws_instance.servers["new_key"]
}
```

### Import блок (импорт существующих ресурсов)

```hcl
# Terraform 1.5+ синтаксис импорта
import {
  to = aws_instance.example
  id = "i-1234567890abcdef"
}

resource "aws_instance" "example" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
  # остальная конфигурация
}

# Импорт с for_each
import {
  for_each = toset(["vpc-123", "vpc-456"])
  to       = aws_vpc.imported[each.key]
  id       = each.key
}
```

### Check блок (валидация инфраструктуры)

```hcl
# Проверка состояния инфраструктуры
check "health_check" {
  data "http" "terraform_io" {
    url = "https://www.terraform.io"
  }
  
  assert {
    condition     = data.http.terraform_io.status_code == 200
    error_message = "Terraform website недоступен"
  }
}

check "database_backup" {
  data "aws_db_instance" "main" {
    db_instance_identifier = "main-db"
  }
  
  assert {
    condition     = data.aws_db_instance.main.backup_retention_period >= 7
    error_message = "Backup retention должен быть минимум 7 дней"
  }
}
```

### Lifecycle hooks

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
  
  lifecycle {
    # Создать новый ресурс перед удалением старого
    create_before_destroy = true
    
    # Предотвратить удаление ресурса
    prevent_destroy = true
    
    # Игнорировать изменения указанных атрибутов
    ignore_changes = [
      tags["LastUpdated"],
      user_data,
    ]
    
    # Игнорировать все изменения
    # ignore_changes = all
    
    # Заменить при изменении другого ресурса
    replace_triggered_by = [
      aws_security_group.web.id
    ]
    
    # Precondition - проверка перед применением
    precondition {
      condition     = var.ami_id != ""
      error_message = "AMI ID не может быть пустым"
    }
    
    # Postcondition - проверка после применения
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance должен получить публичный IP"
    }
  }
}

# Precondition и Postcondition в data source
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  lifecycle {
    postcondition {
      condition     = self.architecture == "x86_64"
      error_message = "AMI должен быть x86_64"
    }
  }
}

# Precondition в output
output "instance_url" {
  value = "https://${aws_instance.web.public_ip}"
  
  precondition {
    condition     = aws_instance.web.public_ip != ""
    error_message = "Instance должен иметь публичный IP для формирования URL"
  }
}
```

---

## Best Practices

### Организация кода

```hcl
# Структура проекта:
# .
# ├── main.tf              # Основные ресурсы
# ├── variables.tf         # Входные переменные
# ├── outputs.tf           # Выходные значения
# ├── locals.tf            # Локальные значения (опционально)
# ├── data.tf              # Data sources (опционально)
# ├── providers.tf         # Конфигурация провайдеров
# ├── versions.tf          # Версии Terraform и провайдеров
# ├── terraform.tfvars     # Значения переменных (НЕ коммитить с секретами!)
# ├── modules/             # Локальные модули
# │   ├── vpc/
# │   ├── compute/
# │   └── database/
# └── environments/        # Конфигурации окружений
#     ├── prod/
#     ├── staging/
#     └── dev/

# versions.tf - всегда явно указывайте версии
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# providers.tf - конфигурация провайдеров отдельно
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = local.common_tags
  }
}

# variables.tf - все переменные в одном месте
variable "environment" {
  description = "Название окружения (dev, staging, prod)"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment должен быть: dev, staging, или prod"
  }
}

variable "instance_count" {
  description = "Количество инстансов"
  type        = number
  default     = 1
  
  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "Instance count должен быть между 1 и 10"
  }
}

# locals.tf - переиспользуемые значения
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = var.project_name
    Owner       = var.owner_team
  }
  
  name_prefix = "${var.project_name}-${var.environment}"
  
  # Вычисляемые значения
  is_production = var.environment == "prod"
  instance_type = local.is_production ? "t2.large" : "t2.micro"
}
```

### Именование

```hcl
# ✅ Хорошие практики именования:

# 1. Используйте snake_case для переменных, locals, outputs
variable "instance_type" {}
locals {
  subnet_cidr = "10.0.1.0/24"
}
output "vpc_id" {}

# 2. Используйте осмысленные имена
# ❌ Плохо
variable "x" {}
resource "aws_instance" "a" {}

# ✅ Хорошо
variable "web_server_count" {}
resource "aws_instance" "web_server" {}

# 3. Для ресурсов - используйте тип без префикса провайдера
# ❌ Плохо
resource "aws_instance" "aws_web_server" {}

# ✅ Хорошо
resource "aws_instance" "web" {}
resource "aws_security_group" "web" {}

# 4. Используйте "this" для single-instance ресурсов в модулях
resource "aws_vpc" "this" {}
resource "aws_subnet" "this" {}

# 5. Используйте "main" для главного ресурса конфигурации
resource "aws_vpc" "main" {}

# 6. Префиксы для типов ресурсов
resource "aws_instance" "web" {}      # веб-сервер
resource "aws_instance" "worker" {}   # worker
resource "aws_db_instance" "main" {}  # база данных
```

### Управление состоянием (State)

```hcl
# Используйте remote backend
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    
    # Включите versioning на S3 bucket
    # Включите encryption на S3 bucket
  }
}

# Разделяйте state по окружениям
# environments/prod/backend.tf
terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}

# environments/staging/backend.tf
terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "staging/terraform.tfstate"
    region = "us-east-1"
  }
}

# Или используйте workspaces
# terraform workspace new prod
# terraform workspace new staging
# terraform workspace select prod
```

### Безопасность

```hcl
# 1. Никогда не храните секреты в коде
# ❌ Плохо
resource "aws_db_instance" "main" {
  password = "MyPassword123!"  # НИКОГДА ТАК НЕ ДЕЛАЙТЕ
}

# ✅ Хорошо - используйте переменные
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

resource "aws_db_instance" "main" {
  password = var.db_password
}

# ✅ Или используйте AWS Secrets Manager / Parameter Store
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}

# 2. Помечайте чувствительные данные как sensitive
variable "api_key" {
  type      = string
  sensitive = true
}

output "database_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}

# 3. Не коммитьте .tfvars с секретами
# Добавьте в .gitignore:
# *.tfvars
# !terraform.tfvars.example
# .terraform/
# terraform.tfstate*

# 4. Используйте .tfvars.example как шаблон
# terraform.tfvars.example
# db_password = "REPLACE_WITH_ACTUAL_PASSWORD"
# api_key     = "REPLACE_WITH_ACTUAL_KEY"

# 5. Ограничивайте доступ к state файлам
# - S3 bucket policy для state
# - IAM роли с минимальными правами
# - Включите MFA для критичных операций
```

### Модульность и переиспользование

```hcl
# 1. Создавайте переиспользуемые модули
# ❌ Плохо - всё в одном файле
resource "aws_vpc" "prod" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc" "staging" {
  cidr_block = "10.1.0.0/16"
}

resource "aws_vpc" "dev" {
  cidr_block = "10.2.0.0/16"
}

# ✅ Хорошо - используйте модули
module "vpc" {
  source = "./modules/vpc"
  
  for_each = {
    prod    = "10.0.0.0/16"
    staging = "10.1.0.0/16"
    dev     = "10.2.0.0/16"
  }
  
  name       = "${each.key}-vpc"
  cidr_block = each.value
  
  tags = {
    Environment = each.key
  }
}

# 2. Делайте модули параметризуемыми
# ❌ Плохо - хардкод значений в модуле
resource "aws_instance" "web" {
  ami           = "ami-12345"  # хардкод
  instance_type = "t2.micro"   # хардкод
}

# ✅ Хорошо - параметры через переменные
variable "ami_id" {
  type = string
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
}

# 3. Группируйте связанные ресурсы в модули
# modules/
# ├── networking/  # VPC, subnets, route tables
# ├── compute/     # EC2, ASG, Launch Templates
# ├── database/    # RDS, DynamoDB
# └── security/    # Security Groups, NACLs, IAM
```

### Документация и комментарии

```hcl
# 1. Документируйте переменные
variable "instance_count" {
  description = "Количество EC2 инстансов для создания. Должно быть между 1 и 10."
  type        = number
  default     = 3
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Instance count должен быть между 1 и 10."
  }
}

# 2. Добавляйте комментарии к сложной логике
locals {
  # Вычисляем CIDR блоки для подсетей в каждой AZ
  # Формула: начальный CIDR + (индекс AZ * количество подсетей на AZ)
  subnet_cidrs = flatten([
    for az_index, az in var.availability_zones : [
      for subnet_index in range(var.subnets_per_az) :
      cidrsubnet(
        var.vpc_cidr,
        8,
        (az_index * var.subnets_per_az) + subnet_index
      )
    ]
  ])
}

# 3. Документируйте outputs
output "instance_ids" {
  description = "Список ID всех созданных EC2 инстансов. Используйте для настройки мониторинга и логирования."
  value       = aws_instance.web[*].id
}

# 4. Создавайте README.md для модулей
# modules/vpc/README.md:
# # VPC Module
# 
# Создаёт VPC с public и private подсетями в нескольких AZ.
# 
# ## Usage
# ```hcl
# module "vpc" {
#   source = "./modules/vpc"
#   
#   vpc_name = "production"
#   vpc_cidr = "10.0.0.0/16"
#   azs      = ["us-east-1a", "us-east-1b"]
# }
# ```
# 
# ## Inputs
# | Name | Description | Type | Default | Required |
# |------|-------------|------|---------|----------|
# | vpc_name | Name of VPC | string | n/a | yes |
# | vpc_cidr | CIDR block | string | n/a | yes |
```

### Тестирование и валидация

```hcl
# 1. Используйте validation блоки
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  type = string
  
  validation {
    condition     = can(regex("^t[23]\\.(nano|micro|small|medium|large|xlarge|2xlarge)$", var.instance_type))
    error_message = "Instance type must be a valid t2 or t3 type."
  }
}

# 2. Используйте precondition и postcondition
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  lifecycle {
    precondition {
      condition     = data.aws_ami.selected.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }
    
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP address."
    }
  }
}

# 3. Используйте check блоки для валидации
check "health_check" {
  data "http" "app_health" {
    url = "https://${aws_instance.web.public_ip}/health"
  }
  
  assert {
    condition     = data.http.app_health.status_code == 200
    error_message = "Application health check failed."
  }
}

# 4. Тестируйте с разными значениями переменных
# terraform plan -var="environment=dev"
# terraform plan -var="environment=staging"
# terraform plan -var="environment=prod"
```

### Производительность и оптимизация

```hcl
# 1. Используйте data sources эффективно
# ❌ Плохо - повторяющиеся data sources
resource "aws_instance" "web1" {
  ami = data.aws_ami.ubuntu.id  # Каждый раз новый запрос
}

resource "aws_instance" "web2" {
  ami = data.aws_ami.ubuntu.id
}

# ✅ Хорошо - один data source, многократное использование
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

locals {
  ubuntu_ami_id = data.aws_ami.ubuntu.id
}

resource "aws_instance" "web1" {
  ami = local.ubuntu_ami_id
}

resource "aws_instance" "web2" {
  ami = local.ubuntu_ami_id
}

# 2. Используйте depends_on только когда необходимо
# Terraform автоматически определяет зависимости через ссылки
# ❌ Плохо - ненужный depends_on
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id
  
  depends_on = [aws_subnet.main]  # Не нужен!
}

# ✅ Хорошо - неявная зависимость
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id
}

# depends_on нужен только для неявных зависимостей
resource "aws_iam_role_policy" "example" {
  role = aws_iam_role.example.id
  
  # Ждём пока роль распространится в AWS
  depends_on = [aws_iam_role.example]
}

# 3. Группируйте изменения
# Вместо множества мелких apply, делайте их батчами
# terraform plan -out=tfplan
# terraform apply tfplan

# 4. Используйте -target только для дебага
# ❌ Не используйте постоянно
# terraform apply -target=aws_instance.web

# ✅ Используйте для всей конфигурации
# terraform apply
```

### Работа с множественными окружениями

```hcl
# Способ 1: Workspaces
# terraform workspace new prod
# terraform workspace new staging
# terraform workspace select prod

locals {
  environment = terraform.workspace
  
  instance_count = {
    prod    = 5
    staging = 2
    dev     = 1
  }
  
  count = local.instance_count[local.environment]
}

# Способ 2: Separate directories (рекомендуется)
# environments/
# ├── prod/
# │   ├── main.tf
# │   ├── variables.tf
# │   └── terraform.tfvars
# ├── staging/
# │   ├── main.tf
# │   ├── variables.tf
# │   └── terraform.tfvars
# └── dev/
#     ├── main.tf
#     ├── variables.tf
#     └── terraform.tfvars

# Способ 3: Использование tfvars файлов
# terraform apply -var-file="prod.tfvars"
# terraform apply -var-file="staging.tfvars"

# prod.tfvars
environment    = "prod"
instance_count = 5
instance_type  = "t2.large"

# staging.tfvars
environment    = "staging"
instance_count = 2
instance_type  = "t2.medium"
```

### Обработка ошибок и edge cases

```hcl
# 1. Используйте try() для безопасного доступа
locals {
  # ❌ Может упасть если ключа нет
  value = var.config["optional_key"]
  
  # ✅ Безопасный доступ с fallback
  safe_value = try(var.config["optional_key"], "default")
  
  # Вложенный доступ
  nested = try(var.config.nested.value, "default")
}

# 2. Используйте can() для проверки
locals {
  # Проверяем можно ли преобразовать в число
  is_number = can(tonumber(var.value))
  
  # Проверяем корректность regex
  is_valid_email = can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", var.email))
  
  # Условная логика на основе can()
  port = can(tonumber(var.port)) ? tonumber(var.port) : 80
}

# 3. Используйте coalesce для первого непустого значения
locals {
  # Первое непустое значение
  value = coalesce(var.optional_value, var.fallback_value, "default")
  
  # Для списков
  list_value = coalescelist(var.optional_list, var.fallback_list, ["default"])
}

# 4. Обрабатывайте null значения
variable "optional_config" {
  type    = map(string)
  default = null
  nullable = true
}

locals {
  config = var.optional_config != null ? var.optional_config : {}
}

resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
  
  # Условное создание блока
  dynamic "credit_specification" {
    for_each = var.enable_unlimited_credits != null && var.enable_unlimited_credits ? [1] : []
    
    content {
      cpu_credits = "unlimited"
    }
  }
}

# 5. Защита от пустых коллекций
locals {
  # Проверка что список не пустой
  subnet_ids = length(var.subnet_ids) > 0 ? var.subnet_ids : data.aws_subnets.default.ids
}

resource "aws_lb" "main" {
  # Убедитесь что есть хотя бы 2 подсети
  subnets = length(local.subnet_ids) >= 2 ? local.subnet_ids : concat(
    local.subnet_ids,
    [data.aws_subnet.additional.id]
  )
  
  lifecycle {
    precondition {
      condition     = length(local.subnet_ids) >= 2
      error_message = "Load balancer requires at least 2 subnets."
    }
  }
}
```

### Советы по отладке

```hcl
# 1. Используйте terraform console для тестирования выражений
# terraform console
# > var.instance_count
# > local.subnet_cidrs
# > aws_instance.web[*].id

# 2. Добавляйте временные outputs для отладки
output "debug_subnet_cidrs" {
  value = local.subnet_cidrs
}

output "debug_for_each_keys" {
  value = keys(aws_instance.servers)
}

# 3. Используйте terraform show для просмотра state
# terraform show
# terraform show -json | jq '.values.root_module.resources'

# 4. Логируйте с разным уровнем детализации
# export TF_LOG=TRACE
# export TF_LOG=DEBUG
# export TF_LOG=INFO
# export TF_LOG_PATH="terraform.log"
# terraform apply

# 5. Используйте terraform graph для визуализации зависимостей
# terraform graph | dot -Tsvg > graph.svg

# 6. Проверяйте план перед apply
# terraform plan -out=tfplan
# terraform show tfplan
# terraform apply tfplan
```

---

## Полезные паттерны и примеры

### Условное создание ресурсов

```hcl
# Паттерн 1: через count
variable "create_vpc" {
  type    = bool
  default = true
}

resource "aws_vpc" "main" {
  count = var.create_vpc ? 1 : 0
  
  cidr_block = "10.0.0.0/16"
}

# Обращение к условному ресурсу
locals {
  vpc_id = var.create_vpc ? aws_vpc.main[0].id : data.aws_vpc.existing.id
}

# Паттерн 2: через for_each с пустой коллекцией
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  for_each = var.enable_monitoring ? toset(["alarm"]) : toset([])
  
  alarm_name          = "high-cpu"
  comparison_operator = "GreaterThanThreshold"
  threshold           = 80
}
```

### Создание ресурсов из CSV/JSON

```hcl
# Чтение CSV файла
locals {
  # users.csv:
  # name,email,role
  # alice,alice@example.com,admin
  # bob,bob@example.com,developer
  
  users_csv = file("${path.module}/users.csv")
  users     = csvdecode(local.users_csv)
}

resource "aws_iam_user" "users" {
  for_each = { for user in local.users : user.name => user }
  
  name = each.value.name
  
  tags = {
    Email = each.value.email
    Role  = each.value.role
  }
}

# Чтение JSON файла
locals {
  # config.json:
  # {
  #   "servers": [
  #     {"name": "web1", "type": "t2.micro"},
  #     {"name": "web2", "type": "t2.small"}
  #   ]
  # }
  
  config  = jsondecode(file("${path.module}/config.json"))
  servers = local.config.servers
}

resource "aws_instance" "servers" {
  for_each = { for server in local.servers : server.name => server }
  
  ami           = data.aws_ami.ubuntu.id
  instance_type = each.value.type
  
  tags = {
    Name = each.key
  }
}
```

### Динамическое создание правил Security Group

```hcl
variable "ingress_rules" {
  type = list(object({
    description = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    {
      description = "HTTP"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description = "HTTPS"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]
}

resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    
    content {
      description = ingress.value.description
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

### Создание ресурсов в нескольких регионах

```hcl
# providers.tf
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "us_west"
  region = "us-west-2"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

# Модуль для каждого региона
module "vpc_us_east" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.us_east
  }
  
  vpc_name = "us-east-vpc"
  vpc_cidr = "10.0.0.0/16"
}

module "vpc_us_west" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.us_west
  }
  
  vpc_name = "us-west-vpc"
  vpc_cidr = "10.1.0.0/16"
}

# Или через for_each
locals {
  regions = {
    us_east = "us-east-1"
    us_west = "us-west-2"
    eu_west = "eu-west-1"
  }
}

# Требует dynamic provider configuration (experimental)
```

### Генерация конфигурационных файлов

```hcl
# Генерация ansible inventory
resource "local_file" "ansible_inventory" {
  content = templatefile("${path.module}/templates/inventory.tpl", {
    web_servers = aws_instance.web[*].public_ip
    db_servers  = aws_instance.db[*].private_ip
  })
  
  filename = "${path.module}/inventory.ini"
}

# templates/inventory.tpl
# [web]
# %{ for ip in web_servers ~}
# ${ip}
# %{ endfor ~}
# 
# [database]
# %{ for ip in db_servers ~}
# ${ip}
# %{ endfor ~}

# Генерация конфигурации приложения
resource "local_file" "app_config" {
  content = yamlencode({
    database = {
      host     = aws_db_instance.main.address
      port     = aws_db_instance.main.port
      name     = aws_db_instance.main.db_name
      username = aws_db_instance.main.username
    }
    redis = {
      host = aws_elasticache_cluster.redis.cache_nodes[0].address
      port = aws_elasticache_cluster.redis.cache_nodes[0].port
    }
    s3 = {
      bucket = aws_s3_bucket.assets.id
      region = aws_s3_bucket.assets.region
    }
  })
  
  filename = "${path.module}/config/app.yaml"
}
```

### Blue-Green Deployment паттерн

```hcl
variable "active_environment" {
  type    = string
  default = "blue"
  
  validation {
    condition     = contains(["blue", "green"], var.active_environment)
    error_message = "Active environment must be blue or green."
  }
}

# Blue environment
module "environment_blue" {
  source = "./modules/app-environment"
  
  name              = "blue"
  instance_count    = var.active_environment == "blue" ? 3 : 0
  ami_id            = var.ami_id
  target_group_arn  = aws_lb_target_group.main.arn
}

# Green environment
module "environment_green" {
  source = "./modules/app-environment"
  
  name              = "green"
  instance_count    = var.active_environment == "green" ? 3 : 0
  ami_id            = var.ami_id
  target_group_arn  = aws_lb_target_group.main.arn
}

# Для переключения:
# 1. terraform apply -var="active_environment=green"  # Создаёт green
# 2. Проверка green окружения
# 3. terraform apply -var="active_environment=green"  # Удаляет blue
```

---

## Быстрая справка по командам Terraform

```bash
# Инициализация
terraform init                    # Инициализация директории
terraform init -upgrade           # Обновление провайдеров

# Планирование
terraform plan                    # Показать план изменений
terraform plan -out=tfplan        # Сохранить план в файл
terraform plan -var="key=value"   # С переменной
terraform plan -var-file="prod.tfvars"  # С файлом переменных

# Применение
terraform apply                   # Применить изменения
terraform apply tfplan            # Применить сохранённый план
terraform apply -auto-approve     # Без подтверждения
terraform apply -target=resource.name  # Только указанный ресурс

# Уничтожение
terraform destroy                 # Удалить всю инфраструктуру
terraform destroy -auto-approve   # Без подтверждения
terraform destroy -target=resource.name  # Только указанный ресурс

# Просмотр состояния
terraform show                    # Показать текущее состояние
terraform show tfplan             # Показать сохранённый план
terraform state list              # Список ресурсов в state
terraform state show resource.name  # Детали ресурса

# Работа с state
terraform state mv old.name new.name     # Переименовать ресурс
terraform state rm resource.name         # Удалить из state
terraform state pull                     # Скачать remote state
terraform state push                     # Загрузить state

# Импорт
terraform import resource.name id        # Импортировать ресурс

# Workspace
terraform workspace list          # Список workspaces
terraform workspace new name      # Создать workspace
terraform workspace select name   # Переключиться на workspace
terraform workspace delete name   # Удалить workspace

# Форматирование и валидация
terraform fmt                     # Форматировать файлы
terraform fmt -recursive          # Рекурсивно
terraform validate                # Валидация конфигурации

# Вывод значений
terraform output                  # Все outputs
terraform output name             # Конкретный output
terraform output -json            # В JSON формате
terraform output -raw name        # Без кавычек

# Консоль для тестирования
terraform console                 # Интерактивная консоль

# Обновление
terraform get                     # Скачать модули
terraform get -update             # Обновить модули

# Граф зависимостей
terraform graph                   # DOT формат
terraform graph | dot -Tsvg > graph.svg  # Визуализация

# Логирование
export TF_LOG=TRACE              # Уровень логирования
export TF_LOG_PATH=terraform.log # Путь к лог-файлу

# Версия
terraform version                # Версия Terraform
terraform version -json          # В JSON формате
```

---

## Шпаргалка по функциям (краткая)

```hcl
# Числовые: abs, ceil, floor, max, min, pow, log, signum
# Строковые: format, join, split, lower, upper, trim, replace, regex
# Коллекции: concat, contains, distinct, element, flatten, length, merge, reverse, sort
# Кодирование: base64encode, base64decode, jsonencode, jsondecode, yamlencode, yamldecode
# Файлы: file, fileexists, fileset, templatefile
# Дата: timestamp, formatdate, timeadd
# Хеш: md5, sha256, sha512
# IP: cidrhost, cidrnetmask, cidrsubnet
# Типы: tobool, tolist, tomap, tonumber, toset, tostring, try, can
```

Это максимально подробная шпаргалка по HCL! Она готова для использования в Git и Notion. 🚀