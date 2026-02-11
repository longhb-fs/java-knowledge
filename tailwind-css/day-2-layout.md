# Day 2: Layout — Flexbox + Grid + Positioning

> Mục tiêu: Master cách bố trí layout với Flexbox và Grid trong Tailwind.

---

## 1. Display

Trước khi vào Flex/Grid, cần hiểu các giá trị display cơ bản:

```html
<div class="block">Block - chiếm full width, xuống dòng</div>
<span class="inline">Inline - không xuống dòng, width theo content</span>
<span class="inline-block">Inline-block - inline nhưng có thể set width/height</span>
<div class="hidden">Hidden - display: none (ẩn hoàn toàn)</div>

<!--
🔄 CSS tương đương:
.block { display: block; }
.inline { display: inline; }
.inline-block { display: inline-block; }
.hidden { display: none; }
-->
```

---

## 2. Flexbox

### 💡 Flexbox là gì?

Flexbox giúp bạn sắp xếp các items **theo 1 chiều** (ngang HOẶC dọc) một cách linh hoạt.

```
┌─────────────────────────────────────────────────────────┐
│ FLEX CONTAINER (parent)                                 │
│ ┌──────────────────────────────────────────────────────┐│
│ │   [Item 1]    [Item 2]    [Item 3]                  ││
│ │                                                      ││
│ │   ← ─ ─ ─ ─ ─ MAIN AXIS (default: horizontal) ─ ─ → ││
│ │                                                      ││
│ │              ↕ CROSS AXIS (vertical)                 ││
│ └──────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### 🔄 So sánh CSS ↔ Tailwind: Flexbox

**CSS truyền thống:**
```css
.container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
}
```

**Tailwind:**
```html
<div class="flex justify-between items-center gap-4">
  <!-- flex items here -->
</div>
```

### 📐 Bật Flex và Direction

```html
<!-- Bật flexbox -->
<div class="flex">
  <div>Item 1</div>
  <div>Item 2</div>
  <div>Item 3</div>
</div>

<!--
📐 Kết quả mặc định (flex-row):
[Item 1] [Item 2] [Item 3]   ← Items nằm ngang
-->


<!-- Flex direction -->
<div class="flex flex-row">Ngang (default) →</div>
<div class="flex flex-row-reverse">Ngang ngược ←</div>
<div class="flex flex-col">Dọc ↓</div>
<div class="flex flex-col-reverse">Dọc ngược ↑</div>

<!--
📐 flex-col:
┌─────────┐
│ Item 1  │
│ Item 2  │
│ Item 3  │
└─────────┘
-->
```

### 📐 Justify Content (Căn theo Main Axis)

```
💡 MAIN AXIS là hướng items xếp theo:
   - flex-row: Main axis = NGANG (←→)
   - flex-col: Main axis = DỌC (↕)

justify-* điều khiển căn chỉnh theo MAIN AXIS
```

```html
<!-- justify-start (default) -->
<div class="flex justify-start">
  <div>A</div>
  <div>B</div>
  <div>C</div>
</div>
<!--
📐 |[A][B][C]          |
         ↑ Items dồn về đầu
-->


<!-- justify-center -->
<div class="flex justify-center">
  <div>A</div>
  <div>B</div>
  <div>C</div>
</div>
<!--
📐 |     [A][B][C]     |
         ↑ Items ở giữa
-->


<!-- justify-end -->
<div class="flex justify-end">
  <div>A</div>
  <div>B</div>
  <div>C</div>
</div>
<!--
📐 |          [A][B][C]|
              ↑ Items dồn về cuối
-->


<!-- justify-between -->
<div class="flex justify-between">
  <div>A</div>
  <div>B</div>
  <div>C</div>
</div>
<!--
📐 |[A]      [B]      [C]|
      ↑ Item đầu và cuối dính edge, còn lại chia đều

🔥 HAY DÙNG cho navbar: Logo trái, menu giữa, button phải
-->


<!-- justify-around -->
<div class="flex justify-around">
  <div>A</div>
  <div>B</div>
  <div>C</div>
