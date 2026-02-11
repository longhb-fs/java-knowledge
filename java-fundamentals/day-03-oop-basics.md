# Day 3: OOP Basics (Lập trình hướng đối tượng — Cơ bản)

## Mục tiêu hôm nay

Sau ngày hôm nay, bạn sẽ:
- Hiểu **Class** (lớp) và **Object** (đối tượng) — 2 khái niệm nền tảng nhất của Java
- Biết cách dùng **Constructor** (hàm khởi tạo) để tạo object
- Viết được **Methods** (phương thức) — hành động của object
- Hiểu **Access Modifiers** (phạm vi truy cập): public, private, protected
- Biết dùng từ khóa `this` và **static**

### Tại sao cần học OOP?

**OOP (Object-Oriented Programming)** = Lập trình hướng đối tượng. Đây là cách tổ chức code **giống thế giới thực**: mọi thứ đều là "đối tượng" có đặc điểm và hành động.

> 💡 **Ví dụ đời thường**: Trong quản lý nhân sự:
> - **Nhân viên** là đối tượng, có đặc điểm (tên, tuổi, lương) và hành động (làm việc, nghỉ phép)
> - **Phòng ban** là đối tượng, có đặc điểm (tên phòng, số người) và hành động (thêm nhân viên, tính lương)
>
> OOP giúp code **dễ hiểu**, **dễ mở rộng**, **dễ bảo trì**. Không dùng OOP → code dài, khó sửa, dễ bug.

---

## 1. Class và Object — Bản thiết kế và Sản phẩm

### 1.1. Khái niệm

```
Class (Lớp)  = BẢN THIẾT KẾ — mô tả "cái gì đó" gồm những gì
Object (Đối tượng) = SẢN PHẨM THỰC TẾ — được tạo ra từ bản thiết kế

Ví dụ thực tế:
┌────────────────┐          ┌──────────────────────┐
│ Class "XeHoi"  │          │ Object "xeCuaToi"    │
│ (bản thiết kế) │  ──tạo─→ │ (chiếc xe thực tế)   │
│                │          │                      │
│ - thuongHieu   │          │ - thuongHieu: Toyota  │
│ - mauSac       │          │ - mauSac: Đen         │
│ - namSanXuat   │          │ - namSanXuat: 2024    │
│                │          │                      │
│ + khoidong()   │          │ + khoidong()          │
│ + dungLai()    │          │ + dungLai()           │
└────────────────┘          └──────────────────────┘

Từ 1 bản thiết kế, có thể tạo NHIỀU sản phẩm:
Class XeHoi → Object xe1 (Toyota Đen), Object xe2 (Honda Trắng), Object xe3 (BMW Xanh)
```

### 1.2. Định nghĩa Class — viết "bản thiết kế"

```java
// File: XeHoi.java
// Class = bản thiết kế cho một chiếc xe

public class XeHoi {

    // ── FIELDS (Thuộc tính) ── là "đặc điểm" của xe
    String thuongHieu;   // Thương hiệu: Toyota, Honda...
    String mauSac;       // Màu sắc: Đen, Trắng...
    int namSanXuat;      // Năm sản xuất: 2024
    double gia;          // Giá: 500000000

    // ── METHODS (Phương thức) ── là "hành động" xe có thể làm
    void khoiDong() {
        System.out.println(thuongHieu + " đang khởi động...");
    }

    void dungLai() {
        System.out.println(thuongHieu + " đã dừng.");
    }

    void hienThiThongTin() {
        System.out.println("Thương hiệu: " + thuongHieu);
        System.out.println("Màu sắc: " + mauSac);
        System.out.println("Năm SX: " + namSanXuat);
        System.out.printf("Giá: %,.0f VND%n", gia);
    }
}
```

> 💡 **Thuật ngữ quan trọng**:
> - **Field** (trường/thuộc tính) = đặc điểm, dữ liệu của object → `thuongHieu`, `mauSac`
> - **Method** (phương thức) = hành động object có thể làm → `khoiDong()`, `dungLai()`
> - Field + Method gọi chung là **Members** (thành viên) của class

### 1.3. Tạo Object — "sản xuất" sản phẩm từ bản thiết kế

