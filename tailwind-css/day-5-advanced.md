# Day 5: Advanced Techniques + Best Practices

> Mục tiêu: Master animations, gradients, filters, arbitrary values, performance optimization, và best practices.

---

## 1. Transitions — Cách tạo hiệu ứng mượt

### 💡 Transition là gì?

Transition làm cho thay đổi CSS diễn ra từ từ thay vì tức thì:

```
┌─────────────────────────────────────────────────────────────┐
│  KHÔNG CÓ TRANSITION                                        │
│                                                              │
│  Button (blue) ─────[click]────→ Button (dark blue)         │
│                  Thay đổi ngay lập tức                       │
│                                                              │
│  CÓ TRANSITION                                               │
│                                                              │
│  Button (blue) ~~~[0.2s]~~~→ Button (dark blue)             │
│                  Chuyển màu mượt mà                          │
└─────────────────────────────────────────────────────────────┘
```

### 🔄 CSS vs Tailwind

**CSS truyền thống:**
```css
.button {
  background-color: #3b82f6;
  transition: background-color 0.3s ease-out;
}

.button:hover {
  background-color: #2563eb;
}
```

**Tailwind:**
```html
<button class="bg-blue-500 transition-colors duration-300 ease-out
               hover:bg-blue-600">
  Hover me
</button>
```

### Transition Properties

```html
<!-- Transition WHAT? -->
<div class="transition">
  Default: color, background-color, border-color, text-decoration-color,
           fill, stroke, opacity, box-shadow, transform, filter,
           backdrop-filter
</div>

<div class="transition-none">No transition</div>
<div class="transition-all">Everything animates</div>
<div class="transition-colors">Only colors</div>
<div class="transition-opacity">Only opacity</div>
<div class="transition-shadow">Only box-shadow</div>
<div class="transition-transform">Only transform</div>
```

### Duration — Bao lâu?

```html
<!--
Duration scale:
┌──────────────┬─────────┐
│ Class        │ Time    │
├──────────────┼─────────┤
│ duration-75  │ 75ms    │
│ duration-100 │ 100ms   │
│ duration-150 │ 150ms   │  ← Micro-interactions (buttons, links)
│ duration-200 │ 200ms   │
│ duration-300 │ 300ms   │  ← Standard (most UI elements)
│ duration-500 │ 500ms   │  ← Emphasis (modals, drawers)
│ duration-700 │ 700ms   │
│ duration-1000│ 1000ms  │  ← Dramatic (page transitions)
└──────────────┴─────────┘
-->

<button class="transition duration-150 hover:bg-blue-600">Fast</button>
<button class="transition duration-300 hover:bg-blue-600">Normal</button>
<button class="transition duration-500 hover:bg-blue-600">Slow</button>
```

### Timing Function — Kiểu chuyển động

```html
<!--
Timing functions visualized:

ease-linear:    ────────────────────→  Constant speed
                [    |    |    |    ]

ease-in:        ─────────────────→→→  Slow start, fast end
                [  |   |    |      ]

ease-out:       →→→─────────────────  Fast start, slow end
                [      |    |   |  ]

ease-in-out:    ───→→→→→→→───────  Slow-fast-slow
                [  |    |    |  ]
-->

<div class="transition ease-linear">Constant speed</div>
<div class="transition ease-in">Slow start (feels heavy)</div>
<div class="transition ease-out">Slow end (feels natural)</div>
<div class="transition ease-in-out">Default, balanced</div>
```

### Delay — Chờ trước khi bắt đầu

```html
<!-- Staggered animation effect -->
<div class="space-y-2">
  <div class="transition hover:translate-x-4 delay-0">Item 1</div>
  <div class="transition hover:translate-x-4 delay-75">Item 2</div>
  <div class="transition hover:translate-x-4 delay-150">Item 3</div>
  <div class="transition hover:translate-x-4 delay-300">Item 4</div>
</div>
```

---

## 2. Transforms — Di chuyển, xoay, scale

### 💡 Transform là gì?

Transform thay đổi vị trí/hình dạng element mà KHÔNG ảnh hưởng layout:

```
┌─────────────────────────────────────────────────────────────┐
│  TRANSFORM KHÔNG ẢNH HƯỞNG LAYOUT                            │
│                                                              │
│  ┌─────┐     ┌─────┐     ┌─────┐                            │
│  │  A  │     │  B  │     │  C  │      ← Original positions  │
│  └─────┘     └─────┘     └─────┘                            │
│                                                              │
│  Sau khi B được transform: scale(1.5)                        │
│                                                              │
│  ┌─────┐   ┌───────┐   ┌─────┐                              │
│  │  A  │   │       │   │  C  │      ← A và C không di chuyển│
│  └─────┘   │   B   │   └─────┘         B chồng lên nhưng    │
│            │       │                    không đẩy siblings   │
│            └───────┘                                         │
└─────────────────────────────────────────────────────────────┘
```

### Scale — Phóng to/thu nhỏ

