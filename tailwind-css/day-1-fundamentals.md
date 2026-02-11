# Day 1: Spacing, Colors, Typography, Sizing

> Mục tiêu: Hiểu rõ cách Tailwind hoạt động thông qua những utility class dùng nhiều nhất.

---

## 1. Spacing (Padding & Margin)

### 💡 Hiểu hệ thống spacing của Tailwind

Tailwind dùng một **scale cố định** thay vì để bạn tự đặt số pixel. Điều này giúp design consistent.

```
┌──────────────────────────────────────────────────────────┐
│  CÔNG THỨC: Số × 4px = Giá trị thực                      │
│                                                          │
│  p-1  = 1 × 4px = 4px    = 0.25rem                       │
│  p-2  = 2 × 4px = 8px    = 0.5rem                        │
│  p-4  = 4 × 4px = 16px   = 1rem                          │
│  p-8  = 8 × 4px = 32px   = 2rem                          │
│  p-16 = 16 × 4px = 64px  = 4rem                          │
└──────────────────────────────────────────────────────────┘
```

### 🔄 So sánh CSS ↔ Tailwind

**CSS truyền thống:**
```css
.box {
  padding: 16px;           /* Tự chọn số */
  margin-top: 24px;        /* Dễ inconsistent: 24px ở đây, 25px ở kia */
  margin-bottom: 32px;
}
```

**Tailwind:**
```html
<div class="p-4 mt-6 mb-8">
  <!--
  p-4  = padding: 16px (4 × 4)
  mt-6 = margin-top: 24px (6 × 4)
  mb-8 = margin-bottom: 32px (8 × 4)
  -->
</div>
```

### 📐 Bảng spacing đầy đủ

| Class | Giá trị | Pixels | Khi nào dùng |
|-------|---------|--------|--------------|
| `p-0` | 0 | 0px | Reset padding |
| `p-px` | 1px | 1px | Border compensation |
| `p-0.5` | 0.125rem | 2px | Rất nhỏ |
| `p-1` | 0.25rem | 4px | Nhỏ |
| `p-2` | 0.5rem | 8px | **Hay dùng** - icon spacing |
| `p-3` | 0.75rem | 12px | Input padding |
| `p-4` | 1rem | 16px | **Hay dùng nhất** - card padding |
| `p-5` | 1.25rem | 20px | Medium |
| `p-6` | 1.5rem | 24px | **Hay dùng** - section padding |
| `p-8` | 2rem | 32px | Large sections |
| `p-10` | 2.5rem | 40px | Hero sections |
| `p-12` | 3rem | 48px | Page padding |
| `p-16` | 4rem | 64px | Large spacing |
| `p-20` | 5rem | 80px | Very large |
| `p-24` | 6rem | 96px | Huge |

### 📐 Hiểu Direction (Hướng)

```
                    pt (padding-top)
                         ↑
                    ┌─────────┐
     pl (left)  ←   │    p    │   →  pr (right)
                    │  (all)  │
                    └─────────┘
                         ↓
                    pb (padding-bottom)


📝 QUY TẮC ĐẶT TÊN:

p  = padding (tất cả 4 phía)
px = padding x-axis (left + right) - NGANG
py = padding y-axis (top + bottom) - DỌC
pt = padding-top
pr = padding-right
pb = padding-bottom
pl = padding-left

Tương tự cho margin: m, mx, my, mt, mr, mb, ml
```

### 🔥 Ví dụ thực tế với giải thích

```html
<!-- Ví dụ 1: Button -->
<button class="px-4 py-2">
  Click me
</button>
<!--
📐 Phân tích:
┌──────────────────────────┐
│←─16px─→ Click me ←─16px─→│  px-4 = padding trái/phải 16px
│    ↑                     │
│   8px                    │  py-2 = padding trên/dưới 8px
│    ↓                     │
└──────────────────────────┘

🔄 CSS tương đương:
button {
  padding-left: 16px;
  padding-right: 16px;
  padding-top: 8px;
  padding-bottom: 8px;
}
Hoặc shorthand: padding: 8px 16px;
-->


<!-- Ví dụ 2: Card -->
<div class="p-6">
  <h2 class="mb-4">Title</h2>
  <p>Content here</p>
</div>
<!--
📐 Phân tích:
┌────────────────────────────────┐
│            24px                │  p-6 = padding tất cả phía 24px
│   ┌────────────────────────┐   │
│   │ Title                  │   │
│   │         ↓ 16px (mb-4)  │   │
│   │ Content here           │   │
│   └────────────────────────┘   │
│            24px                │
└────────────────────────────────┘
-->


<!-- Ví dụ 3: Center horizontally -->
<div class="mx-auto max-w-md">
  Centered content
</div>
<!--
💡 mx-auto = margin-left: auto; margin-right: auto;
   Đây là cách classic để center một block element.

🔄 CSS tương đương:
div {
  margin-left: auto;
  margin-right: auto;
  max-width: 28rem;
}
-->
```

