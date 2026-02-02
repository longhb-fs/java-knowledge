# Day 2: Operators & Control Flow

## Mục tiêu
- Các loại operators trong Java
- Câu lệnh điều kiện (if, switch)
- Vòng lặp (for, while, do-while)
- Arrays cơ bản

---

## 1. Operators (Toán tử)

### 1.1. Arithmetic Operators (Toán tử số học)

```java
int a = 10, b = 3;

System.out.println("a + b = " + (a + b));  // 13
System.out.println("a - b = " + (a - b));  // 7
System.out.println("a * b = " + (a * b));  // 30
System.out.println("a / b = " + (a / b));  // 3 (chia nguyên)
System.out.println("a % b = " + (a % b));  // 1 (phần dư)

// Chia số thực
double c = 10.0, d = 3.0;
System.out.println("c / d = " + (c / d));  // 3.333...

// Lưu ý chia nguyên
int result = 7 / 2;      // 3
double result2 = 7 / 2;  // 3.0 (vẫn là 3 vì 7 và 2 là int)
double result3 = 7.0 / 2; // 3.5
double result4 = (double) 7 / 2; // 3.5
```

### 1.2. Assignment Operators (Toán tử gán)

```java
int x = 10;

x += 5;   // x = x + 5;  → x = 15
x -= 3;   // x = x - 3;  → x = 12
x *= 2;   // x = x * 2;  → x = 24
x /= 4;   // x = x / 4;  → x = 6
x %= 4;   // x = x % 4;  → x = 2
```

### 1.3. Increment/Decrement Operators

```java
int a = 5;

// Post-increment: dùng giá trị trước, tăng sau
System.out.println(a++);  // In 5, sau đó a = 6

// Pre-increment: tăng trước, dùng sau
System.out.println(++a);  // a = 7, in 7

// Post-decrement
int b = 5;
System.out.println(b--);  // In 5, sau đó b = 4

// Pre-decrement
System.out.println(--b);  // b = 3, in 3

// Ví dụ phức tạp
int x = 5;
int y = x++ + ++x;
// x++ → dùng 5, x thành 6
// ++x → x thành 7, dùng 7
// y = 5 + 7 = 12
// x = 7
```

### 1.4. Comparison Operators (Toán tử so sánh)

```java
int a = 10, b = 20;

System.out.println(a == b);  // false (bằng)
System.out.println(a != b);  // true  (khác)
System.out.println(a > b);   // false (lớn hơn)
System.out.println(a < b);   // true  (nhỏ hơn)
System.out.println(a >= b);  // false (lớn hơn hoặc bằng)
System.out.println(a <= b);  // true  (nhỏ hơn hoặc bằng)

// So sánh String - PHẢI dùng equals()
String s1 = "hello";
String s2 = "hello";
String s3 = new String("hello");

System.out.println(s1 == s2);      // true (cùng reference trong String pool)
System.out.println(s1 == s3);      // false (khác reference)
System.out.println(s1.equals(s3)); // true (so sánh nội dung)
```

### 1.5. Logical Operators (Toán tử logic)

```java
boolean a = true, b = false;

// AND - cả hai đều true thì true
System.out.println(a && b);  // false
System.out.println(a & b);   // false (không short-circuit)

// OR - một trong hai true thì true
System.out.println(a || b);  // true
System.out.println(a | b);   // true (không short-circuit)

// NOT - đảo ngược
System.out.println(!a);      // false

// XOR - khác nhau thì true
System.out.println(a ^ b);   // true

// Short-circuit evaluation
int x = 5;
boolean result = (x > 10) && (++x > 5);
// x > 10 là false → không đánh giá ++x
// x vẫn = 5

boolean result2 = (x < 10) || (++x > 5);
// x < 10 là true → không đánh giá ++x
// x vẫn = 5
```

### 1.6. Ternary Operator (Toán tử 3 ngôi)

```java
// condition ? valueIfTrue : valueIfFalse

int age = 20;
String status = (age >= 18) ? "Adult" : "Minor";
System.out.println(status);  // Adult

// Nested ternary (không khuyến khích)
int score = 75;
String grade = (score >= 90) ? "A" :
               (score >= 80) ? "B" :
               (score >= 70) ? "C" :
               (score >= 60) ? "D" : "F";
System.out.println(grade);  // C
```

