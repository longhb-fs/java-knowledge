# Day 4: OOP Pillars (4 Trụ cột của Lập trình Hướng đối tượng)

## Mục tiêu hôm nay

Sau ngày hôm nay, bạn sẽ hiểu 4 trụ cột quan trọng nhất của OOP:
- **Inheritance** (Kế thừa) — "con thừa hưởng từ cha"
- **Polymorphism** (Đa hình) — "cùng 1 hành động, mỗi đối tượng làm khác nhau"
- **Encapsulation** (Đóng gói) — "giấu bên trong, chỉ lộ cái cần thiết"
- **Abstraction** (Trừu tượng) — "chỉ mô tả PHẢI LÀM GÌ, không nói LÀM THẾ NÀO"

### Tại sao 4 trụ cột này quan trọng?

Đây là **nền tảng** của mọi ứng dụng Java. Không hiểu 4 cái này = không viết được code Java chuyên nghiệp. Phỏng vấn Java chắc chắn hỏi.

> 💡 **Ví dụ đời thường**: Tưởng tượng hệ thống quản lý nhân viên:
> - **Kế thừa**: Nhân viên Full-time, Part-time, Freelancer đều "là" Nhân viên → kế thừa từ class Nhân viên
> - **Đa hình**: Gọi `tinhLuong()` → Full-time tính khác, Part-time tính khác, Freelancer tính khác
> - **Đóng gói**: Lương là `private`, chỉ được xem qua `getLuong()`, không sửa trực tiếp
> - **Trừu tượng**: Biết "mọi nhân viên PHẢI có `tinhLuong()`", nhưng mỗi loại tự quyết cách tính

---

## 1. Inheritance (Kế thừa) — "Con thừa hưởng từ Cha"

### Tại sao cần kế thừa?

Khi nhiều class có chung đặc điểm, thay vì viết lại code → **kế thừa** để dùng lại code từ class cha.

> 💡 **Ví dụ đời thường**: Con người thừa hưởng đặc điểm từ cha mẹ (màu mắt, chiều cao). Trong code, class con thừa hưởng fields và methods từ class cha.

```
Không kế thừa (❌ lặp code):        Có kế thừa (✅ gọn gàng):
┌──────────────┐                    ┌──────────────┐
│ Cho           │                    │ DongVat      │ ← Class CHA
│ - ten         │                    │ - ten        │
│ - tuoi        │                    │ - tuoi       │
│ + an()        │                    │ + an()       │
│ + ngu()       │                    │ + ngu()      │
│ + sua()       │                    └──────┬───────┘
└──────────────┘                           │ extends (kế thừa)
┌──────────────┐                    ┌──────┴───────┐
│ Meo           │                    │ Cho          │ ← Class CON
│ - ten         │ ← LẶP!            │ + sua()      │   Tự có thêm sua()
│ - tuoi        │ ← LẶP!            └──────────────┘   Kế thừa ten, tuoi, an(), ngu()
│ + an()        │ ← LẶP!
│ + ngu()       │ ← LẶP!
│ + keu()       │
└──────────────┘
```

### 1.1. Cú pháp kế thừa: `extends`

```java
// ── CLASS CHA (Parent/Superclass) ──
public class DongVat {
    protected String ten;    // protected = class con truy cập được
    protected int tuoi;

    public DongVat(String ten, int tuoi) {
        this.ten = ten;
        this.tuoi = tuoi;
    }

    public void an() {
        System.out.println(ten + " đang ăn.");
    }

    public void ngu() {
        System.out.println(ten + " đang ngủ.");
    }

    public void hienThiThongTin() {
        System.out.println("Tên: " + ten + ", Tuổi: " + tuoi);
    }
}

// ── CLASS CON (Child/Subclass) ── kế thừa từ DongVat
public class Cho extends DongVat {
//                    └── "extends" = "mở rộng từ" / "kế thừa từ"
    private String giong;  // Field riêng của Cho

    public Cho(String ten, int tuoi, String giong) {
        super(ten, tuoi);    // ← Gọi constructor của class CHA
        this.giong = giong;
    }

    // Method RIÊNG của Cho (class cha không có)
    public void sua() {
        System.out.println(ten + " sủa: Gâu gâu!");
    }

    // OVERRIDE = "ghi đè" method của class cha
    @Override
    public void hienThiThongTin() {
        super.hienThiThongTin();  // Gọi method gốc của cha trước
        System.out.println("Giống: " + giong);
    }
}
```

