# Git Шпаргалка для Production

## 🚀 Быстрый старт (основные команды)

### Базовая настройка
```bash
# Первичная настройка
git config --global user.name "Ваше Имя"
git config --global user.email "your.email@example.com"

# Проверка настроек
git config --list
git config user.name
git config user.email

# Настройка редактора по умолчанию
git config --global core.editor "vim"

# Полезные алиасы
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.last 'log -1 HEAD'
git config --global alias.unstage 'reset HEAD --'
```

### Создание и клонирование репозитория
```bash
# Инициализация нового репозитория
git init

# Клонирование существующего репозитория
git clone https://github.com/username/repo.git
git clone https://github.com/username/repo.git new-folder-name

# Клонирование конкретной ветки
git clone -b branch-name https://github.com/username/repo.git

# Клонирование с ограничением глубины истории (быстрее)
git clone --depth 1 https://github.com/username/repo.git
```

## 📝 Основной рабочий процесс

### Проверка статуса
```bash
# Полный статус
git status

# Краткий статус
git status -s

# Показать изменения в конкретной ветке
git status -sb
```

### Добавление файлов в staging
```bash
# Добавить конкретный файл
git add filename.txt

# Добавить все файлы
git add .

# Добавить все файлы определенного типа
git add *.js

# Добавить все файлы в директории
git add src/

# Интерактивное добавление (выбор конкретных изменений)
git add -p

# Добавить все измененные файлы (без новых)
git add -u

# Добавить все (включая удаленные)
git add -A
```

### Просмотр изменений
```bash
# Изменения в рабочей директории (не staged)
git diff

# Изменения в staging area
git diff --staged
git diff --cached

# Изменения в конкретном файле
git diff filename.txt

# Изменения между ветками
git diff branch1..branch2

# Изменения между коммитами
git diff commit1 commit2

# Статистика изменений
git diff --stat
```

### Коммиты
```bash
# Базовый коммит
git commit -m "Описание изменений"

# Коммит с подробным описанием
git commit -m "Краткое описание" -m "Подробное описание того, что и зачем было сделано"

# Добавить все изменения и закоммитить
git commit -am "Описание"

# Изменить последний коммит (message или файлы)
git commit --amend

# Изменить только сообщение последнего коммита
git commit --amend -m "Новое сообщение"

# Добавить файлы в последний коммит без изменения message
git add forgotten-file.txt
git commit --amend --no-edit

# Пустой коммит (для триггера CI/CD)
git commit --allow-empty -m "Trigger CI"
```

## 🌿 Работа с ветками

### Создание и переключение
```bash
# Список веток
git branch                    # локальные
git branch -r                 # удаленные
git branch -a                 # все ветки
git branch -v                 # с последним коммитом

# Создать новую ветку
git branch feature-name

# Создать и переключиться на новую ветку
git checkout -b feature-name
git switch -c feature-name    # новый синтаксис

# Переключиться на существующую ветку
git checkout feature-name
git switch feature-name       # новый синтаксис

# Создать ветку от конкретного коммита
git checkout -b feature-name commit-hash

# Создать ветку от другой ветки
git checkout -b new-feature main
```

### Удаление веток
```bash
# Удалить локальную ветку (только если слита)
git branch -d feature-name

# Принудительно удалить локальную ветку
git branch -D feature-name

# Удалить удаленную ветку
git push origin --delete feature-name
git push origin :feature-name   # старый синтаксис

# Очистить ссылки на удаленные ветки
git fetch --prune
git remote prune origin
```

### Переименование веток
```bash
# Переименовать текущую ветку
git branch -m new-name

# Переименовать другую ветку
git branch -m old-name new-name

# Обновить удаленную ветку после переименования
git push origin :old-name new-name
git push origin -u new-name
```

## 🔄 Синхронизация с удаленным репозиторием

### Настройка remote
```bash
# Показать удаленные репозитории
git remote -v

# Добавить удаленный репозиторий
git remote add origin https://github.com/username/repo.git

# Изменить URL удаленного репозитория
git remote set-url origin https://github.com/username/new-repo.git

# Удалить удаленный репозиторий
git remote remove origin

# Переименовать remote
git remote rename old-name new-name
```

### Fetch - получение изменений без слияния
```bash
# Получить все изменения со всех веток
git fetch

# Получить изменения с конкретного remote
git fetch origin

# Получить конкретную ветку
git fetch origin branch-name

# Получить и удалить устаревшие ссылки
git fetch --prune
```