</div>
<!--
📐 |  [A]    [B]    [C]  |
      ↑ Khoảng cách 2 bên mỗi item bằng nhau
-->


<!-- justify-evenly -->
<div class="flex justify-evenly">
  <div>A</div>
  <div>B</div>
  <div>C</div>
</div>
<!--
📐 |   [A]   [B]   [C]   |
        ↑ Tất cả khoảng cách đều bằng nhau
-->
```

### 📐 Align Items (Căn theo Cross Axis)

```
💡 CROSS AXIS vuông góc với Main Axis:
   - flex-row: Cross axis = DỌC (↕)
   - flex-col: Cross axis = NGANG (←→)

items-* điều khiển căn chỉnh theo CROSS AXIS
```

```html
<!-- Container cần có height để thấy hiệu ứng -->
<div class="flex items-start h-32 bg-gray-100">
  <div class="bg-blue-500 p-4">Short</div>
  <div class="bg-blue-500 p-4">Medium<br>text</div>
  <div class="bg-blue-500 p-4">Tall<br>text<br>here</div>
</div>
<!--
📐 items-start:
┌─────────────────────────────────┐
│[Short][Medium ][Tall  ]         │
│       [text   ][text  ]         │
│                [here  ]         │
│                                 │
└─────────────────────────────────┘
↑ Items căn trên
-->


<div class="flex items-center h-32 bg-gray-100">
  <!-- same content -->
</div>
<!--
📐 items-center:
┌─────────────────────────────────┐
│                                 │
│[Short][Medium ][Tall  ]         │
│       [text   ][text  ]         │
│                [here  ]         │
└─────────────────────────────────┘
↑ Items căn giữa (vertical center)
-->


<div class="flex items-end h-32 bg-gray-100">
  <!-- same content -->
</div>
<!--
📐 items-end:
┌─────────────────────────────────┐
│                                 │
│                [Tall  ]         │
│       [Medium ][text  ]         │
│[Short][text   ][here  ]         │
└─────────────────────────────────┘
↑ Items căn dưới
-->


<div class="flex items-stretch h-32 bg-gray-100">
  <!-- same content -->
</div>
<!--
📐 items-stretch (default):
┌─────────────────────────────────┐
│[Short][Medium ][Tall  ]         │
│       [text   ][text  ]         │
│                [here  ]         │
│ ↑      ↑       ↑                │
└─────────────────────────────────┘
↑ Items stretch để fill height
-->
```

### 📐 Gap (Khoảng cách giữa items)

```html
<div class="flex gap-4">
  <div>A</div>
  <div>B</div>
  <div>C</div>
</div>
<!--
📐 [A]──16px──[B]──16px──[C]

🔄 CSS tương đương:
.container { gap: 16px; }
-->


<!-- Gap khác nhau cho x và y -->
<div class="flex flex-wrap gap-x-4 gap-y-2">
  <!-- items -->
</div>
<!--
💡 gap-x-4 = column-gap: 16px (horizontal)
   gap-y-2 = row-gap: 8px (vertical)

   Hữu ích khi items wrap xuống dòng mới
-->
```

### 📐 Flex Wrap

```html
<!-- Mặc định: nowrap (items bị nén lại) -->
<div class="flex flex-nowrap">
  <!-- items will shrink to fit -->
</div>

<!-- Wrap: items xuống dòng khi hết chỗ -->
<div class="flex flex-wrap">
  <!-- items wrap to next line -->
</div>
<!--
📐 Container 300px, mỗi item 100px:

flex-nowrap:
|[A][B][C]| ← Items bị nén vào 300px

flex-wrap:
|[A][B][C]|
|[D][E]   |  ← Items wrap xuống dòng
-->
```

### 📐 Flex Item Properties

```html
<div class="flex">
  <!-- flex-1: Grow để fill không gian còn lại -->
  <div class="flex-1 bg-blue-200">Grows</div>
  <div class="w-32 bg-blue-400">Fixed 128px</div>
</div>
<!--
📐 |[  Grows (fills remaining)  ][Fixed 128px]|

💡 flex-1 = flex: 1 1 0%
   Nghĩa là: grow=1, shrink=1, basis=0
