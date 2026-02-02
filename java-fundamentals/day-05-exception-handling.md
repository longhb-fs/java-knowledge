# Day 5: Exception Handling

## Mục tiêu
- Hiểu Exception hierarchy
- try-catch-finally
- throw và throws
- Custom exceptions
- Best practices

---

## 1. Exception Hierarchy

```
Throwable
├── Error (không nên catch)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
│
└── Exception
    ├── RuntimeException (Unchecked)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ArithmeticException
    │   ├── ClassCastException
    │   └── IllegalArgumentException
    │
    └── Checked Exceptions
        ├── IOException
        ├── SQLException
        ├── FileNotFoundException
        └── ClassNotFoundException
```

### Checked vs Unchecked

| Type | Compile-time check | Must handle? | Example |
|------|-------------------|--------------|---------|
| Checked | ✅ | ✅ Must try-catch or throws | IOException |
| Unchecked | ❌ | ❌ Optional | NullPointerException |

---

## 2. try-catch-finally

### 2.1. Basic try-catch

```java
public class ExceptionDemo {
    public static void main(String[] args) {
        try {
            int result = 10 / 0;  // ArithmeticException
            System.out.println("Result: " + result);
        } catch (ArithmeticException e) {
            System.out.println("Error: Cannot divide by zero!");
            System.out.println("Message: " + e.getMessage());
        }

        System.out.println("Program continues...");
    }
}
```

### 2.2. Multiple catch blocks

```java
public void processData(String[] args) {
    try {
        int index = Integer.parseInt(args[0]);
        int[] numbers = {1, 2, 3};
        int value = numbers[index];
        int result = 100 / value;
        System.out.println("Result: " + result);

    } catch (ArrayIndexOutOfBoundsException e) {
        System.out.println("Invalid index!");

    } catch (NumberFormatException e) {
        System.out.println("Invalid number format!");

    } catch (ArithmeticException e) {
        System.out.println("Cannot divide by zero!");

    } catch (Exception e) {
        // Catch-all (đặt cuối cùng)
        System.out.println("An error occurred: " + e.getMessage());
    }
}
```

### 2.3. Multi-catch (Java 7+)

```java
try {
    // code that may throw exceptions
} catch (IOException | SQLException e) {
    System.out.println("IO or SQL error: " + e.getMessage());
} catch (Exception e) {
    System.out.println("Other error: " + e.getMessage());
}
```

### 2.4. finally block

```java
FileInputStream fis = null;
try {
    fis = new FileInputStream("file.txt");
    // Read file
} catch (FileNotFoundException e) {
    System.out.println("File not found!");
} catch (IOException e) {
    System.out.println("Error reading file!");
} finally {
    // ALWAYS executes (cleanup)
    if (fis != null) {
        try {
            fis.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    System.out.println("Cleanup done.");
}
```

### 2.5. try-with-resources (Java 7+)

```java
// AutoCloseable resources tự động close
try (FileInputStream fis = new FileInputStream("file.txt");
     BufferedReader reader = new BufferedReader(new InputStreamReader(fis))) {

    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }

} catch (FileNotFoundException e) {
    System.out.println("File not found!");
} catch (IOException e) {
    System.out.println("Error reading file!");
}
// fis và reader tự động close
```

### 2.6. Nested try-catch

```java
try {
    System.out.println("Outer try");

    try {
        int result = 10 / 0;
    } catch (ArithmeticException e) {
        System.out.println("Inner catch: " + e.getMessage());
        throw new RuntimeException("Wrapped exception", e);
    }

} catch (RuntimeException e) {
    System.out.println("Outer catch: " + e.getMessage());
}
```

---

## 3. throw và throws

### 3.1. throw - ném exception

```java
public class ValidationUtils {

    public static void validateAge(int age) {
        if (age < 0) {
            throw new IllegalArgumentException("Age cannot be negative");
        }
        if (age > 150) {
            throw new IllegalArgumentException("Age cannot exceed 150");
        }
        System.out.println("Age is valid: " + age);
    }

    public static void main(String[] args) {
        try {
            validateAge(-5);
        } catch (IllegalArgumentException e) {
            System.out.println("Validation error: " + e.getMessage());
        }
    }
}
```

### 3.2. throws - khai báo exception

