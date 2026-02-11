# Day 3: Responsive Design + States

> Mục tiêu: Master responsive breakpoints, hover/focus states, và dark mode.

---

## 1. Responsive Design

### 💡 Mobile-First là gì?

Tailwind sử dụng **Mobile-First** approach: styles mặc định apply cho màn hình nhỏ nhất, sau đó thêm styles cho màn hình lớn hơn.

```
┌────────────────────────────────────────────────────────────┐
│ MOBILE-FIRST THINKING                                      │
│                                                            │
│ 1. Viết styles cho mobile TRƯỚC (không có prefix)          │
│ 2. Thêm styles cho tablet với prefix md:                   │
│ 3. Thêm styles cho desktop với prefix lg:                  │
│                                                            │
│ ❌ SAI: "Ẩn element này trên mobile"                       │
│    → Viết: class="hidden" (ẩn mặc định)                    │
│           rồi: class="hidden md:block" (hiện từ tablet)    │
│                                                            │
│ ✅ ĐÚNG: "Hiện element này từ tablet trở lên"              │
│    → Viết: class="hidden md:block"                         │
└────────────────────────────────────────────────────────────┘
```

### 📐 Breakpoint System

```
┌─────────────────────────────────────────────────────────────┐
│ PREFIX │ MIN-WIDTH │ THIẾT BỊ        │ CSS                  │
├─────────────────────────────────────────────────────────────┤
│ (none) │ 0px       │ Mobile          │ Default styles       │
│ sm:    │ 640px     │ Large phones    │ @media (min-width: 640px)  │
│ md:    │ 768px     │ Tablets         │ @media (min-width: 768px)  │
│ lg:    │ 1024px    │ Laptops         │ @media (min-width: 1024px) │
│ xl:    │ 1280px    │ Desktops        │ @media (min-width: 1280px) │
│ 2xl:   │ 1536px    │ Large screens   │ @media (min-width: 1536px) │
└─────────────────────────────────────────────────────────────┘

💡 Prefix có nghĩa: "Từ breakpoint này TRỞ LÊN"
   md:flex = "Từ 768px trở lên, apply display: flex"
```

### 🔄 So sánh CSS ↔ Tailwind

**CSS Media Queries:**
```css
.card {
  padding: 16px;
}

@media (min-width: 768px) {
  .card {
    padding: 24px;
  }
}

@media (min-width: 1024px) {
  .card {
    padding: 32px;
  }
}
```

**Tailwind:**
```html
<div class="p-4 md:p-6 lg:p-8">
  Card content
</div>
<!--
📐 Phân tích:
- Mobile (< 768px): padding 16px
- Tablet (≥ 768px): padding 24px
- Desktop (≥ 1024px): padding 32px
-->
```

### 📐 Responsive Patterns chi tiết

#### Pattern 1: Font size responsive

```html
<h1 class="text-2xl md:text-4xl lg:text-6xl font-bold">
  Responsive Heading
</h1>
<!--
📐 Kết quả:
Mobile:  24px (text-2xl)
Tablet:  36px (text-4xl)
Desktop: 60px (text-6xl)

💡 Headings thường cần lớn hơn nhiều trên desktop
   vì có nhiều không gian hơn.
-->
```

#### Pattern 2: Layout direction responsive

```html
<div class="flex flex-col md:flex-row gap-4">
  <div class="md:w-1/3">Sidebar</div>
  <div class="md:w-2/3">Main content</div>
</div>
<!--
📐 Mobile (< 768px):
┌─────────────────┐
│ Sidebar         │  ← Stack dọc (flex-col)
├─────────────────┤
│ Main content    │
└─────────────────┘

📐 Tablet+ (≥ 768px):
┌───────┬─────────────────┐
│Sidebar│ Main content    │  ← Ngang (flex-row)
│  1/3  │      2/3        │
└───────┴─────────────────┘
-->
```

#### Pattern 3: Grid columns responsive