### 🔥 Space Between Children

Khi có nhiều elements và muốn khoảng cách đều giữa chúng:

```html
<!-- Cách 1: Dùng space-x (horizontal) -->
<div class="flex space-x-4">
  <span>A</span>
  <span>B</span>
  <span>C</span>
</div>
<!--
📐 Kết quả:
[A]──16px──[B]──16px──[C]

💡 space-x-4 tự động thêm margin-left: 16px cho mọi child TRỪ child đầu tiên
-->


<!-- Cách 2: Dùng space-y (vertical) -->
<div class="space-y-4">
  <p>Line 1</p>
  <p>Line 2</p>
  <p>Line 3</p>
</div>
<!--
📐 Kết quả:
┌─────────┐
│ Line 1  │
│   ↓16px │
│ Line 2  │
│   ↓16px │
│ Line 3  │
└─────────┘
-->


<!-- Cách 3: Dùng gap (với Flex/Grid) - KHUYẾN KHÍCH -->
<div class="flex gap-4">
  <span>A</span>
  <span>B</span>
  <span>C</span>
</div>
<!--
💡 gap-4 = gap: 16px;
   Hoạt động với cả flex và grid.
   Đây là cách modern hơn space-x/space-y.
-->
```

### ⚠️ Negative Margin

```html
<div class="-mt-4">
  Pull up 16px
</div>
<!--
💡 Dấu trừ (-) phía trước = negative margin
   -mt-4 = margin-top: -16px;

   Dùng khi muốn element "đè" lên element phía trên.
-->
```

---

## 2. Colors

### 💡 Hiểu hệ thống màu của Tailwind

Tailwind có **22 màu**, mỗi màu có **11 shades** (độ đậm nhạt) từ 50 đến 950.

```
┌────────────────────────────────────────────────────────────┐
│  SHADE SCALE (độ đậm)                                      │
│                                                            │
│  50   100  200  300  400  500  600  700  800  900  950     │
│  ○────○────○────○────○────●────○────○────○────○────○       │
│  ↑                        ↑                        ↑       │
│  Nhạt nhất           Base color              Đậm nhất      │
│  (backgrounds)       (buttons)               (text)        │
└────────────────────────────────────────────────────────────┘
```

### 📐 Bảng màu có sẵn

```
GRAY TONES (màu xám):
├── slate   → Xám hơi xanh dương (modern, tech)
├── gray    → Xám trung tính
├── zinc    → Xám hơi ấm
├── neutral → Xám thuần túy
└── stone   → Xám ấm (hơi nâu)

COLORS (màu sắc):
├── red     → Đỏ (errors, danger)
├── orange  → Cam
├── amber   → Hổ phách (warnings)
├── yellow  → Vàng
├── lime    → Xanh lá chanh
├── green   → Xanh lá (success)
├── emerald → Xanh ngọc
├── teal    → Xanh mòng két
├── cyan    → Xanh cyan
├── sky     → Xanh da trời
├── blue    → Xanh dương (primary, links)
├── indigo  → Xanh chàm
├── violet  → Tím violet
├── purple  → Tím
├── fuchsia → Hồng tím
├── pink    → Hồng
└── rose    → Hồng đỏ
```

### 🔄 So sánh CSS ↔ Tailwind

**CSS truyền thống:**
```css
.card {
  background-color: #ffffff;
  color: #111827;
  border: 1px solid #e5e7eb;
}

.card:hover {
  background-color: #f9fafb;
}

.error-text {
  color: #dc2626;
}
```

**Tailwind:**
```html
<div class="bg-white text-gray-900 border border-gray-200 hover:bg-gray-50">
  <span class="text-red-600">Error message</span>
</div>
```

### 📐 Cách áp dụng màu