```html
<!--
Scale values:
┌──────────────┬─────────┬────────────────────┐
│ Class        │ Value   │ Effect             │
├──────────────┼─────────┼────────────────────┤
│ scale-0      │ 0       │ Invisible          │
│ scale-50     │ 0.5     │ Half size          │
│ scale-75     │ 0.75    │ 3/4 size           │
│ scale-90     │ 0.9     │ Slightly smaller   │
│ scale-95     │ 0.95    │ Subtle shrink      │
│ scale-100    │ 1       │ Normal             │
│ scale-105    │ 1.05    │ Subtle grow        │
│ scale-110    │ 1.1     │ Slightly larger    │
│ scale-125    │ 1.25    │ 1.25x              │
│ scale-150    │ 1.5     │ 1.5x               │
└──────────────┴─────────┴────────────────────┘
-->

<!-- Card hover effect -->
<div class="transition-transform duration-300 hover:scale-105">
  Card grows slightly on hover
</div>

<!-- Button press effect -->
<button class="transition-transform active:scale-95">
  Shrinks when pressed
</button>

<!-- Scale X or Y only -->
<div class="hover:scale-x-110">Wider on hover</div>
<div class="hover:scale-y-110">Taller on hover</div>
```

### Rotate — Xoay

```html
<!--
Rotation:
       0°
        │
-90° ───┼─── 90°
        │
      180°
-->

<div class="hover:rotate-45">Rotate 45°</div>
<div class="hover:rotate-90">Rotate 90°</div>
<div class="hover:rotate-180">Rotate 180°</div>
<div class="hover:-rotate-45">Rotate -45° (counter-clockwise)</div>

<!-- Loading spinner -->
<div class="animate-spin w-8 h-8 border-4 border-blue-500 border-t-transparent rounded-full">
</div>

<!-- Icon rotation on hover -->
<button class="group">
  <svg class="transition-transform group-hover:rotate-90">
    <!-- icon -->
  </svg>
  Settings
</button>
```

### Translate — Di chuyển

```html
<!--
Translate directions:
               translate-y-4 (down)
                     ↓
translate-x-4 ←  [element]  → -translate-x-4
    (left)           ↑            (right)
              -translate-y-4 (up)

Values follow spacing scale: 4 = 1rem = 16px
-->

<div class="hover:translate-x-4">Move right 1rem</div>
<div class="hover:-translate-x-4">Move left 1rem</div>
<div class="hover:translate-y-2">Move down 0.5rem</div>
<div class="hover:-translate-y-2">Move up 0.5rem</div>

<!-- Card lift effect -->
<div class="transition duration-300 hover:-translate-y-1 hover:shadow-lg">
  Card floats up on hover
</div>

<!-- Slide in from right -->
<div class="translate-x-full opacity-0
            group-hover:translate-x-0 group-hover:opacity-100
            transition duration-300">
  Slides in when parent hovered
</div>
```

### Origin — Điểm tâm transform

```html
<!--
Transform origin positions:
┌────────────────────────────────┐
│ origin-top-left    origin-top    origin-top-right │
│         ·─────────────·─────────────·             │
│         │                           │             │
│ origin- │                           │ origin-     │
│ left    ·       origin-center       ·  right      │
│         │            ·              │             │
│         │                           │             │
│         ·─────────────·─────────────·             │
│ origin-bottom-left  origin-bottom  origin-bottom-right │
└────────────────────────────────────┘

Default: origin-center
-->

<!-- Rotate from corner -->
<div class="origin-top-left hover:rotate-12">
  Rotates around top-left corner
</div>

<!-- Dropdown animation from top -->
<div class="origin-top scale-y-0 group-hover:scale-y-100 transition">
  Opens from top like a dropdown
</div>
```

### 🔥 Transform Patterns thực tế

```html
<!-- 1. Card with lift effect -->
<div class="bg-white rounded-xl shadow-md p-6
            transition duration-300 ease-out
            hover:-translate-y-1 hover:shadow-xl">
  <h3 class="font-semibold">Card Title</h3>
  <p class="text-gray-600">Card content</p>
</div>

<!-- 2. Image zoom on hover -->
<div class="overflow-hidden rounded-lg">
  <img src="..." class="w-full h-64 object-cover
                        transition duration-500 ease-out
                        hover:scale-110" />
</div>

<!-- 3. Button with press effect -->
<button class="px-4 py-2 bg-blue-500 text-white rounded-lg
               transition-all duration-150
               hover:bg-blue-600
               active:scale-95 active:bg-blue-700">
  Click me
</button>

<!-- 4. Sidebar slide-in -->
<aside class="fixed inset-y-0 left-0 w-64 bg-white shadow-lg
              transform -translate-x-full
              transition duration-300 ease-out
              data-[open=true]:translate-x-0">
  Sidebar content
</aside>

<!-- 5. Modal scale-in -->
<div class="fixed inset-0 flex items-center justify-center">
  <div class="bg-white rounded-xl p-6
              transform scale-95 opacity-0
              transition duration-200 ease-out
              data-[open=true]:scale-100 data-[open=true]:opacity-100">
    Modal content
  </div>
</div>
```

