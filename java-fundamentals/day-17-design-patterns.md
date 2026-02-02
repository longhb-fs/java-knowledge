# Day 17: Design Patterns

## Mục tiêu
- Creational patterns
- Structural patterns
- Behavioral patterns

---

## 1. Singleton

```java
// Eager initialization
public class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    private EagerSingleton() {}
    public static EagerSingleton getInstance() { return INSTANCE; }
}

// Lazy initialization (thread-safe)
public class LazySingleton {
    private static volatile LazySingleton instance;
    private LazySingleton() {}

    public static LazySingleton getInstance() {
        if (instance == null) {
            synchronized (LazySingleton.class) {
                if (instance == null) {
                    instance = new LazySingleton();
                }
            }
        }
        return instance;
    }
}

// Enum singleton (recommended)
public enum EnumSingleton {
    INSTANCE;
    public void doSomething() { }
}
```

---

## 2. Factory

```java
// Simple Factory
public class ShapeFactory {
    public static Shape create(String type) {
        return switch (type) {
            case "circle" -> new Circle();
            case "rectangle" -> new Rectangle();
            case "triangle" -> new Triangle();
            default -> throw new IllegalArgumentException("Unknown shape: " + type);
        };
    }
}

// Factory Method
public abstract class Dialog {
    public void render() {
        Button button = createButton();
        button.render();
    }
    protected abstract Button createButton();
}

public class WindowsDialog extends Dialog {
    @Override
    protected Button createButton() {
        return new WindowsButton();
    }
}

// Abstract Factory
public interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

public class WindowsFactory implements GUIFactory {
    public Button createButton() { return new WindowsButton(); }
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}
```

---

## 3. Builder

```java
public class User {
    private final String name;
    private final int age;
    private final String email;
    private final String phone;

    private User(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.email = builder.email;
        this.phone = builder.phone;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String name;
        private int age;
        private String email;
        private String phone;

        public Builder name(String name) {
            this.name = name;
            return this;
        }
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        public Builder email(String email) {
            this.email = email;
            return this;
        }
        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }
        public User build() {
            return new User(this);
        }
    }
}

// Usage
User user = User.builder()
    .name("John")
    .age(25)
    .email("john@email.com")
    .build();
```

---

## 4. Adapter

```java
// Target interface
public interface MediaPlayer {
    void play(String filename);
}

// Adaptee
public class VlcPlayer {
    public void playVlc(String filename) { /* ... */ }
}

// Adapter
public class MediaAdapter implements MediaPlayer {
    private VlcPlayer vlcPlayer = new VlcPlayer();

    @Override
    public void play(String filename) {
        vlcPlayer.playVlc(filename);
    }
}
```

---

## 5. Decorator

```java
// Component
public interface Coffee {
    double getCost();
    String getDescription();
}

// Concrete Component
public class SimpleCoffee implements Coffee {
    public double getCost() { return 2.0; }
    public String getDescription() { return "Simple Coffee"; }
}

// Decorator
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    public CoffeeDecorator(Coffee coffee) { this.coffee = coffee; }
}

// Concrete Decorators
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }
    public double getCost() { return coffee.getCost() + 0.5; }
    public String getDescription() { return coffee.getDescription() + ", Milk"; }
}

// Usage
Coffee coffee = new MilkDecorator(new SimpleCoffee());
// Cost: 2.5, Desc: "Simple Coffee, Milk"
```

---

## 6. Strategy

```java
// Strategy interface
public interface PaymentStrategy {
    void pay(double amount);
}

// Concrete strategies
public class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    public CreditCardPayment(String cardNumber) { this.cardNumber = cardNumber; }
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " with credit card");
    }
}

public class PayPalPayment implements PaymentStrategy {
    private String email;
    public PayPalPayment(String email) { this.email = email; }
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " with PayPal");
    }
}

// Context
public class ShoppingCart {
    private PaymentStrategy paymentStrategy;

    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }

    public void checkout(double amount) {
        paymentStrategy.pay(amount);
    }
}

// Usage
ShoppingCart cart = new ShoppingCart();
cart.setPaymentStrategy(new CreditCardPayment("1234-5678"));
cart.checkout(100);
```

---

## 7. Observer

```java
// Observer interface
public interface Observer {
    void update(String message);
}

// Subject
public class NewsAgency {
    private List<Observer> observers = new ArrayList<>();
    private String news;

    public void addObserver(Observer observer) {
        observers.add(observer);
    }

    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }

    public void setNews(String news) {
        this.news = news;
        notifyObservers();
    }

    private void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(news);
        }
    }
}

// Concrete Observer
public class NewsChannel implements Observer {
    private String name;
    public NewsChannel(String name) { this.name = name; }

    @Override
    public void update(String message) {
        System.out.println(name + " received: " + message);
    }
}
```

---

## 8. Template Method

```java
public abstract class DataProcessor {
    // Template method
    public final void process() {
        readData();
        processData();
        saveData();
    }

    protected abstract void readData();
    protected abstract void processData();

    protected void saveData() {
        System.out.println("Saving data to database...");
    }
}

public class CsvProcessor extends DataProcessor {
    protected void readData() { System.out.println("Reading CSV..."); }
    protected void processData() { System.out.println("Processing CSV..."); }
}

public class XmlProcessor extends DataProcessor {
    protected void readData() { System.out.println("Reading XML..."); }
    protected void processData() { System.out.println("Processing XML..."); }
}
```

---

## 9. Bài tập thực hành

### Bài 1: Logger Singleton
Tạo thread-safe Logger singleton.

### Bài 2: Pizza Builder
Implement Pizza builder với nhiều toppings.

### Bài 3: Notification System
Implement Observer pattern cho notification system.

### Bài 4: File Exporter
Implement Strategy pattern cho export PDF/Excel/CSV.

---

## Navigation

- [← Day 16: Reflection](./day-16-reflection-annotations.md)
- [Day 18: Memory & GC →](./day-18-memory-gc.md)