```html
<!-- 1. TEXT COLOR -->
<p class="text-gray-900">Dark text (cho headings)</p>
<p class="text-gray-600">Medium text (cho body)</p>
<p class="text-gray-400">Light text (cho placeholder)</p>
<p class="text-blue-600">Blue text (cho links)</p>
<p class="text-red-500">Red text (cho errors)</p>

<!--
🔄 CSS tương đương:
.text-gray-900 { color: #111827; }
.text-blue-600 { color: #2563eb; }
-->


<!-- 2. BACKGROUND COLOR -->
<div class="bg-white">White background</div>
<div class="bg-gray-100">Light gray (page background)</div>
<div class="bg-blue-500">Blue (buttons)</div>
<div class="bg-red-50">Light red (error alert background)</div>

<!--
🔄 CSS tương đương:
.bg-white { background-color: #ffffff; }
.bg-blue-500 { background-color: #3b82f6; }
-->


<!-- 3. BORDER COLOR -->
<div class="border border-gray-200">Light border</div>
<div class="border-2 border-blue-500">Thick blue border</div>

<!--
💡 Lưu ý: Phải có "border" hoặc "border-2" (width) trước
   rồi mới apply border color.

🔄 CSS tương đương:
.border { border-width: 1px; }
.border-gray-200 { border-color: #e5e7eb; }
-->


<!-- 4. OPACITY với màu -->
<div class="bg-black/50">50% transparent black</div>
<div class="bg-blue-500/75">75% opacity blue</div>
<div class="text-white/80">80% opacity white text</div>

<!--
💡 Dùng /[số] để set opacity cho màu
   bg-black/50 = background: rgba(0, 0, 0, 0.5);
-->
```

### 🔥 Pattern: Chọn shade nào?