### 1.7. Bitwise Operators (Toán tử bitwise)

```java
int a = 5;  // 0101 in binary
int b = 3;  // 0011 in binary

System.out.println(a & b);  // 1  (0001) - AND
System.out.println(a | b);  // 7  (0111) - OR
System.out.println(a ^ b);  // 6  (0110) - XOR
System.out.println(~a);     // -6 - NOT

// Shift operators
System.out.println(a << 1); // 10 (1010) - left shift
System.out.println(a >> 1); // 2  (0010) - right shift

// Ứng dụng: nhân/chia cho 2^n nhanh
int x = 8;
System.out.println(x << 2); // 32 (8 * 4 = 8 * 2^2)
System.out.println(x >> 1); // 4  (8 / 2 = 8 / 2^1)
```

---

## 2. Conditional Statements (Câu lệnh điều kiện)

### 2.1. if Statement

```java
int age = 20;

// if đơn giản
if (age >= 18) {
    System.out.println("You are an adult");
}

// if-else
if (age >= 18) {
    System.out.println("Adult");
} else {
    System.out.println("Minor");
}

// if-else if-else
int score = 85;
if (score >= 90) {
    System.out.println("Grade: A");
} else if (score >= 80) {
    System.out.println("Grade: B");
} else if (score >= 70) {
    System.out.println("Grade: C");
} else if (score >= 60) {
    System.out.println("Grade: D");
} else {
    System.out.println("Grade: F");
}

// Nested if
int num = 15;
if (num > 0) {
    if (num % 2 == 0) {
        System.out.println("Positive even");
    } else {
        System.out.println("Positive odd");
    }
} else {
    System.out.println("Not positive");
}
```

### 2.2. switch Statement

```java
// Switch với int/char/String
int day = 3;

switch (day) {
    case 1:
        System.out.println("Monday");
        break;
    case 2:
        System.out.println("Tuesday");
        break;
    case 3:
        System.out.println("Wednesday");
        break;
    case 4:
        System.out.println("Thursday");
        break;
    case 5:
        System.out.println("Friday");
        break;
    case 6:
    case 7:
        System.out.println("Weekend");
        break;
    default:
        System.out.println("Invalid day");
}

// Switch với String (Java 7+)
String fruit = "apple";
switch (fruit) {
    case "apple":
        System.out.println("Red fruit");
        break;
    case "banana":
        System.out.println("Yellow fruit");
        break;
    default:
        System.out.println("Unknown fruit");
}
```

### 2.3. Switch Expression (Java 14+)

```java
// Switch expression - arrow syntax
int day = 3;
String dayName = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 3 -> "Wednesday";
    case 4 -> "Thursday";
    case 5 -> "Friday";
    case 6, 7 -> "Weekend";
    default -> "Invalid";
};
System.out.println(dayName);  // Wednesday

// Switch với yield (cho block code)
String result = switch (day) {
    case 1, 2, 3, 4, 5 -> {
        System.out.println("Processing weekday...");
        yield "Weekday";
    }
    case 6, 7 -> {
        System.out.println("Processing weekend...");
        yield "Weekend";
    }
    default -> "Invalid";
};
```

---

## 3. Loops (Vòng lặp)

### 3.1. for Loop

```java
// Basic for loop
for (int i = 0; i < 5; i++) {
    System.out.println("i = " + i);
}
// Output: 0, 1, 2, 3, 4

// Đếm ngược
for (int i = 5; i > 0; i--) {
    System.out.println(i);
}
// Output: 5, 4, 3, 2, 1

// Bước nhảy
for (int i = 0; i <= 10; i += 2) {
    System.out.println(i);  // 0, 2, 4, 6, 8, 10
}

// Nhiều biến
for (int i = 0, j = 10; i < j; i++, j--) {
    System.out.println("i=" + i + ", j=" + j);
}

// Infinite loop (cẩn thận!)
// for (;;) {
//     System.out.println("Forever...");
// }
```