### Pull - получение и слияние изменений
```bash
# Получить и слить изменения текущей ветки
git pull

# Получить с конкретного remote
git pull origin main

# Pull с rebase вместо merge
git pull --rebase

# Pull только если fast-forward возможен
git pull --ff-only

# Установить поведение по умолчанию для pull
git config pull.rebase false  # merge (по умолчанию)
git config pull.rebase true   # rebase
git config pull.ff only       # только fast-forward
```

### Push - отправка изменений
```bash
# Отправить текущую ветку на origin
git push

# Отправить конкретную ветку
git push origin branch-name

# Первая отправка новой ветки (установить upstream)
git push -u origin branch-name
git push --set-upstream origin branch-name

# Отправить все ветки
git push --all

# Отправить теги
git push --tags

# Принудительная отправка (ОПАСНО!)
git push --force

# Безопасная принудительная отправка (проверяет, что не перезапишет чужие коммиты)
git push --force-with-lease

# Удалить удаленную ветку
git push origin --delete branch-name
```

## 🔀 Слияние веток (Merge)

### Базовое слияние
```bash
# Слить другую ветку в текущую
git merge feature-branch

# Слить с созданием merge commit (даже если возможен fast-forward)
git merge --no-ff feature-branch

# Слить только если возможен fast-forward
git merge --ff-only feature-branch

# Отменить слияние (если есть конфликты)
git merge --abort
```

### Разрешение конфликтов
```bash
# 1. Увидеть конфликтующие файлы
git status

# 2. Открыть файл и найти маркеры конфликта:
# <<<<<<< HEAD
# ваши изменения
# =======
# изменения из другой ветки
# >>>>>>> feature-branch

# 3. Отредактировать файл, удалив маркеры и выбрав нужный код

# 4. Добавить разрешенный файл
git add resolved-file.txt

# 5. Завершить слияние
git commit

# Использовать версию из текущей ветки
git checkout --ours filename.txt

# Использовать версию из сливаемой ветки
git checkout --theirs filename.txt

# Посмотреть конфликты
git diff --name-only --diff-filter=U
```

## 📐 Rebase - перемещение веток

### Базовый rebase
```bash
# Переместить текущую ветку на верх другой ветки
git rebase main

# Переместить feature-branch на верх main
git checkout feature-branch
git rebase main

# Интерактивный rebase (редактирование истории)
git rebase -i HEAD~3  # последние 3 коммита

# Продолжить rebase после разрешения конфликтов
git rebase --continue

# Пропустить текущий коммит при rebase
git rebase --skip

# Отменить rebase
git rebase --abort
```

### Интерактивный rebase - команды
```bash
# В редакторе будут команды:
# pick   - использовать коммит
# reword - использовать коммит, но изменить сообщение
# edit   - использовать коммит, но остановиться для правки
# squash - объединить с предыдущим коммитом
# fixup  - как squash, но отбросить сообщение
# drop   - удалить коммит

# Пример: объединить последние 3 коммита
git rebase -i HEAD~3
# Изменить pick на squash для последних двух коммитов
```

### Автосквош коммитов
```bash
# Создать коммит, помеченный для автосквоша
git commit --fixup=<commit-hash>

# Автоматически сквошить fixup коммиты
git rebase -i --autosquash main
```

## 🏷️ Теги

```bash
# Создать легковесный тег
git tag v1.0.0

# Создать аннотированный тег (рекомендуется)
git tag -a v1.0.0 -m "Release version 1.0.0"

# Создать тег для конкретного коммита
git tag -a v1.0.0 commit-hash -m "Release"

# Посмотреть список тегов
git tag
git tag -l "v1.*"  # поиск по паттерну

# Посмотреть информацию о теге
git show v1.0.0

# Отправить тег на remote
git push origin v1.0.0

# Отправить все теги
git push --tags

# Удалить локальный тег
git tag -d v1.0.0

# Удалить удаленный тег
git push origin --delete v1.0.0

# Переключиться на тег
git checkout v1.0.0
```

## ⏪ Отмена изменений

### Отмена изменений в рабочей директории
```bash
# Отменить изменения в конкретном файле
git checkout -- filename.txt
git restore filename.txt  # новый синтаксис

# Отменить все изменения в рабочей директории
git checkout -- .
git restore .

# Удалить неотслеживаемые файлы
git clean -f              # файлы
git clean -fd             # файлы и директории
git clean -n              # показать, что будет удалено (dry run)
```

### Отмена staging
```bash
# Убрать файл из staging (но сохранить изменения)
git reset HEAD filename.txt
git restore --staged filename.txt  # новый синтаксис

# Убрать все файлы из staging
git reset HEAD
```