```java
// Sử dụng:
Cho cho = new Cho("Milu", 3, "Corgi");
cho.an();              // "Milu đang ăn."    ← KẾ THỪA từ DongVat
cho.ngu();             // "Milu đang ngủ."   ← KẾ THỪA từ DongVat
cho.sua();             // "Milu sủa: Gâu gâu!" ← RIÊNG của Cho
cho.hienThiThongTin(); // Tên: Milu, Tuổi: 3  ← GHI ĐÈ (override)
                        // Giống: Corgi
```

### 1.2. `super` — "gọi lên class cha"

```java
public class Meo extends DongVat {
    private boolean oTrongNha;  // Mèo nhà hay mèo hoang?

    public Meo(String ten, int tuoi, boolean oTrongNha) {
        super(ten, tuoi);  // super() = gọi constructor CHA → BẮT BUỘC nếu cha không có default constructor
        this.oTrongNha = oTrongNha;
    }

    @Override
    public void an() {
        super.an();  // Gọi method an() của CHA trước
        System.out.println(ten + " thích ăn cá.");  // Rồi thêm hành vi riêng
    }
}

// Kết quả:
Meo meo = new Meo("Miu", 2, true);
meo.an();
// "Miu đang ăn."      ← từ super.an()
// "Miu thích ăn cá."  ← thêm riêng
```

> 💡 **Mẹo nhớ**: `this` = "chính tôi", `super` = "cha tôi". `this()` gọi constructor của mình, `super()` gọi constructor của cha.

### 1.3. Chuỗi kế thừa (Inheritance Chain)

```java
// Cấp 1: Ông
public class PhuongTien {
    protected String hang;
    public void khoiDong() { System.out.println("Phương tiện khởi động..."); }
}

// Cấp 2: Cha (kế thừa Ông)
public class OTo extends PhuongTien {
    protected int soCua;
    public void chay() { System.out.println("Ô tô đang chạy..."); }
}

// Cấp 3: Con (kế thừa Cha, gián tiếp kế thừa Ông)
public class XeTheThao extends OTo {
    private int tocDoMax;
    public void duaXe() { System.out.println("Đua xe " + tocDoMax + " km/h!"); }
}

// XeTheThao có TẤT CẢ methods: khoiDong() + chay() + duaXe()
XeTheThao ferrari = new XeTheThao();
ferrari.khoiDong();  // Từ PhuongTien (ông)
ferrari.chay();      // Từ OTo (cha)
ferrari.duaXe();     // Của chính mình
```

### 1.4. Quy tắc quan trọng về kế thừa

```java
// ❌ Java KHÔNG hỗ trợ đa kế thừa (multiple inheritance) với class
public class Con extends Cha1, Cha2 { }  // LỖI!

// ✅ Nhưng CÓ THỂ implement nhiều interface (sẽ học phần Abstraction)
public class Con extends Cha implements GiaoDien1, GiaoDien2 { }

// ❌ Không thể kế thừa class đánh dấu final
public final class KhongChoKeThua { }
public class Con extends KhongChoKeThua { }  // LỖI!

// ❌ Không thể override method đánh dấu final
public class Cha {
    public final void khongDoiDuoc() { }  // Method này không ai ghi đè được
}
```

> 🔥 **Quy tắc vàng**: Java chỉ cho kế thừa ĐƠN (1 class cha). Muốn "đa kế thừa" → dùng Interface.

---

## 2. Polymorphism (Đa hình) — "Cùng hành động, khác cách thực hiện"

### Tại sao cần đa hình?

> 💡 **Ví dụ đời thường**: "Nói" là 1 hành động, nhưng:
> - Chó "nói" → Gâu gâu
> - Mèo "nói" → Meo meo
> - Vịt "nói" → Quạc quạc
>
> Cùng hành động `phatRaTieng()`, mỗi loại động vật làm KHÁC NHAU. Đó là đa hình!