---

## 3. Built-in Animations

### 4 animations có sẵn

```html
<!-- 1. SPIN: Xoay 360° liên tục -->
<!--    Use case: Loading spinners -->
<div class="animate-spin w-8 h-8 border-4 border-blue-500 border-t-transparent rounded-full">
</div>

<!-- 2. PING: Pulse ra ngoài rồi mất -->
<!--    Use case: Notification indicators -->
<span class="relative flex h-3 w-3">
  <span class="animate-ping absolute inline-flex h-full w-full
               rounded-full bg-red-400 opacity-75"></span>
  <span class="relative inline-flex rounded-full h-3 w-3 bg-red-500"></span>
</span>

<!-- 3. PULSE: Opacity fade in/out -->
<!--    Use case: Skeleton loading -->
<div class="animate-pulse space-y-4">
  <div class="h-4 bg-gray-200 rounded w-3/4"></div>
  <div class="h-4 bg-gray-200 rounded w-1/2"></div>
  <div class="h-4 bg-gray-200 rounded w-5/6"></div>
</div>

<!-- 4. BOUNCE: Nhảy lên xuống -->
<!--    Use case: Call-to-action, scroll indicators -->
<div class="animate-bounce">
  <svg class="w-6 h-6"><!-- down arrow --></svg>
</div>
```

### Skeleton Loading Pattern

```html
<!-- Full skeleton card -->
<div class="animate-pulse">
  <!-- Image placeholder -->
  <div class="bg-gray-200 h-48 rounded-t-xl"></div>

  <div class="p-6 space-y-4">
    <!-- Title placeholder -->
    <div class="h-6 bg-gray-200 rounded w-3/4"></div>

    <!-- Text lines -->
    <div class="space-y-2">
      <div class="h-4 bg-gray-200 rounded"></div>
      <div class="h-4 bg-gray-200 rounded w-5/6"></div>
    </div>

    <!-- Button placeholder -->
    <div class="h-10 bg-gray-200 rounded-lg w-32"></div>
  </div>
</div>
```

---

## 4. Custom Animations

### Thêm animations trong config

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      animation: {
        // Format: 'name duration timing-function iteration-count'

        // Fade animations
        'fade-in': 'fadeIn 0.3s ease-out',
        'fade-out': 'fadeOut 0.3s ease-out',
        'fade-in-up': 'fadeInUp 0.4s ease-out',
        'fade-in-down': 'fadeInDown 0.4s ease-out',

        // Slide animations
        'slide-in-left': 'slideInLeft 0.3s ease-out',
        'slide-in-right': 'slideInRight 0.3s ease-out',
        'slide-in-up': 'slideInUp 0.3s ease-out',
        'slide-in-down': 'slideInDown 0.3s ease-out',

        // Scale animations
        'scale-in': 'scaleIn 0.2s ease-out',
        'scale-out': 'scaleOut 0.2s ease-out',

        // Special animations
        'wiggle': 'wiggle 1s ease-in-out infinite',
        'float': 'float 3s ease-in-out infinite',
        'heartbeat': 'heartbeat 1.5s ease-in-out infinite',
      },

      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        fadeOut: {
          '0%': { opacity: '1' },
          '100%': { opacity: '0' },
        },
        fadeInUp: {
          '0%': { opacity: '0', transform: 'translateY(10px)' },
          '100%': { opacity: '1', transform: 'translateY(0)' },
        },
        fadeInDown: {
          '0%': { opacity: '0', transform: 'translateY(-10px)' },
          '100%': { opacity: '1', transform: 'translateY(0)' },
        },
        slideInLeft: {
          '0%': { transform: 'translateX(-100%)' },
          '100%': { transform: 'translateX(0)' },
        },
        slideInRight: {
          '0%': { transform: 'translateX(100%)' },
          '100%': { transform: 'translateX(0)' },
        },
        slideInUp: {
          '0%': { transform: 'translateY(100%)' },
          '100%': { transform: 'translateY(0)' },
        },
        slideInDown: {
          '0%': { transform: 'translateY(-100%)' },
          '100%': { transform: 'translateY(0)' },
        },
        scaleIn: {
          '0%': { transform: 'scale(0.9)', opacity: '0' },
          '100%': { transform: 'scale(1)', opacity: '1' },
        },
        scaleOut: {
          '0%': { transform: 'scale(1)', opacity: '1' },
          '100%': { transform: 'scale(0.9)', opacity: '0' },
        },
        wiggle: {
          '0%, 100%': { transform: 'rotate(-3deg)' },
          '50%': { transform: 'rotate(3deg)' },
        },
        float: {
          '0%, 100%': { transform: 'translateY(0)' },
          '50%': { transform: 'translateY(-10px)' },
        },
        heartbeat: {
          '0%, 100%': { transform: 'scale(1)' },
          '50%': { transform: 'scale(1.05)' },
        },
      },
    },
  },
}
```

### Sử dụng custom animations:

```html
<!-- Page load animations -->
<h1 class="animate-fade-in-up">Welcome</h1>