```html
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
  <div class="bg-white p-4 rounded-lg shadow">Card 1</div>
  <div class="bg-white p-4 rounded-lg shadow">Card 2</div>
  <div class="bg-white p-4 rounded-lg shadow">Card 3</div>
  <div class="bg-white p-4 rounded-lg shadow">Card 4</div>
</div>
<!--
📐 Kết quả theo màn hình:

Mobile (< 640px):     Small (≥ 640px):
┌───────────────┐     ┌───────┬───────┐
│     Card 1    │     │Card 1 │Card 2 │
├───────────────┤     ├───────┼───────┤
│     Card 2    │     │Card 3 │Card 4 │
├───────────────┤     └───────┴───────┘
│     Card 3    │
├───────────────┤     Large (≥ 1024px):
│     Card 4    │     ┌─────┬─────┬─────┐
└───────────────┘     │ C1  │ C2  │ C3  │
                      ├─────┴─────┴─────┤
                      │       C4        │
                      └─────────────────┘

                      XL (≥ 1280px):
                      ┌────┬────┬────┬────┐
                      │ C1 │ C2 │ C3 │ C4 │
                      └────┴────┴────┴────┘
-->
```

#### Pattern 4: Show/Hide elements

```html
<!-- Hiện trên mobile, ẩn từ tablet -->
<div class="block md:hidden">
  Mobile menu button
</div>
<!--
📐 Mobile: Visible
   Tablet+: Hidden
-->


<!-- Ẩn trên mobile, hiện từ tablet -->
<nav class="hidden md:flex gap-6">
  <a href="#">Home</a>
  <a href="#">About</a>
  <a href="#">Contact</a>
</nav>
<!--
📐 Mobile: Hidden
   Tablet+: Visible as flex
-->


<!-- Chỉ hiện trên tablet (không mobile, không desktop) -->
<div class="hidden md:block lg:hidden">
  Tablet only content
</div>
<!--
📐 Mobile: Hidden
   Tablet: Visible
   Desktop+: Hidden
-->
```

#### Pattern 5: Responsive spacing

```html
<section class="py-12 md:py-16 lg:py-24">
  <div class="px-4 md:px-6 lg:px-8">
    <h2 class="mb-4 md:mb-6 lg:mb-8">Section Title</h2>
    <p>Content here...</p>
  </div>
</section>
<!--
💡 Spacing tăng dần theo screen size:
   - Nhiều không gian hơn → padding/margin lớn hơn
   - Tạo cảm giác "thoáng" trên màn hình lớn
-->
```

#### Pattern 6: Responsive container

```html
<div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
  <!-- Content -->
</div>
<!--
📐 Phân tích:
- max-w-7xl: Không rộng quá 1280px
- mx-auto: Center horizontally
- px-4: Mobile có padding 16px 2 bên
- sm:px-6: Từ 640px có padding 24px
- lg:px-8: Từ 1024px có padding 32px

💡 Padding tăng dần để content không dính edge
-->
```

---

## 2. State Modifiers

### 💡 States là gì?

States là những trạng thái của element khi user tương tác (hover, focus, click...) hoặc trạng thái của form (disabled, invalid...).

### 📐 Hover, Focus, Active

```html
<!-- Hover: khi rê chuột lên -->
<button class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded">
  Hover me
</button>
<!--
🔄 CSS tương đương:
.button {
  background-color: #3b82f6;
}
.button:hover {
  background-color: #2563eb;
}
-->


<!-- Focus: khi element được focus (click vào input, tab đến button) -->
<input
  type="text"
  class="border border-gray-300 px-4 py-2 rounded
         focus:border-blue-500 focus:ring-2 focus:ring-blue-500/20
         focus:outline-none"
  placeholder="Focus on me"
/>
<!--
📐 Khi focus:
- Border chuyển xanh
- Thêm ring (glow effect) màu xanh nhạt
- Bỏ outline mặc định của browser

🔄 CSS tương đương:
input:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.2);
  outline: none;
}
-->


<!-- Active: khi đang click/press -->
<button class="bg-blue-500 hover:bg-blue-600 active:bg-blue-700
               active:scale-95 transition-all text-white px-4 py-2 rounded">
  Click me
</button>
<!--
📐 Khi click:
- Background đậm hơn nữa
- Scale nhỏ lại 95% (hiệu ứng "nhấn")
-->


<!-- Focus-visible: chỉ khi focus bằng keyboard (không phải click) -->
<button class="focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500">
  Keyboard accessible
</button>
<!--
💡 focus-visible giúp:
- Click bằng chuột: không có ring (cleaner UI)
- Tab bằng keyboard: có ring (accessibility)
-->
```

