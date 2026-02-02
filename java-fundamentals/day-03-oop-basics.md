# Day 3: OOP Basics

## Mục tiêu
- Hiểu Class và Object
- Constructor
- Methods
- Access Modifiers
- `this` keyword
- Static members

---

## 1. Class và Object

### 1.1. Khái niệm

```
Class = Blueprint (bản thiết kế)
Object = Instance (thực thể từ bản thiết kế)

Ví dụ thực tế:
- Class "Car" là bản thiết kế xe
- Object "myCar" là chiếc xe cụ thể (Toyota Camry, màu đen, ...)
```

### 1.2. Định nghĩa Class

```java
// File: Car.java
public class Car {
    // Fields (thuộc tính)
    String brand;
    String color;
    int year;
    double price;

    // Methods (phương thức)
    void start() {
        System.out.println("Car is starting...");
    }

    void stop() {
        System.out.println("Car stopped.");
    }

    void displayInfo() {
        System.out.println("Brand: " + brand);
        System.out.println("Color: " + color);
        System.out.println("Year: " + year);
        System.out.println("Price: $" + price);
    }
}
```

### 1.3. Tạo Object

```java
public class Main {
    public static void main(String[] args) {
        // Tạo object
        Car myCar = new Car();

        // Gán giá trị cho fields
        myCar.brand = "Toyota";
        myCar.color = "Black";
        myCar.year = 2023;
        myCar.price = 30000;

        // Gọi methods
        myCar.displayInfo();
        myCar.start();
        myCar.stop();

        // Tạo nhiều objects
        Car car1 = new Car();
        car1.brand = "Honda";

        Car car2 = new Car();
        car2.brand = "BMW";
    }
}
```

### 1.4. Class với nhiều file

```
src/
├── Car.java
├── Student.java
├── Product.java
└── Main.java
```

---

## 2. Constructor

### 2.1. Default Constructor

```java
public class Person {
    String name;
    int age;

    // Default constructor (tự động tạo nếu không có constructor nào)
    public Person() {
        // Khởi tạo mặc định
        name = "Unknown";
        age = 0;
    }
}

// Sử dụng
Person p = new Person();  // Gọi default constructor
```

### 2.2. Parameterized Constructor

```java
public class Person {
    String name;
    int age;

    // Constructor với tham số
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

// Sử dụng
Person p = new Person("John", 25);
```

### 2.3. Multiple Constructors (Overloading)

```java
public class Person {
    String name;
    int age;
    String email;

    // Constructor 1: No params
    public Person() {
        this.name = "Unknown";
        this.age = 0;
        this.email = "";
    }

    // Constructor 2: Name only
    public Person(String name) {
        this.name = name;
        this.age = 0;
        this.email = "";
    }

    // Constructor 3: Name and age
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
        this.email = "";
    }

    // Constructor 4: All params
    public Person(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }
}

// Sử dụng
Person p1 = new Person();
Person p2 = new Person("John");
Person p3 = new Person("John", 25);
Person p4 = new Person("John", 25, "john@email.com");
```

### 2.4. Constructor Chaining

```java
public class Person {
    String name;
    int age;
    String email;

    public Person() {
        this("Unknown", 0, "");  // Gọi constructor 3 params
    }

    public Person(String name) {
        this(name, 0, "");  // Gọi constructor 3 params
    }

    public Person(String name, int age) {
        this(name, age, "");  // Gọi constructor 3 params
    }

    public Person(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }
}
```

---

## 3. Methods

### 3.1. Cấu trúc Method

```java
accessModifier returnType methodName(parameters) {
    // method body
    return value;  // nếu có return type
}
```

### 3.2. Các loại Methods

```java
public class Calculator {

    // Method không có return value
    public void printHello() {
        System.out.println("Hello!");
    }

    // Method có return value
    public int add(int a, int b) {
        return a + b;
    }

    // Method với nhiều tham số
    public double calculateAverage(double... numbers) {
        double sum = 0;
        for (double num : numbers) {
            sum += num;
        }
        return sum / numbers.length;
    }

    // Method return object
    public int[] getArray() {
        return new int[]{1, 2, 3, 4, 5};
    }
}
```

### 3.3. Method Overloading