### Отмена коммитов
```bash
# Отменить последний коммит, сохранив изменения в staging
git reset --soft HEAD~1

# Отменить последний коммит, сохранив изменения в рабочей директории
git reset HEAD~1
git reset --mixed HEAD~1  # то же самое

# Отменить последний коммит, удалив все изменения (ОПАСНО!)
git reset --hard HEAD~1

# Отменить несколько коммитов
git reset --hard HEAD~3

# Отменить до конкретного коммита
git reset --hard commit-hash

# Создать новый коммит, отменяющий изменения (безопасно для публичных веток)
git revert HEAD
git revert commit-hash

# Отменить несколько коммитов
git revert HEAD~3..HEAD
```

### Восстановление удаленных коммитов
```bash
# Показать историю всех действий
git reflog

# Восстановить из reflog
git reset --hard HEAD@{2}
git checkout HEAD@{1}
```

## 📜 Просмотр истории

### Базовый лог
```bash
# Стандартная история
git log

# Краткая история (одна строка на коммит)
git log --oneline

# С графом веток
git log --graph --oneline --all

# Красивый граф (алиас)
git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit

# Ограничение количества коммитов
git log -5
git log -n 5

# История за период
git log --since="2 weeks ago"
git log --after="2024-01-01"
git log --until="2024-12-31"

# История конкретного автора
git log --author="John"

# Поиск по сообщению коммита
git log --grep="fix"

# История конкретного файла
git log -- filename.txt

# Показать изменения в коммитах
git log -p
git log -p filename.txt
```

### Просмотр коммитов
```bash
# Показать конкретный коммит
git show commit-hash

# Показать последний коммит
git show HEAD

# Показать изменения в файле из коммита
git show commit-hash:path/to/file.txt

# Список файлов в коммите
git show --name-only commit-hash
git show --stat commit-hash
```

### История по веткам
```bash
# Коммиты в branch1, но не в branch2
git log branch1..branch2

# Коммиты в текущей ветке, но не в main
git log main..HEAD

# Все коммиты, доступные из branch1 или branch2
git log branch1...branch2
```

## 🔍 Поиск и blame

### Поиск в истории
```bash
# Найти коммиты, изменившие количество вхождений строки
git log -S "function_name"

# Найти коммиты, изменившие регулярное выражение
git log -G "regex_pattern"

# Поиск в файлах
git grep "search_term"
git grep -n "search_term"  # с номерами строк
```

### Blame - кто изменял файл
```bash
# Показать, кто изменял каждую строку
git blame filename.txt

# С игнорированием whitespace
git blame -w filename.txt

# Для конкретного диапазона строк
git blame -L 10,20 filename.txt

# Показать email вместо имени
git blame -e filename.txt
```

### Поиск бага - bisect
```bash
# Начать бинарный поиск
git bisect start

# Пометить текущий коммит как плохой
git bisect bad

# Пометить известный хороший коммит
git bisect good commit-hash

# Git автоматически переключится на средний коммит
# Тестируете и помечаете:
git bisect good  # или
git bisect bad

# Завершить поиск
git bisect reset
```

## 🗃️ Stash - временное сохранение

```bash
# Сохранить текущие изменения
git stash
git stash save "описание изменений"

# Сохранить включая неотслеживаемые файлы
git stash -u
git stash --include-untracked

# Сохранить все, включая игнорируемые файлы
git stash -a
git stash --all

# Список stash'ей
git stash list

# Посмотреть содержимое stash
git stash show
git stash show -p stash@{0}

# Применить последний stash (сохранив его)
git stash apply

# Применить конкретный stash
git stash apply stash@{2}

# Применить и удалить последний stash
git stash pop

# Создать ветку из stash
git stash branch branch-name stash@{0}

# Удалить stash
git stash drop stash@{0}

# Очистить все stash'и
git stash clear
```

## 🍒 Cherry-pick - выборочное применение коммитов

```bash
# Применить конкретный коммит к текущей ветке
git cherry-pick commit-hash

# Применить несколько коммитов
git cherry-pick commit1 commit2

# Применить диапазон коммитов
git cherry-pick commit1..commit2

# Cherry-pick без автоматического коммита
git cherry-pick -n commit-hash
git cherry-pick --no-commit commit-hash

# Продолжить после разрешения конфликтов
git cherry-pick --continue

# Отменить cherry-pick
git cherry-pick --abort
```

## 🏗️ Submodules - подмодули

```bash
# Добавить submodule
git submodule add https://github.com/user/repo.git path/to/submodule

# Клонировать репозиторий с submodules
git clone --recursive https://github.com/user/repo.git

# Инициализировать submodules в уже клонированном репозитории
git submodule init
git submodule update

# Обновить все submodules
git submodule update --remote

# Обновить конкретный submodule
git submodule update --remote path/to/submodule

# Удалить submodule
git submodule deinit path/to/submodule
git rm path/to/submodule
rm -rf .git/modules/path/to/submodule
```