<!-- Modal animation -->
<div class="animate-scale-in">Modal content</div>

<!-- Sidebar animation -->
<aside class="animate-slide-in-left">Sidebar</aside>

<!-- Attention-grabbing -->
<button class="animate-wiggle">Click me!</button>

<!-- Floating elements -->
<div class="animate-float">
  <img src="cloud.svg" />
</div>
```

---

## 5. Filters & Effects

### Blur

```html
<!--
Blur scale:
┌────────────┬─────────┐
│ Class      │ Radius  │
├────────────┼─────────┤
│ blur-none  │ 0       │
│ blur-sm    │ 4px     │
│ blur       │ 8px     │
│ blur-md    │ 12px    │
│ blur-lg    │ 16px    │
│ blur-xl    │ 24px    │
│ blur-2xl   │ 40px    │
│ blur-3xl   │ 64px    │
└────────────┴─────────┘
-->

<!-- Blur an image -->
<img class="blur-md" src="..." />

<!-- Blur text for spoiler -->
<p class="blur-sm hover:blur-none transition-all">
  Spoiler content revealed on hover
</p>
```

### Backdrop Blur (Glassmorphism)

```html
<!--
Backdrop blur: Làm mờ PHÍA SAU element

┌────────────────────────────────────┐
│  Background Image                  │
│  ┌──────────────────────────────┐  │
│  │ backdrop-blur-md             │  │
│  │ bg-white/30                  │  │  ← Glass effect!
│  │                              │  │
│  │ Content here                 │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
-->

<!-- Glass card -->
<div class="backdrop-blur-md bg-white/30 p-6 rounded-xl border border-white/20">
  <h3 class="font-semibold">Glass Card</h3>
  <p class="text-gray-700">Frosted glass effect</p>
</div>

<!-- Frosted navbar -->
<nav class="fixed top-0 inset-x-0 z-50
            backdrop-blur-lg bg-white/80 dark:bg-gray-900/80
            border-b border-gray-200/50">
  <div class="max-w-7xl mx-auto px-4 h-16 flex items-center">
    Logo and navigation
  </div>
</nav>

<!-- Modal backdrop -->
<div class="fixed inset-0 backdrop-blur-sm bg-black/50 flex items-center justify-center">
  <div class="bg-white rounded-xl p-6">
    Modal content
  </div>
</div>
```

### Grayscale, Sepia, Invert

```html
<!-- Grayscale: Black & white -->
<img class="grayscale" src="..." />
<img class="grayscale hover:grayscale-0 transition duration-300" src="..." />

<!-- Sepia: Vintage tone -->
<img class="sepia" src="..." />

<!-- Invert: Đảo màu -->
<img class="invert" src="..." />
<img class="dark:invert" src="light-logo.svg" />  <!-- Logo cho dark mode -->
```

### Brightness & Contrast

```html
<!--
Brightness: 0 = black, 100 = normal, 200 = 2x bright
Contrast: 0 = gray, 100 = normal, 200 = high contrast
-->

<!-- Darken image for text overlay -->
<div class="relative">
  <img class="brightness-50" src="..." />
  <h1 class="absolute inset-0 flex items-center justify-center text-white text-4xl">
    Title over dark image
  </h1>
</div>

<!-- Brighten on hover -->
<img class="brightness-90 hover:brightness-100 transition" src="..." />

<!-- High contrast -->
<img class="contrast-125" src="..." />
```

### Drop Shadow (cho transparent images)

```html
<!--
drop-shadow vs shadow:

shadow (box-shadow):
┌─────────────────┐
│ ███████████████ │  ← Shadow theo hình chữ nhật
│ ███   PNG   ███ │
│ ███████████████ │
└─────────────────┘

drop-shadow:
     ███
   █████████       ← Shadow theo hình dạng thực của image
     ███
-->

<img class="drop-shadow-md" src="logo.png" />
<img class="drop-shadow-lg" src="character.png" />
<img class="drop-shadow-2xl" src="product.png" />

<!-- Icon with shadow -->
<svg class="drop-shadow-md text-blue-500">
  <!-- icon path -->
</svg>
```

---

## 6. Gradients

### 💡 Cách gradient hoạt động

```
┌─────────────────────────────────────────────────────────────┐
│  GRADIENT DIRECTION                                          │
│                                                              │
│  bg-gradient-to-r (right)                                    │
│  [from-blue-500 ────────────────────────→ to-purple-500]    │
│                                                              │
│  bg-gradient-to-b (bottom)                                   │
│  from-blue-500                                               │
│       │                                                      │
│       │                                                      │
│       ↓                                                      │
│  to-purple-500                                               │
│                                                              │
│  bg-gradient-to-br (bottom-right)                            │
│  from-blue-500                                               │
│       ╲                                                      │
│         ╲                                                    │
│           ↘                                                  │
│       to-purple-500                                          │
│                                                              │
│  ALL DIRECTIONS:                                             │
│  to-t (top), to-b (bottom), to-l (left), to-r (right)       │
│  to-tl, to-tr, to-bl, to-br (corners)                       │
└─────────────────────────────────────────────────────────────┘
```

### Basic Gradients

```html
<!-- Two-color gradient -->
<div class="bg-gradient-to-r from-blue-500 to-purple-500">
  Left to right: Blue → Purple
