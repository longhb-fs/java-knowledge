# Day 2: OOP — Class, Kế thừa, Đa hình, Interface

> Gộp từ bản 19 ngày: Day 3 (OOP Basics) + Day 4 (OOP Pillars)
> 📖 Đọc sâu: [day-03-oop-basics.md](../java-fundamentals/day-03-oop-basics.md) | [day-04-oop-pillars.md](../java-fundamentals/day-04-oop-pillars.md)

---

## 1. Class & Object — Nền tảng

```java
public class Employee {
    // === Fields (Thuộc tính) ===
    private String name;          // private = chỉ truy cập trong class này
    private double salary;
    private static int count = 0; // static = chia sẻ giữa tất cả instances

    // === Constructor (Hàm khởi tạo) ===
    public Employee(String name, double salary) {
        this.name = name;         // this = object hiện tại
        this.salary = salary;
        count++;
    }

    // Constructor không tham số
    public Employee() {
        this("Unknown", 0);       // Gọi constructor khác bằng this()
    }

    // === Getter / Setter ===
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public double getSalary() { return salary; }

    // === Method ===
    public double annualSalary() {
        return salary * 12;
    }

    // === Static method — gọi mà KHÔNG cần tạo object ===
    public static int getCount() {
        return count;
    }

    // === toString — hiển thị object đẹp khi print ===
    @Override
    public String toString() {
        return "Employee{name='" + name + "', salary=" + salary + "}";
    }
}
```

```java
// Tạo object
Employee emp = new Employee("An", 15_000_000);
System.out.println(emp.getName());         // "An"
System.out.println(emp.annualSalary());    // 180000000
System.out.println(Employee.getCount());   // 1 (gọi static method qua class name)
```

---

## 2. Bốn trụ cột OOP

### 2.1. Encapsulation (Đóng gói)

> Ẩn dữ liệu bên trong, chỉ cho phép truy cập qua getter/setter.

| Access Modifier | Class | Package | Subclass | Mọi nơi |
|----------------|-------|---------|----------|---------|
| `private` | ✅ | ❌ | ❌ | ❌ |
| (default) | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

```java
// ❌ SAI: Field public → ai cũng sửa được → không kiểm soát
public class Account {
    public double balance;  // Ai cũng set balance = -9999 được!
}

// ✅ ĐÚNG: Field private + setter có validation
public class Account {
    private double balance;

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be > 0");
        this.balance += amount;
    }
}
```

### 2.2. Inheritance (Kế thừa)

> Class con kế thừa fields + methods từ class cha. Dùng `extends`.

```java
// Class cha
public class Animal {
    protected String name;

    public Animal(String name) {
        this.name = name;
    }

    public void eat() {
        System.out.println(name + " đang ăn");
    }
}

// Class con — kế thừa Animal
public class Dog extends Animal {

    public Dog(String name) {
        super(name);   // Gọi constructor cha — BẮT BUỘC ở dòng đầu tiên
    }

    // Override — ghi đè method của cha
    @Override
    public void eat() {
        System.out.println(name + " đang gặm xương");
    }

    // Method riêng của Dog
    public void bark() {
        System.out.println("Gâu gâu!");
    }
}
```

⚠️ **Java chỉ cho phép kế thừa đơn** — 1 class chỉ extends 1 class cha. Muốn "đa kế thừa" → dùng Interface.

### 2.3. Polymorphism (Đa hình)

> Cùng 1 method nhưng hành vi khác nhau tùy theo kiểu thực tế của object.

```java
Animal myAnimal = new Dog("Rex");  // Kiểu khai báo: Animal, Kiểu thực tế: Dog
myAnimal.eat();    // → "Rex đang gặm xương" (gọi Dog.eat(), KHÔNG phải Animal.eat())
// myAnimal.bark(); // ❌ COMPILE ERROR — vì kiểu khai báo là Animal, không có bark()

// Ứng dụng: xử lý danh sách hỗn hợp
List<Animal> animals = List.of(new Dog("Rex"), new Cat("Miu"));
for (Animal a : animals) {
    a.eat();  // Mỗi con vật ăn theo cách riêng → đa hình!
}
```

### 2.4. Abstraction (Trừu tượng)

> Ẩn chi tiết triển khai, chỉ lộ interface. Dùng `abstract class` hoặc `interface`.

```java
// Abstract class — có thể có code triển khai + abstract method
public abstract class Shape {
    protected String color;

    public Shape(String color) { this.color = color; }

    // Abstract method — KHÔNG có body, class con PHẢI override
    public abstract double area();

    // Concrete method — có sẵn body
    public String getColor() { return color; }
}

public class Circle extends Shape {
    private double radius;

    public Circle(double radius, String color) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;  // BẮT BUỘC implement
    }
}
```