### 2.1. Method Overriding (Đa hình lúc chạy — Runtime Polymorphism)

Class con **ghi đè** method của class cha để làm theo cách riêng:

```java
public class HinhHoc {
    public double tinhDienTich() {
        return 0;  // Mặc định
    }
}

public class HinhTron extends HinhHoc {
    private double banKinh;

    public HinhTron(double banKinh) {
        this.banKinh = banKinh;
    }

    @Override  // ← Đánh dấu "tôi đang ghi đè method của cha"
    public double tinhDienTich() {
        return Math.PI * banKinh * banKinh;  // Công thức riêng hình tròn
    }
}

public class HinhChuNhat extends HinhHoc {
    private double chieuRong, chieuCao;

    public HinhChuNhat(double chieuRong, double chieuCao) {
        this.chieuRong = chieuRong;
        this.chieuCao = chieuCao;
    }

    @Override
    public double tinhDienTich() {
        return chieuRong * chieuCao;  // Công thức riêng hình chữ nhật
    }
}
```

**Sức mạnh của đa hình — cùng 1 vòng lặp, mỗi hình tính khác nhau:**

```java
// Mảng kiểu CHA, chứa object CON → Đa hình!
HinhHoc[] cacHinh = new HinhHoc[3];
cacHinh[0] = new HinhTron(5);           // Hình tròn bán kính 5
cacHinh[1] = new HinhChuNhat(4, 6);     // Hình chữ nhật 4x6
cacHinh[2] = new HinhTron(3);           // Hình tròn bán kính 3

for (HinhHoc hinh : cacHinh) {
    // Java TỰ ĐỘNG gọi đúng method dựa trên kiểu THỰC TẾ của object
    System.out.printf("Diện tích: %.2f%n", hinh.tinhDienTich());
}
// Kết quả:
// Diện tích: 78.54    ← HinhTron tính π×r²
// Diện tích: 24.00    ← HinhChuNhat tính r×c
// Diện tích: 28.27    ← HinhTron tính π×r²
```

> 🔥 **Điểm mạnh**: Viết code 1 lần (`hinh.tinhDienTich()`), nhưng chạy đúng cho MỌI loại hình. Thêm hình mới (tam giác, lục giác...) → chỉ cần tạo class mới, KHÔNG SỬA code cũ!

### 2.2. Upcasting và Downcasting — "chuyển đổi kiểu object"

```java
// ── UPCASTING (con → cha) — TỰ ĐỘNG, luôn an toàn ──
DongVat dongVat = new Cho("Milu", 3, "Corgi");
// Biến kiểu DongVat, nhưng object thực tế là Cho
dongVat.an();    // ✅ OK — DongVat có method an()
// dongVat.sua(); // ❌ LỖI! DongVat không biết method sua() của Cho

// ── DOWNCASTING (cha → con) — THỦ CÔNG, cần kiểm tra ──
if (dongVat instanceof Cho) {        // Kiểm tra: có thực sự là Cho không?
    Cho cho = (Cho) dongVat;         // Ép kiểu rõ ràng
    cho.sua();                        // ✅ OK — giờ gọi được sua()
}

// Java 16+: Pattern matching — gọn hơn
if (dongVat instanceof Cho cho) {    // Kiểm tra VÀ ép kiểu 1 bước
    cho.sua();                        // ✅ OK
}
```

> 💡 **Ví dụ đời thường**: Upcasting = gọi Corgi là "con chó" (đúng nhưng mất chi tiết). Downcasting = biết "con chó" kia thực ra là Corgi (phải kiểm tra trước).

### 2.3. Method Overloading (Đa hình lúc biên dịch — Compile-time)

Đã học ở Day 3. Nhắc lại: **cùng tên, khác tham số**.

```java
public class PhepTinh {
    public int cong(int a, int b) { return a + b; }             // 2 int
    public int cong(int a, int b, int c) { return a + b + c; }  // 3 int
    public double cong(double a, double b) { return a + b; }    // 2 double
}
```