```java
public class Main {
    public static void main(String[] args) {
        // Tạo object (đối tượng) từ class XeHoi
        // Cú pháp: TenClass tenBien = new TenClass();
        XeHoi xeCuaToi = new XeHoi();
        //  │      │        │    └── Gọi Constructor (hàm khởi tạo) → tạo object trong bộ nhớ
        //  │      │        └── "new" = từ khóa tạo object mới
        //  │      └── Tên biến (đặt tùy ý)
        //  └── Kiểu dữ liệu = tên Class

        // Gán giá trị cho các field
        xeCuaToi.thuongHieu = "Toyota";
        xeCuaToi.mauSac = "Đen";
        xeCuaToi.namSanXuat = 2024;
        xeCuaToi.gia = 500000000;

        // Gọi methods
        xeCuaToi.hienThiThongTin();
        xeCuaToi.khoiDong();
        xeCuaToi.dungLai();

        // Tạo object thứ 2 — từ CÙNG class, nhưng dữ liệu KHÁC
        XeHoi xeBanToi = new XeHoi();
        xeBanToi.thuongHieu = "Honda";
        xeBanToi.mauSac = "Trắng";
        xeBanToi.namSanXuat = 2023;
        xeBanToi.gia = 400000000;

        xeBanToi.hienThiThongTin();
    }
}
```

> 🔥 **Nhớ**: `new XeHoi()` = "Tạo 1 chiếc xe mới theo bản thiết kế XeHoi". Mỗi lần `new` = tạo 1 object MỚI, độc lập, chiếm bộ nhớ riêng.

---

## 2. Constructor (Hàm khởi tạo) — "quy trình tạo sản phẩm"

### Tại sao cần Constructor?

Ở ví dụ trên, ta phải gán từng field một (`xeCuaToi.thuongHieu = "Toyota"`). Rất mệt! Constructor giúp **gán giá trị ngay khi tạo object**.

> 💡 **Ví dụ đời thường**: Đặt hàng online — bạn điền tên, địa chỉ, SĐT ngay khi đặt (constructor), không phải đặt đơn trống rồi gọi lại bổ sung từng thông tin.

### 2.1. Default Constructor (Constructor mặc định)

```java
public class NguoiDung {
    String ten;
    int tuoi;

    // Default Constructor — không có tham số
    // Java TỰ ĐỘNG tạo nếu bạn không viết constructor nào
    public NguoiDung() {
        ten = "Chưa đặt tên";
        tuoi = 0;
    }
}

// Sử dụng:
NguoiDung user = new NguoiDung();  // Gọi default constructor
System.out.println(user.ten);      // "Chưa đặt tên"
```

### 2.2. Parameterized Constructor (Constructor có tham số)

```java
public class NguoiDung {
    String ten;
    int tuoi;

    // Constructor CÓ tham số — nhận giá trị khi tạo object
    public NguoiDung(String ten, int tuoi) {
        this.ten = ten;    // this.ten = field, ten = tham số truyền vào
        this.tuoi = tuoi;  // this.tuoi = field, tuoi = tham số truyền vào
    }
}

// Sử dụng — gán giá trị ngay khi tạo, tiện hơn nhiều!
NguoiDung user = new NguoiDung("Nguyễn Văn A", 25);
System.out.println(user.ten);   // "Nguyễn Văn A"
System.out.println(user.tuoi);  // 25
```

> 🔥 **Quy tắc quan trọng**:
> - Constructor có **CÙNG TÊN** với class
> - Constructor **KHÔNG CÓ** kiểu trả về (không có `void`, `int`...)
> - Nếu bạn viết BẤT KỲ constructor nào → Java KHÔNG TỰ tạo default constructor nữa

### 2.3. Multiple Constructors (Nạp chồng Constructor)

Một class có thể có NHIỀU constructor với tham số khác nhau:

```java
public class NguoiDung {
    String ten;
    int tuoi;
    String email;

    // Constructor 1: Không tham số
    public NguoiDung() {
        this.ten = "Chưa đặt tên";
        this.tuoi = 0;
        this.email = "";
    }

    // Constructor 2: Chỉ có tên
    public NguoiDung(String ten) {
        this.ten = ten;
        this.tuoi = 0;
        this.email = "";
    }

    // Constructor 3: Đầy đủ
    public NguoiDung(String ten, int tuoi, String email) {
        this.ten = ten;
        this.tuoi = tuoi;
        this.email = email;
    }
}

// Java tự biết gọi constructor nào dựa vào tham số:
NguoiDung u1 = new NguoiDung();                          // Gọi Constructor 1
NguoiDung u2 = new NguoiDung("An");                      // Gọi Constructor 2
NguoiDung u3 = new NguoiDung("An", 25, "an@email.com");  // Gọi Constructor 3
```

