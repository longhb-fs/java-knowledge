# Day 5: Advanced Techniques + Best Practices

> Mục tiêu: Master animations, performance optimization, advanced patterns, và production best practices.

---

## 1. Transitions & Animations

### Transition Utilities

```html
<!-- Basic transition -->
<button class="transition hover:bg-blue-600">
  Default (colors, bg, borders, shadows, opacity, transform)
</button>

<!-- Specific properties -->
<div class="transition-colors hover:bg-blue-600">Colors only</div>
<div class="transition-opacity hover:opacity-50">Opacity only</div>
<div class="transition-transform hover:scale-105">Transform only</div>
<div class="transition-all hover:scale-105 hover:bg-blue-600">Everything</div>

<!-- Duration -->
<div class="transition duration-75">75ms</div>
<div class="transition duration-150">150ms (default)</div>
<div class="transition duration-300">300ms</div>
<div class="transition duration-500">500ms</div>
<div class="transition duration-700">700ms</div>
<div class="transition duration-1000">1000ms</div>

<!-- Timing function -->
<div class="transition ease-linear">Linear</div>
<div class="transition ease-in">Ease in (slow start)</div>
<div class="transition ease-out">Ease out (slow end)</div>
<div class="transition ease-in-out">Ease in-out (default)</div>

<!-- Delay -->
<div class="transition delay-150">Delay 150ms</div>
<div class="transition delay-300">Delay 300ms</div>
```

### Transform Utilities

```html
<!-- Scale -->
<div class="hover:scale-100">100% (normal)</div>
<div class="hover:scale-105">105%</div>
<div class="hover:scale-110">110%</div>
<div class="hover:scale-125">125%</div>
<div class="hover:scale-150">150%</div>
<div class="hover:scale-75">75%</div>
<div class="hover:scale-x-110">X-axis only</div>
<div class="hover:scale-y-110">Y-axis only</div>

<!-- Rotate -->
<div class="hover:rotate-45">45 degrees</div>
<div class="hover:rotate-90">90 degrees</div>
<div class="hover:rotate-180">180 degrees</div>
<div class="hover:-rotate-45">-45 degrees</div>

<!-- Translate -->
<div class="hover:translate-x-4">Right 1rem</div>
<div class="hover:-translate-x-4">Left 1rem</div>
<div class="hover:translate-y-4">Down 1rem</div>
<div class="hover:-translate-y-4">Up 1rem</div>
<div class="hover:-translate-y-1">Slight lift effect</div>

<!-- Skew -->
<div class="hover:skew-x-12">Skew X</div>
<div class="hover:skew-y-6">Skew Y</div>

<!-- Origin (for rotate/scale) -->
<div class="origin-center">Center (default)</div>
<div class="origin-top-left">Top-left</div>
<div class="origin-bottom-right">Bottom-right</div>
```

### Built-in Animations

```html
<div class="animate-spin">Spinning (360deg infinite)</div>
<div class="animate-ping">Ping (like notification)</div>
<div class="animate-pulse">Pulse (opacity fade)</div>
<div class="animate-bounce">Bounce (up and down)</div>
```

### 🔥 Animation Patterns

```html
<!-- Card hover effect -->
<div class="transition duration-300 ease-out
            hover:-translate-y-1 hover:shadow-xl">
  Card with lift effect
</div>

<!-- Button press effect -->
<button class="transition-transform active:scale-95">
  Press me
</button>

<!-- Image zoom on hover -->
<div class="overflow-hidden rounded-lg">
  <img class="transition duration-500 hover:scale-110" src="..." />
</div>

<!-- Fade in on load -->
<div class="animate-fade-in"> <!-- Custom animation needed -->
  Content
</div>

<!-- Staggered animation -->
<div class="space-y-2">
  <div class="animate-fade-in" style="animation-delay: 0ms">Item 1</div>
  <div class="animate-fade-in" style="animation-delay: 100ms">Item 2</div>
  <div class="animate-fade-in" style="animation-delay: 200ms">Item 3</div>
</div>

<!-- Loading skeleton -->
<div class="animate-pulse space-y-4">
  <div class="h-4 bg-gray-200 rounded w-3/4"></div>
  <div class="h-4 bg-gray-200 rounded w-1/2"></div>
  <div class="h-4 bg-gray-200 rounded w-5/6"></div>
</div>

<!-- Notification badge ping -->
<span class="relative flex h-3 w-3">
  <span class="animate-ping absolute inline-flex h-full w-full
               rounded-full bg-red-400 opacity-75"></span>
  <span class="relative inline-flex rounded-full h-3 w-3 bg-red-500"></span>
</span>
```