## 🔧 Продвинутые техники

### Worktree - несколько рабочих директорий
```bash
# Создать новую рабочую директорию для ветки
git worktree add ../path-to-new-worktree branch-name

# Создать для новой ветки
git worktree add ../hotfix -b hotfix/critical-bug

# Список worktree
git worktree list

# Удалить worktree
git worktree remove ../path-to-worktree

# Очистить устаревшие worktree
git worktree prune
```

### Патчи
```bash
# Создать патч для последних 3 коммитов
git format-patch -3

# Создать патч для коммитов в ветке
git format-patch main..feature-branch

# Применить патч
git apply patch-file.patch

# Применить патч с коммитом
git am patch-file.patch

# Проверить, можно ли применить патч
git apply --check patch-file.patch
```

### Rerere - повторное использование разрешенных конфликтов
```bash
# Включить rerere
git config --global rerere.enabled true

# Git автоматически запоминает, как вы разрешили конфликты,
# и применит те же решения при повторении конфликтов
```

## 📊 Анализ репозитория

```bash
# Статистика коммитов по авторам
git shortlog -sn

# Количество строк по авторам
git log --author="John" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }'

# Самые измененные файлы
git log --pretty=format: --name-only | sort | uniq -c | sort -rg | head -10

# Размер репозитория
git count-objects -vH

# Найти большие файлы в истории
git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | sed -n 's/^blob //p' | sort --numeric-sort --key=2 | tail -n 10
```

## 🛡️ Безопасность и очистка

### Удаление чувствительных данных из истории
```bash
# Удалить файл из всей истории (ОПАСНО!)
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/sensitive-file" \
  --prune-empty --tag-name-filter cat -- --all

# Более современный способ - использовать git filter-repo
# (требует установки: pip install git-filter-repo)
git filter-repo --path path/to/sensitive-file --invert-paths

# После очистки нужно принудительно запушить
git push origin --force --all
git push origin --force --tags
```

### Очистка репозитория
```bash
# Удалить недостижимые объекты
git gc

# Агрессивная очистка
git gc --aggressive --prune=now

# Проверить целостность репозитория
git fsck
```

## 🎯 Best Practices для Production

### 1. Структура коммитов (Conventional Commits)
```
<type>(<scope>): <subject>

<body>

<footer>

Типы:
- feat: новая функциональность
- fix: исправление бага
- docs: изменения в документации
- style: форматирование, пропущенные точки с запятой и т.д.
- refactor: рефакторинг кода
- test: добавление тестов
- chore: обновление задач сборки, настроек и т.д.

Примеры:
feat(auth): add OAuth2 authentication
fix(api): resolve memory leak in user service
docs(readme): update installation instructions
```

### 2. Стратегия ветвления (Git Flow)
```
main (production)
├── develop (разработка)
│   ├── feature/user-authentication
│   ├── feature/payment-integration
│   └── feature/dashboard
├── release/v1.2.0
└── hotfix/critical-security-bug

Правила:
- main: только стабильный production код
- develop: активная разработка
- feature/*: новые фичи (от develop)
- release/*: подготовка релиза (от develop)
- hotfix/*: срочные исправления (от main)
```

### 3. Рабочий процесс в команде
```bash
# 1. Обновить main и develop
git checkout main
git pull origin main
git checkout develop
git pull origin develop

# 2. Создать feature ветку
git checkout -b feature/new-feature develop

# 3. Работать и коммитить
git add .
git commit -m "feat(module): add new feature"

# 4. Регулярно синхронизировать с develop
git fetch origin
git rebase origin/develop

# 5. Перед PR - привести в порядок историю
git rebase -i origin/develop

# 6. Создать Pull Request в develop

# 7. После мерджа - удалить локальную ветку
git branch -d feature/new-feature
```

### 4. Pre-commit проверки
```bash
# Установить pre-commit hooks
# В .git/hooks/pre-commit:
#!/bin/sh
npm run lint
npm test
```

### 5. Защита веток
```bash
# На GitHub/GitLab настроить:
# - Защиту main и develop
# - Требование code review
# - Прохождение CI/CD
# - Запрет force push
```

### 6. Полезные .gitignore паттерны
```gitignore
# Dependencies
node_modules/
vendor/

# Environment
.env
.env.local
*.local

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Build
dist/
build/
*.log

# Secrets
*.key
*.pem
secrets.yml
```