</div>

<!-- Three-color gradient with via -->
<div class="bg-gradient-to-r from-red-500 via-yellow-500 to-green-500">
  Rainbow: Red → Yellow → Green
</div>

<!-- Direction variations -->
<div class="bg-gradient-to-t from-black to-transparent">
  Bottom to top fade (image overlay)
</div>

<div class="bg-gradient-to-br from-pink-500 to-orange-500">
  Diagonal: Top-left to bottom-right
</div>
```

### 🔄 CSS vs Tailwind

**CSS:**
```css
.gradient-bg {
  background: linear-gradient(to right, #3b82f6, #8b5cf6);
}

.gradient-text {
  background: linear-gradient(to right, #3b82f6, #8b5cf6);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
```

**Tailwind:**
```html
<div class="bg-gradient-to-r from-blue-500 to-purple-500">
  Gradient background
</div>

<h1 class="bg-gradient-to-r from-blue-500 to-purple-500
           bg-clip-text text-transparent">
  Gradient text
</h1>
```

### 🔥 Gradient Patterns

```html
<!-- 1. Gradient button -->
<button class="px-6 py-3 rounded-lg text-white font-medium
               bg-gradient-to-r from-blue-500 to-purple-600
               hover:from-blue-600 hover:to-purple-700
               transition-all duration-300">
  Gradient Button
</button>

<!-- 2. Gradient border -->
<div class="p-[2px] bg-gradient-to-r from-pink-500 via-purple-500 to-blue-500 rounded-xl">
  <div class="bg-white dark:bg-gray-900 rounded-[10px] p-6">
    Card with gradient border
  </div>
</div>

<!-- 3. Image overlay -->
<div class="relative">
  <img src="..." class="w-full h-64 object-cover" />
  <div class="absolute inset-0 bg-gradient-to-t from-black/80 via-black/20 to-transparent">
    <div class="absolute bottom-4 left-4 text-white">
      <h3 class="text-xl font-bold">Title</h3>
      <p class="text-sm text-gray-300">Description</p>
    </div>
  </div>
</div>

<!-- 4. Gradient text -->
<h1 class="text-5xl font-bold
           bg-gradient-to-r from-blue-600 via-purple-600 to-pink-600
           bg-clip-text text-transparent">
  Amazing Gradient Text
</h1>

<!-- 5. Mesh gradient background -->
<div class="relative min-h-screen bg-gray-50 overflow-hidden">
  <!-- Gradient blobs -->
  <div class="absolute -top-40 -right-40 w-80 h-80
              bg-purple-300 rounded-full blur-3xl opacity-30"></div>
  <div class="absolute -bottom-40 -left-40 w-80 h-80
              bg-pink-300 rounded-full blur-3xl opacity-30"></div>
  <div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2
              w-96 h-96 bg-blue-300 rounded-full blur-3xl opacity-20"></div>

  <!-- Content -->
  <div class="relative z-10 p-8">
    Content with beautiful mesh gradient background
  </div>
</div>

<!-- 6. Animated gradient -->
<div class="bg-gradient-to-r from-blue-500 via-purple-500 to-pink-500
            bg-[length:200%_auto] animate-gradient">
  Animated gradient background
</div>

<!-- Add to config:
animation: {
  'gradient': 'gradient 3s linear infinite',
},
keyframes: {
  gradient: {
    '0%, 100%': { backgroundPosition: '0% 50%' },
    '50%': { backgroundPosition: '100% 50%' },
  },
},
-->
```

---

## 7. Arbitrary Values

### 💡 Khi nào cần arbitrary values?

Khi giá trị bạn cần KHÔNG có trong Tailwind's default scale:

```html
<!-- Tailwind có: w-64 (256px), w-72 (288px), w-80 (320px) -->
<!-- Nhưng bạn cần exactly 300px -->

<div class="w-[300px]">Exact 300px width</div>
```

### Cú pháp: `property-[value]`

```html
<!-- SIZE -->
<div class="w-[137px]">Exact width</div>
<div class="h-[calc(100vh-80px)]">Viewport minus header</div>
<div class="min-h-[500px]">Minimum height</div>
<div class="max-w-[1400px]">Max width</div>

<!-- COLORS -->
<div class="bg-[#1da1f2]">Twitter blue</div>
<div class="text-[rgb(255,100,50)]">RGB color</div>
<div class="border-[hsl(200,50%,50%)]">HSL color</div>

<!-- TYPOGRAPHY -->
<p class="text-[22px]">Custom font size</p>
<p class="leading-[1.7]">Custom line height</p>
<p class="tracking-[0.05em]">Custom letter spacing</p>
<p class="font-[550]">Custom font weight</p>

<!-- SPACING -->
<div class="p-[5px_10px_15px_20px]">Custom padding shorthand</div>
<div class="m-[clamp(1rem,5vw,3rem)]">Responsive margin</div>
<div class="gap-[7px]">Custom gap</div>

<!-- LAYOUT -->
<div class="grid-cols-[200px_1fr_200px]">Custom grid columns</div>
<div class="grid-rows-[auto_1fr_auto]">Custom grid rows</div>
<div class="z-[999]">Custom z-index</div>

<!-- POSITION -->
<div class="top-[117px]">Custom top</div>
<div class="inset-[10px_20px]">Custom inset</div>

<!-- BORDERS -->
<div class="rounded-[20px]">Custom border radius</div>
<div class="border-[3px]">Custom border width</div>

<!-- SHADOWS -->
<div class="shadow-[0_4px_20px_rgba(0,0,0,0.15)]">Custom shadow</div>
```

### Arbitrary Properties — CSS không có trong Tailwind

```html
<!-- Dùng dấu ngoặc vuông cho CSS property -->
<div class="[mask-type:alpha]">Mask type</div>
<div class="[clip-path:polygon(0_0,100%_0,100%_75%,0_100%)]">Clip path</div>
<div class="[text-wrap:balance]">Text wrap balance</div>
<div class="[writing-mode:vertical-rl]">Vertical text</div>
```

### Arbitrary Values với Modifiers

```html
<!-- Responsive -->
<div class="w-full md:w-[calc(100%-2rem)] lg:w-[800px]">
  Responsive arbitrary values
</div>

<!-- States -->
<button class="bg-blue-500 hover:bg-[#1d4ed8] active:bg-[#1e40af]">
  Custom hover/active colors
</button>

<!-- Dark mode -->
<div class="bg-white dark:bg-[#0f172a]">
  Custom dark mode color
</div>
```

### ⚠️ Khi nào KHÔNG dùng arbitrary values

```html
<!-- ❌ BAD: Dùng arbitrary khi có class có sẵn -->
<div class="p-[16px]">  <!-- Use p-4 instead -->
<div class="w-[100%]">  <!-- Use w-full instead -->
<div class="text-[14px]">  <!-- Use text-sm instead -->

<!-- ✅ GOOD: Chỉ dùng khi thực sự cần giá trị custom -->
<div class="w-[calc(100%-80px)]">  <!-- No equivalent -->
<div class="bg-[#1da1f2]">  <!-- Brand color -->
<div class="grid-cols-[250px_1fr_250px]">  <!-- Specific layout -->
```

---

## 8. Performance Optimization

### Content Configuration

```js
// tailwind.config.js
module.exports = {
  content: [
    // Scan these files for class names
    './src/**/*.{html,js,ts,jsx,tsx,vue}',
    './components/**/*.{js,ts,jsx,tsx}',
    './pages/**/*.{js,ts,jsx,tsx}',

    // Don't forget public HTML
    './public/index.html',
  ],
}
```

### ⚠️ Dynamic Class Names — DANGER!

```html
<!--
Tailwind hoạt động bằng cách SCAN files để tìm class names.
Nó KHÔNG execute JavaScript, chỉ tìm strings.

❌ PROBLEM: Dynamic class names không được detect
-->

<!-- ❌ BAD: Concatenated strings -->
<div :class="`bg-${color}-500`">
  <!-- Tailwind không thấy "bg-red-500", "bg-blue-500"... -->
</div>

<!-- ❌ BAD: Template literals -->
<div class="text-{{ error ? 'red' : 'green' }}-500">
  <!-- Không thấy "text-red-500" hay "text-green-500" -->
</div>

<!-- ✅ GOOD: Complete class names -->
<div :class="error ? 'text-red-500' : 'text-green-500'">
  <!-- Tailwind thấy cả 2 class names -->
</div>

<!-- ✅ GOOD: Object syntax -->
<div :class="{
  'text-red-500': error,
  'text-green-500': !error,
  'bg-red-100': error,
  'bg-green-100': !error,
}">
  <!-- All class names visible -->
</div>

<!-- ✅ GOOD: Mapping object -->
<script>
const colorMap = {
  success: 'bg-green-500 text-white',
  warning: 'bg-yellow-500 text-black',
  error: 'bg-red-500 text-white',
};
</script>
<div :class="colorMap[status]">
  <!-- Class names in colorMap are detected -->
</div>
```

### Safelist — Khi phải dùng dynamic classes

```js
// tailwind.config.js
module.exports = {
  safelist: [
    // Specific classes
    'bg-red-500',
    'bg-green-500',
    'bg-blue-500',

    // Pattern-based
    {
      pattern: /bg-(red|green|blue|yellow)-(100|500|900)/,
      // Generates: bg-red-100, bg-red-500, bg-red-900, bg-green-100...
    },

    // With variants
    {
      pattern: /text-(red|green|blue)-(500|600)/,
      variants: ['hover', 'focus', 'dark'],
      // Generates: text-red-500, hover:text-red-500, dark:text-red-500...
    },
  ],
}
```

### Disable Unused Core Plugins

```js
// tailwind.config.js
module.exports = {
  corePlugins: {
    // Disable if not using
    float: false,
    clear: false,
    skew: false,
    sepia: false,
    // ...etc
  },
}
```

---

## 9. Best Practices

### 1. Consistent Class Ordering

```html
<!--
Recommended order:
1. Layout/Position (flex, grid, relative, absolute...)
2. Sizing (w, h, min, max...)
3. Spacing (p, m, gap...)
4. Background (bg-*)
5. Border (border, rounded...)
6. Typography (text, font...)
7. Effects (shadow, opacity...)
8. States (hover, focus, dark...)
9. Transitions (transition, duration...)
-->

<div class="
  relative flex items-center justify-between    /* Layout */
  w-full max-w-md h-16                          /* Sizing */
  p-4 gap-4                                     /* Spacing */
  bg-white dark:bg-gray-800                     /* Background */
  border border-gray-200 rounded-lg shadow-md   /* Border/Effects */
  text-gray-900 text-sm font-medium             /* Typography */
  hover:bg-gray-50 hover:shadow-lg              /* States */
  transition-all duration-200                   /* Transitions */
">
```

### 2. Use Prettier Plugin for Auto-sorting

```bash
npm install -D prettier-plugin-tailwindcss
```

```json
// .prettierrc
{
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

### 3. Mobile-First Approach

```html
<!-- ✅ GOOD: Start mobile, add larger breakpoints -->
<div class="
  flex flex-col           /* Mobile: vertical stack */
  md:flex-row             /* Tablet+: horizontal */
  lg:justify-between      /* Desktop+: space between */
">

<!-- ❌ BAD: Desktop-first (harder to maintain) -->
<div class="
  flex flex-row justify-between    /* Desktop default */
  md:flex-col                      /* What?? Smaller = column? */
  sm:flex-col                      /* Confusing! */
">
```

### 4. Extract Components for Repeated Patterns

```tsx
// ✅ GOOD: Extract to component when used 3+ times
function Badge({ variant, children }) {
  const styles = {
    success: 'bg-green-100 text-green-800',
    warning: 'bg-yellow-100 text-yellow-800',
    error: 'bg-red-100 text-red-800',
  };

  return (
    <span className={`px-2 py-1 rounded-full text-xs font-medium ${styles[variant]}`}>
      {children}
    </span>
  );
}

// Usage
<Badge variant="success">Active</Badge>
<Badge variant="error">Failed</Badge>
```

### 5. Use Design Tokens

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        // ✅ Semantic tokens (good)
        surface: {
          DEFAULT: '#ffffff',
          secondary: '#f9fafb',
        },
        content: {
          DEFAULT: '#111827',
          secondary: '#6b7280',
        },
        accent: {
          DEFAULT: '#3b82f6',
          hover: '#2563eb',
        },

        // ❌ Avoid using raw colors in code
        // Instead of bg-blue-500, use bg-accent
      },
    },
  },
}
```

```html
<!-- ✅ GOOD: Semantic class names -->
<div class="bg-surface text-content">
  <button class="bg-accent hover:bg-accent-hover text-white">
    Action
  </button>