### Custom Animations (Config)

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      animation: {
        'fade-in': 'fadeIn 0.5s ease-out',
        'fade-in-up': 'fadeInUp 0.5s ease-out',
        'slide-in-right': 'slideInRight 0.3s ease-out',
        'scale-in': 'scaleIn 0.2s ease-out',
        'wiggle': 'wiggle 1s ease-in-out infinite',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        fadeInUp: {
          '0%': { opacity: '0', transform: 'translateY(10px)' },
          '100%': { opacity: '1', transform: 'translateY(0)' },
        },
        slideInRight: {
          '0%': { transform: 'translateX(100%)' },
          '100%': { transform: 'translateX(0)' },
        },
        scaleIn: {
          '0%': { transform: 'scale(0.9)', opacity: '0' },
          '100%': { transform: 'scale(1)', opacity: '1' },
        },
        wiggle: {
          '0%, 100%': { transform: 'rotate(-3deg)' },
          '50%': { transform: 'rotate(3deg)' },
        },
      },
    },
  },
}
```

---

## 2. Filters & Effects

### Blur

```html
<div class="blur-none">No blur</div>
<div class="blur-sm">Slight blur</div>
<div class="blur">Default blur</div>
<div class="blur-md">Medium blur</div>
<div class="blur-lg">Large blur</div>
<div class="blur-xl">Extra large blur</div>
<div class="blur-2xl">2XL blur</div>
<div class="blur-3xl">3XL blur</div>
```

### Backdrop Blur (Glassmorphism)

```html
<div class="backdrop-blur-md bg-white/30 p-6 rounded-xl">
  Glass effect card
</div>

<!-- Frosted glass navbar -->
<nav class="fixed top-0 inset-x-0
            backdrop-blur-lg bg-white/80 dark:bg-gray-900/80
            border-b border-gray-200/50">
  Navigation content
</nav>
```

### Grayscale, Sepia, Invert

```html
<img class="grayscale" src="..." />        <!-- B&W -->
<img class="grayscale-0" src="..." />      <!-- Full color -->
<img class="hover:grayscale-0 grayscale transition" src="..." />  <!-- Reveal on hover -->

<img class="sepia" src="..." />            <!-- Sepia tone -->
<img class="invert" src="..." />           <!-- Invert colors -->
```

### Brightness & Contrast

```html
<img class="brightness-50" src="..." />   <!-- Darker -->
<img class="brightness-100" src="..." />  <!-- Normal -->
<img class="brightness-150" src="..." />  <!-- Brighter -->

<img class="contrast-50" src="..." />     <!-- Low contrast -->
<img class="contrast-100" src="..." />    <!-- Normal -->
<img class="contrast-150" src="..." />    <!-- High contrast -->
```

### Drop Shadow (for irregular shapes)

```html
<!-- Works on images with transparency -->
<img class="drop-shadow-md" src="logo.png" />
<img class="drop-shadow-lg" src="logo.png" />
<img class="drop-shadow-2xl" src="logo.png" />
```

---

## 3. Gradients

### Linear Gradients

```html
<!-- Direction -->
<div class="bg-gradient-to-r from-blue-500 to-purple-500">Left to Right</div>
<div class="bg-gradient-to-l from-blue-500 to-purple-500">Right to Left</div>
<div class="bg-gradient-to-t from-blue-500 to-purple-500">Bottom to Top</div>
<div class="bg-gradient-to-b from-blue-500 to-purple-500">Top to Bottom</div>
<div class="bg-gradient-to-br from-blue-500 to-purple-500">Top-left to Bottom-right</div>
<div class="bg-gradient-to-tr from-blue-500 to-purple-500">Bottom-left to Top-right</div>

