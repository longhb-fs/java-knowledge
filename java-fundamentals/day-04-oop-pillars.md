# Day 4: OOP Pillars

## Mục tiêu
- Inheritance (Kế thừa)
- Polymorphism (Đa hình)
- Encapsulation (Đóng gói)
- Abstraction (Trừu tượng)

---

## 1. Inheritance (Kế thừa)

### 1.1. Khái niệm

```
Parent Class (Superclass) → Child Class (Subclass)

Animal (parent)
  ├── Dog (child)
  ├── Cat (child)
  └── Bird (child)

Keyword: extends
```

### 1.2. Basic Inheritance

```java
// Parent class
public class Animal {
    protected String name;
    protected int age;

    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public void eat() {
        System.out.println(name + " is eating.");
    }

    public void sleep() {
        System.out.println(name + " is sleeping.");
    }

    public void displayInfo() {
        System.out.println("Name: " + name + ", Age: " + age);
    }
}

// Child class
public class Dog extends Animal {
    private String breed;

    public Dog(String name, int age, String breed) {
        super(name, age);  // Gọi constructor của parent
        this.breed = breed;
    }

    // Method riêng của Dog
    public void bark() {
        System.out.println(name + " is barking: Woof!");
    }

    // Override method từ parent
    @Override
    public void displayInfo() {
        super.displayInfo();  // Gọi method của parent
        System.out.println("Breed: " + breed);
    }
}

// Sử dụng
Dog dog = new Dog("Buddy", 3, "Labrador");
dog.eat();        // Inherited from Animal
dog.sleep();      // Inherited from Animal
dog.bark();       // Dog's own method
dog.displayInfo(); // Overridden method
```

### 1.3. `super` Keyword

```java
public class Cat extends Animal {
    private boolean isIndoor;

    public Cat(String name, int age, boolean isIndoor) {
        super(name, age);  // Gọi constructor của parent
        this.isIndoor = isIndoor;
    }

    @Override
    public void eat() {
        super.eat();  // Gọi method eat() của parent
        System.out.println(name + " prefers fish.");
    }

    public void meow() {
        System.out.println(name + " says: Meow!");
    }
}
```

### 1.4. Inheritance Chain

```java
// Level 1
public class Vehicle {
    protected String brand;

    public void start() {
        System.out.println("Vehicle starting...");
    }
}

// Level 2
public class Car extends Vehicle {
    protected int numDoors;

    public void drive() {
        System.out.println("Car driving...");
    }
}

// Level 3
public class SportsCar extends Car {
    private int topSpeed;

    public void race() {
        System.out.println("Racing at " + topSpeed + " km/h!");
    }
}

// SportsCar có tất cả methods: start(), drive(), race()
SportsCar ferrari = new SportsCar();
ferrari.start();  // From Vehicle
ferrari.drive();  // From Car
ferrari.race();   // Own method
```

### 1.5. Lưu ý về Inheritance

```java
// ❌ Java không hỗ trợ multiple inheritance với class
public class Child extends Parent1, Parent2 { }  // Error!

// ✅ Nhưng có thể implement nhiều interfaces
public class Child extends Parent implements Interface1, Interface2 { }

// ❌ Không thể kế thừa final class
public final class FinalClass { }
public class Child extends FinalClass { }  // Error!

// ❌ Không thể override final method
public class Parent {
    public final void cannotOverride() { }
}
```

---

## 2. Polymorphism (Đa hình)

### 2.1. Method Overriding (Runtime Polymorphism)

```java
public class Shape {
    public double getArea() {
        return 0;
    }
}

public class Circle extends Shape {
    private double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public double getArea() {
        return Math.PI * radius * radius;
    }
}

public class Rectangle extends Shape {
    private double width, height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public double getArea() {
        return width * height;
    }
}

// Polymorphism in action
Shape[] shapes = new Shape[3];
shapes[0] = new Circle(5);
shapes[1] = new Rectangle(4, 6);
shapes[2] = new Circle(3);

for (Shape shape : shapes) {
    // Gọi đúng method dựa trên actual type
    System.out.println("Area: " + shape.getArea());
}
```

### 2.2. Upcasting và Downcasting