> 💡 **Overloading vs Overriding**:
> - **Overloading** = CÙNG class, cùng tên, KHÁC tham số (compile-time)
> - **Overriding** = class CON ghi đè method class CHA, CÙNG tham số (runtime)

---

## 3. Encapsulation (Đóng gói) — "Giấu bên trong, chỉ lộ cái cần thiết"

### Tại sao cần đóng gói?

> 💡 **Ví dụ đời thường**: Bạn dùng tivi → nhấn nút bật/tắt/chuyển kênh (public methods). Bạn KHÔNG CẦN biết mạch điện bên trong hoạt động thế nào (private fields). Nếu ai cũng mở được tivi ra sửa mạch → dễ hỏng!
>
> Đóng gói = **ẩn chi tiết bên trong**, chỉ cung cấp **cổng giao tiếp** an toàn.

### 3.1. Cách thực hiện

```java
public class NhanVien {
    // ① PRIVATE fields — ẩn dữ liệu, không ai truy cập trực tiếp
    private String maNV;
    private String ten;
    private double luong;

    // ② PUBLIC constructor — "cổng vào" duy nhất để tạo object
    public NhanVien(String maNV, String ten, double luong) {
        this.maNV = maNV;
        this.setTen(ten);      // Dùng setter để có validation
        this.setLuong(luong);
    }

    // ③ PUBLIC getters — cho phép ĐỌC có kiểm soát
    public String getMaNV() { return maNV; }
    public String getTen() { return ten; }
    public double getLuong() { return luong; }

    // ④ PUBLIC setters với VALIDATION — cho phép GHI có kiểm tra
    public void setTen(String ten) {
        if (ten != null && ten.length() >= 2) {
            this.ten = ten;
        } else {
            throw new IllegalArgumentException("Tên phải có ít nhất 2 ký tự!");
        }
    }

    public void setLuong(double luong) {
        if (luong >= 0) {
            this.luong = luong;
        } else {
            throw new IllegalArgumentException("Lương không được âm!");
        }
    }

    // ⑤ PUBLIC business methods — hành vi nghiệp vụ
    public void tangLuong(double phanTram) {
        if (phanTram > 0 && phanTram <= 50) {
            this.luong += this.luong * (phanTram / 100);
        }
    }
}
```

```java
// Sử dụng:
NhanVien nv = new NhanVien("NV01", "An", 15000000);
// nv.luong = -999;        // ❌ LỖI BIÊN DỊCH! luong là private
nv.setLuong(-999);         // ❌ Ném exception! "Lương không được âm!"
nv.tangLuong(10);          // ✅ OK — tăng 10% qua method an toàn
System.out.println(nv.getLuong()); // Đọc lương qua getter
```

> 🔥 **Quy tắc**: Fields → `private`. Truy cập → qua `public` getter/setter có validation. Đây là quy tắc **BẮT BUỘC** trong code chuyên nghiệp.

### 3.2. Immutable Class (Class bất biến) — object KHÔNG THỂ thay đổi sau khi tạo

```java
// final class = không ai kế thừa được
public final class DiaChi {
    // final fields = không thể gán lại sau khi khởi tạo
    private final String thanhPho;
    private final String quan;
    private final String duong;

    public DiaChi(String thanhPho, String quan, String duong) {
        this.thanhPho = thanhPho;
        this.quan = quan;
        this.duong = duong;
    }

    // CHỈ CÓ getter, KHÔNG CÓ setter → không ai sửa được
    public String getThanhPho() { return thanhPho; }
    public String getQuan() { return quan; }
    public String getDuong() { return duong; }

    // Muốn "thay đổi" → tạo object MỚI
    public DiaChi doiQuan(String quanMoi) {
        return new DiaChi(this.thanhPho, quanMoi, this.duong);
    }
}

// Sử dụng:
DiaChi dc = new DiaChi("HCM", "Quận 1", "Nguyễn Huệ");
// dc.thanhPho = "HN";  // ❌ LỖI! private + final
DiaChi dc2 = dc.doiQuan("Quận 7");  // ✅ Tạo object MỚI, dc gốc KHÔNG đổi
```