```java
import java.io.*;

public class FileUtils {

    // Khai báo method có thể throw IOException
    public static String readFile(String path) throws IOException {
        BufferedReader reader = new BufferedReader(new FileReader(path));
        StringBuilder content = new StringBuilder();
        String line;

        while ((line = reader.readLine()) != null) {
            content.append(line).append("\n");
        }
        reader.close();

        return content.toString();
    }

    // Caller phải handle exception
    public static void main(String[] args) {
        try {
            String content = readFile("data.txt");
            System.out.println(content);
        } catch (IOException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

### 3.3. Re-throwing exceptions

```java
public void processFile(String path) throws IOException {
    try {
        // Process file
        readFile(path);
    } catch (IOException e) {
        System.out.println("Logging error: " + e.getMessage());
        throw e;  // Re-throw sau khi log
    }
}

// Hoặc wrap trong exception khác
public void processFile2(String path) {
    try {
        readFile(path);
    } catch (IOException e) {
        throw new RuntimeException("Failed to process file: " + path, e);
    }
}
```

---

## 4. Custom Exceptions

### 4.1. Checked Exception

```java
public class InsufficientFundsException extends Exception {

    private double amount;
    private double balance;

    public InsufficientFundsException(String message) {
        super(message);
    }

    public InsufficientFundsException(double amount, double balance) {
        super(String.format("Insufficient funds: tried to withdraw %.2f but balance is %.2f",
            amount, balance));
        this.amount = amount;
        this.balance = balance;
    }

    public double getAmount() {
        return amount;
    }

    public double getBalance() {
        return balance;
    }
}
```

### 4.2. Unchecked Exception

```java
public class InvalidEmailException extends RuntimeException {

    private String email;

    public InvalidEmailException(String email) {
        super("Invalid email format: " + email);
        this.email = email;
    }

    public InvalidEmailException(String message, Throwable cause) {
        super(message, cause);
    }

    public String getEmail() {
        return email;
    }
}
```

### 4.3. Sử dụng Custom Exception

```java
public class BankAccount {
    private String accountNumber;
    private double balance;

    public BankAccount(String accountNumber, double initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }

    public void withdraw(double amount) throws InsufficientFundsException {
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        if (amount > balance) {
            throw new InsufficientFundsException(amount, balance);
        }
        balance -= amount;
        System.out.printf("Withdrawn: $%.2f. New balance: $%.2f%n", amount, balance);
    }

    public static void main(String[] args) {
        BankAccount account = new BankAccount("123", 1000);

        try {
            account.withdraw(1500);
        } catch (InsufficientFundsException e) {
            System.out.println("Error: " + e.getMessage());
            System.out.printf("Tried: $%.2f, Available: $%.2f%n",
                e.getAmount(), e.getBalance());
        }
    }
}
```

---

## 5. Exception Methods

```java
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    // getMessage() - lấy message
    System.out.println("Message: " + e.getMessage());

    // toString() - class name + message
    System.out.println("ToString: " + e.toString());

    // printStackTrace() - in stack trace
    e.printStackTrace();

    // getStackTrace() - lấy stack trace dạng array
    StackTraceElement[] stackTrace = e.getStackTrace();
    for (StackTraceElement element : stackTrace) {
        System.out.println("  at " + element);
    }

    // getCause() - lấy exception gốc (nếu có)
    Throwable cause = e.getCause();
}
```

---

## 6. Common Exceptions

### 6.1. NullPointerException

```java
String str = null;

// ❌ Gây NullPointerException
// int length = str.length();

// ✅ Check null trước
if (str != null) {
    int length = str.length();
}

// ✅ Hoặc dùng Objects.requireNonNull()
String str2 = Objects.requireNonNull(str, "String cannot be null");

// ✅ Hoặc dùng Optional (Java 8+)
Optional<String> optStr = Optional.ofNullable(str);
int length = optStr.map(String::length).orElse(0);
```

### 6.2. ArrayIndexOutOfBoundsException

```java
int[] arr = {1, 2, 3};

// ❌ Index ngoài range
// int value = arr[5];

// ✅ Check index
int index = 5;
if (index >= 0 && index < arr.length) {
    int value = arr[index];
}
```

### 6.3. NumberFormatException

```java
String input = "abc";

// ❌ Không phải số
// int num = Integer.parseInt(input);

// ✅ Try-catch
try {
    int num = Integer.parseInt(input);
} catch (NumberFormatException e) {
    System.out.println("Invalid number format");
}

// ✅ Hoặc validate trước
if (input.matches("-?\\d+")) {
    int num = Integer.parseInt(input);
}
```

### 6.4. ClassCastException

```java
Object obj = "Hello";

// ❌ Cast sai type
// Integer num = (Integer) obj;

// ✅ Check với instanceof
if (obj instanceof Integer) {
    Integer num = (Integer) obj;
}

// ✅ Java 16+ pattern matching
if (obj instanceof Integer num) {
    System.out.println(num);
}
```

---

## 7. Best Practices

### 7.1. Specific vs Generic Exception

```java
// ❌ Quá generic
try {
    // code
} catch (Exception e) {
    // handle
}