```java
// Upcasting - tự động (child → parent)
Animal animal = new Dog("Buddy", 3, "Labrador");
animal.eat();   // ✅ OK
// animal.bark();  // ❌ Error! Animal không có bark()

// Downcasting - phải cast thủ công (parent → child)
if (animal instanceof Dog) {
    Dog dog = (Dog) animal;
    dog.bark();  // ✅ OK
}

// Java 16+ pattern matching
if (animal instanceof Dog dog) {
    dog.bark();  // ✅ OK, auto-cast
}
```

### 2.3. Method Overloading (Compile-time Polymorphism)

```java
public class Calculator {
    // Same name, different parameters
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
```

### 2.4. Covariant Return Type

```java
public class Animal {
    public Animal reproduce() {
        return new Animal();
    }
}

public class Dog extends Animal {
    @Override
    public Dog reproduce() {  // Return type cụ thể hơn
        return new Dog();
    }
}
```

---

## 3. Encapsulation (Đóng gói)

### 3.1. Concept

```
Encapsulation = Data hiding + Bundling data with methods

Ẩn implementation details, expose only what's necessary
```

### 3.2. Implementation

```java
public class Employee {
    // Private fields - hidden from outside
    private String id;
    private String name;
    private double salary;
    private String department;

    // Constructor
    public Employee(String id, String name, double salary) {
        this.id = id;
        this.setName(name);
        this.setSalary(salary);
    }

    // Getters - controlled access
    public String getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public double getSalary() {
        return salary;
    }

    // Setters with validation
    public void setName(String name) {
        if (name != null && name.length() >= 2) {
            this.name = name;
        } else {
            throw new IllegalArgumentException("Name must be at least 2 characters");
        }
    }

    public void setSalary(double salary) {
        if (salary >= 0) {
            this.salary = salary;
        } else {
            throw new IllegalArgumentException("Salary cannot be negative");
        }
    }

    // Business method
    public void raiseSalary(double percentage) {
        if (percentage > 0 && percentage <= 50) {
            this.salary += this.salary * (percentage / 100);
        }
    }
}
```

### 3.3. Immutable Class

```java
public final class ImmutablePerson {
    private final String name;
    private final int age;
    private final List<String> hobbies;

    public ImmutablePerson(String name, int age, List<String> hobbies) {
        this.name = name;
        this.age = age;
        // Defensive copy
        this.hobbies = new ArrayList<>(hobbies);
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    public List<String> getHobbies() {
        // Return copy, not original
        return new ArrayList<>(hobbies);
    }
}
```

---

## 4. Abstraction (Trừu tượng)

### 4.1. Abstract Class

```java
// Abstract class - không thể tạo instance
public abstract class Shape {
    protected String color;

    public Shape(String color) {
        this.color = color;
    }

    // Abstract method - không có body
    public abstract double getArea();
    public abstract double getPerimeter();

    // Concrete method - có body
    public void displayColor() {
        System.out.println("Color: " + color);
    }
}

// Concrete class - phải implement abstract methods
public class Circle extends Shape {
    private double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double getArea() {
        return Math.PI * radius * radius;
    }

    @Override
    public double getPerimeter() {
        return 2 * Math.PI * radius;
    }
}

public class Rectangle extends Shape {
    private double width, height;

    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }

    @Override
    public double getArea() {
        return width * height;
    }

    @Override
    public double getPerimeter() {
        return 2 * (width + height);
    }
}

// Sử dụng
// Shape s = new Shape("Red");  // ❌ Error! Cannot instantiate abstract class
Shape circle = new Circle("Red", 5);
Shape rect = new Rectangle("Blue", 4, 6);

circle.displayColor();  // Color: Red
System.out.println(circle.getArea());  // 78.54...
```

### 4.2. Interface

```java
// Interface - 100% abstract (trước Java 8)
public interface Drawable {
    void draw();  // implicitly public abstract
}

public interface Resizable {
    void resize(double factor);
}

// Class có thể implement nhiều interfaces
public class Circle implements Drawable, Resizable {
    private double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public void draw() {
        System.out.println("Drawing circle with radius " + radius);
    }

    @Override
    public void resize(double factor) {
        radius *= factor;
    }
}
```

### 4.3. Interface với default và static methods (Java 8+)