</div>

<!-- ❌ BAD: Hard to maintain if brand color changes -->
<div class="bg-white text-gray-900">
  <button class="bg-blue-500 hover:bg-blue-600 text-white">
    Action
  </button>
</div>
```

---

## 10. Common Patterns

### Responsive Hide/Show

```html
<!-- Mobile only -->
<div class="block md:hidden">Mobile menu icon</div>

<!-- Desktop only -->
<div class="hidden md:block">Desktop navigation</div>

<!-- Tablet only -->
<div class="hidden md:block lg:hidden">Tablet specific</div>
```

### Truncate Text

```html
<!-- Single line truncate -->
<p class="truncate">
  Very long text that will be truncated with ellipsis...
</p>

<!-- Multi-line clamp -->
<p class="line-clamp-2">
  Long text that spans multiple lines will be limited to 2 lines
  with an ellipsis at the end of the second line...
</p>

<p class="line-clamp-3">Three lines max...</p>
```

### Aspect Ratio

```html
<div class="aspect-video">16:9 (videos)</div>
<div class="aspect-square">1:1 (avatars, thumbnails)</div>
<div class="aspect-[4/3]">4:3 (custom ratio)</div>
<div class="aspect-[21/9]">Ultra-wide</div>
```

### Sticky Elements

```html
<!-- Sticky header -->
<header class="sticky top-0 z-50 bg-white shadow">
  Navigation