### 3.2. while Loop

```java
// while - kiểm tra điều kiện trước
int count = 0;
while (count < 5) {
    System.out.println("Count: " + count);
    count++;
}

// Đọc input đến khi gặp "quit"
Scanner scanner = new Scanner(System.in);
String input = "";
while (!input.equals("quit")) {
    System.out.print("Enter command: ");
    input = scanner.nextLine();
    System.out.println("You entered: " + input);
}
```

### 3.3. do-while Loop

```java
// do-while - thực hiện ít nhất 1 lần
int count = 0;
do {
    System.out.println("Count: " + count);
    count++;
} while (count < 5);

// Ví dụ: menu
Scanner scanner = new Scanner(System.in);
int choice;
do {
    System.out.println("1. Option A");
    System.out.println("2. Option B");
    System.out.println("0. Exit");
    System.out.print("Choice: ");
    choice = scanner.nextInt();

    switch (choice) {
        case 1 -> System.out.println("Selected A");
        case 2 -> System.out.println("Selected B");
        case 0 -> System.out.println("Goodbye!");
        default -> System.out.println("Invalid choice");
    }
} while (choice != 0);
```

### 3.4. for-each Loop (Enhanced for)

```java
// Duyệt mảng
int[] numbers = {1, 2, 3, 4, 5};
for (int num : numbers) {
    System.out.println(num);
}

// Duyệt String array
String[] fruits = {"apple", "banana", "cherry"};
for (String fruit : fruits) {
    System.out.println(fruit);
}

// Không thể modify index
// Không thể đi ngược
```

### 3.5. break và continue

```java
// break - thoát khỏi vòng lặp
for (int i = 0; i < 10; i++) {
    if (i == 5) {
        break;  // Thoát khi i = 5
    }
    System.out.println(i);
}
// Output: 0, 1, 2, 3, 4

// continue - bỏ qua iteration hiện tại
for (int i = 0; i < 10; i++) {
    if (i % 2 == 0) {
        continue;  // Bỏ qua số chẵn
    }
    System.out.println(i);
}
// Output: 1, 3, 5, 7, 9

// Labeled break/continue - cho nested loops
outer:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (i == 1 && j == 1) {
            break outer;  // Thoát cả 2 vòng lặp
        }
        System.out.println("i=" + i + ", j=" + j);
    }
}
```

---

## 4. Arrays (Mảng)

### 4.1. Khai báo và khởi tạo

```java
// Cách 1: Khai báo rồi khởi tạo
int[] numbers;
numbers = new int[5];  // Mảng 5 phần tử, default = 0

// Cách 2: Khai báo và khởi tạo
int[] numbers2 = new int[5];

// Cách 3: Khởi tạo với giá trị
int[] numbers3 = {1, 2, 3, 4, 5};

// Cách 4: Tường minh
int[] numbers4 = new int[]{1, 2, 3, 4, 5};

// Default values
int[] intArr = new int[3];      // [0, 0, 0]
double[] doubleArr = new double[3];  // [0.0, 0.0, 0.0]
boolean[] boolArr = new boolean[3];  // [false, false, false]
String[] strArr = new String[3];     // [null, null, null]
```

### 4.2. Truy cập và modify

```java
int[] numbers = {10, 20, 30, 40, 50};

// Truy cập phần tử (index từ 0)
System.out.println(numbers[0]);  // 10
System.out.println(numbers[2]);  // 30
System.out.println(numbers[4]);  // 50

// Modify phần tử
numbers[1] = 25;
System.out.println(numbers[1]);  // 25

// Độ dài mảng
System.out.println(numbers.length);  // 5

// Lỗi ArrayIndexOutOfBoundsException
// System.out.println(numbers[5]);  // Error!
// System.out.println(numbers[-1]); // Error!
```

### 4.3. Duyệt mảng

```java
int[] numbers = {10, 20, 30, 40, 50};

// Cách 1: for loop
for (int i = 0; i < numbers.length; i++) {
    System.out.println("Index " + i + ": " + numbers[i]);
}

// Cách 2: for-each
for (int num : numbers) {
    System.out.println(num);
}

// Cách 3: while
int i = 0;
while (i < numbers.length) {
    System.out.println(numbers[i]);
    i++;
}
```