<!-- Three color stops -->
<div class="bg-gradient-to-r from-red-500 via-yellow-500 to-green-500">
  Rainbow
</div>

<!-- Gradient text -->
<h1 class="text-4xl font-bold
           bg-gradient-to-r from-blue-600 to-purple-600
           bg-clip-text text-transparent">
  Gradient Text
</h1>
```

### 🔥 Gradient Patterns

```html
<!-- Button with gradient -->
<button class="bg-gradient-to-r from-blue-500 to-purple-600
               hover:from-blue-600 hover:to-purple-700
               text-white px-6 py-3 rounded-lg">
  Gradient Button
</button>

<!-- Card with gradient border -->
<div class="p-[2px] bg-gradient-to-r from-pink-500 via-purple-500 to-blue-500 rounded-xl">
  <div class="bg-white dark:bg-gray-900 rounded-xl p-6">
    Content inside
  </div>
</div>

<!-- Gradient overlay on image -->
<div class="relative">
  <img src="..." class="w-full h-64 object-cover" />
  <div class="absolute inset-0 bg-gradient-to-t from-black/80 to-transparent">
    <div class="absolute bottom-4 left-4 text-white">
      <h3 class="font-bold">Title</h3>
    </div>
  </div>
</div>

<!-- Mesh gradient background -->
<div class="relative bg-gray-50 overflow-hidden">
  <div class="absolute -top-40 -right-40 w-80 h-80
              bg-purple-300 rounded-full blur-3xl opacity-30"></div>
  <div class="absolute -bottom-40 -left-40 w-80 h-80
              bg-pink-300 rounded-full blur-3xl opacity-30"></div>
  <div class="relative z-10 p-8">
    Content with mesh gradient background
  </div>
</div>
```

---

## 4. Arbitrary Values

Khi cần giá trị không có trong default scale:

```html
<!-- Arbitrary values với [] -->
<div class="w-[137px]">Exact width</div>
<div class="h-[calc(100vh-80px)]">Calculated height</div>
<div class="top-[117px]">Custom position</div>
<div class="bg-[#1da1f2]">Custom color (Twitter blue)</div>
<div class="text-[22px]">Custom font size</div>
<div class="p-[5px_10px_15px_20px]">Custom padding</div>
<div class="grid-cols-[200px_1fr_200px]">Custom grid</div>
<div class="font-[600]">Custom font weight</div>
<div class="z-[999]">Custom z-index</div>

<!-- Arbitrary properties với [] -->
<div class="[mask-type:alpha]">CSS property not in Tailwind</div>
<div class="[clip-path:polygon(0_0,100%_0,100%_75%,0_100%)]">Clip path</div>

<!-- With modifiers -->
<div class="hover:bg-[#ff6b6b]">Custom hover color</div>
<div class="md:w-[calc(100%-2rem)]">Responsive arbitrary</div>
```

---

## 5. Performance Optimization

### Production Build

```bash
# Tailwind automatically purges unused CSS in production
npm run build
```

### Content Configuration

```js
// tailwind.config.js
module.exports = {
  content: [
    './src/**/*.{html,js,ts,jsx,tsx,vue}',
    './components/**/*.{js,ts,jsx,tsx}',
    // Include any files that use Tailwind classes
  ],
  // ...
}
```

### ⚠️ Avoid Dynamic Class Names

```html
<!-- ❌ BAD: Dynamic class names won't be detected -->
<div class="text-{{ error ? 'red' : 'green' }}-500">

<!-- ❌ BAD: Concatenated class names -->
<div :class="`bg-${color}-500`">