</header>

<!-- Sticky sidebar -->
<aside class="sticky top-20 h-fit">
  <!-- top-20 = below header height -->
  Sidebar content
</aside>
```

### Full-height Page Layout

```html
<div class="min-h-screen flex flex-col">
  <header class="h-16 shrink-0">Header</header>
  <main class="flex-1">Main content grows to fill</main>
  <footer class="h-20 shrink-0">Footer</footer>
</div>
```

### Center Everything

```html
<!-- Flex center -->
<div class="flex items-center justify-center h-screen">
  Centered content
</div>

<!-- Grid center -->
<div class="grid place-items-center h-screen">
  Centered content
</div>
```

---

## 11. Debugging Tips

### Outline All Elements

```html
<!-- Add to body temporarily -->
<body class="[&_*]:outline [&_*]:outline-1 [&_*]:outline-red-500/20">
```

### Responsive Breakpoint Indicator

```html
<!-- Shows current breakpoint -->
<div class="fixed bottom-4 right-4 bg-black text-white px-2 py-1 text-xs rounded z-[9999]">
  <span class="sm:hidden">XS</span>
  <span class="hidden sm:inline md:hidden">SM</span>
  <span class="hidden md:inline lg:hidden">MD</span>
  <span class="hidden lg:inline xl:hidden">LG</span>
  <span class="hidden xl:inline 2xl:hidden">XL</span>
  <span class="hidden 2xl:inline">2XL</span>