### 📐 Form States

```html
<!-- Disabled -->
<button
  class="bg-blue-500 text-white px-4 py-2 rounded
         disabled:bg-gray-400 disabled:cursor-not-allowed"
  disabled
>
  Disabled Button
</button>
<!--
📐 Khi disabled:
- Background xám
- Cursor hiện "not-allowed"
-->


<!-- Invalid (HTML5 validation) -->
<input
  type="email"
  class="border px-4 py-2 rounded
         invalid:border-red-500 invalid:text-red-600"
  value="not-an-email"
/>
<!--
📐 Khi giá trị invalid:
- Border đỏ
- Text đỏ

💡 Browser tự validate dựa trên type="email"
-->


<!-- Required -->
<input
  type="text"
  class="border px-4 py-2 rounded required:border-red-500"
  required
/>


<!-- Placeholder shown -->
<input
  class="border px-4 py-2 rounded
         placeholder-shown:border-gray-300
         not-placeholder-shown:border-green-500"
  placeholder="Type something..."
/>
<!--
📐
- Còn placeholder: border xám
- Có value (no placeholder): border xanh lá
-->
```

### 📐 List Item States: first, last, odd, even

```html
<ul class="divide-y">
  <li class="py-4 first:pt-0 last:pb-0">Item 1</li>
  <li class="py-4 first:pt-0 last:pb-0">Item 2</li>
  <li class="py-4 first:pt-0 last:pb-0">Item 3</li>
</ul>
<!--
📐
- first:pt-0 → Item đầu không có padding-top
- last:pb-0 → Item cuối không có padding-bottom
- Giúp danh sách không có spacing thừa
-->


<!-- Zebra stripes table -->
<table class="w-full">
  <tbody>
    <tr class="odd:bg-white even:bg-gray-50">
      <td class="p-4">Row 1</td>
    </tr>
    <tr class="odd:bg-white even:bg-gray-50">
      <td class="p-4">Row 2</td>
    </tr>
    <tr class="odd:bg-white even:bg-gray-50">
      <td class="p-4">Row 3</td>
    </tr>
  </tbody>
</table>
<!--
📐
Row 1: white background (odd)
Row 2: gray background (even)
Row 3: white background (odd)
...
-->
```

### 📐 Group & Peer Modifiers

#### Group: Style child dựa trên parent state

```html
<div class="group cursor-pointer p-4 rounded-lg border
            hover:bg-gray-50 hover:border-blue-500 transition-all">
  <h3 class="font-semibold text-gray-900 group-hover:text-blue-600">
    Card Title
  </h3>
  <p class="text-gray-500 group-hover:text-gray-700">
    Card description that changes on hover
  </p>
  <span class="text-blue-600 opacity-0 group-hover:opacity-100 transition-opacity">
    Read more →
  </span>
</div>
<!--
📐 Khi hover vào CARD (parent):
- Card background → gray-50
- Card border → blue
- Title → blue (group-hover:text-blue-600)
- Description → darker
- "Read more" → xuất hiện

💡 class="group" trên parent
   class="group-hover:..." trên children

🔄 CSS tương đương:
.group:hover .title { color: blue; }
.group:hover .description { color: darker; }
-->
```

#### Peer: Style sibling dựa trên sibling khác

```html
<div class="relative">
  <input
    type="email"
    id="email"
    class="peer border px-4 py-2 rounded w-full
           focus:border-blue-500"
    placeholder=" "
  />
  <label
    for="email"
    class="absolute left-4 top-2 text-gray-500 transition-all
           peer-placeholder-shown:top-2 peer-placeholder-shown:text-base
           peer-focus:top-0 peer-focus:text-xs peer-focus:text-blue-500
           peer-focus:-translate-y-full peer-focus:bg-white peer-focus:px-1"
  >
    Email address
  </label>
</div>
<!--
📐 Floating label pattern:
1. Input trống (placeholder shown): Label ở vị trí bình thường
2. Input focus: Label bay lên trên, nhỏ lại, đổi màu

💡 class="peer" trên input
   class="peer-focus:..." trên label

⚠️ QUAN TRỌNG: peer element PHẢI đứng TRƯỚC element cần style
-->


<!-- Peer cho validation message -->
<div>
  <input
    type="email"
    class="peer border px-4 py-2 rounded"
    placeholder="Enter email"
  />
  <p class="invisible peer-invalid:visible text-red-500 text-sm mt-1">
    Please enter a valid email address
  </p>
</div>
<!--
📐
- Input valid: Error message invisible
- Input invalid: Error message visible
-->
```