### 7. Работа с большими файлами (Git LFS)
```bash
# Установка Git LFS
git lfs install

# Отслеживать файлы
git lfs track "*.psd"
git lfs track "*.zip"

# Коммитить .gitattributes
git add .gitattributes
git commit -m "chore: add git lfs tracking"
```

### 8. Алиасы для продуктивности
```bash
# Добавить в ~/.gitconfig
[alias]
    st = status -sb
    co = checkout
    br = branch
    ci = commit
    unstage = reset HEAD --
    last = log -1 HEAD
    visual = log --graph --oneline --all
    amend = commit --amend --no-edit
    undo = reset HEAD~1 --mixed
    cleanup = !git branch --merged | grep -v '\\*\\|master\\|main\\|develop' | xargs -n 1 git branch -d
    sync = !git fetch --all --prune && git pull
```

### 9. Работа с конфиденциальными данными
```bash
# НИКОГДА не коммитить:
- .env файлы
- API ключи
- Пароли
- Сертификаты
- Приватные ключи
- Credentials

# Использовать:
- Environment variables
- Secret managers (AWS Secrets Manager, Vault)
- .env.example для документации
```

### 10. Резервное копирование
```bash
# Создать bundle всего репозитория
git bundle create repo-backup.bundle --all

# Восстановить из bundle
git clone repo-backup.bundle restored-repo
```

## 🚨 Частые проблемы и решения

### Случайно закоммитили в main вместо feature ветки
```bash
# Создать новую ветку с текущим состоянием
git branch feature-branch

# Откатить main на коммит до ошибочного
git reset --hard HEAD~1

# Переключиться на feature ветку
git checkout feature-branch
```

### Забыли добавить файл в последний коммит
```bash
git add forgotten-file.txt
git commit --amend --no-edit
```

### Нужно изменить старый коммит
```bash
# Интерактивный rebase
git rebase -i HEAD~5

# Изменить pick на edit для нужного коммита
# Внести изменения
git add changed-files
git commit --amend
git rebase --continue
```

### Случайно удалили ветку
```bash
# Найти последний коммит ветки в reflog
git reflog

# Восстановить ветку
git checkout -b recovered-branch HEAD@{X}
```

### Конфликты при pull
```bash
# Вариант 1: merge
git pull --no-rebase

# Вариант 2: rebase (чище история)
git pull --rebase

# При конфликтах:
# 1. Решить конфликты
# 2. git add resolved-files
# 3. git rebase --continue
```

### Отменить push (если никто не успел спуллить)
```bash
git reset --hard HEAD~1
git push --force-with-lease
```

## 📚 Полезные команды для ежедневной работы

```bash
# Быстрая проверка статуса всех репозиториев
find . -name ".git" -type d | while read dir; do cd "$dir/.."; echo "$(pwd)"; git status -s; done

# Создать архив ветки
git archive --format=zip --output=project.zip main

# Экспорт последних N коммитов в патчи
git format-patch -10

# Показать коммиты, не запушенные на origin
git log origin/main..HEAD

# Показать файлы, отличающиеся от origin
git diff --name-only origin/main

# Временно игнорировать изменения в файле
git update-index --assume-unchanged file.txt

# Вернуть отслеживание
git update-index --no-assume-unchanged file.txt
```

---

## 🎓 Дополнительные ресурсы