### 2.4. Constructor Chaining — "gọi chéo" constructor

Để tránh lặp code, constructor này có thể gọi constructor khác bằng `this()`:

```java
public class NguoiDung {
    String ten;
    int tuoi;
    String email;

    public NguoiDung() {
        this("Chưa đặt tên", 0, "");  // Gọi constructor 3 tham số
    }

    public NguoiDung(String ten) {
        this(ten, 0, "");  // Gọi constructor 3 tham số
    }

    public NguoiDung(String ten, int tuoi, String email) {
        // Constructor "gốc" — mọi constructor khác gọi về đây
        this.ten = ten;
        this.tuoi = tuoi;
        this.email = email;
    }
}
```

> 💡 **Lợi ích**: Code gán giá trị chỉ viết 1 lần (trong constructor đầy đủ nhất). Sửa 1 chỗ = sửa hết.

---

## 3. Methods (Phương thức) — "hành động" của Object

### 3.1. Cấu trúc Method

```java
// phamViTruyCap  kiểuTrảVề  tênMethod(thamSố) {
//    ...code...
//    return giáTrị;  // nếu có kiểu trả về
// }

public int tinhTong(int a, int b) {
//  │     │     │          └── Tham số (parameters) — dữ liệu đầu vào
//  │     │     └── Tên method (dùng camelCase)
//  │     └── Kiểu trả về: int, String, void (không trả gì)...
//  └── Phạm vi: public, private, protected
    return a + b;  // Trả về kết quả
}
```

### 3.2. Các loại Methods

```java
public class MayTinh {

    // ① Method KHÔNG trả về giá trị (void) — chỉ thực hiện hành động
    public void inLoiChao() {
        System.out.println("Xin chào!");
        // Không cần return
    }

    // ② Method CÓ trả về giá trị — tính toán rồi trả kết quả
    public int cong(int a, int b) {
        return a + b;  // Trả về tổng
    }

    // ③ Method với varargs (số tham số thay đổi)
    // double... = "nhận nhiều số double, không biết trước bao nhiêu"
    public double tinhTrungBinh(double... cacSo) {
        double tong = 0;
        for (double so : cacSo) {
            tong += so;
        }
        return tong / cacSo.length;
    }
}

// Sử dụng:
MayTinh mt = new MayTinh();
mt.inLoiChao();                           // "Xin chào!"
int ketQua = mt.cong(3, 5);              // ketQua = 8
double tb = mt.tinhTrungBinh(8, 9, 7, 10); // tb = 8.5 (truyền bao nhiêu số cũng được)
```

### 3.3. Method Overloading (Nạp chồng phương thức) — cùng tên, khác tham số

```java
public class PhepTinh {
    // Cùng tên "cong", nhưng KHÁC KIỂU/SỐ LƯỢNG tham số
    // Java tự biết gọi method nào dựa vào tham số truyền vào

    public int cong(int a, int b) {          // 2 số nguyên
        return a + b;
    }

    public int cong(int a, int b, int c) {   // 3 số nguyên
        return a + b + c;
    }

    public double cong(double a, double b) { // 2 số thực
        return a + b;
    }

    public String cong(String a, String b) { // 2 chuỗi
        return a + b;                         // Nối chuỗi
    }
}

// Sử dụng:
PhepTinh pt = new PhepTinh();
System.out.println(pt.cong(1, 2));           // 3       → gọi method 2 int
System.out.println(pt.cong(1, 2, 3));        // 6       → gọi method 3 int
System.out.println(pt.cong(1.5, 2.5));       // 4.0     → gọi method 2 double
System.out.println(pt.cong("Xin", " chào")); // Xin chào → gọi method 2 String
```

> 💡 **Mẹo nhớ**: Overloading = "CÙNG TÊN, KHÁC tham số". Java phân biệt bằng kiểu và số lượng tham số, KHÔNG phân biệt bằng kiểu trả về.

### 3.4. Pass by Value — Java truyền tham số kiểu gì?