<!-- ✅ GOOD: Complete class names -->
<div :class="error ? 'text-red-500' : 'text-green-500'">

<!-- ✅ GOOD: Object syntax -->
<div :class="{
  'text-red-500': error,
  'text-green-500': !error
}">

<!-- ✅ GOOD: If you MUST use dynamic, safelist them -->
```

### Safelist (When dynamic classes are needed)

```js
// tailwind.config.js
module.exports = {
  safelist: [
    'bg-red-500',
    'bg-green-500',
    'bg-blue-500',
    // Pattern-based
    {
      pattern: /bg-(red|green|blue)-(100|500|900)/,
    },
    // With variants
    {
      pattern: /text-(red|green|blue)-(500)/,
      variants: ['hover', 'dark'],
    },
  ],
}
```

### Reduce Bundle Size

```js
// tailwind.config.js
module.exports = {
  // Disable unused core plugins
  corePlugins: {
    float: false,       // Nếu không dùng float
    clear: false,
    skew: false,
    // ...
  },
}
```

---

## 6. Best Practices

### 1. Consistent Ordering

```html
<!-- Recommended order -->
<div class="
  relative flex items-center justify-between    /* Layout/Position */
  w-full max-w-md h-16                          /* Sizing */
  p-4 gap-4                                     /* Spacing */
  bg-white dark:bg-gray-800                     /* Background */
  border border-gray-200 rounded-lg shadow-md   /* Border/Shadow */
  text-gray-900 text-sm font-medium             /* Typography */
  hover:bg-gray-50                              /* States */
  transition-colors duration-200                /* Animation */
">
```

### 2. Use Semantic Class Names for Components

```css
/* When @apply makes sense */
@layer components {
  .btn-primary {
    @apply bg-blue-500 text-white px-4 py-2 rounded-lg
           hover:bg-blue-600 transition-colors;
  }
}
```

```html
<!-- Clear intent -->
<button class="btn-primary">Submit</button>
```

### 3. Extract Repeated Patterns

```tsx
// React: Component extraction
const Badge = ({ variant, children }) => {
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
};
```

### 4. Use Design Tokens

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        // Semantic tokens
        surface: {
          DEFAULT: '#ffffff',
          secondary: '#f9fafb',
          dark: '#1f2937',
        },
        content: {
          DEFAULT: '#111827',
          secondary: '#6b7280',
          inverse: '#ffffff',
        },
        accent: {
          DEFAULT: '#3b82f6',
          hover: '#2563eb',
        },
      },
    },
  },
}
```

```html
<div class="bg-surface text-content">
  <p class="text-content-secondary">Semantic colors</p>
  <button class="bg-accent hover:bg-accent-hover text-content-inverse">
    Action
  </button>
</div>
```

### 5. Breakpoint Strategy

```html
<!-- Mobile-first: Start small, add larger -->
<div class="
  flex flex-col          /* Mobile: vertical */
  md:flex-row            /* Tablet+: horizontal */
  lg:justify-between     /* Desktop+: space between */
">

<!-- NOT desktop-first (harder to maintain) -->
```

---

## 7. Common Patterns

### Responsive Hide/Show

```html
<!-- Show on mobile only -->
<div class="block md:hidden">Mobile only</div>

<!-- Show on desktop only -->
<div class="hidden md:block">Desktop only</div>

<!-- Show on tablet only -->
<div class="hidden md:block lg:hidden">Tablet only</div>
```

### Truncate Text

```html
<!-- Single line -->
<p class="truncate">Very long text that will be truncated...</p>

<!-- Multi-line (line clamp) -->
<p class="line-clamp-2">
  Long text that spans multiple lines will be limited to 2 lines
  with an ellipsis at the end...
</p>
```

### Aspect Ratio

```html
<div class="aspect-video">16:9 ratio</div>
<div class="aspect-square">1:1 ratio</div>
<div class="aspect-[4/3]">4:3 ratio</div>
```

### Sticky Elements