```java
public class MathUtils {

    // Cùng tên, khác tham số
    public int add(int a, int b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public String add(String a, String b) {
        return a + b;
    }
}

// Sử dụng
MathUtils math = new MathUtils();
System.out.println(math.add(1, 2));        // 3
System.out.println(math.add(1, 2, 3));     // 6
System.out.println(math.add(1.5, 2.5));    // 4.0
System.out.println(math.add("Hello ", "World"));  // Hello World
```

### 3.4. Pass by Value

```java
public class PassByValueDemo {

    // Primitive: pass by value
    public void changeValue(int x) {
        x = 100;  // Chỉ thay đổi copy
    }

    // Object: pass reference by value
    public void changeObject(Person p) {
        p.name = "Changed";  // Thay đổi object gốc
    }

    public void reassignObject(Person p) {
        p = new Person("New");  // Không ảnh hưởng object gốc
    }

    public static void main(String[] args) {
        PassByValueDemo demo = new PassByValueDemo();

        // Test primitive
        int num = 10;
        demo.changeValue(num);
        System.out.println(num);  // Vẫn là 10

        // Test object modification
        Person person = new Person();
        person.name = "Original";
        demo.changeObject(person);
        System.out.println(person.name);  // "Changed"

        // Test object reassignment
        demo.reassignObject(person);
        System.out.println(person.name);  // Vẫn là "Changed"
    }
}
```

---

## 4. Access Modifiers

### 4.1. Các loại Access Modifiers

| Modifier | Class | Package | Subclass | World |
|----------|-------|---------|----------|-------|
| `public` | ✅ | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| (default) | ✅ | ✅ | ❌ | ❌ |
| `private` | ✅ | ❌ | ❌ | ❌ |

### 4.2. Ví dụ

```java
public class Person {
    public String publicField;       // Accessible everywhere
    protected String protectedField; // Same package + subclasses
    String defaultField;             // Same package only
    private String privateField;     // Same class only

    public void publicMethod() { }
    protected void protectedMethod() { }
    void defaultMethod() { }
    private void privateMethod() { }
}
```

### 4.3. Best Practices

```java
public class BankAccount {
    // Private fields - encapsulation
    private String accountNumber;
    private double balance;

    // Public constructor
    public BankAccount(String accountNumber, double initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }

    // Public getters
    public String getAccountNumber() {
        return accountNumber;
    }

    public double getBalance() {
        return balance;
    }

    // Public methods với business logic
    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }

    public boolean withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
            return true;
        }
        return false;
    }
}
```

---

## 5. `this` Keyword

### 5.1. Tham chiếu đến instance hiện tại

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        // this.field để phân biệt với parameter
        this.name = name;
        this.age = age;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;  // this có thể bỏ ở đây
    }
}
```

### 5.2. Gọi constructor khác

```java
public class Rectangle {
    private int width;
    private int height;

    public Rectangle() {
        this(1, 1);  // Gọi constructor 2 params
    }

    public Rectangle(int side) {
        this(side, side);  // Hình vuông
    }

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
}
```

### 5.3. Return this (Method Chaining)

```java
public class StringBuilder2 {
    private String value = "";

    public StringBuilder2 append(String s) {
        value += s;
        return this;  // Return chính object này
    }

    public StringBuilder2 appendLine(String s) {
        value += s + "\n";
        return this;
    }

    public String build() {
        return value;
    }
}

// Sử dụng - Method chaining
String result = new StringBuilder2()
    .append("Hello ")
    .append("World")
    .appendLine("!")
    .append("How are you?")
    .build();
```

---

## 6. Static Members

### 6.1. Static Fields

```java
public class Counter {
    // Static field - shared across all instances
    private static int count = 0;

    // Instance field - unique to each instance
    private int id;

    public Counter() {
        count++;
        this.id = count;
    }

    public static int getCount() {
        return count;
    }

    public int getId() {
        return id;
    }
}

// Sử dụng
Counter c1 = new Counter();  // count = 1, c1.id = 1
Counter c2 = new Counter();  // count = 2, c2.id = 2
Counter c3 = new Counter();  // count = 3, c3.id = 3