> 💡 **Khi nào dùng Immutable?** Khi dữ liệu KHÔNG NÊN thay đổi sau khi tạo: địa chỉ, tiền tệ, ngày tháng. `String` trong Java cũng là immutable!

---

## 4. Abstraction (Trừu tượng) — "Chỉ nói PHẢI LÀM GÌ, không nói LÀM THẾ NÀO"

### Tại sao cần trừu tượng?

> 💡 **Ví dụ đời thường**: Bạn biết "ô tô PHẢI CÓ khả năng chạy". Nhưng cách chạy thì:
> - Xe xăng: đốt nhiên liệu
> - Xe điện: dùng pin
> - Xe hybrid: kết hợp cả hai
>
> "Phải chạy được" = **trừu tượng** (abstract). Cách chạy cụ thể = **cài đặt** (implementation).

### 4.1. Abstract Class — class "chưa hoàn chỉnh"

Abstract class = **bản thiết kế chưa đầy đủ**. Không thể tạo object trực tiếp, phải có class con hoàn thiện nốt.

```java
// abstract class = KHÔNG THỂ new trực tiếp
public abstract class HinhHoc {
    protected String mauSac;

    public HinhHoc(String mauSac) {
        this.mauSac = mauSac;
    }

    // abstract method = CHỈ KHAI BÁO, không có code bên trong
    // Class con BẮT BUỘC phải viết code (implement)
    public abstract double tinhDienTich();
    public abstract double tinhChuVi();

    // Method thường (concrete) = CÓ code bên trong, class con kế thừa được
    public void hienThiMau() {
        System.out.println("Màu sắc: " + mauSac);
    }
}

// Class con PHẢI implement tất cả abstract methods
public class HinhTron extends HinhHoc {
    private double banKinh;

    public HinhTron(String mauSac, double banKinh) {
        super(mauSac);  // Gọi constructor cha
        this.banKinh = banKinh;
    }

    @Override
    public double tinhDienTich() {
        return Math.PI * banKinh * banKinh;  // Viết code cụ thể cho hình tròn
    }

    @Override
    public double tinhChuVi() {
        return 2 * Math.PI * banKinh;
    }
}

public class HinhChuNhat extends HinhHoc {
    private double rong, cao;

    public HinhChuNhat(String mauSac, double rong, double cao) {
        super(mauSac);
        this.rong = rong;
        this.cao = cao;
    }

    @Override
    public double tinhDienTich() { return rong * cao; }

    @Override
    public double tinhChuVi() { return 2 * (rong + cao); }
}
```

```java
// Sử dụng:
// HinhHoc h = new HinhHoc("Đỏ");  // ❌ LỖI! Không thể new abstract class

HinhHoc tron = new HinhTron("Đỏ", 5);
HinhHoc vuong = new HinhChuNhat("Xanh", 4, 6);

tron.hienThiMau();  // "Màu sắc: Đỏ"    ← method thường, kế thừa từ cha
System.out.printf("Diện tích tròn: %.2f%n", tron.tinhDienTich());   // 78.54
System.out.printf("Diện tích vuông: %.2f%n", vuong.tinhDienTich()); // 24.00
```

### 4.2. Interface — "Hợp đồng" class PHẢI tuân thủ

Interface = **danh sách hành động class PHẢI CÓ**. Không chứa code (trước Java 8), chỉ khai báo method.

> 💡 **Ví dụ đời thường**: Ổ cắm USB là "interface" — bất kỳ thiết bị nào (chuột, bàn phím, USB) muốn kết nối máy tính đều PHẢI có đầu cắm USB. Interface quy định "bạn PHẢI CÓ khả năng gì", không quan tâm bên trong làm thế nào.

