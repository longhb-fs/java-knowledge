# Tailwind CSS v4 — Hướng Dẫn Chi Tiết 5 Ngày

> **Utility-first CSS framework** - Thay vì viết CSS custom, bạn dùng các class có sẵn để style trực tiếp trong HTML.
>
> 📌 **Tài liệu này dành cho Tailwind CSS v4.1** (phiên bản mới nhất, 2025)

---

## Tailwind là gì? Tại sao nên học?

### Vấn đề với CSS truyền thống

Khi viết CSS truyền thống, bạn thường gặp những vấn đề sau:

```css
/* 1. PHẢI ĐẶT TÊN CLASS - Tốn thời gian nghĩ tên */
.card-wrapper { }
.card-container { }
.card-inner { }
.card-content-wrapper { }  /* ??? */

/* 2. FILE CSS PHÌNH TO - Chỉ thêm, ít khi xóa */
/* File styles.css: 500 lines → 1000 lines → 3000 lines... */

/* 3. DEAD CSS - Xóa HTML nhưng quên xóa CSS */
.old-component { }  /* Component đã xóa nhưng CSS vẫn còn */

/* 4. SPECIFICITY WAR - Class không apply, phải dùng !important */
.button { color: blue; }
.header .button { color: red; }
.header .nav .button { color: green !important; }  /* Nightmare */

/* 5. CONTEXT SWITCHING - Nhảy qua lại giữa HTML và CSS */
/* Mở file HTML → thấy class → mở file CSS → tìm class → sửa → quay lại HTML... */
```

### Tailwind giải quyết như thế nào?

```html
<!-- Tailwind: Viết style TRỰC TIẾP trong HTML -->
<button class="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600">
  Click me
</button>

<!--
Giải thích từng class:
├── bg-blue-500    → background-color: #3b82f6 (màu xanh)
├── text-white     → color: white
├── px-4           → padding-left: 1rem; padding-right: 1rem
├── py-2           → padding-top: 0.5rem; padding-bottom: 0.5rem
├── rounded        → border-radius: 0.25rem
└── hover:bg-blue-600 → khi hover: background đậm hơn
-->
```

**Lợi ích:**
| Vấn đề CSS | Tailwind giải quyết |
|------------|---------------------|
| Đặt tên class | Không cần - dùng utility có sẵn |
| File CSS phình to | File CSS chỉ ~10KB (chỉ build class được dùng) |
| Dead CSS | Xóa HTML = xóa luôn styles |
| Specificity war | Tất cả utility đều flat, không conflict |
| Context switching | Viết ngay trong HTML, không cần mở file CSS |

---

## So sánh trực quan: CSS vs Tailwind

### Ví dụ 1: Button đơn giản

**CSS truyền thống:**
```css
/* styles.css */
.btn-primary {
  background-color: #3b82f6;
  color: white;
  padding: 8px 16px;
  border-radius: 8px;
  font-weight: 500;
  transition: background-color 0.2s;
}

.btn-primary:hover {
  background-color: #2563eb;
}
```

```html
<!-- index.html -->
<button class="btn-primary">Click me</button>
```

**Tailwind:**
```html
<!-- Tất cả trong 1 file HTML -->
<button class="bg-blue-500 text-white px-4 py-2 rounded-lg font-medium
               transition-colors hover:bg-blue-600">
  Click me
</button>
```

### Ví dụ 2: Card component

**CSS truyền thống:**
```css
.card {
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  padding: 24px;
  max-width: 300px;
}

.card-title {
  font-size: 20px;
  font-weight: 600;
  margin-bottom: 8px;
}

.card-description {
  color: #6b7280;
  font-size: 14px;
}
```

**Tailwind:**
```html
<div class="bg-white rounded-xl shadow-md p-6 max-w-sm">
  <h2 class="text-xl font-semibold mb-2">Card Title</h2>
  <p class="text-gray-500 text-sm">Card description here</p>
</div>

<!--
Mapping chi tiết:
├── bg-white      = background: white
├── rounded-xl    = border-radius: 12px
├── shadow-md     = box-shadow: 0 4px 6px rgba(0,0,0,0.1)
├── p-6           = padding: 24px (6 × 4px)
├── max-w-sm      = max-width: 384px
├── text-xl       = font-size: 20px
├── font-semibold = font-weight: 600
├── mb-2          = margin-bottom: 8px
├── text-gray-500 = color: #6b7280
└── text-sm       = font-size: 14px
-->
```

---

## Công thức nhớ Tailwind

Tailwind có quy tắc đặt tên rất consistent. Nắm được công thức này, bạn sẽ đoán được 80% class names:

```
┌─────────────────────────────────────────────────────────┐
│  [property]-[value]                                      │
│  [property]-[direction]-[value]                          │
│  [property]-[color]-[shade]                              │
└─────────────────────────────────────────────────────────┘

VÍ DỤ:

1. Property + Value:
   ├── text-center     → text-align: center
   ├── font-bold       → font-weight: bold
   └── rounded-lg      → border-radius: large

2. Property + Direction + Value:
   ├── p-4             → padding: 1rem (all sides)
   ├── px-4            → padding-left + padding-right: 1rem
   ├── pt-4            → padding-top: 1rem
   └── mt-8            → margin-top: 2rem

3. Property + Color + Shade:
   ├── bg-blue-500     → background: #3b82f6 (blue, medium)
   ├── text-red-600    → color: #dc2626 (red, darker)
   └── border-gray-300 → border-color: #d1d5db (gray, light)

4. State + Property:
   ├── hover:bg-blue-600   → :hover { background: darker blue }
   ├── focus:ring-2        → :focus { ring: 2px }
   └── dark:bg-gray-800    → dark mode { background: dark gray }

5. Responsive + Property:
   ├── md:flex         → @media (min-width: 768px) { display: flex }
   ├── lg:text-xl      → @media (min-width: 1024px) { font-size: xl }
   └── sm:hidden       → @media (min-width: 640px) { display: none }
```

---

## Ai nên học tài liệu này?

| Đối tượng | Phù hợp? | Ghi chú |
|-----------|----------|---------|
| Dev đã biết CSS, muốn học Tailwind | ✅ **Rất phù hợp** | Target chính |
| Fullstack dev muốn làm UI nhanh | ✅ **Rất phù hợp** | Không cần master CSS |
| Fresher biết HTML/CSS cơ bản | ✅ Phù hợp | Cần đọc kỹ phần giải thích |
| Người chưa biết CSS | ⚠️ Nên học CSS cơ bản trước | Cần hiểu box model, flexbox |

---

## Lộ trình 5 ngày

| Ngày | Chủ đề | Bạn sẽ học |
|------|--------|-----------|
| **1** | [Fundamentals](./day-1-fundamentals.md) | Spacing, Colors, Typography, Sizing - những thứ dùng nhiều nhất |
| **2** | [Layout](./day-2-layout.md) | Flexbox, Grid, Position - cách bố trí layout |
| **3** | [Responsive + States](./day-3-responsive-states.md) | Mobile-first, hover/focus, dark mode |
| **4** | [Components](./day-4-components.md) | Tạo reusable components, customize config |
| **5** | [Advanced](./day-5-advanced.md) | Animations, gradients, performance, best practices |

---

## Setup nhanh để bắt đầu học

### Cách 1: CDN (Nhanh nhất - chỉ để học)

Tạo file `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Learn Tailwind</title>
  <!-- Thêm dòng này là dùng được Tailwind -->
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-100 p-8">

  <h1 class="text-3xl font-bold text-blue-600 mb-4">
    Hello Tailwind!
  </h1>

  <p class="text-gray-600">
    Nếu bạn thấy text màu xanh và nền xám, Tailwind đã hoạt động.
  </p>

</body>
</html>
```

Mở file này trong browser → Thấy kết quả ngay.

> ⚠️ **Lưu ý:** CDN chỉ dùng để học/prototype. Production phải dùng build tool.

### Cách 2: Vite + Tailwind v4 (Cho project thật)

```bash
# Bước 1: Tạo project Vite
npm create vite@latest my-tailwind-project -- --template vanilla
cd my-tailwind-project

# Bước 2: Cài Tailwind v4
npm install tailwindcss @tailwindcss/vite
```

**Bước 3:** Sửa file `vite.config.js`:
```js
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [
    tailwindcss(),
  ],
})
```

**Bước 4:** Sửa file `src/style.css`:
```css
@import "tailwindcss";
```

```bash
# Bước 5: Chạy dev server
npm run dev
```

> 💡 **Thay đổi lớn từ v3 → v4:**
> - Không cần `tailwind.config.js` - config trực tiếp trong CSS với `@theme`
> - Không cần PostCSS - dùng Vite plugin
> - Thay `@tailwind base/components/utilities` bằng `@import "tailwindcss"`

---

## Quy ước trong tài liệu

| Ký hiệu | Ý nghĩa |
|---------|---------|
| 💡 | Giải thích chi tiết / Tips quan trọng |
| ⚠️ | Cảnh báo / Dễ nhầm lẫn |
| 🔥 | Pattern hay dùng trong thực tế |
| ❌ / ✅ | Cách sai / Cách đúng |
| 📐 | Hình minh họa |
| 🔄 | So sánh CSS ↔ Tailwind |

---

## Navigation

| Ngày | Link |
|------|------|
| Day 1 | [Fundamentals →](./day-1-fundamentals.md) |
| Day 2 | [Layout →](./day-2-layout.md) |
| Day 3 | [Responsive + States →](./day-3-responsive-states.md) |
| Day 4 | [Components →](./day-4-components.md) |
| Day 5 | [Advanced →](./day-5-advanced.md) |