// ✅ Specific exceptions
try {
    // code
} catch (FileNotFoundException e) {
    // handle file not found
} catch (IOException e) {
    // handle other IO errors
}
```

### 7.2. Không bỏ trống catch block

```java
// ❌ Empty catch - nuốt exception
try {
    // code
} catch (Exception e) {
    // Doing nothing!
}

// ✅ Ít nhất log exception
try {
    // code
} catch (Exception e) {
    logger.error("Error occurred", e);
    // Hoặc rethrow
    throw new RuntimeException("Failed to process", e);
}
```

### 7.3. Clean up resources

```java
// ❌ Resource leak
FileInputStream fis = new FileInputStream("file.txt");
// Nếu exception xảy ra ở đây, fis không được close

// ✅ try-with-resources
try (FileInputStream fis = new FileInputStream("file.txt")) {
    // use fis
}  // auto-close
```

### 7.4. Throw early, catch late

```java
// Throw early - validate đầu method
public void processUser(User user) {
    if (user == null) {
        throw new IllegalArgumentException("User cannot be null");
    }
    if (user.getName() == null || user.getName().isEmpty()) {
        throw new IllegalArgumentException("User name is required");
    }
    // Process...
}

// Catch late - handle ở layer phù hợp
public void handleRequest() {
    try {
        service.processUser(user);
    } catch (IllegalArgumentException e) {
        showErrorToUser(e.getMessage());
    }
}
```

### 7.5. Use standard exceptions

```java
// ✅ IllegalArgumentException - tham số không hợp lệ
if (age < 0) {
    throw new IllegalArgumentException("Age cannot be negative");
}

// ✅ IllegalStateException - trạng thái không hợp lệ
if (!isInitialized) {
    throw new IllegalStateException("Object not initialized");
}

// ✅ UnsupportedOperationException - operation không được hỗ trợ
throw new UnsupportedOperationException("This method is not implemented");

// ✅ NullPointerException - giá trị null không được phép
Objects.requireNonNull(param, "Parameter cannot be null");
```

---

## 8. Exception trong thực tế

### 8.1. Service Layer

```java
public class UserService {
    private UserRepository repository;

    public User findById(Long id) throws UserNotFoundException {
        return repository.findById(id)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + id));
    }

    public User create(UserCreateRequest request) {
        validateRequest(request);

        if (repository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException(request.getEmail());
        }

        User user = new User(request);
        return repository.save(user);
    }

    private void validateRequest(UserCreateRequest request) {
        if (request.getName() == null || request.getName().isBlank()) {
            throw new ValidationException("Name is required");
        }
        if (request.getEmail() == null || !request.getEmail().contains("@")) {
            throw new ValidationException("Valid email is required");
        }
    }
}
```

### 8.2. Controller Layer

```java
public class UserController {
    private UserService userService;

    public Response getUser(Long id) {
        try {
            User user = userService.findById(id);
            return Response.ok(user);
        } catch (UserNotFoundException e) {
            return Response.notFound(e.getMessage());
        } catch (Exception e) {
            return Response.error("Internal server error");
        }
    }

    public Response createUser(UserCreateRequest request) {
        try {
            User user = userService.create(request);
            return Response.created(user);
        } catch (ValidationException e) {
            return Response.badRequest(e.getMessage());
        } catch (DuplicateEmailException e) {
            return Response.conflict(e.getMessage());
        }
    }
}
```

---

## 9. Bài tập thực hành

### Bài 1: Custom Exception cho ATM
Tạo hệ thống ATM với các exception:
- `InsufficientBalanceException`
- `InvalidAmountException`
- `CardBlockedException`
- `DailyLimitExceededException`

```java
public class ATM {
    public void withdraw(String cardNumber, double amount)
        throws InsufficientBalanceException, InvalidAmountException,
               CardBlockedException, DailyLimitExceededException {
        // Implementation
    }
}
```

---

### Bài 2: File Processor
Tạo class đọc và xử lý file với exception handling đúng cách:
- Đọc file text
- Parse từng dòng thành object
- Log lỗi chi tiết
- Trả về danh sách objects hợp lệ

```
File format:
name,age,email
John,25,john@email.com
Invalid Line
Jane,30,jane@email.com
```

---

### Bài 3: Validation Framework
Tạo framework validation đơn giản:

```java
public class Validator {
    public static void validate(Object obj) throws ValidationException {
        // Check @NotNull, @Min, @Max, @Email annotations
    }
}

public class User {
    @NotNull
    private String name;

    @Min(0) @Max(150)
    private int age;