---

## 3. Interface

```java
// Interface — "hợp đồng" quy định phải có những method gì
public interface Flyable {
    void fly();                                    // abstract (mặc định)
    default void land() { System.out.println("Đáp xuống"); }  // Java 8+: có body
    static boolean canFly(Object obj) { return obj instanceof Flyable; }  // static method
}

public interface Swimmable {
    void swim();
}

// Class có thể implements NHIỀU interfaces (giải quyết vấn đề đa kế thừa)
public class Duck extends Animal implements Flyable, Swimmable {
    public Duck(String name) { super(name); }

    @Override public void fly() { System.out.println(name + " bay"); }
    @Override public void swim() { System.out.println(name + " bơi"); }
}
```

### Abstract Class vs Interface — Khi nào dùng?

| Tiêu chí | Abstract Class | Interface |
|----------|---------------|-----------|
| Kế thừa | Chỉ extends 1 | implements nhiều |
| Constructor | Có | Không |
| Fields | Mọi loại | Chỉ `public static final` |
| Methods | abstract + concrete | abstract + default + static |
| **Dùng khi** | **Các class liên quan chặt** (Animal → Dog, Cat) | **Khả năng chung** (Flyable, Serializable) |

💡 **Quy tắc ngón tay cái:** Mối quan hệ **"là gì" (is-a)** → abstract class. Mối quan hệ **"có thể làm gì" (can-do)** → interface.

---

## 4. Record, Sealed Class, Enum

### 4.1. Record (Java 16+) — Immutable data class

```java
// TRƯỚC: Phải viết constructor, getter, equals, hashCode, toString
// SAU: 1 dòng!
public record Point(int x, int y) {}

Point p = new Point(1, 2);
p.x();       // 1 (getter tự động, KHÔNG có prefix "get")
p.y();       // 2
// p.x = 5;  // ❌ ERROR — Record là immutable
```

### 4.2. Enum — Tập hợp hằng số cố định

```java
public enum Status {
    PENDING, APPROVED, REJECTED;

    public boolean isFinal() {
        return this == APPROVED || this == REJECTED;
    }
}

Status s = Status.APPROVED;
s.name();      // "APPROVED"
s.ordinal();   // 1 (vị trí, bắt đầu từ 0)
```

### 4.3. Sealed Class (Java 17+) — Giới hạn ai được kế thừa

```java
public sealed class Payment permits CreditCard, BankTransfer, Cash {}

public final class CreditCard extends Payment {}    // final = không ai kế thừa tiếp
public final class BankTransfer extends Payment {}
public final class Cash extends Payment {}
// public class Bitcoin extends Payment {}  // ❌ ERROR — không nằm trong permits
```

---

## 5. Những keyword quan trọng

| Keyword | Ý nghĩa | Ví dụ |
|---------|---------|-------|
| `this` | Object hiện tại | `this.name = name;` |
| `super` | Truy cập class cha | `super.eat();` |
| `static` | Thuộc về class (không cần tạo object) | `Math.max(a, b)` |
| `final` | Không thể thay đổi/ghi đè/kế thừa | `final int X = 10;` |
| `abstract` | Chưa triển khai, class con phải override | `abstract void draw();` |
| `instanceof` | Kiểm tra kiểu | `if (a instanceof Dog d) { d.bark(); }` |

---

## 6. Equals, HashCode, Comparable

```java
public class Employee implements Comparable<Employee> {
    private String id;
    private String name;

    // equals + hashCode — BẮT BUỘC override nếu dùng làm key trong HashMap/HashSet
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee e)) return false;
        return Objects.equals(id, e.id);  // So sánh theo id
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);  // PHẢI consistent với equals
    }

    // Comparable — sắp xếp tự nhiên
    @Override
    public int compareTo(Employee other) {
        return this.name.compareTo(other.name);  // Sắp xếp theo tên
    }
}
```

💡 **Quy tắc:** Nếu override `equals()` → PHẢI override `hashCode()`. Hai object `equals()` = true → PHẢI cùng `hashCode()`.

---

## 7. Bài tập

1. **Hệ thống quản lý:** Tạo abstract class `Shape` với method `area()` và `perimeter()`. Implement `Circle`, `Rectangle`, `Triangle`.
2. **Đa hình:** Tạo `List<Shape>`, thêm các hình khác nhau, tính tổng diện tích.
3. **Interface:** Tạo interface `Payable` với method `calculatePay()`. Implement cho `FullTimeEmployee` và `Contractor`.

---

## Navigation

- [← Day 1: Basics](./day-1-basics.md)
- [Day 3: Exception + String + Collection →](./day-3-exception-string-collection.md)