> 🔥 **Câu hỏi phỏng vấn kinh điển**: Java là "pass by value" hay "pass by reference"?
> **Đáp án: LUÔN là pass by value!** Nhưng cần hiểu rõ:

```java
public class TruyenThamSo {

    // ① Primitive: truyền BẢN SAO giá trị → thay đổi KHÔNG ảnh hưởng biến gốc
    public void thayDoiSo(int x) {
        x = 100;  // Chỉ thay đổi bản sao, biến gốc không đổi
    }

    // ② Object: truyền BẢN SAO địa chỉ → thay đổi NỘI DUNG sẽ ảnh hưởng
    public void thayDoiTen(NguoiDung nd) {
        nd.ten = "Đã sửa";  // Thay đổi nội dung → ảnh hưởng object gốc!
    }

    // ③ Object: gán lại biến → KHÔNG ảnh hưởng
    public void ganLai(NguoiDung nd) {
        nd = new NguoiDung("Mới");  // Gán lại biến cục bộ → object gốc KHÔNG đổi
    }

    public static void main(String[] args) {
        TruyenThamSo demo = new TruyenThamSo();

        // Test ①: Primitive
        int so = 10;
        demo.thayDoiSo(so);
        System.out.println(so);  // Vẫn là 10! (bản sao bị thay, gốc không đổi)

        // Test ②: Thay đổi nội dung object
        NguoiDung user = new NguoiDung("An", 25, "");
        demo.thayDoiTen(user);
        System.out.println(user.ten);  // "Đã sửa" (nội dung bị thay đổi)

        // Test ③: Gán lại object
        demo.ganLai(user);
        System.out.println(user.ten);  // Vẫn "Đã sửa" (gán lại không ảnh hưởng gốc)
    }
}
```

> 💡 **Ví dụ đời thường**: Bạn cho bạn bè **bản photo chìa khóa nhà** (pass by value of reference). Bạn bè có thể **vào nhà sửa đồ** (thay đổi nội dung object). Nhưng nếu bạn bè **vứt bản photo và lấy chìa khóa nhà khác** (gán lại), nhà bạn vẫn nguyên.

---

## 4. Access Modifiers (Phạm vi truy cập) — "ai được phép dùng gì?"

### Tại sao cần Access Modifiers?

Giống như nhà bạn có **phòng khách** (ai cũng vào được) và **phòng ngủ** (chỉ gia đình). Code cũng cần phân quyền: cái nào public (công khai), cái nào private (riêng tư).

### 4.1. Bảng phạm vi truy cập

| Modifier | Trong class | Cùng package | Class con | Bất kỳ đâu |
|----------|:-----------:|:------------:|:---------:|:-----------:|
| `public` | ✅ | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| (default — không viết gì) | ✅ | ✅ | ❌ | ❌ |
| `private` | ✅ | ❌ | ❌ | ❌ |

> 💡 **Mẹo nhớ** (từ rộng → hẹp):
> - `public` = "Công khai" — ai cũng xem được (như bảng tin)
> - `protected` = "Bảo vệ" — gia đình + họ hàng (cùng package + class con)
> - `(default)` = "Nội bộ" — chỉ hàng xóm cùng khu (cùng package)
> - `private` = "Riêng tư" — chỉ mình tôi (trong class)

### 4.2. Ví dụ thực tế: Tài khoản ngân hàng

```java
public class TaiKhoanNganHang {
    // private = riêng tư → không ai bên ngoài class này truy cập trực tiếp được
    private String soTaiKhoan;
    private double soDu;

    // public constructor — ai cũng có thể tạo tài khoản
    public TaiKhoanNganHang(String soTaiKhoan, double soDuBanDau) {
        this.soTaiKhoan = soTaiKhoan;
        this.soDu = soDuBanDau;
    }

    // public getter — cho phép XEM số dư (nhưng không sửa trực tiếp)
    public double getSoDu() {
        return soDu;
    }

    // public method — nạp tiền có KIỂM TRA (validation)
    public void napTien(double soTien) {
        if (soTien > 0) {
            soDu += soTien;
            System.out.printf("Đã nạp %,.0f VND. Số dư: %,.0f VND%n", soTien, soDu);
        } else {
            System.out.println("Số tiền không hợp lệ!");
        }
    }

    // public method — rút tiền có KIỂM TRA
    public boolean rutTien(double soTien) {
        if (soTien > 0 && soTien <= soDu) {
            soDu -= soTien;
            System.out.printf("Đã rút %,.0f VND. Số dư: %,.0f VND%n", soTien, soDu);
            return true;
        }
        System.out.println("Rút tiền thất bại! Số dư không đủ.");
        return false;
    }
}

// Sử dụng:
TaiKhoanNganHang tk = new TaiKhoanNganHang("001", 10000000);
// tk.soDu = -999999;  // ❌ LỖI BIÊN DỊCH! soDu là private, không truy cập được
tk.napTien(5000000);    // ✅ OK — thông qua method public có validation
tk.rutTien(20000000);   // ✅ OK — method sẽ kiểm tra và từ chối (số dư không đủ)
```