System.out.println(Counter.getCount());  // 3
System.out.println(c1.getId());  // 1
System.out.println(c2.getId());  // 2
```

### 6.2. Static Methods

```java
public class MathUtils {

    // Static method - gọi không cần tạo object
    public static int add(int a, int b) {
        return a + b;
    }

    public static int max(int a, int b) {
        return (a > b) ? a : b;
    }

    public static double power(double base, int exp) {
        double result = 1;
        for (int i = 0; i < exp; i++) {
            result *= base;
        }
        return result;
    }
}

// Sử dụng - gọi qua class name
int sum = MathUtils.add(5, 3);
int maximum = MathUtils.max(10, 20);
double squared = MathUtils.power(2, 10);  // 1024
```

### 6.3. Static Block

```java
public class DatabaseConfig {
    private static String url;
    private static String username;
    private static String password;

    // Static block - chạy 1 lần khi class được load
    static {
        System.out.println("Loading database config...");
        url = "jdbc:mysql://localhost:3306/mydb";
        username = "root";
        password = "password";
        System.out.println("Config loaded!");
    }

    public static String getUrl() {
        return url;
    }
}

// Khi truy cập class lần đầu, static block chạy
String url = DatabaseConfig.getUrl();
```

### 6.4. Static vs Instance

```java
public class Demo {
    private int instanceVar = 10;
    private static int staticVar = 20;

    // Instance method
    public void instanceMethod() {
        System.out.println(instanceVar);  // ✅ OK
        System.out.println(staticVar);    // ✅ OK
        staticMethod();                    // ✅ OK
    }

    // Static method
    public static void staticMethod() {
        // System.out.println(instanceVar);  // ❌ Error!
        System.out.println(staticVar);       // ✅ OK
        // instanceMethod();                  // ❌ Error!
    }
}
```

---

## 7. Getters và Setters

### 7.1. Encapsulation Pattern

```java
public class Student {
    // Private fields
    private String name;
    private int age;
    private double gpa;

    // Constructor
    public Student(String name, int age, double gpa) {
        this.setName(name);
        this.setAge(age);
        this.setGpa(gpa);
    }

    // Getters
    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    public double getGpa() {
        return gpa;
    }

    // Setters với validation
    public void setName(String name) {
        if (name != null && !name.trim().isEmpty()) {
            this.name = name;
        }
    }

    public void setAge(int age) {
        if (age >= 0 && age <= 150) {
            this.age = age;
        }
    }

    public void setGpa(double gpa) {
        if (gpa >= 0.0 && gpa <= 4.0) {
            this.gpa = gpa;
        }
    }
}
```

### 7.2. Read-Only và Write-Only

```java
public class ReadOnlyExample {
    private final String id;  // final = không thể thay đổi

    public ReadOnlyExample(String id) {
        this.id = id;
    }

    // Chỉ có getter, không có setter
    public String getId() {
        return id;
    }
}

public class WriteOnlyExample {
    private String password;

    // Chỉ có setter, không có getter
    public void setPassword(String password) {
        this.password = hashPassword(password);
    }

    private String hashPassword(String password) {
        // Hash logic
        return "hashed_" + password;
    }
}
```

---

## 8. Bài tập thực hành

### Bài 1: Class Student
Tạo class `Student` với:
- Fields: id, name, age, gpa
- Constructors: default, full params
- Getters/Setters với validation
- Method `displayInfo()`
- Method `isPassed()` - GPA >= 2.0

```java
// Expected usage:
Student s1 = new Student("S001", "John", 20, 3.5);
s1.displayInfo();
System.out.println("Passed: " + s1.isPassed());

s1.setAge(-5);  // Should not change (invalid)
s1.setGpa(5.0); // Should not change (invalid)
```

---

### Bài 2: Class BankAccount
Tạo class `BankAccount` với:
- Fields: accountNumber, ownerName, balance
- Static field: totalAccounts (đếm số tài khoản)
- Methods: deposit(), withdraw(), transfer(BankAccount target, double amount)
- Validation cho các methods

```java
// Expected usage:
BankAccount acc1 = new BankAccount("001", "John", 1000);
BankAccount acc2 = new BankAccount("002", "Jane", 500);

