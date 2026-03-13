# Полная шпаргалка по Go (Golang)

## 📋 Содержание
1. [Соглашения об именовании](#соглашения-об-именовании)
2. [Базовый синтаксис](#базовый-синтаксис)
3. [Типы данных](#типы-данных)
4. [Функции](#функции)
5. [Структуры и методы](#структуры-и-методы)
6. [Интерфейсы](#интерфейсы)
7. [Горутины и каналы](#горутины-и-каналы)
8. [Обработка ошибок](#обработка-ошибок)
9. [Работа с файлами и IO](#работа-с-файлами-и-io)
10. [Тестирование](#тестирование)
11. [Модули и зависимости](#модули-и-зависимости)
12. [Лучшие практики](#лучшие-практики)

---

## Соглашения об именовании

### Переменные
```go
// Локальные переменные - camelCase
var userName string
age := 25
isActive := true

// Экспортируемые (публичные) переменные - PascalCase
var MaxConnections = 100
var DefaultTimeout = 30 * time.Second

// Константы - PascalCase или SCREAMING_SNAKE_CASE для глобальных
const Pi = 3.14159
const MAX_RETRY_COUNT = 3

// Короткие имена для циклов и временных переменных
for i := 0; i < 10; i++ {}
for _, v := range items {}

// Акронимы пишутся заглавными
var userID int     // не userId
var httpClient     // не httpClient
var URL string     // не Url
```

### Функции
```go
// Приватные функции - camelCase
func calculateTotal() {}
func getUserByID(id int) {}

// Публичные функции - PascalCase
func NewClient() {}
func GetUserProfile() {}

// Геттеры без префикса Get (если не требуется)
func (u *User) Name() string { return u.name }

// Сеттеры с префиксом Set
func (u *User) SetName(name string) { u.name = name }
```

### Типы и структуры
```go
// Типы - PascalCase
type User struct {}
type RequestHandler func(w http.ResponseWriter, r *http.Request)

// Интерфейсы - существительное или -er суффикс
type Reader interface {}
type Writer interface {}
type UserRepository interface {}
```

### Пакеты
```go
// Короткие, понятные имена, lowercase
package user
package httpclient
package database

// Избегайте:
// - util, common, base (слишком общие)
// - underscores или CamelCase
```

---

## Базовый синтаксис

### Структура программы
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    fmt.Println("Hello, World!")
}
```

### Объявление переменных
```go
// Полная форма
var name string = "John"
var age int = 30

// Короткая форма (только внутри функций)
name := "John"
age := 30

// Множественное объявление
var (
    host = "localhost"
    port = 8080
    ssl  = false
)

// Множественное присваивание
x, y := 10, 20
a, b = b, a  // swap

// Игнорирование значений
_, err := someFunction()

// Zero values
var s string    // ""
var i int       // 0
var b bool      // false
var p *int      // nil
```

### Константы
```go
const Pi = 3.14159

const (
    StatusPending  = "pending"
    StatusApproved = "approved"
    StatusRejected = "rejected"
)

// iota - автоинкремент
const (
    Sunday = iota  // 0
    Monday         // 1
    Tuesday        // 2
)

const (
    _ = iota
    KB = 1 << (10 * iota)  // 1024
    MB                      // 1048576
    GB                      // 1073741824
)
```

---

## Типы данных

### Базовые типы
```go
// Целые числа
var i int = 42           // platform dependent (32 or 64 bit)
var i8 int8 = 127        // -128 to 127
var i16 int16 = 32767
var i32 int32 = 2147483647
var i64 int64 = 9223372036854775807

var ui uint = 42         // unsigned
var ui8 uint8 = 255      // 0 to 255 (byte)
var ui16 uint16 = 65535
var ui32 uint32
var ui64 uint64

// Вещественные числа
var f32 float32 = 3.14
var f64 float64 = 3.141592653589793

// Комплексные числа
var c64 complex64 = 1 + 2i
var c128 complex128 = complex(1, 2)

// Булевы
var b bool = true

// Строки
var s string = "Hello, 世界"
var raw string = `Multi-line
string with\n no escaping`

// Байты и руны
var bt byte = 'A'        // alias для uint8
var r rune = '世'         // alias для int32 (Unicode code point)
```

### Массивы
```go
// Фиксированный размер
var arr [5]int
arr[0] = 10

// Инициализация
arr := [5]int{1, 2, 3, 4, 5}
arr := [...]int{1, 2, 3}  // размер определяется автоматически

// Многомерные массивы
matrix := [3][3]int{
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9},
}
```

### Слайсы (динамические массивы)
```go
// Создание
var s []int                    // nil slice
s = []int{}                    // пустой slice (не nil)
s = []int{1, 2, 3}
s = make([]int, 5)            // длина 5, cap 5
s = make([]int, 5, 10)        // длина 5, cap 10

// Операции
s = append(s, 4)              // добавить элемент
s = append(s, 5, 6, 7)        // добавить несколько
s = append(s, other...)       // добавить другой slice

// Слайсинг
sub := s[1:4]                 // элементы с индекса 1 до 3
sub := s[:3]                  // с начала до индекса 2
sub := s[2:]                  // с индекса 2 до конца
sub := s[:]                   // весь slice

// Копирование
dst := make([]int, len(src))
copy(dst, src)

// Длина и ёмкость
len(s)  // количество элементов
cap(s)  // ёмкость
```

### Map (словари)
```go
// Создание
var m map[string]int              // nil map (нельзя использовать!)
m = map[string]int{}              // пустая map
m = make(map[string]int)          // пустая map
m = map[string]int{               // с инициализацией
    "apple":  5,
    "banana": 3,
}

// Операции
m["orange"] = 7                   // добавить/обновить
value := m["apple"]               // получить (0 если нет)
value, exists := m["apple"]       // проверить существование
delete(m, "banana")               // удалить
len(m)                            // количество элементов

// Итерация
for key, value := range m {
    fmt.Println(key, value)
}

// Только ключи
for key := range m {
    fmt.Println(key)
}
```

### Указатели
```go
// Объявление и получение адреса
var p *int
x := 42
p = &x                // p указывает на x

// Разыменование
fmt.Println(*p)       // читаем значение
*p = 21               // изменяем значение

// new - выделяет память и возвращает указатель
p := new(int)         // *int указывающий на 0

// Указатели на структуры
type Point struct { X, Y int }
p := &Point{10, 20}
fmt.Println(p.X)      // автоматическое разыменование
```

---

## Функции

### Базовый синтаксис
```go
// Простая функция
func add(a int, b int) int {
    return a + b
}

// Сокращённая запись параметров одного типа
func add(a, b int) int {
    return a + b
}

// Несколько возвращаемых значений
func swap(a, b string) (string, string) {
    return b, a
}

// Именованные возвращаемые значения
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        err = errors.New("division by zero")
        return  // возвращает result=0, err
    }
    result = a / b
    return  // naked return
}

// Вариадические функции
func sum(numbers ...int) int {
    total := 0
    for _, n := range numbers {
        total += n
    }
    return total
}
result := sum(1, 2, 3, 4, 5)
```

### Функции высшего порядка
```go
// Функция как параметр
func apply(fn func(int) int, value int) int {
    return fn(value)
}

// Возвращение функции
func multiplier(factor int) func(int) int {
    return func(x int) int {
        return x * factor
    }
}
double := multiplier(2)
result := double(5)  // 10

// Замыкания
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}
c := counter()
fmt.Println(c())  // 1
fmt.Println(c())  // 2
```

### Defer, Panic, Recover
```go
// defer - выполняется перед выходом из функции
func example() {
    defer fmt.Println("world")
    fmt.Println("hello")
    // Output: hello world
}

// Множественные defer (выполняются в обратном порядке - LIFO)
func multiDefer() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
    // Output: 3 2 1
}

// Типичное использование - закрытие ресурсов
func readFile(path string) error {
    file, err := os.Open(path)
    if err != nil {
        return err
    }
    defer file.Close()  // гарантированно закроется
    
    // работа с файлом
    return nil
}

// panic - критическая ошибка
func mustConnect() {
    if !connected {
        panic("not connected")
    }
}

// recover - перехват panic
func safeExecute(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    
    fn()
    return nil
}
```

---

## Структуры и методы

### Определение структур
```go
// Простая структура
type User struct {
    ID        int
    FirstName string
    LastName  string
    Email     string
    Age       int
}

// Создание экземпляров
u1 := User{1, "John", "Doe", "john@example.com", 30}
u2 := User{
    ID:        2,
    FirstName: "Jane",
    Email:     "jane@example.com",
}
u3 := new(User)  // указатель на User с zero values
u4 := &User{}    // тоже указатель

// Вложенные структуры
type Address struct {
    Street  string
    City    string
    ZipCode string
}

type Employee struct {
    User             // встроенная структура (embedding)
    Position  string
    Salary    float64
    Address   Address  // обычное поле
}

e := Employee{
    User: User{
        ID:        1,
        FirstName: "John",
    },
    Position: "Developer",
}
// Доступ к полям встроенной структуры
fmt.Println(e.FirstName)  // через embedding
```

### Теги структур
```go
type User struct {
    ID        int    `json:"id" db:"user_id" validate:"required"`
    FirstName string `json:"first_name" db:"first_name"`
    LastName  string `json:"last_name,omitempty" db:"last_name"`
    Email     string `json:"email" validate:"required,email"`
    Password  string `json:"-" db:"password_hash"`  // игнорировать в JSON
}

// Использование
import "encoding/json"

user := User{ID: 1, FirstName: "John"}
jsonData, _ := json.Marshal(user)
// {"id":1,"first_name":"John"}
```

### Методы
```go
// Метод с value receiver
func (u User) FullName() string {
    return u.FirstName + " " + u.LastName
}

// Метод с pointer receiver (может изменять структуру)
func (u *User) SetEmail(email string) {
    u.Email = email
}

// Правило: используйте pointer receiver если:
// 1. Метод изменяет receiver
// 2. Структура большая (для эффективности)
// 3. Для консистентности (если есть хотя бы один pointer receiver)

// Constructor pattern
func NewUser(firstName, lastName, email string) *User {
    return &User{
        FirstName: firstName,
        LastName:  lastName,
        Email:     email,
    }
}
```

---

## Интерфейсы

### Определение интерфейсов
```go
// Интерфейс - набор методов
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Композиция интерфейсов
type ReadWriter interface {
    Reader
    Writer
}

// Пустой интерфейс (любой тип)
var i interface{}
i = 42
i = "hello"
i = struct{}{}

// any - alias для interface{} (Go 1.18+)
var a any = "anything"
```

### Реализация интерфейсов
```go
// Неявная реализация (duck typing)
type MyWriter struct {
    buffer []byte
}

func (w *MyWriter) Write(p []byte) (n int, err error) {
    w.buffer = append(w.buffer, p...)
    return len(p), nil
}

// MyWriter автоматически реализует Writer
var w Writer = &MyWriter{}

// Проверка реализации интерфейса во время компиляции
var _ Writer = (*MyWriter)(nil)
```

### Type assertion и type switch
```go
// Type assertion
var i interface{} = "hello"
s := i.(string)           // panic если не string
s, ok := i.(string)       // ok = true/false, нет panic

// Type switch
func describe(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("Integer: %d\n", v)
    case string:
        fmt.Printf("String: %s\n", v)
    case bool:
        fmt.Printf("Boolean: %t\n", v)
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}
```

### Стандартные интерфейсы
```go
// Stringer - для строкового представления
type Stringer interface {
    String() string
}

func (u User) String() string {
    return fmt.Sprintf("%s %s <%s>", u.FirstName, u.LastName, u.Email)
}

// Error
type error interface {
    Error() string
}

// Sort interface
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

---

## Горутины и каналы

### Горутины
```go
// Запуск горутины
go func() {
    fmt.Println("Running in goroutine")
}()

// С параметрами
go processData(data)

// Анонимная функция с параметрами
go func(name string) {
    fmt.Println("Hello", name)
}("World")

// WaitGroup для ожидания завершения
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Println("Worker", id)
    }(i)
}

wg.Wait()  // ждём завершения всех горутин
```

### Каналы
```go
// Создание каналов
ch := make(chan int)           // unbuffered
ch := make(chan int, 10)       // buffered (ёмкость 10)
ch := make(chan string, 100)

// Отправка и получение
ch <- 42          // отправить в канал (блокирует если канал полон)
value := <-ch     // получить из канала (блокирует если канал пуст)

// Закрытие канала
close(ch)

// Проверка закрытия
value, ok := <-ch  // ok = false если канал закрыт

// Итерация по каналу (до закрытия)
for value := range ch {
    fmt.Println(value)
}

// Только для чтения/записи
func produce(ch chan<- int) {  // только запись
    ch <- 42
}

func consume(ch <-chan int) {  // только чтение
    value := <-ch
}
```

### Select
```go
// Ожидание нескольких каналов
select {
case msg := <-ch1:
    fmt.Println("Received from ch1:", msg)
case msg := <-ch2:
    fmt.Println("Received from ch2:", msg)
case ch3 <- value:
    fmt.Println("Sent to ch3")
default:
    fmt.Println("No communication")
}

// Timeout pattern
select {
case result := <-ch:
    // обработка
case <-time.After(5 * time.Second):
    fmt.Println("Timeout")
}

// Завершение горутин
done := make(chan struct{})
go func() {
    for {
        select {
        case <-done:
            return
        default:
            // работа
        }
    }
}()
close(done)  // сигнал завершения
```

### Синхронизация
```go
import "sync"

// Mutex
var (
    counter int
    mu      sync.Mutex
)

mu.Lock()
counter++
mu.Unlock()

// RWMutex (множественное чтение, эксклюзивная запись)
var (
    data   map[string]string
    rwmu   sync.RWMutex
)

// Чтение
rwmu.RLock()
value := data[key]
rwmu.RUnlock()

// Запись
rwmu.Lock()
data[key] = value
rwmu.Unlock()

// Once - выполнить только один раз
var once sync.Once
once.Do(func() {
    // инициализация
})

// Atomic operations
import "sync/atomic"

var counter int64
atomic.AddInt64(&counter, 1)
value := atomic.LoadInt64(&counter)
atomic.StoreInt64(&counter, 0)
```

### Паттерны concurrency
```go
// Worker pool
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        results <- j * 2
    }
}

jobs := make(chan int, 100)
results := make(chan int, 100)

// Запуск workers
for w := 1; w <= 3; w++ {
    go worker(w, jobs, results)
}

// Отправка задач
for j := 1; j <= 5; j++ {
    jobs <- j
}
close(jobs)

// Получение результатов
for r := 1; r <= 5; r++ {
    <-results
}

// Pipeline pattern
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Использование
for n := range square(generator(1, 2, 3, 4)) {
    fmt.Println(n)
}
```

---

## Обработка ошибок

### Базовые принципы
```go
// Создание ошибки
err := errors.New("something went wrong")
err := fmt.Errorf("failed to process %s: %w", filename, originalErr)

// Проверка ошибки
if err != nil {
    return err  // или обработать
}

// Игнорирование ошибки (избегайте!)
_ = someFunction()
```

### Кастомные ошибки
```go
// Простой тип ошибки
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// Использование
func validate(age int) error {
    if age < 0 {
        return &ValidationError{
            Field:   "age",
            Message: "must be non-negative",
        }
    }
    return nil
}

// Расширенная ошибка со stack trace
type AppError struct {
    Code    int
    Message string
    Err     error
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("[%d] %s: %v", e.Code, e.Message, e.Err)
    }
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}

func (e *AppError) Unwrap() error {
    return e.Err
}
```

### Работа с ошибками (Go 1.13+)
```go
// Wrapping errors
if err != nil {
    return fmt.Errorf("failed to read config: %w", err)
}

// Unwrapping
originalErr := errors.Unwrap(err)

// Проверка типа ошибки
var validationErr *ValidationError
if errors.As(err, &validationErr) {
    fmt.Println("Validation error:", validationErr.Field)
}

// Проверка наличия конкретной ошибки в цепочке
if errors.Is(err, os.ErrNotExist) {
    fmt.Println("File not found")
}
```

### Паттерны обработки
```go
// Early return
func process(data string) error {
    if data == "" {
        return errors.New("empty data")
    }
    
    result, err := step1(data)
    if err != nil {
        return fmt.Errorf("step1 failed: %w", err)
    }
    
    if err := step2(result); err != nil {
        return fmt.Errorf("step2 failed: %w", err)
    }
    
    return nil
}

// Error accumulation
type MultiError []error

func (m MultiError) Error() string {
    var msgs []string
    for _, err := range m {
        msgs = append(msgs, err.Error())
    }
    return strings.Join(msgs, "; ")
}

func validateAll(items []Item) error {
    var errs MultiError
    for i, item := range items {
        if err := item.Validate(); err != nil {
            errs = append(errs, fmt.Errorf("item %d: %w", i, err))
        }
    }
    if len(errs) > 0 {
        return errs
    }
    return nil
}

// Sentinel errors
var (
    ErrNotFound    = errors.New("not found")
    ErrInvalidInput = errors.New("invalid input")
    ErrUnauthorized = errors.New("unauthorized")
)
```

---

## Работа с файлами и IO

### Чтение файлов
```go
import (
    "os"
    "io"
    "bufio"
)

// Чтение всего файла
data, err := os.ReadFile("file.txt")
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(data))

// Чтение с использованием os.Open
file, err := os.Open("file.txt")
if err != nil {
    log.Fatal(err)
}
defer file.Close()

// Чтение в буфер
buffer := make([]byte, 1024)
n, err := file.Read(buffer)

// Построчное чтение
scanner := bufio.NewScanner(file)
for scanner.Scan() {
    line := scanner.Text()
    fmt.Println(line)
}
if err := scanner.Err(); err != nil {
    log.Fatal(err)
}

// Чтение всего в строку
content, err := io.ReadAll(file)
```

### Запись файлов
```go
// Запись всего файла
data := []byte("Hello, World!")
err := os.WriteFile("file.txt", data, 0644)

// Создание и запись
file, err := os.Create("file.txt")
if err != nil {
    log.Fatal(err)
}
defer file.Close()

file.WriteString("Hello, World!\n")
file.Write([]byte("More data\n"))

// Буферизованная запись
writer := bufio.NewWriter(file)
writer.WriteString("Buffered data\n")
writer.Flush()  // важно!

// Append к существующему файлу
file, err := os.OpenFile("file.txt", os.O_APPEND|os.O_WRONLY, 0644)
```

### Работа с директориями
```go
// Создание директории
os.Mkdir("mydir", 0755)
os.MkdirAll("path/to/dir", 0755)  // с родительскими

// Удаление
os.Remove("file.txt")
os.RemoveAll("dir")  // рекурсивно

// Проверка существования
if _, err := os.Stat("file.txt"); os.IsNotExist(err) {
    fmt.Println("File does not exist")
}

// Чтение директории
entries, err := os.ReadDir(".")
for _, entry := range entries {
    fmt.Println(entry.Name(), entry.IsDir())
}

// Рекурсивный обход
filepath.Walk(".", func(path string, info os.FileInfo, err error) error {
    if err != nil {
        return err
    }
    fmt.Println(path, info.Size())
    return nil
})
```

### JSON
```go
import "encoding/json"

// Маршалинг (struct -> JSON)
type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

p := Person{Name: "John", Age: 30}
jsonData, err := json.Marshal(p)
// {"name":"John","age":30}

// Pretty print
jsonData, err := json.MarshalIndent(p, "", "  ")

// Анмаршалинг (JSON -> struct)
var p2 Person
err := json.Unmarshal(jsonData, &p2)

// Работа с неизвестной структурой
var result map[string]interface{}
json.Unmarshal(jsonData, &result)

// Энкодер/декодер для потоков
encoder := json.NewEncoder(writer)
encoder.Encode(p)

decoder := json.NewDecoder(reader)
var p3 Person
decoder.Decode(&p3)
```

---

## Тестирование

### Unit тесты
```go
// math.go
package math

func Add(a, b int) int {
    return a + b
}

// math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5
    
    if result != expected {
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}

// Table-driven tests
func TestAddTableDriven(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, -1, -2},
        {"zero", 0, 5, 5},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("got %d, want %d", result, tt.expected)
            }
        })
    }
}

// Subtests
func TestMath(t *testing.T) {
    t.Run("Add", func(t *testing.T) {
        if Add(2, 3) != 5 {
            t.Error("Add failed")
        }
    })
    
    t.Run("Subtract", func(t *testing.T) {
        if Subtract(5, 3) != 2 {
            t.Error("Subtract failed")
        }
    })
}
```

### Benchmarks
```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}

// С setup
func BenchmarkComplexOperation(b *testing.B) {
    // Setup
    data := generateTestData()
    
    b.ResetTimer()  // сброс таймера после setup
    
    for i := 0; i < b.N; i++ {
        processData(data)
    }
}

// Запуск: go test -bench=.
// Запуск с memory profiling: go test -bench=. -benchmem
```

### Примеры (Examples)
```go
func ExampleAdd() {
    result := Add(2, 3)
    fmt.Println(result)
    // Output: 5
}

func ExampleAdd_negative() {
    result := Add(-1, -1)
    fmt.Println(result)
    // Output: -2
}
```

### Mocking и тестирование интерфейсов
```go
// Определяем интерфейс
type UserRepository interface {
    GetUser(id int) (*User, error)
    SaveUser(user *User) error
}

// Реальная реализация
type DBRepository struct {
    db *sql.DB
}

func (r *DBRepository) GetUser(id int) (*User, error) {
    // реальная работа с БД
}

// Mock для тестов
type MockRepository struct {
    Users map[int]*User
    Err   error
}

func (m *MockRepository) GetUser(id int) (*User, error) {
    if m.Err != nil {
        return nil, m.Err
    }
    return m.Users[id], nil
}

func (m *MockRepository) SaveUser(user *User) error {
    if m.Err != nil {
        return m.Err
    }
    m.Users[user.ID] = user
    return nil
}

// Тест с mock
func TestUserService(t *testing.T) {
    mock := &MockRepository{
        Users: map[int]*User{
            1: {ID: 1, Name: "John"},
        },
    }
    
    service := NewUserService(mock)
    user, err := service.GetUserByID(1)
    
    if err != nil {
        t.Fatal(err)
    }
    if user.Name != "John" {
        t.Errorf("expected John, got %s", user.Name)
    }
}
```

### Test helpers и утилиты
```go
// Helper функции
func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper()  // помечаем как helper
    if got != want {
        t.Errorf("got %v, want %v", got, want)
    }
}

func assertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatal(err)
    }
}

// Использование
func TestSomething(t *testing.T) {
    result, err := someFunction()
    assertNoError(t, err)
    assertEqual(t, result, expectedValue)
}

// Temporary files для тестов
func TestFileOperation(t *testing.T) {
    tmpfile, err := os.CreateTemp("", "test")
    if err != nil {
        t.Fatal(err)
    }
    defer os.Remove(tmpfile.Name())
    
    // тесты с tmpfile
}
```

### Coverage
```bash
# Запуск тестов с coverage
go test -cover

# Генерация coverage profile
go test -coverprofile=coverage.out

# Просмотр coverage в браузере
go tool cover -html=coverage.out

# Coverage для всех пакетов
go test -cover ./...
```

---

## Модули и зависимости

### Инициализация модуля
```bash
# Создание нового модуля
go mod init github.com/username/projectname

# Создаётся go.mod файл:
# module github.com/username/projectname
# 
# go 1.21
```

### Управление зависимостями
```bash
# Добавление зависимости (автоматически при импорте)
go get github.com/gin-gonic/gin

# Конкретная версия
go get github.com/gin-gonic/gin@v1.9.0

# Последняя версия
go get -u github.com/gin-gonic/gin

# Обновление всех зависимостей
go get -u ./...

# Удаление неиспользуемых зависимостей
go mod tidy

# Скачать зависимости
go mod download

# Vendor (копировать зависимости в проект)
go mod vendor

# Проверка зависимостей
go mod verify

# Просмотр графа зависимостей
go mod graph
```

### Структура go.mod
```go
module github.com/username/projectname

go 1.21

require (
    github.com/gin-gonic/gin v1.9.0
    github.com/lib/pq v1.10.9
)

require (
    // indirect dependencies
    github.com/golang/protobuf v1.5.3 // indirect
)

// Замена зависимости
replace github.com/old/module => github.com/new/module v1.0.0

// Локальная замена
replace github.com/mymodule => ../mymodule

// Исключение версии
exclude github.com/broken/module v1.5.0
```

### Организация проекта
```
myproject/
├── go.mod
├── go.sum
├── main.go
├── cmd/
│   └── app/
│       └── main.go          # точка входа
├── internal/                # приватный код (недоступен извне)
│   ├── domain/             # бизнес-логика
│   │   ├── user.go
│   │   └── order.go
│   ├── repository/         # работа с данными
│   │   ├── user_repo.go
│   │   └── order_repo.go
│   └── service/            # сервисный слой
│       ├── user_service.go
│       └── order_service.go
├── pkg/                     # публичный переиспользуемый код
│   ├── validator/
│   └── logger/
├── api/                     # API определения
│   ├── http/
│   └── grpc/
├── configs/                 # конфигурационные файлы
├── migrations/              # миграции БД
├── scripts/                 # скрипты
├── test/                    # интеграционные тесты
└── docs/                    # документация
```

---

## Лучшие практики

### Именование
```go
// ✅ Хорошо
var userCount int
var httpClient *http.Client
var apiURL string

func getUserByID(id int) (*User, error)
func (s *Server) Start() error

// ❌ Плохо
var user_count int
var HTTPClient *http.Client
var apiUrl string

func get_user_by_id(id int) (*User, error)
func (s *Server) start() error  // если должен быть публичным
```

### Обработка ошибок
```go
// ✅ Хорошо - проверяем все ошибки
data, err := readFile()
if err != nil {
    return fmt.Errorf("failed to read file: %w", err)
}

// ✅ Хорошо - добавляем контекст
if err := validate(input); err != nil {
    return fmt.Errorf("validation failed for %s: %w", input.Name, err)
}

// ❌ Плохо - игнорируем ошибки
data, _ := readFile()

// ❌ Плохо - потеря контекста
if err != nil {
    return err
}
```

### Структуры и конструкторы
```go
// ✅ Хорошо - конструктор с валидацией
type Config struct {
    host string
    port int
    db   *sql.DB
}

func NewConfig(host string, port int) (*Config, error) {
    if host == "" {
        return nil, errors.New("host is required")
    }
    if port <= 0 {
        return nil, errors.New("port must be positive")
    }
    
    return &Config{
        host: host,
        port: port,
    }, nil
}

// ✅ Хорошо - functional options pattern
type ServerOption func(*Server)

func WithTimeout(timeout time.Duration) ServerOption {
    return func(s *Server) {
        s.timeout = timeout
    }
}

func NewServer(addr string, opts ...ServerOption) *Server {
    s := &Server{
        addr:    addr,
        timeout: 30 * time.Second,  // default
    }
    
    for _, opt := range opts {
        opt(s)
    }
    
    return s
}

// Использование
server := NewServer(":8080", 
    WithTimeout(60*time.Second),
    WithMaxConnections(100),
)
```

### Интерфейсы
```go
// ✅ Хорошо - маленькие интерфейсы
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// ✅ Хорошо - accept interfaces, return structs
func ProcessData(r io.Reader) (*Result, error) {
    // ...
}

// ❌ Плохо - слишком большой интерфейс
type DataStore interface {
    Create(item Item) error
    Read(id int) (Item, error)
    Update(item Item) error
    Delete(id int) error
    List() ([]Item, error)
    Count() int
    Truncate() error
    // ... еще 10 методов
}

// Лучше разбить:
type ItemReader interface {
    Read(id int) (Item, error)
    List() ([]Item, error)
}

type ItemWriter interface {
    Create(item Item) error
    Update(item Item) error
    Delete(id int) error
}
```

### Горутины и каналы
```go
// ✅ Хорошо - контролируемое завершение
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return
        case job := <-jobs:
            process(job)
        }
    }
}

// ✅ Хорошо - bounded concurrency
const maxWorkers = 10
sem := make(chan struct{}, maxWorkers)

for _, item := range items {
    sem <- struct{}{}  // acquire
    go func(item Item) {
        defer func() { <-sem }()  // release
        process(item)
    }(item)
}

// ❌ Плохо - неконтролируемое создание горутин
for _, item := range items {
    go process(item)  // может создать тысячи горутин
}

// ❌ Плохо - горутина без механизма завершения
go func() {
    for {
        doWork()
        time.Sleep(time.Second)
    }
}()  // утечка горутины
```

### Context
```go
import "context"

// ✅ Хорошо - передаём context первым параметром
func fetchData(ctx context.Context, url string) (*Data, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    resp, err := http.DefaultClient.Do(req)
    // ...
}

// ✅ Хорошо - таймауты
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

data, err := fetchData(ctx, url)

// ✅ Хорошо - отмена по требованию
ctx, cancel := context.WithCancel(context.Background())
go func() {
    <-stopChan
    cancel()
}()

// ✅ Хорошо - передача метаданных
ctx = context.WithValue(ctx, "requestID", uuid.New())
```

### Логирование
```go
// ✅ Хорошо - структурированное логирование
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

logger.Info("user logged in",
    "user_id", userID,
    "ip", remoteAddr,
    "duration_ms", elapsed.Milliseconds(),
)

// ✅ Хорошо - разные уровни
logger.Debug("processing request", "id", reqID)
logger.Info("request completed", "status", 200)
logger.Warn("slow query", "duration", time.Second)
logger.Error("database error", "error", err)

// ❌ Плохо - неструктурированное
log.Printf("User %d logged in from %s", userID, remoteAddr)
```

### Производительность
```go
// ✅ Хорошо - переиспользование буферов
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process() {
    buf := bufPool.Get().(*bytes.Buffer)
    defer bufPool.Put(buf)
    buf.Reset()
    
    // использование buf
}

// ✅ Хорошо - предварительное выделение памяти
items := make([]Item, 0, expectedSize)

// ✅ Хорошо - избегаем копирования больших структур
func ProcessUser(u *User) {  // pointer
    // ...
}

// ❌ Плохо - копирование при каждом вызове
func ProcessUser(u User) {  // value copy
    // ...
}

// ✅ Хорошо - используем strings.Builder для конкатенации
var sb strings.Builder
for _, s := range parts {
    sb.WriteString(s)
}
result := sb.String()

// ❌ Плохо - множественная конкатенация
var result string
for _, s := range parts {
    result += s  // создаёт новую строку каждый раз
}
```

### Организация кода
```go
// ✅ Хорошо - группировка импортов
import (
    // Стандартная библиотека
    "context"
    "fmt"
    "time"
    
    // Внешние зависимости
    "github.com/gin-gonic/gin"
    "github.com/lib/pq"
    
    // Внутренние пакеты
    "myapp/internal/domain"
    "myapp/internal/repository"
)

// ✅ Хорошо - группировка объявлений
const (
    StatusPending  = "pending"
    StatusApproved = "approved"
)

var (
    ErrNotFound = errors.New("not found")
    ErrInvalid  = errors.New("invalid")
)

// ✅ Хорошо - комментарии для экспортируемых элементов
// User represents a user in the system.
// It contains personal information and authentication details.
type User struct {
    ID    int
    Email string
}

// NewUser creates a new User with the given email.
// It returns an error if the email is invalid.
func NewUser(email string) (*User, error) {
    // ...
}
```

### Тестирование
```go
// ✅ Хорошо - table-driven tests
func TestValidate(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr bool
    }{
        {"valid email", "user@example.com", false},
        {"invalid email", "not-an-email", true},
        {"empty string", "", true},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := Validate(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("got error = %v, wantErr = %v", err, tt.wantErr)
            }
        })
    }
}

// ✅ Хорошо - тестируем интерфейсы, а не реализации
type Storage interface {
    Save(data []byte) error
    Load() ([]byte, error)
}

func TestService(t *testing.T) {
    storage := &MockStorage{}  // mock реализация
    service := NewService(storage)
    // тесты
}
```

---

## Полезные пакеты стандартной библиотеки

### strings
```go
import "strings"

strings.Contains("hello", "ell")        // true
strings.HasPrefix("hello", "he")       // true
strings.HasSuffix("hello", "lo")       // true
strings.ToUpper("hello")               // "HELLO"
strings.ToLower("HELLO")               // "hello"
strings.Split("a,b,c", ",")           // []string{"a", "b", "c"}
strings.Join([]string{"a", "b"}, ",") // "a,b"
strings.Trim(" hello ", " ")          // "hello"
strings.Replace("hello", "l", "L", 2) // "heLLo"
strings.ReplaceAll("hello", "l", "L") // "heLLo"
strings.Count("hello", "l")           // 2
```

### strconv
```go
import "strconv"

// String to int
i, err := strconv.Atoi("42")
i, err := strconv.ParseInt("42", 10, 64)

// Int to string
s := strconv.Itoa(42)
s := strconv.FormatInt(42, 10)

// String to float
f, err := strconv.ParseFloat("3.14", 64)

// Float to string
s := strconv.FormatFloat(3.14159, 'f', 2, 64) // "3.14"

// Boolean
b, err := strconv.ParseBool("true")
s := strconv.FormatBool(true)
```

### time
```go
import "time"

// Текущее время
now := time.Now()

// Создание времени
t := time.Date(2024, time.January, 1, 12, 0, 0, 0, time.UTC)

// Парсинг
t, err := time.Parse("2006-01-02", "2024-01-01")
t, err := time.Parse(time.RFC3339, "2024-01-01T12:00:00Z")

// Форматирование (магическая дата: Jan 2 15:04:05 2006 MST)
s := now.Format("2006-01-02 15:04:05")
s := now.Format(time.RFC3339)

// Арифметика
future := now.Add(24 * time.Hour)
past := now.Add(-1 * time.Hour)
duration := future.Sub(now)

// Сравнение
if t1.Before(t2) {}
if t1.After(t2) {}
if t1.Equal(t2) {}

// Sleep и таймеры
time.Sleep(2 * time.Second)

timer := time.NewTimer(5 * time.Second)
<-timer.C  // блокируется на 5 секунд

ticker := time.NewTicker(time.Second)
defer ticker.Stop()
for t := range ticker.C {
    fmt.Println("Tick at", t)
}
```

### regexp
```go
import "regexp"

// Компиляция
re := regexp.MustCompile(`\d+`)

// Поиск
matched := re.MatchString("abc123")        // true
found := re.FindString("abc123def456")     // "123"
all := re.FindAllString("abc123def456", -1) // ["123", "456"]

// Группы
re := regexp.MustCompile(`(\w+)@(\w+\.\w+)`)
matches := re.FindStringSubmatch("user@example.com")
// ["user@example.com", "user", "example.com"]

// Замена
result := re.ReplaceAllString("abc123def456", "X")
// "abcXdefX"
```

### sort
```go
import "sort"

// Сортировка встроенных типов
ints := []int{3, 1, 4, 1, 5}
sort.Ints(ints)

strings := []string{"c", "a", "b"}
sort.Strings(strings)

// Проверка отсортированности
isSorted := sort.IntsAreSorted(ints)

// Поиск
index := sort.SearchInts(ints, 4)

// Кастомная сортировка
sort.Slice(items, func(i, j int) bool {
    return items[i].Priority < items[j].Priority
})

// Реализация sort.Interface
type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }

sort.Sort(ByAge(people))
```

---

## Быстрая справка по командам

```bash
# Компиляция и запуск
go run main.go              # Запустить программу
go build                    # Собрать бинарник
go build -o myapp          # Собрать с именем
go install                  # Установить в $GOPATH/bin

# Тестирование
go test                     # Запустить тесты
go test -v                  # Подробный вывод
go test -cover              # С coverage
go test -bench=.            # Запустить benchmarks
go test ./...               # Все пакеты рекурсивно

# Модули
go mod init                 # Инициализировать модуль
go mod tidy                 # Очистить зависимости
go get package              # Добавить зависимость
go mod download             # Скачать зависимости

# Форматирование и проверки
go fmt ./...                # Форматировать код
goimports -w .              # Форматировать + исправить импорты
go vet ./...                # Статический анализ
golint ./...                # Линтер (требует установки)
golangci-lint run           # Мета-линтер

# Документация
go doc package              # Документация пакета
go doc package.Function     # Документация функции
godoc -http=:6060          # Запустить локальный сервер документации

# Производительность
go test -cpuprofile cpu.prof
go test -memprofile mem.prof
go tool pprof cpu.prof      # Анализ профиля

# Другое
go version                  # Версия Go
go env                      # Переменные окружения
go list -m all              # Список зависимостей
go clean                    # Очистить кэш сборки
```

---

## Полезные ссылки

- **Официальная документация**: https://go.dev/doc/
- **Effective Go**: https://go.dev/doc/effective_go
- **Go by Example**: https://gobyexample.com/
- **Go Playground**: https://go.dev/play/
- **Style Guide**: https://google.github.io/styleguide/go/
- **Package Documentation**: https://pkg.go.dev/

---

*Эта шпаргалка покрывает основные аспекты Go. Сохраните её для быстрого доступа к синтаксису и лучшим практикам!*