> 🔥 **Nguyên tắc vàng**: Fields luôn để `private`, truy cập qua `public` getter/setter. Đây gọi là **Encapsulation** (Đóng gói) — 1 trong 4 trụ cột OOP (sẽ học kỹ ở Day 4).

---

## 5. `this` Keyword — "chính tôi"

### Tại sao cần `this`?

`this` = "**chính object hiện tại**". Dùng khi tên tham số TRÙNG với tên field.

### 5.1. Phân biệt field và parameter

```java
public class NguoiDung {
    private String ten;   // ← field (thuộc tính của object)
    private int tuoi;

    public NguoiDung(String ten, int tuoi) {
        // "ten" ở đây là tham số truyền vào
        // "this.ten" là field của object
        this.ten = ten;    // Gán tham số "ten" cho field "this.ten"
        this.tuoi = tuoi;

        // Nếu KHÔNG dùng this:
        // ten = ten;  // ❌ Java hiểu = gán tham số cho chính nó → field không đổi!
    }

    public void setTen(String ten) {
        this.ten = ten;  // this.ten = field, ten = tham số mới
    }
}
```

### 5.2. Gọi constructor khác

```java
public class HinhChuNhat {
    private int chieuRong;
    private int chieuCao;

    // Constructor hình vuông (1 tham số)
    public HinhChuNhat(int canh) {
        this(canh, canh);  // Gọi constructor 2 tham số, truyền canh cho cả 2
    }

    // Constructor đầy đủ (2 tham số)
    public HinhChuNhat(int chieuRong, int chieuCao) {
        this.chieuRong = chieuRong;
        this.chieuCao = chieuCao;
    }
}

// Sử dụng:
HinhChuNhat hv = new HinhChuNhat(5);     // Hình vuông 5x5
HinhChuNhat hcn = new HinhChuNhat(4, 6); // Hình chữ nhật 4x6
```

### 5.3. Method Chaining — "nối chuỗi lệnh"

Khi method `return this`, ta có thể gọi liên tiếp nhiều method trên 1 dòng:

```java
public class TaoTinNhan {
    private String nguoiGui = "";
    private String nguoiNhan = "";
    private String noiDung = "";

    public TaoTinNhan tuNguoi(String nguoiGui) {
        this.nguoiGui = nguoiGui;
        return this;  // Trả về chính object này → có thể gọi method tiếp
    }

    public TaoTinNhan denNguoi(String nguoiNhan) {
        this.nguoiNhan = nguoiNhan;
        return this;
    }

    public TaoTinNhan noiDung(String noiDung) {
        this.noiDung = noiDung;
        return this;
    }

    public void gui() {
        System.out.printf("Từ: %s → Đến: %s%nNội dung: %s%n", nguoiGui, nguoiNhan, noiDung);
    }
}

// Sử dụng — gọi liên tiếp trên 1 dòng, rất gọn!
new TaoTinNhan()
    .tuNguoi("An")
    .denNguoi("Bình")
    .noiDung("Chào bạn!")
    .gui();
// Kết quả: Từ: An → Đến: Bình
//          Nội dung: Chào bạn!
```

> 💡 Pattern này rất phổ biến trong thực tế: `StringBuilder`, `Stream API`, và các thư viện như Lombok Builder.

---

## 6. Static Members — "thuộc về Class, không thuộc về Object"

### Tại sao cần static?

Đôi khi dữ liệu/hành động thuộc về **cả class**, không phải từng object riêng lẻ.