```java
public interface Vehicle {
    // Abstract method
    void start();
    void stop();

    // Default method - có body
    default void honk() {
        System.out.println("Beep beep!");
    }

    // Static method
    static void printInfo() {
        System.out.println("This is a vehicle interface");
    }
}

public class Car implements Vehicle {
    @Override
    public void start() {
        System.out.println("Car starting...");
    }

    @Override
    public void stop() {
        System.out.println("Car stopping...");
    }

    // Có thể override default method
    @Override
    public void honk() {
        System.out.println("Car honking: HONK HONK!");
    }
}

// Sử dụng
Car car = new Car();
car.start();
car.honk();
Vehicle.printInfo();  // Static method
```

### 4.4. Abstract Class vs Interface

| Feature | Abstract Class | Interface |
|---------|----------------|-----------|
| Methods | Abstract + Concrete | Abstract + Default (Java 8+) |
| Fields | Any type | public static final only |
| Constructor | ✅ Yes | ❌ No |
| Multiple inheritance | ❌ No | ✅ Yes |
| Access modifiers | Any | public (implicitly) |
| When to use | "is-a" + shared code | "can-do" capability |

```java
// Abstract class: shared state + behavior
abstract class Animal {
    protected String name;
    protected int age;

    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public abstract void makeSound();

    public void sleep() {
        System.out.println(name + " is sleeping");
    }
}

// Interface: capability
interface Swimmable {
    void swim();
}

interface Flyable {
    void fly();
}

// Duck is an Animal that can swim and fly
class Duck extends Animal implements Swimmable, Flyable {
    public Duck(String name, int age) {
        super(name, age);
    }

    @Override
    public void makeSound() {
        System.out.println("Quack!");
    }

    @Override
    public void swim() {
        System.out.println(name + " is swimming");
    }

    @Override
    public void fly() {
        System.out.println(name + " is flying");
    }
}
```

### 4.5. Sealed Classes (Java 17+)

```java
// Sealed class - giới hạn subclasses
public sealed class Shape permits Circle, Rectangle, Triangle {
    // ...
}

public final class Circle extends Shape { }
public final class Rectangle extends Shape { }
public non-sealed class Triangle extends Shape { }  // Cho phép extend tiếp
```

---

## 5. Object Class

### 5.1. Methods của Object

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    // Override toString()
    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + "}";
    }

    // Override equals()
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Person person = (Person) obj;
        return age == person.age && Objects.equals(name, person.name);
    }

    // Override hashCode()
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}

// Sử dụng
Person p1 = new Person("John", 25);
Person p2 = new Person("John", 25);

System.out.println(p1);  // Person{name='John', age=25}
System.out.println(p1.equals(p2));  // true
System.out.println(p1 == p2);  // false (different objects)
```

### 5.2. getClass() và instanceof

```java
Animal animal = new Dog("Buddy", 3);

// getClass()
System.out.println(animal.getClass());  // class Dog
System.out.println(animal.getClass().getName());  // com.example.Dog
System.out.println(animal.getClass().getSimpleName());  // Dog

// instanceof
System.out.println(animal instanceof Dog);     // true
System.out.println(animal instanceof Animal);  // true
System.out.println(animal instanceof Cat);     // false
```

---

## 6. Bài tập thực hành

### Bài 1: Hệ thống nhân viên
Tạo hệ thống quản lý nhân viên:

```
Employee (abstract)
├── FullTimeEmployee
│   - baseSalary
│   - calculateSalary() = baseSalary
├── PartTimeEmployee
│   - hourlyRate, hoursWorked
│   - calculateSalary() = hourlyRate * hoursWorked
└── Contractor
    - projectFee
    - calculateSalary() = projectFee
```

Yêu cầu:
- Abstract method: calculateSalary()
- Method: displayInfo()
- Tạo array Employee[] và tính tổng lương

---

### Bài 2: Hệ thống hình học
Tạo hệ thống tính toán hình học:

```
interface Drawable { void draw(); }
interface Measurable { double getArea(); double getPerimeter(); }

Shape (abstract) implements Drawable, Measurable
├── Circle
├── Rectangle
├── Triangle
└── Square extends Rectangle
```

---

### Bài 3: Hệ thống thanh toán
Tạo hệ thống xử lý thanh toán:

```
interface Payable { void processPayment(double amount); }