- [Git Documentation](https://git-scm.com/doc)
- [Atlassian Git Tutorials](https://www.atlassian.com/git/tutorials)
- [GitHub Git Cheat Sheet](https://training.github.com/downloads/github-git-cheat-sheet.pdf)
- [Learn Git Branching](https://learngitbranching.js.org/)

**Помните:** `git reflog` - ваш спасательный круг! Почти любую ошибку можно исправить!

---

## 🔐 SSH ключи для Git

### Генерация SSH ключей
```bash
# Создать новый SSH ключ
ssh-keygen -t ed25519 -C "your.email@example.com"

# Для старых систем без ed25519
ssh-keygen -t rsa -b 4096 -C "your.email@example.com"

# Запустить ssh-agent
eval "$(ssh-agent -s)"

# Добавить ключ в ssh-agent
ssh-add ~/.ssh/id_ed25519

# Скопировать публичный ключ (Linux/Mac)
cat ~/.ssh/id_ed25519.pub | pbcopy  # macOS
cat ~/.ssh/id_ed25519.pub | xclip   # Linux

# Проверить соединение
ssh -T git@github.com
ssh -T git@gitlab.com
```

### Настройка нескольких SSH ключей
```bash
# Создать ~/.ssh/config
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github

Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_ed25519_gitlab

Host work-github
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
```

## 🎨 Git Hooks для автоматизации

### Pre-commit (проверка перед коммитом)
```bash
# .git/hooks/pre-commit
#!/bin/sh

echo "Running pre-commit checks..."

# Проверка eslint
npm run lint
if [ $? -ne 0 ]; then
    echo "ESLint failed. Please fix errors before committing."
    exit 1
fi

# Проверка тестов
npm test
if [ $? -ne 0 ]; then
    echo "Tests failed. Please fix tests before committing."
    exit 1
fi

# Проверка на TODO/FIXME в staged файлах
if git diff --cached | grep -E "TODO|FIXME"; then
    echo "Warning: Found TODO or FIXME in staged files"
    read -p "Continue? (y/n) " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        exit 1
    fi
fi

echo "Pre-commit checks passed!"
```

### Commit-msg (проверка сообщения коммита)
```bash
# .git/hooks/commit-msg
#!/bin/sh

commit_msg=$(cat "$1")

# Проверка формата Conventional Commits
pattern="^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,50}"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "Error: Commit message doesn't follow Conventional Commits format"
    echo "Format: <type>(<scope>): <subject>"
    echo "Example: feat(auth): add OAuth2 support"
    exit 1
fi
```

### Pre-push (проверка перед push)
```bash
# .git/hooks/pre-push
#!/bin/sh

echo "Running pre-push checks..."

# Запретить push в main/master напрямую
protected_branch='main'
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')

if [ "$protected_branch" = "$current_branch" ]; then
    echo "Direct push to $protected_branch is not allowed!"
    echo "Please create a Pull Request."
    exit 1
fi

# Проверка, что все тесты проходят
npm run test:ci
if [ $? -ne 0 ]; then
    echo "Tests failed. Push aborted."
    exit 1
fi

echo "Pre-push checks passed!"
```

## 🌐 Работа с GitHub/GitLab через CLI

### GitHub CLI (gh)
```bash
# Установка
# macOS: brew install gh
# Linux: см. https://github.com/cli/cli#installation

# Аутентификация
gh auth login

# Создать репозиторий
gh repo create my-project --public --clone

# Список PR
gh pr list

# Создать PR
gh pr create --title "Add new feature" --body "Description"

# Создать PR с автозаполнением
gh pr create --fill

# Просмотреть PR
gh pr view 123

# Checkout PR локально
gh pr checkout 123

# Мердж PR
gh pr merge 123 --squash

# Создать issue
gh issue create --title "Bug report" --body "Description"

# Список issues
gh issue list

# Просмотреть Actions
gh run list
gh run view 123456

# Клонировать репозиторий
gh repo clone owner/repo
```

### GitLab CLI (glab)
```bash
# Установка
# macOS: brew install glab
# Linux: см. https://gitlab.com/gitlab-org/cli

# Аутентификация
glab auth login

# Создать MR
glab mr create --title "Feature" --description "Description"

# Список MR
glab mr list

# Просмотреть MR
glab mr view 123

# Создать issue
glab issue create --title "Bug"

# Список pipelines
glab ci list
```

## 🏢 Корпоративные настройки Git

### Настройка для работы с корпоративным прокси
```bash
# HTTP прокси
git config --global http.proxy http://proxy.company.com:8080
git config --global https.proxy https://proxy.company.com:8080

# С аутентификацией
git config --global http.proxy http://username:password@proxy.com:8080

# Отключить прокси
git config --global --unset http.proxy
git config --global --unset https.proxy

# Прокси для конкретного домена
git config --global http.https://github.com.proxy http://proxy.com:8080
```

### Настройка разных учетных записей для разных проектов
```bash
# В ~/.gitconfig
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/personal/"]
    path = ~/.gitconfig-personal

# В ~/.gitconfig-work
[user]
    name = Your Name
    email = work@company.com

[core]
    sshCommand = ssh -i ~/.ssh/id_rsa_work

# В ~/.gitconfig-personal
[user]
    name = Your Name
    email = personal@gmail.com

[core]
    sshCommand = ssh -i ~/.ssh/id_rsa_personal
```

## 📈 Performance и оптимизация

### Ускорение работы с большими репозиториями
```bash
# Частичное клонирование (без истории)
git clone --depth 1 https://github.com/user/repo.git

# Клонирование только одной ветки
git clone --single-branch --branch main https://github.com/user/repo.git

# Sparse checkout (только нужные директории)
git clone --filter=blob:none --sparse https://github.com/user/repo.git
cd repo
git sparse-checkout init --cone
git sparse-checkout set src/app

# Включить filesystem monitor (для огромных репозиториев)
git config core.fsmonitor true

# Включить параллельное скачивание
git config fetch.parallel 10

# Оптимизация garbage collection
git config gc.auto 256

# Включить commit-graph для быстрого доступа
git config feature.manyFiles true
git commit-graph write --reachable
```

### Maintenance для больших репозиториев
```bash
# Включить автоматическое обслуживание
git maintenance start

# Ручной запуск оптимизации
git maintenance run --task=gc
git maintenance run --task=commit-graph

# Проверка производительности
time git status
time git log --oneline -100
```

## 🔄 CI/CD интеграция

### GitLab CI пример (.gitlab-ci.yml)
```yaml
stages:
  - test
  - build
  - deploy

variables:
  GIT_DEPTH: 10  # Shallow clone

test:
  stage: test
  script:
    - npm install
    - npm run lint
    - npm test
  only:
    - merge_requests
    - main

build:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
  only:
    - main

deploy:
  stage: deploy
  script:
    - ./deploy.sh
  only:
    - main
  when: manual
```

### GitHub Actions пример (.github/workflows/ci.yml)
```yaml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0  # Полная история для SonarQube и т.д.
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Lint
      run: npm run lint
    
    - name: Test
      run: npm test
    
    - name: Build
      run: npm run build
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: dist
        path: dist/
```

## 🛠️ Продвинутые сценарии

### Разделение большого коммита
```bash
# 1. Начать интерактивный rebase
git rebase -i HEAD~1

# 2. Изменить pick на edit
# 3. Откатить коммит, сохранив изменения
git reset HEAD^

# 4. Добавлять и коммитить по частям
git add -p  # интерактивное добавление
git commit -m "Part 1"
git add file2.txt
git commit -m "Part 2"

# 5. Продолжить rebase
git rebase --continue
```

### Синхронизация форка с upstream
```bash
# 1. Добавить upstream remote
git remote add upstream https://github.com/original/repo.git

# 2. Получить изменения
git fetch upstream

# 3. Переключиться на main
git checkout main

# 4. Слить изменения из upstream
git merge upstream/main

# 5. Отправить в свой форк
git push origin main

# Альтернатива через rebase
git rebase upstream/main
git push origin main --force-with-lease
```

### Перенос коммитов между репозиториями
```bash
# Из репозитория A
git format-patch -1 <commit-hash>

# В репозитории B
git am < 0001-commit-name.patch

# Или напрямую через cherry-pick
# В репозитории B
git remote add repo-a /path/to/repo-a
git fetch repo-a
git cherry-pick <commit-hash>
```

### Изменение автора коммитов
```bash
# Для последнего коммита
git commit --amend --author="New Name <new.email@example.com>"

# Для нескольких коммитов
git rebase -i HEAD~5
# Изменить pick на edit для нужных коммитов
# Для каждого:
git commit --amend --author="New Name <new.email@example.com>" --no-edit
git rebase --continue

# Для всей ветки (изменить OLD_EMAIL на NEW)
git filter-branch --env-filter '
if [ "$GIT_COMMITTER_EMAIL" = "old@email.com" ]; then
    export GIT_COMMITTER_NAME="New Name"
    export GIT_COMMITTER_EMAIL="new@email.com"
fi
if [ "$GIT_AUTHOR_EMAIL" = "old@email.com" ]; then
    export GIT_AUTHOR_NAME="New Name"
    export GIT_AUTHOR_EMAIL="new@email.com"
fi
' --tag-name-filter cat -- --branches --tags
```

## 🎯 Чеклист перед важными операциями

### Перед Force Push
```bash
✓ Убедиться, что никто другой не работает с веткой
✓ Использовать --force-with-lease вместо --force
✓ Сделать backup ветки: git branch backup-branch
✓ Предупредить команду
✓ Проверить, что пушите правильную ветку: git branch --show-current
```

### Перед Rebase
```bash
✓ Закоммитить или stash все изменения
✓ Убедиться, что на правильной ветке
✓ Сделать backup: git branch backup-before-rebase
✓ Обновить целевую ветку: git fetch origin main
✓ Быть готовым к разрешению конфликтов
```

### Перед Merge в main
```bash
✓ Все тесты проходят локально
✓ Code review выполнен
✓ CI/CD pipeline зеленый
✓ Обновлена документация
✓ main обновлен: git pull origin main
✓ Feature ветка ребейзнута на main
✓ Нет конфликтов
```

## 💡 Pro Tips

```bash
# 1. Быстрое переключение на предыдущую ветку
git checkout -
git switch -

# 2. Показать список файлов в коммите
git show --name-only <commit-hash>

# 3. Найти, когда строка была удалена
git log -S "deleted text" --source --all

# 4. Посмотреть, что будет push'нуто
git diff --stat origin/main HEAD

# 5. Создать пустую ветку (без истории)
git checkout --orphan new-branch
git rm -rf .

# 6. Экспортировать измененные файлы
git diff --name-only main feature | xargs tar -czf changes.tar.gz

# 7. Подсчитать количество коммитов
git rev-list --count HEAD

# 8. Найти самый большой файл в репозитории
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  awk '/^blob/ {print substr($0,6)}' | \
  sort --numeric-sort --key=2 | \
  tail -n 10

# 9. Показать ветки с последним коммитом
git for-each-ref --sort=-committerdate refs/heads/ --format='%(committerdate:short) %(refname:short)'

# 10. Автоматически stash перед rebase
git config rebase.autoStash true

# 11. Красивый граф в терминале
git log --all --decorate --oneline --graph

# 12. Показать содержимое файла из другой ветки
git show branch-name:path/to/file.txt

# 13. Сравнить файл между ветками
git diff branch1:file.txt branch2:file.txt

# 14. Быстрый коммит с добавлением всех файлов
git commit -am "message"

# 15. Посмотреть изменения после последнего pull
git log HEAD@{1}..HEAD@{0}
```

## ⚡ Скрипты для автоматизации

### Скрипт для создания feature ветки
```bash
#!/bin/bash
# create-feature.sh

if [ -z "$1" ]; then
    echo "Usage: ./create-feature.sh feature-name"
    exit 1
fi

FEATURE_NAME=$1

echo "Creating feature branch: feature/$FEATURE_NAME"

git checkout main
git pull origin main
git checkout -b feature/$FEATURE_NAME
git push -u origin feature/$FEATURE_NAME

echo "Feature branch created and pushed!"
echo "Branch: feature/$FEATURE_NAME"
```

### Скрипт для очистки старых веток
```bash
#!/bin/bash
# cleanup-branches.sh

echo "Fetching from origin..."
git fetch --prune

echo "Merged local branches (excluding main, develop):"
git branch --merged | grep -vE "^\*|main|develop"

read -p "Delete these branches? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    git branch --merged | grep -vE "^\*|main|develop" | xargs -n 1 git branch -d
    echo "Branches deleted!"
fi
```

### Скрипт для синхронизации всех репозиториев
```bash
#!/bin/bash
# sync-all-repos.sh

find . -name ".git" -type d | while read gitdir; do
    repo=$(dirname "$gitdir")
    echo "Syncing $repo..."
    cd "$repo"
    
    current_branch=$(git branch --show-current)
    
    git fetch --all --prune
    
    if [ "$current_branch" = "main" ] || [ "$current_branch" = "master" ]; then
        git pull --rebase
    fi
    
    cd - > /dev/null
done

echo "All repositories synced!"
```

## 🆘 Экстренные команды (Когда все сломалось)

```bash
# 1. Полный откат к состоянию origin
git fetch origin
git reset --hard origin/main
git clean -fd

# 2. Восстановление после неудачного rebase
git rebase --abort
git reset --hard ORIG_HEAD

# 3. Восстановление удаленной ветки (в течение 30 дней)
git reflog
git checkout -b recovered-branch <commit-hash>

# 4. Отмена последнего push (если никто не спулил)
git reset --hard HEAD~1
git push --force-with-lease

# 5. Восстановление файла из любого коммита
git checkout <commit-hash> -- path/to/file

# 6. Полный reset репозитория (ЯДЕРНАЯ ОПЦИЯ)
git fetch origin
git reset --hard origin/main
git clean -fdx  # Удалит ВСЕ неотслеживаемые файлы

# 7. Если git полностью сломался
rm -rf .git
git init
git remote add origin <url>
git fetch
git reset --hard origin/main
```

---

## 📖 Полезные ссылки для углубленного изучения

- **Официальная документация**: https://git-scm.com/doc
- **Pro Git Book** (бесплатно): https://git-scm.com/book/en/v2
- **Git Flight Rules**: https://github.com/k88hudson/git-flight-rules
- **Conventional Commits**: https://www.conventionalcommits.org/
- **Semantic Versioning**: https://semver.org/
- **GitHub Flow**: https://docs.github.com/en/get-started/quickstart/github-flow
- **Git Flow**: https://nvie.com/posts/a-successful-git-branching-model/

---

**Последний совет**: Используйте `git help <command>` для подробной справки по любой команде!

Например: `git help commit`, `git help rebase`, `git help log`

Эта шпаргалка охватывает 95% ежедневных сценариев работы с Git в production-окружении. Удачи! 🚀