> 💡 **Ví dụ đời thường**:
> - `static` = "Số lượng xe Toyota đã sản xuất" — thuộc về hãng Toyota, không thuộc từng chiếc xe
> - Không static = "Màu sắc của chiếc xe này" — mỗi xe có màu riêng

### 6.1. Static Fields — biến dùng chung cho TẤT CẢ object

```java
public class BoiDem {
    // Static field — DÙNG CHUNG, tất cả object chia sẻ 1 giá trị
    private static int tongSo = 0;

    // Instance field — RIÊNG mỗi object
    private int id;

    public BoiDem() {
        tongSo++;         // Tăng biến chung
        this.id = tongSo; // Gán id riêng cho object này
    }

    // Static method — gọi qua TÊN CLASS, không cần tạo object
    public static int getTongSo() {
        return tongSo;
    }

    public int getId() {
        return id;
    }
}

// Sử dụng:
BoiDem a = new BoiDem();  // tongSo = 1, a.id = 1
BoiDem b = new BoiDem();  // tongSo = 2, b.id = 2
BoiDem c = new BoiDem();  // tongSo = 3, c.id = 3

System.out.println(BoiDem.getTongSo());  // 3 — gọi qua TÊN CLASS
System.out.println(a.getId());            // 1
System.out.println(b.getId());            // 2
```

```
Bộ nhớ:
┌──────────────────────┐
│ Class BoiDem (shared) │
│ tongSo = 3            │
└──────────────────────┘
     ↑        ↑        ↑
┌────────┐ ┌────────┐ ┌────────┐
│ a      │ │ b      │ │ c      │
│ id = 1 │ │ id = 2 │ │ id = 3 │
└────────┘ └────────┘ └────────┘
3 object, mỗi cái có id riêng, nhưng chia sẻ 1 tongSo
```

### 6.2. Static Methods — gọi không cần tạo object

```java
public class ToanHoc {
    // Static method — gọi trực tiếp qua tên class
    public static int cong(int a, int b) {
        return a + b;
    }

    public static int max(int a, int b) {
        return (a > b) ? a : b;
    }

    public static double luiThua(double co, int mu) {
        double ketQua = 1;
        for (int i = 0; i < mu; i++) {
            ketQua *= co;
        }
        return ketQua;
    }
}

// Sử dụng — gọi qua TÊN CLASS, KHÔNG CẦN new
int tong = ToanHoc.cong(5, 3);           // 8
int soLon = ToanHoc.max(10, 20);         // 20
double ketQua = ToanHoc.luiThua(2, 10);  // 1024.0
```

> 💡 **Bạn đã dùng static mà không biết**: `Math.PI`, `Math.sqrt()`, `Arrays.sort()` — toàn static methods!

### 6.3. Static vs Instance — khác nhau thế nào?

```java
public class Demo {
    private int bienRieng = 10;         // Instance — mỗi object có riêng
    private static int bienChung = 20;  // Static — dùng chung

    // Instance method — CÓ THỂ truy cập cả 2
    public void methodRieng() {
        System.out.println(bienRieng);   // ✅ OK
        System.out.println(bienChung);   // ✅ OK
    }

    // Static method — CHỈ truy cập static
    public static void methodChung() {
        // System.out.println(bienRieng); // ❌ LỖI! Static không truy cập instance
        System.out.println(bienChung);    // ✅ OK
    }
}
```

> 🔥 **Quy tắc**: Static method KHÔNG THỂ truy cập instance members. Vì static thuộc về Class (tồn tại trước khi có object), còn instance thuộc về Object (chỉ tồn tại khi `new`).

---

## 7. Getters và Setters — "cổng vào" cho private fields

### Tại sao cần Getter/Setter?

Fields nên để `private` (đóng gói). Getter cho phép **đọc**, Setter cho phép **ghi có kiểm tra**.