    @Email
    private String email;
}
```

---

### Bài 4: Retry Mechanism
Tạo mechanism retry khi gặp exception:

```java
public class RetryHelper {
    public static <T> T retry(Callable<T> task, int maxAttempts, long delayMs)
        throws Exception {
        // Retry logic
    }
}

// Usage
String result = RetryHelper.retry(() -> fetchData(), 3, 1000);
```

---

## 10. Đáp án tham khảo

<details>
<summary>Bài 1: Custom Exception cho ATM</summary>

```java
// Custom Exceptions
public class InsufficientBalanceException extends Exception {
    public InsufficientBalanceException(double requested, double available) {
        super(String.format("Insufficient balance. Requested: %.2f, Available: %.2f",
            requested, available));
    }
}

public class InvalidAmountException extends Exception {
    public InvalidAmountException(String message) {
        super(message);
    }
}

public class CardBlockedException extends Exception {
    public CardBlockedException(String cardNumber) {
        super("Card is blocked: " + cardNumber);
    }
}

public class DailyLimitExceededException extends Exception {
    public DailyLimitExceededException(double limit) {
        super("Daily withdrawal limit exceeded: " + limit);
    }
}

// ATM Class
public class ATM {
    private Map<String, Double> balances = new HashMap<>();
    private Map<String, Boolean> blockedCards = new HashMap<>();
    private Map<String, Double> dailyWithdrawals = new HashMap<>();
    private static final double DAILY_LIMIT = 5000;

    public void withdraw(String cardNumber, double amount)
            throws InsufficientBalanceException, InvalidAmountException,
                   CardBlockedException, DailyLimitExceededException {

        // Check blocked
        if (blockedCards.getOrDefault(cardNumber, false)) {
            throw new CardBlockedException(cardNumber);
        }

        // Check amount
        if (amount <= 0) {
            throw new InvalidAmountException("Amount must be positive");
        }
        if (amount % 50 != 0) {
            throw new InvalidAmountException("Amount must be multiple of 50");
        }

        // Check daily limit
        double todayWithdrawal = dailyWithdrawals.getOrDefault(cardNumber, 0.0);
        if (todayWithdrawal + amount > DAILY_LIMIT) {
            throw new DailyLimitExceededException(DAILY_LIMIT);
        }

        // Check balance
        double balance = balances.getOrDefault(cardNumber, 0.0);
        if (amount > balance) {
            throw new InsufficientBalanceException(amount, balance);
        }

        // Process withdrawal
        balances.put(cardNumber, balance - amount);
        dailyWithdrawals.put(cardNumber, todayWithdrawal + amount);
        System.out.printf("Withdrawn $%.2f. New balance: $%.2f%n",
            amount, balances.get(cardNumber));
    }

    public static void main(String[] args) {
        ATM atm = new ATM();
        atm.balances.put("1234", 10000.0);

        try {
            atm.withdraw("1234", 500);
            atm.withdraw("1234", 6000);  // Exceeds daily limit
        } catch (InsufficientBalanceException e) {
            System.out.println("Balance error: " + e.getMessage());
        } catch (InvalidAmountException e) {
            System.out.println("Amount error: " + e.getMessage());
        } catch (CardBlockedException e) {
            System.out.println("Card error: " + e.getMessage());
        } catch (DailyLimitExceededException e) {
            System.out.println("Limit error: " + e.getMessage());
        }
    }
}
```
</details>

<details>
<summary>Bài 4: Retry Mechanism</summary>

```java
import java.util.concurrent.Callable;

public class RetryHelper {

    public static <T> T retry(Callable<T> task, int maxAttempts, long delayMs)
            throws Exception {

        Exception lastException = null;

        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                System.out.println("Attempt " + attempt + "...");
                return task.call();

            } catch (Exception e) {
                lastException = e;
                System.out.println("Attempt " + attempt + " failed: " + e.getMessage());

                if (attempt < maxAttempts) {
                    System.out.println("Retrying in " + delayMs + "ms...");
                    Thread.sleep(delayMs);
                }
            }
        }

        throw new Exception("All " + maxAttempts + " attempts failed", lastException);
    }

    public static void main(String[] args) {
        try {
            String result = retry(() -> {
                // Simulate failing operation
                if (Math.random() < 0.7) {
                    throw new RuntimeException("Network error");
                }
                return "Success!";
            }, 5, 1000);

            System.out.println("Result: " + result);

        } catch (Exception e) {
            System.out.println("Final failure: " + e.getMessage());
        }
    }
}
```
</details>

---

## Navigation

- [← Day 4: OOP Pillars](./day-04-oop-pillars.md)
- [Day 6: Strings & Wrappers →](./day-06-strings-wrappers.md)