```java
// Interface = "hợp đồng" khai báo khả năng
public interface VeDuoc {      // "Có khả năng vẽ được"
    void ve();                  // Method trừu tượng (không có code)
}

public interface ThayDoiKichThuoc {  // "Có khả năng thay đổi kích thước"
    void thayDoiKichThuoc(double heSo);
}

// Class CÓ THỂ implement NHIỀU interface (khác với class chỉ extends 1)
public class HinhTronVe extends HinhHoc implements VeDuoc, ThayDoiKichThuoc {
//                                                 └── implement 2 interface

    private double banKinh;

    public HinhTronVe(String mauSac, double banKinh) {
        super(mauSac);
        this.banKinh = banKinh;
    }

    @Override
    public double tinhDienTich() { return Math.PI * banKinh * banKinh; }

    @Override
    public double tinhChuVi() { return 2 * Math.PI * banKinh; }

    @Override
    public void ve() {
        System.out.println("Vẽ hình tròn " + mauSac + " bán kính " + banKinh);
    }

    @Override
    public void thayDoiKichThuoc(double heSo) {
        banKinh *= heSo;
    }
}
```

### 4.3. Interface với default method (Java 8+)

Từ Java 8, interface CÓ THỂ chứa method có code (dùng từ khóa `default`):

```java
public interface PhuongTien {
    // Abstract method — class CON phải implement
    void khoiDong();
    void dungLai();

    // Default method — CÓ code sẵn, class con không bắt buộc override
    default void bom() {
        System.out.println("Bíp bíp!");
    }

    // Static method — gọi qua tên interface
    static void inThongTin() {
        System.out.println("Đây là interface PhuongTien");
    }
}

public class OTo implements PhuongTien {
    @Override
    public void khoiDong() { System.out.println("Ô tô khởi động..."); }

    @Override
    public void dungLai() { System.out.println("Ô tô dừng lại."); }

    // Có thể override default method hoặc không
    @Override
    public void bom() { System.out.println("BÉÉÉÉÉP!"); }
}

// Sử dụng:
OTo xe = new OTo();
xe.khoiDong();     // "Ô tô khởi động..."
xe.bom();          // "BÉÉÉÉÉP!" (đã override)
PhuongTien.inThongTin();  // "Đây là interface PhuongTien" (static method)
```

### 4.4. Abstract Class vs Interface — khi nào dùng cái nào?

| Tiêu chí | Abstract Class | Interface |
|----------|----------------|-----------|
| **Methods** | Có cả abstract + concrete | Abstract + default (Java 8+) |
| **Fields** | Bất kỳ kiểu nào | Chỉ `public static final` (hằng số) |
| **Constructor** | ✅ Có | ❌ Không |
| **Đa kế thừa** | ❌ Chỉ extends 1 | ✅ Implement nhiều |
| **Khi nào dùng?** | **"LÀ GÌ"** + chia sẻ code | **"CÓ KHẢ NĂNG GÌ"** |

> 💡 **Mẹo nhớ**:
> - **Abstract class**: "Chó **LÀ** Động vật" → `class Cho extends DongVat`
> - **Interface**: "Chó **CÓ KHẢ NĂNG** bơi" → `class Cho implements BoiDuoc`

```java
// Ví dụ kết hợp cả hai:
abstract class DongVat {       // "Là" động vật (có tên, tuổi, ăn, ngủ)
    protected String ten;
    public abstract void phatRaTieng();
    public void ngu() { System.out.println(ten + " đang ngủ"); }
}

interface BoiDuoc {            // "Có khả năng" bơi
    void boi();
}

interface BayDuoc {            // "Có khả năng" bay
    void bay();
}

// Vịt LÀ động vật, CÓ THỂ bơi VÀ bay
class Vit extends DongVat implements BoiDuoc, BayDuoc {
    public Vit(String ten) { this.ten = ten; }

    @Override
    public void phatRaTieng() { System.out.println("Quạc quạc!"); }

    @Override
    public void boi() { System.out.println(ten + " đang bơi"); }

    @Override
    public void bay() { System.out.println(ten + " đang bay"); }
}
```

### 4.5. Sealed Classes (Java 17+) — giới hạn ai được kế thừa

```java
// sealed = "đóng dấu" — chỉ cho phép các class được liệt kê kế thừa
public sealed class HinhHoc permits HinhTron, HinhChuNhat, TamGiac {
    // ...
}

public final class HinhTron extends HinhHoc { }        // final = không ai kế thừa tiếp
public final class HinhChuNhat extends HinhHoc { }
public non-sealed class TamGiac extends HinhHoc { }    // non-sealed = cho phép kế thừa tiếp

// public class HinhThang extends HinhHoc { }  // ❌ LỖI! Không nằm trong danh sách permits
```