```java
public class SinhVien {
    private String ten;
    private int tuoi;
    private double diemTB;

    public SinhVien(String ten, int tuoi, double diemTB) {
        this.setTen(ten);
        this.setTuoi(tuoi);
        this.setDiemTB(diemTB);
    }

    // === GETTERS — cho phép ĐỌC ===
    public String getTen() { return ten; }
    public int getTuoi() { return tuoi; }
    public double getDiemTB() { return diemTB; }

    // === SETTERS — cho phép GHI có KIỂM TRA ===
    public void setTen(String ten) {
        if (ten != null && !ten.trim().isEmpty()) {
            this.ten = ten;
        } else {
            System.out.println("Tên không hợp lệ!");
        }
    }

    public void setTuoi(int tuoi) {
        if (tuoi >= 0 && tuoi <= 100) {
            this.tuoi = tuoi;
        } else {
            System.out.println("Tuổi phải từ 0-100!");
        }
    }

    public void setDiemTB(double diemTB) {
        if (diemTB >= 0.0 && diemTB <= 10.0) {
            this.diemTB = diemTB;
        } else {
            System.out.println("Điểm phải từ 0-10!");
        }
    }

    // Method nghiệp vụ
    public boolean datYeuCau() {
        return diemTB >= 5.0;
    }
}

// Sử dụng:
SinhVien sv = new SinhVien("An", 20, 8.5);
sv.setTuoi(-5);     // "Tuổi phải từ 0-100!" → không thay đổi
sv.setDiemTB(15.0); // "Điểm phải từ 0-10!" → không thay đổi
System.out.println(sv.getTuoi());  // Vẫn 20 (giá trị cũ, vì set bị từ chối)
```

> 💡 **Phím tắt trong IntelliJ**: Nhấn **Alt + Insert** (hoặc chuột phải → Generate) → chọn "Getter and Setter" → IntelliJ tự sinh code!

---

## 8. Tóm tắt cuối ngày

| Khái niệm | Giải thích | Ví dụ |
|-----------|-----------|-------|
| Class | "Bản thiết kế" mô tả đặc điểm + hành động | `class XeHoi { ... }` |
| Object | "Sản phẩm" tạo từ bản thiết kế | `XeHoi xe = new XeHoi();` |
| Field | Đặc điểm/dữ liệu của object | `String ten;` |
| Method | Hành động object có thể làm | `void khoiDong() { ... }` |
| Constructor | Hàm khởi tạo — gán giá trị khi tạo object | Cùng tên class, không có kiểu trả về |
| `this` | "Chính object hiện tại" | `this.ten = ten;` |
| `private` | Chỉ truy cập trong class | Fields nên private |
| `public` | Ai cũng truy cập được | Methods, constructors |
| `static` | Thuộc về Class, dùng chung | `static int count;` |
| Getter/Setter | Đọc/ghi có kiểm tra cho private fields | `getTen()`, `setTen()` |
| Overloading | Cùng tên, khác tham số | Nhiều constructor / method |
| Pass by value | Java luôn truyền bản sao | Primitive: copy giá trị, Object: copy địa chỉ |

---

## 9. Bài tập thực hành

### Bài 1: Class SinhVien
Tạo class `SinhVien` với:
- Fields: maSV, ten, tuoi, diemTB
- Constructors: default + đầy đủ
- Getters/Setters với validation (tuổi 16-60, điểm 0-10)
- Method `hienThiThongTin()` và `datYeuCau()` (điểm >= 5.0)

```java
// Kết quả mong muốn:
SinhVien sv = new SinhVien("SV001", "Nguyễn Văn A", 20, 8.5);
sv.hienThiThongTin();
// === Thông tin Sinh viên ===
// Mã SV: SV001
// Tên: Nguyễn Văn A
// Tuổi: 20
// Điểm TB: 8.50
// Đạt yêu cầu: Có
```

---

### Bài 2: Class TaiKhoanNganHang
Tạo class `TaiKhoanNganHang` với:
- Fields: soTK, chuTK, soDu
- Static field: tongSoTK (đếm số tài khoản đã tạo)
- Methods: napTien(), rutTien(), chuyenTien(TaiKhoanNganHang nguoiNhan, double soTien)
- Validation: số tiền > 0, rút không vượt số dư

---

### Bài 3: Class HinhChuNhat
Tạo class `HinhChuNhat` với:
- Fields: chieuRong, chieuCao
- Constructors: default (1,1), hình vuông (1 cạnh), đầy đủ (2 cạnh)
- Methods: tinhDienTich(), tinhChuVi(), laHinhVuong()
- Static method: soSanh(HinhChuNhat h1, HinhChuNhat h2) → trả về hình lớn hơn

---

### Bài 4: Class NhanVien + PhongBan
Tạo 2 classes:
- `PhongBan`: maPhong, tenPhong, soNhanVien
- `NhanVien`: maNV, ten, luong, phongBan
- Khi thêm nhân viên vào phòng ban → tăng soNhanVien