### 📐 Before & After Pseudo-elements

```html
<!-- Required field asterisk -->
<label class="after:content-['*'] after:ml-0.5 after:text-red-500">
  Email
</label>
<!--
📐 Kết quả: "Email *" với dấu * màu đỏ

🔄 CSS tương đương:
label::after {
  content: '*';
  margin-left: 2px;
  color: red;
}
-->


<!-- Decorative line under heading -->
<h2 class="relative pb-4
           after:absolute after:bottom-0 after:left-0
           after:w-16 after:h-1 after:bg-blue-500 after:rounded">
  Section Title
</h2>
<!--
📐 Kết quả:
Section Title
────────      ← Blue underline (16px wide)
-->
```

---

## 3. Dark Mode

### 💡 Setup Dark Mode (Tailwind v4)

Tailwind v4 hỗ trợ 2 cách bật dark mode:

**Cách 1: Media Query (Mặc định)** - Tự động theo OS preference:
```css
/* Không cần config gì - đây là mặc định */
@import "tailwindcss";
```

**Cách 2: Class-based (Toggle bằng JS)** - Khuyến khích:
```css
/* styles.css */
@import "tailwindcss";

/* Override dark variant để dùng class selector */
@custom-variant dark (&:where(.dark, .dark *));
```

```
┌─────────────────────────────────────────────────────────────┐
│ MODE              │ CÁCH HOẠT ĐỘNG                          │
├─────────────────────────────────────────────────────────────┤
│ Media (default)   │ Tự động theo OS setting                 │
│                   │ prefers-color-scheme: dark              │
│                   │ User không thể toggle                   │
├─────────────────────────────────────────────────────────────┤
│ Class-based       │ Bật khi có class "dark" trên <html>     │
│ (@custom-variant) │ User có thể toggle (cần JavaScript)     │
│                   │ ← KHUYẾN KHÍCH: kiểm soát được          │
└─────────────────────────────────────────────────────────────┘

💡 Thay đổi từ v3: Không còn dùng tailwind.config.js
   Thay vào đó dùng @custom-variant trong CSS
```

### 📐 Sử dụng Dark Mode

```html
<!-- Thêm class "dark" vào html để bật dark mode -->
<html class="dark">

<!-- Element với dark mode styles -->
<div class="bg-white dark:bg-gray-900">
  <h1 class="text-gray-900 dark:text-white">
    Heading
  </h1>
  <p class="text-gray-600 dark:text-gray-400">
    Paragraph text
  </p>
</div>
<!--
📐 Light mode (không có class "dark"):
- Background: white
- Heading: dark gray
- Paragraph: medium gray

📐 Dark mode (có class "dark"):
- Background: very dark gray
- Heading: white
- Paragraph: light gray
-->
```

### 📐 Dark Mode Color Mapping

```
┌─────────────────────────────────────────────────────────────┐
│ ELEMENT          │ LIGHT MODE     │ DARK MODE               │
├─────────────────────────────────────────────────────────────┤
│ Page background  │ white, gray-50 │ gray-900, gray-950      │
│ Card background  │ white          │ gray-800                │
│ Primary text     │ gray-900       │ white, gray-100         │
│ Secondary text   │ gray-600       │ gray-400                │
│ Muted text       │ gray-400       │ gray-500                │
│ Borders          │ gray-200       │ gray-700                │
│ Dividers         │ gray-100       │ gray-800                │
│ Hover background │ gray-50        │ gray-800                │
│ Primary button   │ blue-500       │ blue-600                │
│ Button hover     │ blue-600       │ blue-700                │
└─────────────────────────────────────────────────────────────┘
```

### 🔥 Dark Mode Complete Example