---

## 5. Object Class — "Ông tổ" của mọi class

### Tại sao cần biết?

Mọi class trong Java đều **ngầm kế thừa** từ `Object`. Nghĩa là mọi object đều có sẵn 3 method quan trọng: `toString()`, `equals()`, `hashCode()`.

```java
public class NguoiDung {
    private String ten;
    private int tuoi;

    public NguoiDung(String ten, int tuoi) {
        this.ten = ten;
        this.tuoi = tuoi;
    }

    // ① Override toString() — mô tả object dưới dạng chuỗi
    @Override
    public String toString() {
        return "NguoiDung{ten='" + ten + "', tuoi=" + tuoi + "}";
    }

    // ② Override equals() — so sánh NỘI DUNG 2 object
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;                    // Cùng object → bằng
        if (obj == null || getClass() != obj.getClass()) return false;  // Khác class → không bằng
        NguoiDung other = (NguoiDung) obj;               // Ép kiểu
        return tuoi == other.tuoi && ten.equals(other.ten); // So sánh từng field
    }

    // ③ Override hashCode() — LUÔN override cùng equals()
    @Override
    public int hashCode() {
        return java.util.Objects.hash(ten, tuoi);
    }
}

// Sử dụng:
NguoiDung u1 = new NguoiDung("An", 25);
NguoiDung u2 = new NguoiDung("An", 25);

System.out.println(u1);              // NguoiDung{ten='An', tuoi=25} ← toString()
System.out.println(u1.equals(u2));   // true  ← so sánh nội dung
System.out.println(u1 == u2);       // false ← so sánh địa chỉ bộ nhớ (khác object)
```

> 🔥 **Quy tắc**: Override `equals()` thì **BẮT BUỘC** override `hashCode()` cùng. Nếu không → bug khi dùng HashMap, HashSet.

---

## 6. Tóm tắt cuối ngày

| Trụ cột | Nghĩa | Từ khóa | Ví dụ |
|---------|-------|---------|-------|
| **Inheritance** (Kế thừa) | Con thừa hưởng từ cha | `extends`, `super` | `class Cho extends DongVat` |
| **Polymorphism** (Đa hình) | Cùng hành động, khác cách làm | `@Override` | `tinhDienTich()` cho mỗi hình |
| **Encapsulation** (Đóng gói) | Giấu bên trong, lộ cái cần | `private`, getter/setter | Field private + public getter |
| **Abstraction** (Trừu tượng) | Chỉ nói làm gì, không nói cách | `abstract`, `interface` | `abstract void tinhLuong()` |

| Khái niệm | Giải thích |
|-----------|-----------|
| `extends` | Class con kế thừa class cha |
| `super` | Gọi constructor/method của class cha |
| `@Override` | Đánh dấu ghi đè method |
| `abstract class` | Class chưa hoàn chỉnh, không thể new |
| `interface` | "Hợp đồng" khai báo khả năng |
| `implements` | Class cài đặt interface |
| Upcasting | Con → Cha (tự động) |
| Downcasting | Cha → Con (thủ công, cần kiểm tra `instanceof`) |
| Overloading | Cùng tên, khác tham số (compile-time) |
| Overriding | Class con ghi đè method cha (runtime) |
| `sealed` | Giới hạn ai được kế thừa (Java 17+) |

---

## 7. Bài tập thực hành

### Bài 1: Hệ thống nhân viên
Tạo hệ thống tính lương:

```
NhanVien (abstract)
├── NhanVienFullTime    → lương = luongCoBan
├── NhanVienPartTime    → lương = soGio × donGia
└── NhanVienFreelance   → lương = phiDuAn
```

**Yêu cầu:**
- Abstract method: `tinhLuong()`
- Method `hienThiThongTin()` in mã NV, tên, lương
- Tạo mảng `NhanVien[]`, tính tổng lương tất cả