---

### Bài 5: Hệ thống thư viện đơn giản
- `Sach`: maSach, tieuDe, tacGia, conKhong (boolean)
- `ThanhVien`: maTV, ten, sachDangMuon[]
- `ThuVien`: danhSachSach[], danhSachTV[]
- Methods: themSach(), muonSach(), traSach()

---

## 10. Đáp án tham khảo

> ⚠️ **Tự làm trước ít nhất 15 phút trước khi xem đáp án!**

<details>
<summary>Bài 1: Class SinhVien (click để mở)</summary>

```java
public class SinhVien {
    private String maSV;
    private String ten;
    private int tuoi;
    private double diemTB;

    // Default constructor
    public SinhVien() {
        this.maSV = "";
        this.ten = "Chưa đặt tên";
        this.tuoi = 18;
        this.diemTB = 0.0;
    }

    // Full constructor
    public SinhVien(String maSV, String ten, int tuoi, double diemTB) {
        this.maSV = maSV;
        this.setTen(ten);
        this.setTuoi(tuoi);
        this.setDiemTB(diemTB);
    }

    // Getters
    public String getMaSV() { return maSV; }
    public String getTen() { return ten; }
    public int getTuoi() { return tuoi; }
    public double getDiemTB() { return diemTB; }

    // Setters với validation
    public void setTen(String ten) {
        if (ten != null && !ten.trim().isEmpty()) {
            this.ten = ten;
        }
    }

    public void setTuoi(int tuoi) {
        if (tuoi >= 16 && tuoi <= 60) {
            this.tuoi = tuoi;
        }
    }

    public void setDiemTB(double diemTB) {
        if (diemTB >= 0.0 && diemTB <= 10.0) {
            this.diemTB = diemTB;
        }
    }

    // Hiển thị thông tin
    public void hienThiThongTin() {
        System.out.println("=== Thông tin Sinh viên ===");
        System.out.println("Mã SV: " + maSV);
        System.out.println("Tên: " + ten);
        System.out.println("Tuổi: " + tuoi);
        System.out.printf("Điểm TB: %.2f%n", diemTB);
        System.out.println("Đạt yêu cầu: " + (datYeuCau() ? "Có" : "Không"));
    }

    // Kiểm tra đạt yêu cầu
    public boolean datYeuCau() {
        return diemTB >= 5.0;
    }
}
```
</details>

<details>
<summary>Bài 2: Class TaiKhoanNganHang (click để mở)</summary>

```java
public class TaiKhoanNganHang {
    private String soTK;
    private String chuTK;
    private double soDu;

    private static int tongSoTK = 0;  // Đếm tổng số tài khoản

    public TaiKhoanNganHang(String soTK, String chuTK, double soDuBanDau) {
        this.soTK = soTK;
        this.chuTK = chuTK;
        this.soDu = soDuBanDau > 0 ? soDuBanDau : 0;
        tongSoTK++;  // Mỗi lần tạo TK mới → tăng biến đếm chung
    }

    public static int getTongSoTK() { return tongSoTK; }
    public double getSoDu() { return soDu; }

    public void napTien(double soTien) {
        if (soTien > 0) {
            soDu += soTien;
            System.out.printf("Đã nạp %,.0f. Số dư: %,.0f%n", soTien, soDu);
        } else {
            System.out.println("Số tiền không hợp lệ!");
        }
    }

    public boolean rutTien(double soTien) {
        if (soTien > 0 && soTien <= soDu) {
            soDu -= soTien;
            System.out.printf("Đã rút %,.0f. Số dư: %,.0f%n", soTien, soDu);
            return true;
        }
        System.out.println("Rút tiền thất bại!");
        return false;
    }

    public boolean chuyenTien(TaiKhoanNganHang nguoiNhan, double soTien) {
        if (this.rutTien(soTien)) {
            nguoiNhan.napTien(soTien);
            System.out.printf("Đã chuyển %,.0f cho %s%n", soTien, nguoiNhan.chuTK);
            return true;
        }
        return false;
    }
}
```
</details>

---

## Navigation

- [← Day 2: Operators & Control Flow](./day-02-operators-control-flow.md)
- [Day 4: OOP Pillars →](./day-04-oop-pillars.md)