</div>
```

### Visual Grid Overlay

```html
<div class="fixed inset-0 pointer-events-none z-[9999]
            grid grid-cols-12 gap-4 max-w-7xl mx-auto px-4">
  <div class="bg-blue-500/10 h-full"></div>
  <div class="bg-blue-500/10 h-full"></div>
  <!-- repeat 12 times -->
</div>
```

---

## Cheat Sheet — Day 5

```
TRANSITIONS
├── transition, transition-colors, transition-transform, transition-all
├── duration-150, duration-300, duration-500
├── ease-in, ease-out, ease-in-out, ease-linear
└── delay-75, delay-150, delay-300

TRANSFORMS
├── scale-95, scale-100, scale-105, scale-110
├── rotate-45, rotate-90, -rotate-45
├── translate-x-4, -translate-y-2
└── origin-center, origin-top-left

ANIMATIONS (built-in)
├── animate-spin    → Loading spinners
├── animate-ping    → Notifications
├── animate-pulse   → Skeleton loading
└── animate-bounce  → Attention

FILTERS
├── blur-md, blur-lg
├── backdrop-blur-md (glassmorphism)
├── grayscale, sepia, invert
├── brightness-50, brightness-150
└── drop-shadow-lg

GRADIENTS
├── bg-gradient-to-r/l/t/b/tr/br/tl/bl
├── from-X via-Y to-Z
└── bg-clip-text text-transparent (text gradient)

ARBITRARY VALUES
├── w-[300px], h-[calc(100vh-80px)]
├── bg-[#1da1f2], text-[22px]
├── grid-cols-[200px_1fr_200px]
└── [clip-path:polygon(...)]

PERFORMANCE
├── content: [...] in config
├── Avoid dynamic class concatenation
├── safelist: [...] for dynamic classes
└── corePlugins: { float: false }

BEST PRACTICES
├── Consistent class ordering
├── Mobile-first responsive
├── Extract repeated patterns
├── Use semantic design tokens
└── Install prettier-plugin-tailwindcss
```

---

## Bài tập cuối khóa

1. **Landing Page**: Tạo responsive landing page với:
   - Hero section với gradient background và custom animation
   - Features grid với hover transform effects
   - Glassmorphism testimonial cards
   - Dark mode support hoàn chỉnh

2. **Dashboard**: Tạo admin dashboard với:
   - Sticky header với backdrop-blur
   - Sidebar slide-in animation (mobile)
   - Stats cards với skeleton loading
   - Data table với hover states

3. **E-commerce Card**: Tạo product card với:
   - Image zoom on hover
   - Gradient overlay với title
   - "Add to cart" button với press effect
   - Badge với ping animation (sale indicator)

---

## Tài nguyên bổ sung

| Resource | Link | Mô tả |
|----------|------|-------|
| Tailwind Docs | tailwindcss.com/docs | Official documentation |
| Tailwind UI | tailwindui.com | Premium components |
| Headless UI | headlessui.com | Unstyled accessible components |
| Heroicons | heroicons.com | Icons by Tailwind team |
| Tailwind Play | play.tailwindcss.com | Online playground |

---

## 🎓 Hoàn thành Tailwind CSS!

Bạn đã hoàn thành 5 ngày học Tailwind CSS:

| Ngày | Nội dung đã học |
|------|-----------------|
| 1 | Spacing, Colors, Typography, Sizing |
| 2 | Flexbox, Grid, Position |
| 3 | Responsive design, States, Dark mode |
| 4 | @apply, Components, Config customization |
| 5 | Animations, Gradients, Filters, Best practices |

**Tiếp theo nên học:**

| Chủ đề | Lý do |
|--------|-------|
| Component Libraries | daisyUI, Shadcn/ui cho components có sẵn |
| Headless UI | Dropdowns, modals với accessibility |
| Framer Motion | Animation library kết hợp với Tailwind |
| Tailwind + Framework | Deep integration với React/Vue/Angular |

---

## Navigation

- [← Day 4: Components](./day-4-components.md)
- [Overview →](./00-overview.md)