```
💡 QUY TẮC NGÓN TAY CÁI:

┌─────────────────────────────────────────────────────────┐
│ ELEMENT          │ LIGHT MODE      │ DARK MODE         │
├─────────────────────────────────────────────────────────┤
│ Page background  │ white, gray-50  │ gray-900, gray-950│
│ Card background  │ white           │ gray-800          │
│ Primary text     │ gray-900        │ white, gray-100   │
│ Secondary text   │ gray-600        │ gray-400          │
│ Muted text       │ gray-400        │ gray-500          │
│ Borders          │ gray-200        │ gray-700          │
│ Primary button   │ blue-500        │ blue-600          │
│ Button hover     │ blue-600        │ blue-700          │
│ Success          │ green-500       │ green-400         │
│ Error            │ red-500         │ red-400           │
│ Warning          │ amber-500       │ amber-400         │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Typography

### 💡 Font Size

```
┌─────────────────────────────────────────────────────────┐
│ CLASS      │ SIZE         │ KHI NÀO DÙNG               │
├─────────────────────────────────────────────────────────┤
│ text-xs    │ 12px (0.75rem) │ Labels, captions         │
│ text-sm    │ 14px (0.875rem)│ Secondary text, buttons  │
│ text-base  │ 16px (1rem)    │ Body text (default)      │
│ text-lg    │ 18px (1.125rem)│ Lead paragraphs          │
│ text-xl    │ 20px (1.25rem) │ H4, card titles          │
│ text-2xl   │ 24px (1.5rem)  │ H3                       │
│ text-3xl   │ 30px (1.875rem)│ H2                       │
│ text-4xl   │ 36px (2.25rem) │ H1                       │
│ text-5xl   │ 48px (3rem)    │ Hero headings            │
│ text-6xl   │ 60px (3.75rem) │ Display text             │
└─────────────────────────────────────────────────────────┘
```

### 🔄 So sánh CSS ↔ Tailwind

**CSS:**
```css
h1 { font-size: 36px; font-weight: 700; }
p { font-size: 16px; color: #4b5563; }
.small { font-size: 14px; }
```

**Tailwind:**
```html
<h1 class="text-4xl font-bold">Heading</h1>
<p class="text-base text-gray-600">Paragraph</p>
<span class="text-sm">Small text</span>
```

### 📐 Font Weight

```html
<p class="font-thin">100 - Thin</p>
<p class="font-light">300 - Light</p>
<p class="font-normal">400 - Normal (default)</p>
<p class="font-medium">500 - Medium ← HAY DÙNG cho buttons</p>
<p class="font-semibold">600 - Semibold ← HAY DÙNG cho headings</p>
<p class="font-bold">700 - Bold</p>
<p class="font-extrabold">800 - Extrabold</p>
<p class="font-black">900 - Black</p>

<!--
🔄 CSS tương đương:
.font-medium { font-weight: 500; }
.font-semibold { font-weight: 600; }
.font-bold { font-weight: 700; }
-->
```

### 📐 Text Alignment & Transform

```html
<!-- Alignment -->
<p class="text-left">Left aligned (default)</p>
<p class="text-center">Center aligned</p>
<p class="text-right">Right aligned</p>
<p class="text-justify">Justified text</p>

<!-- Transform -->
<p class="uppercase">UPPERCASE</p>
<p class="lowercase">lowercase</p>
<p class="capitalize">Capitalize Each Word</p>
<p class="normal-case">Normal Case (reset)</p>

<!-- Decoration -->
<p class="underline">Underlined text</p>
<p class="line-through">Strikethrough</p>
<p class="no-underline">No underline (reset, dùng cho links)</p>
```

### 🔥 Text Truncation (Cắt text dài)

```html
<!-- Single line truncate -->
<p class="truncate w-48">
  This is a very long text that will be truncated with ellipsis...
</p>
<!--
📐 Kết quả: "This is a very long te..."

🔄 CSS tương đương:
.truncate {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
-->


<!-- Multi-line clamp -->
<p class="line-clamp-2">
  This is a long paragraph that spans multiple lines.
  It will be clamped to show only 2 lines with an
  ellipsis at the end. The rest is hidden.
</p>
<!--
📐 Kết quả:
"This is a long paragraph that spans multiple lines.
It will be clamped to show only 2 lines with an..."

💡 line-clamp-1, line-clamp-2, line-clamp-3, etc.
-->
```

### 📐 Line Height (Leading)

```html
<p class="leading-none">Line height: 1 (tight)</p>
<p class="leading-tight">Line height: 1.25</p>
<p class="leading-snug">Line height: 1.375</p>
<p class="leading-normal">Line height: 1.5 (default)</p>
<p class="leading-relaxed">Line height: 1.625</p>
<p class="leading-loose">Line height: 2 (spacious)</p>

<!--
💡 Dùng leading-relaxed hoặc leading-loose cho body text dài
   để dễ đọc hơn.
-->
```

---

## 4. Sizing (Width & Height)

### 💡 Sizing Scale

Tailwind dùng cùng scale với spacing (số × 4px):

```html
<!-- Fixed width -->
<div class="w-4">16px</div>
<div class="w-8">32px</div>
<div class="w-16">64px</div>
<div class="w-32">128px</div>
<div class="w-64">256px</div>
<div class="w-96">384px</div>

<!-- Percentage width -->
<div class="w-1/2">50%</div>
<div class="w-1/3">33.333%</div>
<div class="w-2/3">66.666%</div>
<div class="w-1/4">25%</div>
<div class="w-3/4">75%</div>
<div class="w-full">100%</div>

<!-- Viewport width -->
<div class="w-screen">100vw (full viewport width)</div>

<!-- Content-based -->
<div class="w-auto">auto (default)</div>
<div class="w-fit">fit-content (shrink to content)</div>
<div class="w-min">min-content</div>
<div class="w-max">max-content</div>
```

### 🔥 Max-Width (Rất hay dùng!)

```html
<!--
💡 max-w-* giới hạn width tối đa, element vẫn responsive.
   Đây là cách tạo container trong Tailwind.
-->

<div class="max-w-xs">320px max</div>
<div class="max-w-sm">384px max</div>
<div class="max-w-md">448px max ← Forms, modals</div>
<div class="max-w-lg">512px max</div>
<div class="max-w-xl">576px max ← Cards</div>
<div class="max-w-2xl">672px max ← Content area</div>
<div class="max-w-4xl">896px max</div>
<div class="max-w-6xl">1152px max</div>
<div class="max-w-7xl">1280px max ← Page container</div>

<!-- Centered container pattern -->
<div class="max-w-4xl mx-auto px-4">
  Centered content with max width and padding
</div>
<!--
📐 Giải thích:
- max-w-4xl: không rộng quá 896px
- mx-auto: center horizontally
- px-4: padding 2 bên để không dính edge trên mobile
-->
```

### 📐 Height

```html
<div class="h-16">64px height</div>
<div class="h-32">128px height</div>
<div class="h-64">256px height</div>
<div class="h-full">100% of parent</div>
<div class="h-screen">100vh (full viewport height)</div>

<!-- Min height - hay dùng cho full page layouts -->
<div class="min-h-screen">
  At least full viewport height
</div>
```

---

## 5. Borders & Rounded

### 📐 Border Width

```html
<div class="border">1px border (default)</div>
<div class="border-2">2px border</div>
<div class="border-4">4px border</div>
<div class="border-8">8px border</div>

<!-- One side only -->
<div class="border-t">Top border only</div>
<div class="border-r">Right border only</div>
<div class="border-b">Bottom border only</div>
<div class="border-l">Left border only</div>
<div class="border-x-2">Left + Right 2px</div>
<div class="border-y">Top + Bottom 1px</div>

<!--
⚠️ QUAN TRỌNG: border-{color} không có nghĩa nếu không có border width!

❌ Sai:
<div class="border-gray-200">Không thấy border!</div>

✅ Đúng:
<div class="border border-gray-200">Có border!</div>
-->
```

### 📐 Border Radius

```
┌────────────────────────────────────────────────────────┐
│ CLASS        │ VALUE        │ VISUAL                   │
├────────────────────────────────────────────────────────┤
│ rounded-none │ 0            │ ┌──────┐                 │
│ rounded-sm   │ 2px          │ ╭──────╮ (barely)        │
│ rounded      │ 4px          │ ╭──────╮                 │
│ rounded-md   │ 6px          │ ╭──────╮ ← Buttons       │
│ rounded-lg   │ 8px          │ ╭──────╮ ← Cards         │
│ rounded-xl   │ 12px         │ ╭──────╮ ← Modern cards  │
│ rounded-2xl  │ 16px         │ ╭──────╮                 │
│ rounded-3xl  │ 24px         │ ╭──────╮                 │
│ rounded-full │ 9999px       │    ●    ← Circles/pills  │
└────────────────────────────────────────────────────────┘
```

```html
<!-- Per-corner control -->
<div class="rounded-t-lg">Top corners only</div>
<div class="rounded-b-lg">Bottom corners only</div>
<div class="rounded-l-lg">Left corners only</div>
<div class="rounded-r-lg">Right corners only</div>
<div class="rounded-tl-lg">Top-left only</div>

<!-- 🔥 Common patterns -->
<button class="rounded-md">Regular button</button>
<div class="rounded-lg">Card</div>
<img class="rounded-full" src="avatar.jpg" /> <!-- Circle avatar -->
<span class="rounded-full px-3 py-1">Pill badge</span>
```

---

## 6. Shadows

### 📐 Box Shadow Scale

```
┌────────────────────────────────────────────────────────┐
│ CLASS       │ USE CASE                                 │
├────────────────────────────────────────────────────────┤
│ shadow-sm   │ Subtle depth (inputs, small cards)       │
│ shadow      │ Default elevation                        │
│ shadow-md   │ Cards, dropdowns ← HAY DÙNG NHẤT         │
│ shadow-lg   │ Modals, popovers                         │
│ shadow-xl   │ Floating elements                        │
│ shadow-2xl  │ High elevation                           │
│ shadow-inner│ Inset shadow (pressed buttons)           │
│ shadow-none │ Remove shadow                            │
└────────────────────────────────────────────────────────┘
```

```html
<!-- 🔥 Practical examples -->

<!-- Card with shadow -->
<div class="bg-white rounded-lg shadow-md p-6">
  Card content
</div>

<!-- Elevated on hover -->
<div class="shadow-md hover:shadow-xl transition-shadow">
  Hover to elevate
</div>

<!--
🔄 CSS tương đương:
.shadow-md {
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1),
              0 2px 4px -2px rgba(0, 0, 0, 0.1);
}
-->
```

---

## 7. Ví dụ tổng hợp: Card Component

Hãy build một card từ đầu, giải thích từng class:

```html
<div class="max-w-sm mx-auto">
  <!--
  max-w-sm: Card không rộng quá 384px
  mx-auto: Center horizontally
  -->

  <div class="bg-white rounded-xl shadow-lg overflow-hidden">
    <!--
    bg-white: Background trắng
    rounded-xl: Border radius 12px (modern look)
    shadow-lg: Shadow lớn để card nổi lên
    overflow-hidden: Ẩn content tràn ra ngoài (quan trọng cho rounded corners)
    -->

    <!-- Image -->
    <img
      src="https://picsum.photos/400/200"
      alt="Card image"
      class="w-full h-48 object-cover"
    />
    <!--
    w-full: Image rộng 100% container
    h-48: Chiều cao cố định 192px
    object-cover: Crop image để fill, không bị méo
    -->

    <!-- Content -->
    <div class="p-6">
      <!-- p-6: Padding 24px tất cả phía -->

      <span class="text-xs font-semibold text-blue-600 uppercase tracking-wide">
        Category
      </span>
      <!--
      text-xs: Font size 12px
      font-semibold: Font weight 600
      text-blue-600: Màu xanh
      uppercase: CHỮ IN HOA
      tracking-wide: Letter spacing rộng hơn
      -->

      <h2 class="mt-2 text-xl font-bold text-gray-900">
        Card Title Here
      </h2>
      <!--
      mt-2: Margin top 8px (cách category)
      text-xl: Font size 20px
      font-bold: Font weight 700
      text-gray-900: Màu text đậm
      -->

      <p class="mt-2 text-gray-600 text-sm leading-relaxed">
        This is a description of the card. It explains what this
        card is about and provides useful information.
      </p>
      <!--
      mt-2: Margin top 8px
      text-gray-600: Màu text nhạt hơn title
      text-sm: Font size 14px
      leading-relaxed: Line height 1.625 (dễ đọc)
      -->

      <!-- Footer -->
      <div class="mt-4 flex items-center justify-between">
        <!--
        mt-4: Margin top 16px
        flex: Display flex
        items-center: Align items vertically center
        justify-between: Space between (price trái, button phải)
        -->

        <span class="text-lg font-bold text-gray-900">$99.00</span>

        <button class="bg-blue-500 text-white px-4 py-2 rounded-lg
                       font-medium hover:bg-blue-600 transition-colors">
          Buy Now
        </button>
        <!--
        bg-blue-500: Background xanh
        text-white: Text trắng
        px-4 py-2: Padding horizontal 16px, vertical 8px
        rounded-lg: Border radius 8px
        font-medium: Font weight 500
        hover:bg-blue-600: Hover thì background đậm hơn
        transition-colors: Smooth transition khi hover
        -->
      </div>
    </div>
  </div>
</div>
```

**Kết quả:**
```
┌────────────────────────────┐
│ [      IMAGE       ]       │
├────────────────────────────┤
│ CATEGORY                   │
│ Card Title Here            │
│                            │
│ This is a description...   │
│                            │
│ $99.00        [Buy Now]    │
└────────────────────────────┘
```

---

## Cheat Sheet — Day 1

```
SPACING
├── p-{n}, m-{n}         → padding/margin all sides
├── px-{n}, py-{n}       → x-axis (horizontal), y-axis (vertical)
├── pt, pr, pb, pl       → individual sides
├── mx-auto              → center horizontally
├── space-x-{n}          → horizontal gap between children
├── space-y-{n}          → vertical gap between children
└── gap-{n}              → gap for flex/grid (preferred)

COLORS
├── text-{color}-{shade} → text color
├── bg-{color}-{shade}   → background color
├── border-{color}-{shade} → border color (need border width!)
└── {color}/50           → opacity modifier (50%)

TYPOGRAPHY
├── text-{size}          → xs, sm, base, lg, xl, 2xl, 3xl...
├── font-{weight}        → thin, light, normal, medium, semibold, bold
├── text-{align}         → left, center, right, justify
├── uppercase/lowercase  → text transform
├── truncate             → single line ellipsis
└── line-clamp-{n}       → multi-line ellipsis

SIZING
├── w-{n}, h-{n}         → fixed width/height
├── w-full, h-full       → 100%
├── w-screen, h-screen   → 100vw, 100vh
├── max-w-{size}         → max-width constraint
└── min-h-screen         → at least full viewport

BORDERS
├── border, border-2     → border width (required!)
├── border-{color}       → border color
├── rounded-{size}       → border radius
└── rounded-full         → circle/pill

SHADOWS
├── shadow-sm            → subtle
├── shadow-md            → cards (most used)
├── shadow-lg            → modals
└── shadow-xl            → floating elements
```

---

## Bài tập thực hành

### Bài 1: Profile Card
Tạo card có:
- Avatar tròn (rounded-full)
- Tên (text-xl, font-bold)
- Bio (text-gray-600, text-sm)
- 2 buttons: Follow (primary) và Message (secondary)

### Bài 2: Pricing Card
Tạo pricing card có:
- Plan name
- Price lớn ($99/month)
- Feature list với checkmarks
- CTA button full width

### Bài 3: Alert Component
Tạo 4 loại alert:
- Success (green)
- Error (red)
- Warning (amber/yellow)
- Info (blue)

Mỗi alert có icon và text.

---

## Navigation

- [← Overview](./00-overview.md)
- [Day 2: Layout →](./day-2-layout.md)