---

### Bài 2: Hệ thống hình học
```
interface VeDuoc { void ve(); }
interface DoLuongDuoc { double tinhDienTich(); double tinhChuVi(); }

HinhHoc (abstract) implements VeDuoc, DoLuongDuoc
├── HinhTron
├── HinhChuNhat
├── TamGiac
└── HinhVuong extends HinhChuNhat
```

---

### Bài 3: Hệ thống thanh toán
```
interface ThanhToanDuoc { void xuLyThanhToan(double soTien); }

ThanhToan (abstract) implements ThanhToanDuoc
├── ThanhToanTheCredit  → cần soThe, cvv, validate()
├── ThanhToanChuyenKhoan → cần soTaiKhoan, tenNganHang
└── ThanhToanViDienTu    → cần maVi, soDu
```

---

### Bài 4: Nhân vật game
```
interface TanCongDuoc { void tanCong(); }
interface HoiMauDuoc { void hoiMau(); }
interface DiChuyenDuoc { void diChuyen(); }

NhanVat (abstract)
├── ChienBinh implements TanCongDuoc, DiChuyenDuoc
├── PhapSu implements TanCongDuoc, HoiMauDuoc, DiChuyenDuoc
├── CungThu implements TanCongDuoc, DiChuyenDuoc
└── ThuTe implements HoiMauDuoc, DiChuyenDuoc
```

---

## 8. Đáp án tham khảo

> ⚠️ **Tự làm trước ít nhất 15 phút!**

<details>
<summary>Bài 1: Hệ thống nhân viên (click để mở)</summary>

```java
// ── Abstract class ──
public abstract class NhanVien {
    protected String maNV;
    protected String ten;

    public NhanVien(String maNV, String ten) {
        this.maNV = maNV;
        this.ten = ten;
    }

    // Abstract method — mỗi loại NV tự quyết cách tính
    public abstract double tinhLuong();

    public void hienThiThongTin() {
        System.out.println("Mã NV: " + maNV);
        System.out.println("Tên: " + ten);
        System.out.printf("Lương: %,.0f VND%n", tinhLuong());
    }
}

// ── Full-time ──
public class NhanVienFullTime extends NhanVien {
    private double luongCoBan;

    public NhanVienFullTime(String maNV, String ten, double luongCoBan) {
        super(maNV, ten);
        this.luongCoBan = luongCoBan;
    }

    @Override
    public double tinhLuong() {
        return luongCoBan;
    }
}

// ── Part-time ──
public class NhanVienPartTime extends NhanVien {
    private double donGia;    // Giá mỗi giờ
    private int soGioLam;     // Số giờ đã làm

    public NhanVienPartTime(String maNV, String ten, double donGia, int soGioLam) {
        super(maNV, ten);
        this.donGia = donGia;
        this.soGioLam = soGioLam;
    }

    @Override
    public double tinhLuong() {
        return donGia * soGioLam;
    }
}

// ── Freelance ──
public class NhanVienFreelance extends NhanVien {
    private double phiDuAn;

    public NhanVienFreelance(String maNV, String ten, double phiDuAn) {
        super(maNV, ten);
        this.phiDuAn = phiDuAn;
    }

    @Override
    public double tinhLuong() {
        return phiDuAn;
    }
}

// ── Main ──
public class Main {
    public static void main(String[] args) {
        NhanVien[] danhSach = {
            new NhanVienFullTime("NV01", "An", 15000000),
            new NhanVienPartTime("NV02", "Bình", 100000, 80),
            new NhanVienFreelance("NV03", "Chi", 20000000)
        };

        double tongLuong = 0;
        for (NhanVien nv : danhSach) {
            nv.hienThiThongTin();
            tongLuong += nv.tinhLuong();  // ← Đa hình! Mỗi loại tính khác nhau
            System.out.println("---");
        }

        System.out.printf("Tổng lương: %,.0f VND%n", tongLuong);
    }
}
```
</details>

---

## Navigation

- [← Day 3: OOP Basics](./day-03-oop-basics.md)
- [Day 5: Exception Handling →](./day-05-exception-handling.md)