### 4.4. Mảng 2 chiều

```java
// Khai báo mảng 2D
int[][] matrix = new int[3][4];  // 3 hàng, 4 cột

// Khởi tạo với giá trị
int[][] matrix2 = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

// Truy cập
System.out.println(matrix2[0][0]);  // 1
System.out.println(matrix2[1][2]);  // 6

// Duyệt mảng 2D
for (int row = 0; row < matrix2.length; row++) {
    for (int col = 0; col < matrix2[row].length; col++) {
        System.out.print(matrix2[row][col] + " ");
    }
    System.out.println();
}

// Jagged array (mảng không đều)
int[][] jagged = {
    {1, 2},
    {3, 4, 5},
    {6, 7, 8, 9}
};
```

### 4.5. Array utilities

```java
import java.util.Arrays;

int[] numbers = {5, 2, 8, 1, 9};

// In mảng
System.out.println(Arrays.toString(numbers));  // [5, 2, 8, 1, 9]

// Sắp xếp
Arrays.sort(numbers);
System.out.println(Arrays.toString(numbers));  // [1, 2, 5, 8, 9]

// Tìm kiếm (mảng phải được sort)
int index = Arrays.binarySearch(numbers, 5);
System.out.println("Found at index: " + index);  // 2

// Fill
int[] arr = new int[5];
Arrays.fill(arr, 10);
System.out.println(Arrays.toString(arr));  // [10, 10, 10, 10, 10]

// Copy
int[] copy = Arrays.copyOf(numbers, 3);  // Copy 3 phần tử đầu
int[] rangeCopy = Arrays.copyOfRange(numbers, 1, 4);  // Copy index 1-3

// So sánh
int[] a = {1, 2, 3};
int[] b = {1, 2, 3};
System.out.println(Arrays.equals(a, b));  // true
```

---

## 5. Bài tập thực hành

### Bài 1: Máy tính đơn giản
Viết chương trình máy tính với các phép: +, -, *, /, %
- Nhập 2 số và phép tính
- Sử dụng switch để xử lý
- Xử lý chia cho 0

```
Enter first number: 10
Enter operator (+, -, *, /, %): /
Enter second number: 3
Result: 10 / 3 = 3.33
```

---

### Bài 2: Kiểm tra số nguyên tố
Viết hàm kiểm tra số nguyên tố và in các số nguyên tố từ 1 đến n.

```
Enter n: 20
Prime numbers from 1 to 20:
2 3 5 7 11 13 17 19
Total: 8 primes
```

---

### Bài 3: Bảng cửu chương
In bảng cửu chương từ 2 đến 9 theo format đẹp.

```
=== Multiplication Table ===

   |  2    3    4    5    6    7    8    9
---+----------------------------------------
 1 |  2    3    4    5    6    7    8    9
 2 |  4    6    8   10   12   14   16   18
...
10 | 20   30   40   50   60   70   80   90
```

---

### Bài 4: Tam giác sao
Vẽ các loại tam giác sao với n dòng.

```
Enter n: 5
Enter type (1-4): 1

Type 1 (Right triangle):
*
**
***
****
*****

Type 2 (Inverted):
*****
****
***
**
*

Type 3 (Pyramid):
    *
   ***
  *****
 *******
*********

Type 4 (Diamond):
    *
   ***
  *****
 *******
*********
 *******
  *****
   ***
    *
```

---

### Bài 5: Thao tác mảng
Viết chương trình với các chức năng:
1. Nhập mảng n phần tử
2. In mảng
3. Tìm min, max
4. Tính tổng, trung bình
5. Đảo ngược mảng
6. Sắp xếp mảng (không dùng Arrays.sort)

```
Enter array size: 5
Enter 5 elements:
Element 0: 3
Element 1: 1
Element 2: 4
Element 3: 1
Element 4: 5

Array: [3, 1, 4, 1, 5]
Min: 1
Max: 5
Sum: 14
Average: 2.8
Reversed: [5, 1, 4, 1, 3]
Sorted: [1, 1, 3, 4, 5]
```