Payment (abstract) implements Payable
├── CreditCardPayment
│   - cardNumber, cvv
│   - validate(), processPayment()
├── BankTransfer
│   - bankAccount, bankName
├── EWallet
    - walletId, balance
```

---

### Bài 4: Game Characters
Tạo hệ thống nhân vật game:

```
interface Attackable { void attack(); }
interface Healable { void heal(); }
interface Movable { void move(); }

GameCharacter (abstract)
├── Warrior implements Attackable, Movable
├── Mage implements Attackable, Healable, Movable
├── Archer implements Attackable, Movable
└── Healer implements Healable, Movable
```

---

### Bài 5: Vehicle Rental System
Tạo hệ thống cho thuê xe:

```
Vehicle (abstract)
├── Car
├── Motorcycle
└── Bicycle

interface Rentable {
    double calculateRentalCost(int days);
    boolean isAvailable();
}
```

---

## 7. Đáp án tham khảo

<details>
<summary>Bài 1: Hệ thống nhân viên</summary>

```java
// Abstract class
public abstract class Employee {
    protected String id;
    protected String name;

    public Employee(String id, String name) {
        this.id = id;
        this.name = name;
    }

    public abstract double calculateSalary();

    public void displayInfo() {
        System.out.println("ID: " + id);
        System.out.println("Name: " + name);
        System.out.printf("Salary: $%.2f%n", calculateSalary());
    }
}

// Full-time employee
public class FullTimeEmployee extends Employee {
    private double baseSalary;

    public FullTimeEmployee(String id, String name, double baseSalary) {
        super(id, name);
        this.baseSalary = baseSalary;
    }

    @Override
    public double calculateSalary() {
        return baseSalary;
    }
}

// Part-time employee
public class PartTimeEmployee extends Employee {
    private double hourlyRate;
    private int hoursWorked;

    public PartTimeEmployee(String id, String name, double hourlyRate, int hoursWorked) {
        super(id, name);
        this.hourlyRate = hourlyRate;
        this.hoursWorked = hoursWorked;
    }

    @Override
    public double calculateSalary() {
        return hourlyRate * hoursWorked;
    }
}

// Contractor
public class Contractor extends Employee {
    private double projectFee;

    public Contractor(String id, String name, double projectFee) {
        super(id, name);
        this.projectFee = projectFee;
    }

    @Override
    public double calculateSalary() {
        return projectFee;
    }
}

// Main
public class Main {
    public static void main(String[] args) {
        Employee[] employees = {
            new FullTimeEmployee("E001", "John", 5000),
            new PartTimeEmployee("E002", "Jane", 25, 80),
            new Contractor("E003", "Bob", 3000)
        };

        double totalSalary = 0;
        for (Employee emp : employees) {
            emp.displayInfo();
            totalSalary += emp.calculateSalary();
            System.out.println("---");
        }

        System.out.printf("Total Salary: $%.2f%n", totalSalary);
    }
}
```
</details>

<details>
<summary>Bài 2: Hệ thống hình học</summary>

```java
// Interfaces
public interface Drawable {
    void draw();
}

public interface Measurable {
    double getArea();
    double getPerimeter();
}

// Abstract class
public abstract class Shape implements Drawable, Measurable {
    protected String color;

    public Shape(String color) {
        this.color = color;
    }
}

// Circle
public class Circle extends Shape {
    private double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double getArea() {
        return Math.PI * radius * radius;
    }

    @Override
    public double getPerimeter() {
        return 2 * Math.PI * radius;
    }

    @Override
    public void draw() {
        System.out.println("Drawing " + color + " circle with radius " + radius);
    }
}

// Rectangle
public class Rectangle extends Shape {
    protected double width;
    protected double height;

    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }

    @Override
    public double getArea() {
        return width * height;
    }

    @Override
    public double getPerimeter() {
        return 2 * (width + height);
    }

    @Override
    public void draw() {
        System.out.println("Drawing " + color + " rectangle " + width + "x" + height);
    }
}

// Square
public class Square extends Rectangle {
    public Square(String color, double side) {
        super(color, side, side);
    }

    @Override
    public void draw() {
        System.out.println("Drawing " + color + " square with side " + width);
    }
}
```
</details>

---

## Navigation

- [← Day 3: OOP Basics](./day-03-oop-basics.md)
- [Day 5: Exception Handling →](./day-05-exception-handling.md)