```html
<header class="sticky top-0 z-50 bg-white shadow">
  Sticky header
</header>

<aside class="sticky top-20 h-fit">
  Sidebar that sticks below header
</aside>
```

### Full-height Page

```html
<div class="min-h-screen flex flex-col">
  <header class="h-16">Header</header>
  <main class="flex-1">Main content (grows to fill)</main>
  <footer class="h-20">Footer</footer>
</div>
```

---

## 8. Debugging Tips

### Outline All Elements

```html
<!-- Add to body for debugging layout -->
<body class="[&_*]:outline [&_*]:outline-1 [&_*]:outline-red-500/20">
```

### Visual Grid Overlay

```html
<div class="fixed inset-0 pointer-events-none z-[9999]
            grid grid-cols-12 gap-4 max-w-7xl mx-auto px-4">
  <div class="bg-blue-500/10"></div>
  <div class="bg-blue-500/10"></div>
  <!-- repeat 12 times -->
</div>
```

### Check Responsive Breakpoint

```html
<div class="fixed bottom-4 right-4 bg-black text-white px-2 py-1 text-xs z-50">
  <span class="sm:hidden">XS</span>
  <span class="hidden sm:inline md:hidden">SM</span>
  <span class="hidden md:inline lg:hidden">MD</span>
  <span class="hidden lg:inline xl:hidden">LG</span>
  <span class="hidden xl:inline 2xl:hidden">XL</span>
  <span class="hidden 2xl:inline">2XL</span>
</div>
```

---

## Cheat Sheet — Day 5

```
TRANSITIONS & TRANSFORMS
├── transition, transition-colors, transition-transform
├── duration-300, delay-150
├── ease-in, ease-out, ease-in-out
├── scale-105, rotate-45, translate-x-4
└── origin-center, origin-top-left

ANIMATIONS
├── animate-spin, animate-ping, animate-pulse, animate-bounce
└── Custom: animation + keyframes in config

FILTERS
├── blur-md, blur-lg
├── backdrop-blur-md (glassmorphism)
├── grayscale, sepia, invert
├── brightness-150, contrast-125
└── drop-shadow-lg

GRADIENTS
├── bg-gradient-to-r from-X to-Y
├── via-X (middle color)
└── bg-clip-text text-transparent (text gradient)

ARBITRARY VALUES
├── w-[137px], h-[calc(100vh-80px)]
├── bg-[#1da1f2], text-[22px]
└── [mask-type:alpha]

PERFORMANCE
├── content: [...] paths in config
├── Avoid dynamic class concatenation
├── safelist: [...] for dynamic classes
└── corePlugins: { float: false } to disable unused

BEST PRACTICES
├── Consistent class ordering
├── Mobile-first responsive
├── Extract components with @apply
├── Use semantic design tokens
└── Component extraction > @apply for complex patterns
```

---

## Bài tập cuối khóa

1. **Landing Page**: Tạo responsive landing page với:
   - Hero section với gradient background và animations
   - Features grid với hover effects
   - Pricing cards với dark mode support
   - Testimonials carousel placeholder
   - Footer với newsletter form

2. **Dashboard**: Tạo admin dashboard với:
   - Responsive sidebar (collapsed on mobile)
   - Stats cards với skeleton loading
   - Data table với hover states
   - Glassmorphism modal

3. **E-commerce Card**: Tạo product card với:
   - Image hover zoom effect
   - Quick view overlay
   - Add to cart animation
   - Dark mode support

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

Bạn đã học xong Tailwind CSS trong 5 ngày. Tiếp theo:

| Chủ đề | Áp dụng |
|--------|---------|
| Component Libraries | daisyUI, Flowbite, Shadcn/ui |
| Headless UI | Accessible dropdowns, modals |
| Tailwind + React/Vue/Angular | Framework integration |
| Animation Libraries | Framer Motion + Tailwind |

---

## Navigation

- [← Day 4: Components](./day-4-components.md)
- [Overview →](./00-overview.md)