```html
<!-- Card with full dark mode support -->
<div class="
  bg-white dark:bg-gray-800
  border border-gray-200 dark:border-gray-700
  rounded-xl shadow-lg dark:shadow-gray-900/30
  p-6
">
  <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
    Card Title
  </h3>

  <p class="mt-2 text-gray-600 dark:text-gray-400">
    This is the card description with proper contrast
    in both light and dark modes.
  </p>

  <div class="mt-4 flex gap-3">
    <button class="
      bg-blue-500 dark:bg-blue-600
      hover:bg-blue-600 dark:hover:bg-blue-700
      text-white px-4 py-2 rounded-lg
      transition-colors
    ">
      Primary
    </button>

    <button class="
      bg-gray-100 dark:bg-gray-700
      hover:bg-gray-200 dark:hover:bg-gray-600
      text-gray-900 dark:text-white
      px-4 py-2 rounded-lg
      transition-colors
    ">
      Secondary
    </button>
  </div>
</div>
```

### 📐 Dark Mode Toggle với JavaScript

```html
<button id="theme-toggle" class="p-2 rounded-lg
       bg-gray-100 dark:bg-gray-800
       hover:bg-gray-200 dark:hover:bg-gray-700">
  <!-- Sun icon (show in dark mode) -->
  <svg class="w-6 h-6 hidden dark:block text-yellow-400">
    <!-- sun path -->
  </svg>
  <!-- Moon icon (show in light mode) -->
  <svg class="w-6 h-6 block dark:hidden text-gray-700">
    <!-- moon path -->
  </svg>
</button>

<script>
  const toggle = document.getElementById('theme-toggle');

  // Toggle dark mode
  toggle.addEventListener('click', () => {
    document.documentElement.classList.toggle('dark');

    // Save preference
    if (document.documentElement.classList.contains('dark')) {
      localStorage.theme = 'dark';
    } else {
      localStorage.theme = 'light';
    }
  });

  // On page load, check saved preference
  if (localStorage.theme === 'dark' ||
      (!('theme' in localStorage) &&
       window.matchMedia('(prefers-color-scheme: dark)').matches)) {
    document.documentElement.classList.add('dark');
  } else {
    document.documentElement.classList.remove('dark');
  }
</script>
```

---

## 4. Stacking Modifiers

Bạn có thể kết hợp nhiều modifiers với nhau:

```html
<!-- Responsive + State -->
<button class="
  bg-blue-500 hover:bg-blue-600
  md:bg-green-500 md:hover:bg-green-600
">
  Blue on mobile, green on tablet+
</button>
<!--
📐 Mobile: blue → hover: darker blue
   Tablet+: green → hover: darker green
-->


<!-- Dark + State -->
<button class="
  bg-white hover:bg-gray-100
  dark:bg-gray-800 dark:hover:bg-gray-700
">
  Different colors in dark mode
</button>


<!-- Responsive + Dark + State -->
<div class="
  p-4 md:p-6 lg:p-8
  bg-white dark:bg-gray-800
  hover:bg-gray-50 dark:hover:bg-gray-700
  text-gray-900 dark:text-white
">
  All the modifiers!
</div>


<!-- Group + Dark -->
<div class="group">
  <span class="
    text-gray-600 group-hover:text-blue-600
    dark:text-gray-400 dark:group-hover:text-blue-400
  ">
    Group hover in dark mode
  </span>
</div>
```

---

## 5. Ví dụ tổng hợp: Responsive Navbar với Dark Mode