acc1.deposit(500);           // Balance: 1500
acc1.withdraw(200);          // Balance: 1300
acc1.transfer(acc2, 300);    // acc1: 1000, acc2: 800

System.out.println(BankAccount.getTotalAccounts());  // 2
```

---

### Bài 3: Class Rectangle
Tạo class `Rectangle` với:
- Fields: width, height
- Constructors: default (1,1), square (side), full (width, height)
- Methods: getArea(), getPerimeter(), isSquare()
- Static method: compare(Rectangle r1, Rectangle r2) - return rectangle lớn hơn

---

### Bài 4: Class Employee với Department
Tạo 2 classes:
- `Department`: id, name, employeeCount
- `Employee`: id, name, salary, department

Yêu cầu:
- Khi thêm Employee vào Department, tăng employeeCount
- Method tính tổng lương của Department

---

### Bài 5: Class Library System
Tạo hệ thống quản lý thư viện đơn giản:
- Class `Book`: isbn, title, author, available
- Class `Member`: id, name, borrowedBooks[]
- Class `Library`: books[], members[]

Methods:
- Library.addBook(Book)
- Library.registerMember(Member)
- Member.borrowBook(Book) - kiểm tra available
- Member.returnBook(Book)

---

## 9. Đáp án tham khảo

<details>
<summary>Bài 1: Class Student</summary>

```java
public class Student {
    private String id;
    private String name;
    private int age;
    private double gpa;

    // Default constructor
    public Student() {
        this.id = "";
        this.name = "Unknown";
        this.age = 0;
        this.gpa = 0.0;
    }

    // Full constructor
    public Student(String id, String name, int age, double gpa) {
        this.id = id;
        this.setName(name);
        this.setAge(age);
        this.setGpa(gpa);
    }

    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public int getAge() { return age; }
    public double getGpa() { return gpa; }

    // Setters with validation
    public void setId(String id) {
        if (id != null && !id.isEmpty()) {
            this.id = id;
        }
    }

    public void setName(String name) {
        if (name != null && !name.trim().isEmpty()) {
            this.name = name;
        }
    }

    public void setAge(int age) {
        if (age >= 0 && age <= 100) {
            this.age = age;
        }
    }

    public void setGpa(double gpa) {
        if (gpa >= 0.0 && gpa <= 4.0) {
            this.gpa = gpa;
        }
    }

    // Methods
    public void displayInfo() {
        System.out.println("=== Student Info ===");
        System.out.println("ID: " + id);
        System.out.println("Name: " + name);
        System.out.println("Age: " + age);
        System.out.printf("GPA: %.2f%n", gpa);
    }

    public boolean isPassed() {
        return gpa >= 2.0;
    }
}
```
</details>

<details>
<summary>Bài 2: Class BankAccount</summary>

```java
public class BankAccount {
    private String accountNumber;
    private String ownerName;
    private double balance;

    private static int totalAccounts = 0;

    public BankAccount(String accountNumber, String ownerName, double initialBalance) {
        this.accountNumber = accountNumber;
        this.ownerName = ownerName;
        this.balance = initialBalance > 0 ? initialBalance : 0;
        totalAccounts++;
    }

    public static int getTotalAccounts() {
        return totalAccounts;
    }

    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
            System.out.printf("Deposited: $%.2f. New balance: $%.2f%n", amount, balance);
        }
    }

    public boolean withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
            System.out.printf("Withdrawn: $%.2f. New balance: $%.2f%n", amount, balance);
            return true;
        }
        System.out.println("Withdrawal failed. Insufficient funds.");
        return false;
    }

    public boolean transfer(BankAccount target, double amount) {
        if (this.withdraw(amount)) {
            target.deposit(amount);
            System.out.printf("Transferred $%.2f to %s%n", amount, target.ownerName);
            return true;
        }
        return false;
    }

    public void displayInfo() {
        System.out.println("Account: " + accountNumber);
        System.out.println("Owner: " + ownerName);
        System.out.printf("Balance: $%.2f%n", balance);
    }
}
```
</details>

---

## Navigation

- [← Day 2: Operators & Control Flow](./day-02-operators-control-flow.md)
- [Day 4: OOP Pillars →](./day-04-oop-pillars.md)