---

### Bài 6: FizzBuzz
In các số từ 1 đến n:
- Chia hết cho 3: in "Fizz"
- Chia hết cho 5: in "Buzz"
- Chia hết cho cả 3 và 5: in "FizzBuzz"
- Còn lại: in số đó

---

### Bài 7: Guess the Number
Tạo game đoán số:
- Random số từ 1-100
- Người chơi đoán, hint "Too high" hoặc "Too low"
- Đếm số lần đoán

---

## 6. Đáp án tham khảo

<details>
<summary>Bài 1: Máy tính</summary>

```java
import java.util.Scanner;

public class Calculator {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.print("Enter first number: ");
        double num1 = scanner.nextDouble();

        System.out.print("Enter operator (+, -, *, /, %): ");
        char operator = scanner.next().charAt(0);

        System.out.print("Enter second number: ");
        double num2 = scanner.nextDouble();

        double result;
        switch (operator) {
            case '+':
                result = num1 + num2;
                break;
            case '-':
                result = num1 - num2;
                break;
            case '*':
                result = num1 * num2;
                break;
            case '/':
                if (num2 == 0) {
                    System.out.println("Error: Division by zero!");
                    return;
                }
                result = num1 / num2;
                break;
            case '%':
                if (num2 == 0) {
                    System.out.println("Error: Division by zero!");
                    return;
                }
                result = num1 % num2;
                break;
            default:
                System.out.println("Invalid operator!");
                return;
        }

        System.out.printf("Result: %.2f %c %.2f = %.2f%n", num1, operator, num2, result);
        scanner.close();
    }
}
```
</details>

<details>
<summary>Bài 2: Số nguyên tố</summary>

```java
import java.util.Scanner;

public class PrimeNumbers {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.print("Enter n: ");
        int n = scanner.nextInt();

        System.out.println("Prime numbers from 1 to " + n + ":");

        int count = 0;
        for (int num = 2; num <= n; num++) {
            if (isPrime(num)) {
                System.out.print(num + " ");
                count++;
            }
        }

        System.out.println("\nTotal: " + count + " primes");
        scanner.close();
    }

    public static boolean isPrime(int num) {
        if (num < 2) return false;
        if (num == 2) return true;
        if (num % 2 == 0) return false;

        for (int i = 3; i <= Math.sqrt(num); i += 2) {
            if (num % i == 0) {
                return false;
            }
        }
        return true;
    }
}
```
</details>

<details>
<summary>Bài 4: Tam giác sao (Pyramid)</summary>

```java
import java.util.Scanner;

public class StarPatterns {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.print("Enter n: ");
        int n = scanner.nextInt();

        // Type 3: Pyramid
        System.out.println("Pyramid:");
        for (int i = 1; i <= n; i++) {
            // In spaces
            for (int j = 0; j < n - i; j++) {
                System.out.print(" ");
            }
            // In stars
            for (int k = 0; k < 2 * i - 1; k++) {
                System.out.print("*");
            }
            System.out.println();
        }

        scanner.close();
    }
}
```
</details>

<details>
<summary>Bài 7: Guess the Number</summary>

```java
import java.util.Random;
import java.util.Scanner;

public class GuessNumber {
    public static void main(String[] args) {
        Random random = new Random();
        Scanner scanner = new Scanner(System.in);

        int secretNumber = random.nextInt(100) + 1;  // 1-100
        int guess;
        int attempts = 0;

        System.out.println("I'm thinking of a number between 1 and 100.");

        do {
            System.out.print("Your guess: ");
            guess = scanner.nextInt();
            attempts++;

            if (guess < secretNumber) {
                System.out.println("Too low!");
            } else if (guess > secretNumber) {
                System.out.println("Too high!");
            } else {
                System.out.println("Congratulations! You got it in " + attempts + " attempts!");
            }
        } while (guess != secretNumber);

        scanner.close();
    }
}
```
</details>

---

## Navigation

- [← Day 1: Setup & Syntax](./day-01-setup-syntax.md)
- [Day 3: OOP Basics →](./day-03-oop-basics.md)