```html
<nav class="
  bg-white dark:bg-gray-900
  border-b border-gray-200 dark:border-gray-800
  sticky top-0 z-50
">
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
    <div class="flex items-center justify-between h-16">

      <!-- Logo -->
      <a href="#" class="text-xl font-bold text-gray-900 dark:text-white">
        Logo
      </a>

      <!-- Desktop Navigation (hidden on mobile) -->
      <div class="hidden md:flex items-center gap-8">
        <a href="#" class="text-gray-600 dark:text-gray-300
                          hover:text-gray-900 dark:hover:text-white
                          transition-colors">
          Home
        </a>
        <a href="#" class="text-gray-600 dark:text-gray-300
                          hover:text-gray-900 dark:hover:text-white
                          transition-colors">
          Features
        </a>
        <a href="#" class="text-gray-600 dark:text-gray-300
                          hover:text-gray-900 dark:hover:text-white
                          transition-colors">
          Pricing
        </a>
      </div>

      <!-- Right side -->
      <div class="flex items-center gap-4">
        <!-- Dark mode toggle -->
        <button id="theme-toggle" class="
          p-2 rounded-lg
          text-gray-500 dark:text-gray-400
          hover:bg-gray-100 dark:hover:bg-gray-800
          transition-colors
        ">
          🌙
        </button>

        <!-- CTA Button (hidden on small mobile) -->
        <button class="
          hidden sm:block
          bg-blue-500 dark:bg-blue-600
          hover:bg-blue-600 dark:hover:bg-blue-700
          text-white px-4 py-2 rounded-lg
          transition-colors
        ">
          Get Started
        </button>

        <!-- Mobile menu button (visible on mobile only) -->
        <button class="md:hidden p-2">
          <svg class="w-6 h-6 text-gray-900 dark:text-white" fill="none" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M4 6h16M4 12h16M4 18h16"/>
          </svg>
        </button>
      </div>
    </div>

    <!-- Mobile Navigation (hidden by default) -->
    <div class="md:hidden hidden" id="mobile-menu">
      <div class="py-4 space-y-2 border-t border-gray-200 dark:border-gray-800">
        <a href="#" class="block py-2 text-gray-600 dark:text-gray-300
                          hover:text-gray-900 dark:hover:text-white">
          Home
        </a>
        <a href="#" class="block py-2 text-gray-600 dark:text-gray-300
                          hover:text-gray-900 dark:hover:text-white">
          Features
        </a>
        <a href="#" class="block py-2 text-gray-600 dark:text-gray-300
                          hover:text-gray-900 dark:hover:text-white">
          Pricing
        </a>
        <button class="w-full mt-4 bg-blue-500 text-white py-2 rounded-lg">
          Get Started
        </button>
      </div>
    </div>
  </div>
</nav>

<!--
📐 BREAKDOWN:

1. RESPONSIVE:
   - Logo: always visible
   - Desktop nav: hidden md:flex (ẩn mobile, hiện tablet+)
   - Mobile menu btn: md:hidden (hiện mobile, ẩn tablet+)
   - CTA button: hidden sm:block (ẩn tiny screens)

2. DARK MODE:
   - Background: white/gray-900
   - Border: gray-200/gray-800
   - Text: gray-600/gray-300 → hover: gray-900/white

3. STATES:
   - Links: hover colors
   - Buttons: hover backgrounds
   - Sticky: top-0 z-50
-->
```

---

## Cheat Sheet — Day 3

```
RESPONSIVE (Mobile-first)
├── (none)    → default (all screens)
├── sm:      → ≥ 640px
├── md:      → ≥ 768px
├── lg:      → ≥ 1024px
├── xl:      → ≥ 1280px
└── 2xl:     → ≥ 1536px

STATES
├── hover:        → mouse over
├── focus:        → focused (click/tab)
├── focus-visible: → keyboard focus only
├── active:       → being clicked
├── disabled:     → disabled elements
├── invalid:      → invalid form inputs
├── first:, last: → first/last child
├── odd:, even:   → zebra striping
├── group-hover:  → child reacts to parent hover
└── peer-*:       → sibling reacts to sibling state

DARK MODE
├── dark:           → dark mode variant
├── darkMode: 'class' → toggle with class="dark" on <html>
└── darkMode: 'media' → follow OS preference

STACKING
├── md:hover:bg-gray-100      → responsive + state
├── dark:hover:bg-gray-700    → dark + state
└── md:dark:hover:bg-gray-600 → all combined
```

---

## Bài tập

### Bài 1: Responsive Hero Section
Tạo hero với:
- Text trái, image phải (desktop)
- Stacked layout (mobile)
- Font size responsive
- Dark mode support

### Bài 2: Interactive Card
Card với:
- Hover: lift effect (shadow-xl, -translate-y-1)
- Group hover: title đổi màu, icon xuất hiện
- Dark mode colors

### Bài 3: Form với States
Form có:
- Focus ring trên inputs
- Invalid state với error message (peer)
- Disabled button state
- Dark mode styling

---

## Navigation

- [← Day 2: Layout](./day-2-layout.md)
- [Day 4: Components →](./day-4-components.md)