-->


<!-- flex-none: Không grow, không shrink -->
<div class="flex">
  <div class="flex-none w-32">Fixed</div>
  <div class="flex-1">Grows</div>
</div>
<!--
💡 flex-none = flex: none
   Item giữ nguyên kích thước
-->


<!-- flex-auto: Grow và shrink, basis=auto -->
<div class="flex">
  <div class="flex-auto">Auto 1</div>
  <div class="flex-auto">Auto 2 with more content</div>
</div>
<!--
💡 flex-auto = flex: 1 1 auto
   Tương tự flex-1 nhưng basis theo content
-->


<!-- flex-grow / flex-shrink riêng lẻ -->
<div class="flex">
  <div class="flex-grow">Will grow</div>
  <div class="flex-grow-0">Won't grow</div>
</div>
```

### 📐 Self Alignment (Override cho từng item)

```html
<div class="flex items-start h-32">
  <div>Follows container (start)</div>
  <div class="self-center">I'm centered!</div>
  <div class="self-end">I'm at the end!</div>
</div>
<!--
📐
┌─────────────────────────────────────────┐
│[Follows]                                │
│          [I'm centered!]                │
│                          [I'm at end!]  │
└─────────────────────────────────────────┘
-->
```

### 🔥 Flexbox Patterns phổ biến

```html
<!-- 🔥 Pattern 1: Center everything -->
<div class="flex items-center justify-center h-screen">
  <div>Perfectly centered on page</div>
</div>
<!--
💡 Đây là cách đơn giản nhất để center cả horizontal và vertical
-->


<!-- 🔥 Pattern 2: Navbar -->
<nav class="flex items-center justify-between px-6 py-4">
  <div class="font-bold text-xl">Logo</div>
  <div class="flex gap-6">
    <a href="#">Home</a>
    <a href="#">About</a>
    <a href="#">Contact</a>
  </div>
  <button class="bg-blue-500 text-white px-4 py-2 rounded">Login</button>
</nav>
<!--
📐 |Logo      Home About Contact      [Login]|
     ↑              ↑                    ↑
   start          center               end
-->


<!-- 🔥 Pattern 3: Card footer pushed to bottom -->
<div class="flex flex-col h-64 p-4 bg-white rounded-lg shadow">
  <h3 class="font-bold">Card Title</h3>
  <p class="text-gray-600">Some content here...</p>
  <div class="mt-auto">  <!-- mt-auto pushes footer to bottom -->
    <button class="w-full bg-blue-500 text-white py-2 rounded">
      Action
    </button>
  </div>
</div>
<!--
📐
┌────────────────────┐
│ Card Title         │
│ Some content...    │
│                    │
│                    │
│ [    Action     ]  │ ← mt-auto đẩy xuống đáy
└────────────────────┘
-->


<!-- 🔥 Pattern 4: Equal width columns -->
<div class="flex gap-4">
  <div class="flex-1 bg-blue-100 p-4">Column 1</div>
  <div class="flex-1 bg-blue-100 p-4">Column 2</div>
  <div class="flex-1 bg-blue-100 p-4">Column 3</div>
</div>
<!--
📐 |[  Col 1  ][  Col 2  ][  Col 3  ]|
              ↑ Tất cả cùng width
-->


<!-- 🔥 Pattern 5: Sidebar + Main content -->
<div class="flex min-h-screen">
  <aside class="w-64 bg-gray-800 text-white p-4">
    Sidebar (fixed width)
  </aside>
  <main class="flex-1 p-6">
    Main content (fills remaining)
  </main>
</div>
```

---

## 3. Grid

### 💡 Grid là gì?

Grid giúp bạn sắp xếp items **theo 2 chiều** (ngang VÀ dọc) cùng lúc.

```
┌─────────────────────────────────────────┐
│  GRID CONTAINER                         │
│  ┌─────────┬─────────┬─────────┐        │
│  │ Cell 1  │ Cell 2  │ Cell 3  │ Row 1  │
│  ├─────────┼─────────┼─────────┤        │
│  │ Cell 4  │ Cell 5  │ Cell 6  │ Row 2  │
│  ├─────────┼─────────┼─────────┤        │
│  │ Cell 7  │ Cell 8  │ Cell 9  │ Row 3  │
│  └─────────┴─────────┴─────────┘        │
│   Col 1     Col 2     Col 3             │
└─────────────────────────────────────────┘
```

### 🔄 Flex vs Grid: Khi nào dùng?

```
┌────────────────────────────────────────────────────────┐
│ DÙNG FLEXBOX khi:                                      │
│ ├── Layout 1 chiều (hàng HOẶC cột)                     │
│ ├── Items có kích thước khác nhau                      │
│ ├── Content-driven (layout theo content)               │
│ └── Navbar, toolbar, buttons group, card footer        │
├────────────────────────────────────────────────────────┤
│ DÙNG GRID khi:                                         │
│ ├── Layout 2 chiều (hàng VÀ cột)                       │
│ ├── Cần alignment chính xác theo grid                  │
│ ├── Layout-driven (content theo layout)                │
│ └── Card grid, image gallery, dashboard, page layout   │
└────────────────────────────────────────────────────────┘
```

### 📐 Basic Grid

```html
<!-- Grid 3 columns -->
<div class="grid grid-cols-3 gap-4">
  <div class="bg-blue-100 p-4">1</div>
  <div class="bg-blue-100 p-4">2</div>
  <div class="bg-blue-100 p-4">3</div>
  <div class="bg-blue-100 p-4">4</div>
  <div class="bg-blue-100 p-4">5</div>
  <div class="bg-blue-100 p-4">6</div>
</div>
<!--
📐 Kết quả:
┌─────┬─────┬─────┐
│  1  │  2  │  3  │
├─────┼─────┼─────┤
│  4  │  5  │  6  │
└─────┴─────┴─────┘

🔄 CSS tương đương:
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
}
-->


<!-- Grid variations -->
<div class="grid grid-cols-1">1 column</div>
<div class="grid grid-cols-2">2 columns</div>
<div class="grid grid-cols-3">3 columns</div>
<div class="grid grid-cols-4">4 columns</div>
<div class="grid grid-cols-6">6 columns</div>
<div class="grid grid-cols-12">12 columns (like Bootstrap)</div>
```

### 📐 Column & Row Span

```html
<div class="grid grid-cols-4 gap-4">
  <!-- Span 2 columns -->
  <div class="col-span-2 bg-blue-200 p-4">Spans 2 columns</div>
  <div class="bg-blue-100 p-4">1 col</div>
  <div class="bg-blue-100 p-4">1 col</div>

  <!-- Span 3 columns -->
  <div class="col-span-3 bg-blue-200 p-4">Spans 3 columns</div>
  <div class="bg-blue-100 p-4">1 col</div>

  <!-- Full width -->
  <div class="col-span-full bg-blue-300 p-4">Full width</div>
</div>
<!--
📐 Kết quả:
┌───────────┬─────┬─────┐
│ Spans 2   │  1  │  1  │
├─────────────────┬─────┤
│ Spans 3         │  1  │
├─────────────────────────┤
│ Full width              │
└─────────────────────────┘
-->


<!-- Row span -->
<div class="grid grid-cols-3 grid-rows-3 gap-4 h-64">
  <div class="row-span-2 bg-blue-200 p-4">
    Spans 2 rows
  </div>
  <div class="bg-blue-100 p-4">Cell</div>
  <div class="bg-blue-100 p-4">Cell</div>
  <div class="col-span-2 row-span-2 bg-blue-300 p-4">
    2×2 cell
  </div>
</div>
<!--
📐 Kết quả:
┌─────┬─────┬─────┐
│     │Cell │Cell │
│ 2   ├───────────┤
│rows │           │
├─────┤   2×2     │
│     │           │
└─────┴───────────┘
-->
```

### 📐 Grid Start/End (Vị trí chính xác)

```html
<div class="grid grid-cols-6 gap-4">
  <!-- Start từ column 2, end ở column 5 -->
  <div class="col-start-2 col-end-5 bg-blue-200 p-4">
    Columns 2-4
  </div>
</div>
<!--
📐
  1     2     3     4     5     6
  └─────┼─────────────────┼─────┘
        │  Columns 2-4    │
        └─────────────────┘
-->


<!-- Hoặc dùng start + span -->
<div class="grid grid-cols-6 gap-4">
  <div class="col-start-2 col-span-3 bg-blue-200 p-4">
    Start 2, span 3
  </div>
</div>
```

### 📐 Grid Alignment

```html
<!-- Căn items trong cells -->
<div class="grid grid-cols-3 gap-4 h-64">
  <!-- justify-items: căn horizontal trong cell -->
  <div class="grid justify-items-start">Left in cell</div>
  <div class="grid justify-items-center">Center in cell</div>
  <div class="grid justify-items-end">Right in cell</div>
</div>


<!-- place-items: shorthand cho cả 2 -->
<div class="grid grid-cols-3 place-items-center h-64">
  <div>Centered</div>
  <div>Centered</div>
  <div>Centered</div>
</div>
<!--
💡 place-items-center = justify-items: center + align-items: center
   Mỗi item được center trong cell của nó
-->
```

### 🔥 Grid Patterns phổ biến

```html
<!-- 🔥 Pattern 1: Responsive card grid -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
  <div class="bg-white rounded-lg shadow p-4">Card 1</div>
  <div class="bg-white rounded-lg shadow p-4">Card 2</div>
  <div class="bg-white rounded-lg shadow p-4">Card 3</div>
  <div class="bg-white rounded-lg shadow p-4">Card 4</div>
</div>
<!--
📐
Mobile:     Tablet:       Desktop:      Large:
[Card 1]    [Card1][Card2] [C1][C2][C3]  [C1][C2][C3][C4]
[Card 2]    [Card3][Card4] [C4]
[Card 3]
[Card 4]
-->


<!-- 🔥 Pattern 2: Holy Grail Layout -->
<div class="grid grid-cols-[200px_1fr_200px] grid-rows-[auto_1fr_auto] min-h-screen">
  <header class="col-span-3 bg-gray-800 text-white p-4">
    Header
  </header>
  <aside class="bg-gray-100 p-4">
    Left Sidebar
  </aside>
  <main class="p-4">
    Main Content
  </main>
  <aside class="bg-gray-100 p-4">
    Right Sidebar
  </aside>
  <footer class="col-span-3 bg-gray-800 text-white p-4">
    Footer
  </footer>
</div>
<!--
📐
┌───────────────────────────────────┐
│            Header                 │
├─────────┬───────────────┬─────────┤
│         │               │         │
│  Left   │     Main      │  Right  │
│  200px  │   Content     │  200px  │
│         │    (1fr)      │         │
├─────────┴───────────────┴─────────┤
│            Footer                 │
└───────────────────────────────────┘
-->


<!-- 🔥 Pattern 3: Image gallery với featured -->
<div class="grid grid-cols-4 grid-rows-2 gap-2">
  <div class="col-span-2 row-span-2 bg-blue-200">
    Featured (large)
  </div>
  <div class="bg-blue-100">Small 1</div>
  <div class="bg-blue-100">Small 2</div>
  <div class="bg-blue-100">Small 3</div>
  <div class="bg-blue-100">Small 4</div>
</div>
<!--
📐
┌───────────┬─────┬─────┐
│           │  S1 │  S2 │
│  Featured ├─────┼─────┤
│           │  S3 │  S4 │
└───────────┴─────┴─────┘
-->


<!-- 🔥 Pattern 4: Auto-fit responsive (không cần breakpoints!) -->
<div class="grid grid-cols-[repeat(auto-fit,minmax(250px,1fr))] gap-6">
  <div class="bg-white rounded-lg shadow p-4">Card 1</div>
  <div class="bg-white rounded-lg shadow p-4">Card 2</div>
  <div class="bg-white rounded-lg shadow p-4">Card 3</div>
  <div class="bg-white rounded-lg shadow p-4">Card 4</div>
</div>
<!--
💡 auto-fit + minmax tự động tính số columns:
   - Mỗi card ít nhất 250px
   - Nhiều cards fit được thì thêm columns
   - Tự responsive mà không cần md:, lg:
-->
```

---

## 4. Positioning

### 💡 Position Types

```html
<div class="static">Static (default) - normal flow</div>
<div class="relative">Relative - offset từ vị trí bình thường</div>
<div class="absolute">Absolute - ra khỏi flow, relative to positioned ancestor</div>
<div class="fixed">Fixed - relative to viewport, không scroll</div>
<div class="sticky">Sticky - normal → fixed khi scroll qua</div>

<!--
🔄 CSS tương đương:
.relative { position: relative; }
.absolute { position: absolute; }
.fixed { position: fixed; }
.sticky { position: sticky; }
-->
```

### 📐 Inset (top, right, bottom, left)

```html
<!-- Position từ edges -->
<div class="absolute top-0 left-0">Top-left corner</div>
<div class="absolute top-0 right-0">Top-right corner</div>
<div class="absolute bottom-0 left-0">Bottom-left corner</div>
<div class="absolute bottom-4 right-4">Bottom-right với offset 16px</div>

<!--
📐 Minh họa:
┌─────────────────────────────────┐
│[top-0 left-0]   [top-0 right-0]│
│                                 │
│                                 │
│                [bottom-4 right-4]│
│[bottom-0 left-0]                │
└─────────────────────────────────┘
-->


<!-- Inset shorthand -->
<div class="absolute inset-0">Fill toàn bộ parent (top/right/bottom/left = 0)</div>
<div class="absolute inset-x-0">Left và Right = 0 (full width)</div>
<div class="absolute inset-y-0">Top và Bottom = 0 (full height)</div>
<div class="absolute inset-4">Cách mỗi edge 16px</div>
```

### 🔥 Positioning Patterns

```html
<!-- 🔥 Pattern 1: Badge trên image -->
<div class="relative inline-block">
  <img src="product.jpg" class="w-64 h-64 object-cover rounded-lg" />
  <span class="absolute top-2 right-2 bg-red-500 text-white text-xs
               px-2 py-1 rounded-full font-medium">
    NEW
  </span>
</div>
<!--
📐
┌────────────────────┐
│            [NEW]   │ ← absolute, cách top-right 8px
│                    │
│      Image         │
│                    │
│                    │
└────────────────────┘

💡 Parent phải có position: relative
   Child có position: absolute sẽ position relative to parent
-->


<!-- 🔥 Pattern 2: Overlay trên image -->
<div class="relative">
  <img src="hero.jpg" class="w-full h-64 object-cover" />
  <div class="absolute inset-0 bg-black/50 flex items-center justify-center">
    <h2 class="text-white text-3xl font-bold">Overlay Text</h2>
  </div>
</div>
<!--
📐
┌────────────────────────────────┐
│                                │
│       Overlay Text             │ ← inset-0 = fill toàn bộ
│        (centered)              │    bg-black/50 = 50% opacity
│                                │
└────────────────────────────────┘
-->


<!-- 🔥 Pattern 3: Sticky header -->
<header class="sticky top-0 z-50 bg-white shadow">
  <nav class="container mx-auto px-4 py-4">
    Navigation content
  </nav>
</header>
<!--
💡 sticky top-0: Dính ở top khi scroll qua
   z-50: Đảm bảo header nằm trên các element khác
-->


<!-- 🔥 Pattern 4: Fixed action button (FAB) -->
<button class="fixed bottom-6 right-6 w-14 h-14
               bg-blue-500 hover:bg-blue-600
               rounded-full shadow-lg
               flex items-center justify-center
               text-white text-2xl">
  +
</button>
<!--
📐
┌─────────────────────┐
│                     │
│     Page content    │
│                     │
│                 [+] │ ← fixed, luôn ở bottom-right
└─────────────────────┘
-->


<!-- 🔥 Pattern 5: Center absolute element -->
<div class="relative h-64">
  <div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2">
    Perfectly centered
  </div>
</div>
<!--
💡 Công thức center absolute:
   1. top-1/2 left-1/2 → đặt góc trên trái ở giữa
   2. -translate-x-1/2 -translate-y-1/2 → dịch lại 50% kích thước của chính nó
-->
```

### 📐 Z-Index

```html
<div class="z-0">Base layer (0)</div>
<div class="z-10">Layer 10</div>
<div class="z-20">Layer 20</div>
<div class="z-30">Layer 30</div>
<div class="z-40">Layer 40</div>
<div class="z-50">Layer 50 ← modals, dropdowns</div>
<div class="-z-10">Behind (negative)</div>

<!--
💡 Z-index chỉ hoạt động với positioned elements (relative, absolute, fixed, sticky)

   Recommended usage:
   - z-10: Dropdowns, tooltips
   - z-20: Sticky elements
   - z-30: Fixed headers
   - z-40: Modals
   - z-50: Toasts, notifications
-->
```

---

## 5. Container & Overflow

### 📐 Container

```html
<!-- Tailwind container class -->
<div class="container mx-auto px-4">
  Content centered with responsive max-width
</div>
<!--
💡 container class tự động set max-width theo breakpoint:
   - sm (640px): max-w-640px
   - md (768px): max-w-768px
   - lg (1024px): max-w-1024px
   - xl (1280px): max-w-1280px
   - 2xl (1536px): max-w-1536px
-->


<!-- Manual container (kiểm soát tốt hơn) -->
<div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
  Content with manual max-width
</div>
<!--
💡 Cách này phổ biến hơn vì:
   - Kiểm soát được max-width chính xác
   - Responsive padding
-->
```

### 📐 Overflow

```html
<div class="overflow-auto">Scroll khi content tràn</div>
<div class="overflow-hidden">Cắt content tràn</div>
<div class="overflow-visible">Hiển thị content tràn</div>
<div class="overflow-scroll">Luôn hiện scrollbar</div>
<div class="overflow-x-auto">Horizontal scroll</div>
<div class="overflow-y-hidden">Ẩn vertical overflow</div>

<!--
💡 overflow-hidden hay dùng với rounded corners:
   Nếu không có overflow-hidden, content bên trong
   có thể "lòi" ra ngoài border-radius
-->
```

---

## Cheat Sheet — Day 2

```
FLEXBOX
├── flex                     → display: flex
├── flex-row, flex-col       → direction
├── flex-wrap                → allow wrapping
├── justify-start/center/end/between/around/evenly  → main axis
├── items-start/center/end/stretch                  → cross axis
├── gap-{n}                  → spacing between items
├── flex-1, flex-none        → grow/shrink behavior
└── self-center              → override alignment for one item

GRID
├── grid                     → display: grid
├── grid-cols-{n}            → define columns (1-12)
├── grid-rows-{n}            → define rows
├── gap-{n}                  → spacing
├── col-span-{n}             → span multiple columns
├── row-span-{n}             → span multiple rows
├── col-start-{n}            → start position
└── place-items-center       → center items in cells

POSITIONING
├── relative                 → relative to normal position
├── absolute                 → relative to positioned parent
├── fixed                    → relative to viewport
├── sticky                   → stick on scroll
├── top/right/bottom/left-{n} → position from edge
├── inset-0                  → fill parent
└── z-{n}                    → stacking order

CONTAINER
├── container mx-auto        → centered container
├── max-w-{size}             → max width constraint
└── overflow-{auto/hidden}   → overflow behavior
```

---

## Bài tập

### Bài 1: Navbar
Tạo navbar với:
- Logo bên trái
- Menu items ở giữa
- Login button bên phải
- Responsive: Menu ẩn trên mobile

### Bài 2: Card Grid
Tạo grid 4 cards:
- 1 column mobile, 2 tablet, 4 desktop
- Gap 24px
- Cards có shadow và rounded corners

### Bài 3: Dashboard Layout
Tạo layout với:
- Fixed sidebar 250px
- Main content fills remaining
- Sticky header
- Footer ở bottom

---

## Navigation

- [← Day 1: Fundamentals](./day-1-fundamentals.md)
- [Day 3: Responsive + States →](./day-3-responsive-states.md)
