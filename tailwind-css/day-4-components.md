# Day 4: Components + Customization

> Mục tiêu: Build reusable components, sử dụng @apply, customize config, và tổ chức code.

---

## 1. Component Patterns

### Khi nào nên tạo component?

```
✅ TẠO COMPONENT khi:
├── Lặp lại nhiều lần (buttons, cards, inputs)
├── Có logic phức tạp (dropdown, modal)
└── Cần maintain consistency

❌ KHÔNG CẦN COMPONENT khi:
├── One-off layouts (hero sections)
├── Simple utility combinations
└── Highly context-specific styles
```

### Pattern 1: Copy/Paste (Simple)

```html
<!-- Chỉ cần copy paste khi dùng lại -->
<button class="bg-blue-500 hover:bg-blue-600 text-white font-medium
               px-4 py-2 rounded-lg transition-colors">
  Button
</button>
```

### Pattern 2: @apply trong CSS (DRY)

```css
/* styles.css */
@layer components {
  .btn {
    @apply px-4 py-2 rounded-lg font-medium transition-colors;
  }

  .btn-primary {
    @apply bg-blue-500 text-white hover:bg-blue-600;
  }

  .btn-secondary {
    @apply bg-gray-200 text-gray-800 hover:bg-gray-300;
  }

  .btn-outline {
    @apply border-2 border-blue-500 text-blue-500
           hover:bg-blue-500 hover:text-white;
  }
}
```

```html
<button class="btn btn-primary">Primary</button>
<button class="btn btn-secondary">Secondary</button>
<button class="btn btn-outline">Outline</button>
```

### Pattern 3: Framework Components (React/Vue/Angular)

```tsx
// Button.tsx (React)
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'outline';
  size?: 'sm' | 'md' | 'lg';
  children: React.ReactNode;
}

const variants = {
  primary: 'bg-blue-500 text-white hover:bg-blue-600',
  secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
  outline: 'border-2 border-blue-500 text-blue-500 hover:bg-blue-500 hover:text-white',
};

const sizes = {
  sm: 'px-3 py-1.5 text-sm',
  md: 'px-4 py-2',
  lg: 'px-6 py-3 text-lg',
};

export function Button({ variant = 'primary', size = 'md', children }: ButtonProps) {
  return (
    <button className={`
      rounded-lg font-medium transition-colors
      ${variants[variant]}
      ${sizes[size]}
    `}>
      {children}
    </button>
  );
}
```

```tsx
// Angular component
@Component({
  selector: 'app-button',
  template: `
    <button [ngClass]="buttonClasses">
      <ng-content></ng-content>
    </button>
  `,
})
export class ButtonComponent {
  @Input() variant: 'primary' | 'secondary' | 'outline' = 'primary';
  @Input() size: 'sm' | 'md' | 'lg' = 'md';

  get buttonClasses(): string {
    const base = 'rounded-lg font-medium transition-colors';
    const variants = {
      primary: 'bg-blue-500 text-white hover:bg-blue-600',
      secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
      outline: 'border-2 border-blue-500 text-blue-500 hover:bg-blue-500 hover:text-white',
    };
    const sizes = {
      sm: 'px-3 py-1.5 text-sm',
      md: 'px-4 py-2',
      lg: 'px-6 py-3 text-lg',
    };
    return `${base} ${variants[this.variant]} ${sizes[this.size]}`;
  }
}
```

---

## 2. Common UI Components

### Buttons

```css
@layer components {
  .btn {
    @apply inline-flex items-center justify-center gap-2
           font-medium rounded-lg transition-all duration-200
           focus:outline-none focus:ring-2 focus:ring-offset-2
           disabled:opacity-50 disabled:cursor-not-allowed;
  }

  /* Sizes */
  .btn-xs { @apply px-2 py-1 text-xs; }
  .btn-sm { @apply px-3 py-1.5 text-sm; }
  .btn-md { @apply px-4 py-2; }
  .btn-lg { @apply px-6 py-3 text-lg; }

  /* Variants */
  .btn-primary {
    @apply bg-blue-500 text-white hover:bg-blue-600 focus:ring-blue-500;
  }
  .btn-danger {
    @apply bg-red-500 text-white hover:bg-red-600 focus:ring-red-500;
  }
  .btn-ghost {
    @apply text-gray-700 hover:bg-gray-100 focus:ring-gray-500;
  }
}
```

### Inputs

```css
@layer components {
  .input {
    @apply w-full px-4 py-2 rounded-lg
           border border-gray-300
           bg-white text-gray-900
           placeholder-gray-400
           focus:border-blue-500 focus:ring-2 focus:ring-blue-500/20
           transition-colors duration-200
           disabled:bg-gray-100 disabled:cursor-not-allowed;
  }

  .input-error {
    @apply border-red-500 focus:border-red-500 focus:ring-red-500/20;
  }

  .label {
    @apply block text-sm font-medium text-gray-700 mb-1;
  }

  .label-required::after {
    @apply text-red-500 ml-0.5;
    content: '*';
  }

  .input-hint {
    @apply text-sm text-gray-500 mt-1;
  }

  .input-error-message {
    @apply text-sm text-red-500 mt-1;
  }
}
```

### Cards

```css
@layer components {
  .card {
    @apply bg-white rounded-xl shadow-md overflow-hidden;
  }

  .card-bordered {
    @apply border border-gray-200 shadow-none;
  }

  .card-header {
    @apply px-6 py-4 border-b border-gray-200;
  }

  .card-body {
    @apply p-6;
  }

  .card-footer {
    @apply px-6 py-4 bg-gray-50 border-t border-gray-200;
  }
}
```

```html
<div class="card">
  <div class="card-header">
    <h3 class="font-semibold">Card Title</h3>
  </div>
  <div class="card-body">
    <p>Card content goes here.</p>
  </div>
  <div class="card-footer">
    <button class="btn btn-primary btn-sm">Action</button>
  </div>
</div>
```

### Badges

```css
@layer components {
  .badge {
    @apply inline-flex items-center px-2.5 py-0.5 rounded-full
           text-xs font-medium;
  }

  .badge-gray { @apply bg-gray-100 text-gray-800; }
  .badge-blue { @apply bg-blue-100 text-blue-800; }
  .badge-green { @apply bg-green-100 text-green-800; }
  .badge-red { @apply bg-red-100 text-red-800; }
  .badge-yellow { @apply bg-yellow-100 text-yellow-800; }

  /* Dot badge */
  .badge-dot::before {
    @apply w-1.5 h-1.5 rounded-full mr-1.5;
    content: '';
  }
  .badge-dot-green::before { @apply bg-green-500; }
  .badge-dot-red::before { @apply bg-red-500; }
}
```

### Alerts

```css
@layer components {
  .alert {
    @apply flex items-start gap-3 p-4 rounded-lg;
  }

  .alert-info {
    @apply bg-blue-50 text-blue-800 border border-blue-200;
  }

  .alert-success {
    @apply bg-green-50 text-green-800 border border-green-200;
  }

  .alert-warning {
    @apply bg-yellow-50 text-yellow-800 border border-yellow-200;
  }

  .alert-error {
    @apply bg-red-50 text-red-800 border border-red-200;
  }
}
```

---

## 3. Tailwind Config Customization

### Extending Theme

```js
// tailwind.config.js
module.exports = {
  content: ['./src/**/*.{html,js,ts,jsx,tsx}'],
  theme: {
    extend: {
      // Add custom colors
      colors: {
        brand: {
          50: '#eff6ff',
          100: '#dbeafe',
          500: '#3b82f6',
          600: '#2563eb',
          900: '#1e3a8a',
        },
        // Single color
        primary: '#3b82f6',
      },

      // Custom fonts
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
        display: ['Poppins', 'sans-serif'],
        mono: ['Fira Code', 'monospace'],
      },

      // Custom spacing
      spacing: {
        '18': '4.5rem',
        '88': '22rem',
        '128': '32rem',
      },

      // Custom border radius
      borderRadius: {
        '4xl': '2rem',
      },

      // Custom shadows
      boxShadow: {
        'soft': '0 2px 15px -3px rgba(0, 0, 0, 0.07)',
        'hard': '0 4px 6px -1px rgba(0, 0, 0, 0.2)',
      },

      // Custom animations
      animation: {
        'fade-in': 'fadeIn 0.3s ease-in-out',
        'slide-up': 'slideUp 0.3s ease-out',
        'spin-slow': 'spin 3s linear infinite',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideUp: {
          '0%': { transform: 'translateY(10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
      },
    },
  },
  plugins: [],
}
```

### Overriding Theme (không dùng extend)

```js
// tailwind.config.js
module.exports = {
  theme: {
    // This REPLACES default screens (not recommended)
    screens: {
      'tablet': '640px',
      'laptop': '1024px',
      'desktop': '1280px',
    },

    // Better: use extend to ADD to defaults
    extend: {
      screens: {
        '3xl': '1600px',
      },
    },
  },
}
```

### Custom Variants

```js
// tailwind.config.js
const plugin = require('tailwindcss/plugin');

module.exports = {
  plugins: [
    plugin(function({ addVariant }) {
      // Add 'hocus' variant (hover + focus)
      addVariant('hocus', ['&:hover', '&:focus']);

      // Add 'not-first' variant
      addVariant('not-first', '&:not(:first-child)');

      // Add 'supports-backdrop' variant
      addVariant('supports-backdrop', '@supports (backdrop-filter: blur(0px))');
    }),
  ],
}
```

```html
<button class="hocus:bg-blue-600">Hover or Focus</button>
<li class="not-first:border-t">List item</li>
```

---

## 4. Plugins

### Official Plugins

```bash
npm install -D @tailwindcss/forms @tailwindcss/typography @tailwindcss/aspect-ratio
```

```js
// tailwind.config.js
module.exports = {
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
    require('@tailwindcss/aspect-ratio'),
  ],
}
```

### @tailwindcss/forms

```html
<!-- Đã styled sẵn form elements -->
<input type="text" class="form-input rounded-md" />
<select class="form-select rounded-md">...</select>
<input type="checkbox" class="form-checkbox rounded" />
<input type="radio" class="form-radio" />
<textarea class="form-textarea rounded-md"></textarea>
```

### @tailwindcss/typography

```html
<!-- Prose: styled content từ CMS/Markdown -->
<article class="prose lg:prose-xl dark:prose-invert">
  <h1>Article Title</h1>
  <p>This paragraph is automatically styled...</p>
  <ul>
    <li>List items too</li>
  </ul>
  <pre><code>Code blocks as well</code></pre>
</article>
```

### Custom Plugin

```js
// tailwind.config.js
const plugin = require('tailwindcss/plugin');

module.exports = {
  plugins: [
    plugin(function({ addComponents, addUtilities, theme }) {
      // Add component classes
      addComponents({
        '.card-custom': {
          backgroundColor: theme('colors.white'),
          borderRadius: theme('borderRadius.lg'),
          padding: theme('spacing.6'),
          boxShadow: theme('boxShadow.xl'),
        },
      });

      // Add utility classes
      addUtilities({
        '.text-shadow': {
          textShadow: '2px 2px 4px rgba(0,0,0,0.1)',
        },
        '.text-shadow-lg': {
          textShadow: '4px 4px 8px rgba(0,0,0,0.2)',
        },
      });
    }),
  ],
}
```

---

## 5. CSS Layers

```css
/* Order matters! base → components → utilities */

@layer base {
  /* Reset & base styles */
  html {
    @apply scroll-smooth;
  }

  body {
    @apply bg-gray-50 text-gray-900 antialiased;
  }

  h1, h2, h3 {
    @apply font-bold;
  }
}

@layer components {
  /* Reusable components */
  .btn { ... }
  .card { ... }
  .input { ... }
}

@layer utilities {
  /* Custom utilities */
  .text-gradient {
    @apply bg-clip-text text-transparent bg-gradient-to-r;
  }
}
```

### Layer Priority

```
base       → lowest priority (can be overridden)
components → medium priority
utilities  → highest priority (wins over components)
```

---

## 6. Ví dụ: Component Library

### Complete Button System

```css
/* components/button.css */
@layer components {
  /* Base button */
  .btn {
    @apply inline-flex items-center justify-center gap-2
           font-medium rounded-lg
           transition-all duration-200
           focus:outline-none focus:ring-2 focus:ring-offset-2
           disabled:opacity-50 disabled:pointer-events-none;
  }

  /* Sizes */
  .btn-xs { @apply h-7 px-2 text-xs; }
  .btn-sm { @apply h-8 px-3 text-sm; }
  .btn-md { @apply h-10 px-4 text-sm; }
  .btn-lg { @apply h-12 px-6 text-base; }
  .btn-xl { @apply h-14 px-8 text-lg; }

  /* Solid variants */
  .btn-primary {
    @apply bg-blue-500 text-white
           hover:bg-blue-600 active:bg-blue-700
           focus:ring-blue-500;
  }

  .btn-secondary {
    @apply bg-gray-100 text-gray-900
           hover:bg-gray-200 active:bg-gray-300
           focus:ring-gray-500;
  }

  .btn-success {
    @apply bg-green-500 text-white
           hover:bg-green-600 active:bg-green-700
           focus:ring-green-500;
  }

  .btn-danger {
    @apply bg-red-500 text-white
           hover:bg-red-600 active:bg-red-700
           focus:ring-red-500;
  }

  /* Outline variants */
  .btn-outline-primary {
    @apply border-2 border-blue-500 text-blue-500
           hover:bg-blue-500 hover:text-white
           focus:ring-blue-500;
  }

  /* Ghost variants */
  .btn-ghost {
    @apply text-gray-600
           hover:bg-gray-100
           focus:ring-gray-500;
  }

  /* Icon button */
  .btn-icon {
    @apply !p-0 aspect-square;
  }
  .btn-icon.btn-sm { @apply w-8; }
  .btn-icon.btn-md { @apply w-10; }
  .btn-icon.btn-lg { @apply w-12; }

  /* Loading state */
  .btn-loading {
    @apply relative text-transparent pointer-events-none;
  }
  .btn-loading::after {
    @apply absolute inset-0 flex items-center justify-center;
    content: '';
    @apply w-4 h-4 border-2 border-current border-t-transparent rounded-full animate-spin;
  }

  /* Full width */
  .btn-block {
    @apply w-full;
  }
}
```

```html
<!-- Usage -->
<button class="btn btn-primary btn-md">Primary Button</button>
<button class="btn btn-secondary btn-sm">Small Secondary</button>
<button class="btn btn-outline-primary btn-lg">Large Outline</button>
<button class="btn btn-danger btn-md btn-loading">Loading...</button>
<button class="btn btn-ghost btn-icon btn-md">
  <svg>...</svg>
</button>
```

---

## 7. Tổ chức CSS Files

```
src/
├── styles/
│   ├── main.css              # Entry point
│   ├── base/
│   │   ├── reset.css         # Custom resets
│   │   └── typography.css    # Base typography
│   ├── components/
│   │   ├── buttons.css
│   │   ├── forms.css
│   │   ├── cards.css
│   │   └── modals.css
│   └── utilities/
│       └── custom.css        # Custom utilities
```

```css
/* main.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Base */
@import './base/reset.css';
@import './base/typography.css';

/* Components */
@import './components/buttons.css';
@import './components/forms.css';
@import './components/cards.css';
@import './components/modals.css';

/* Custom utilities */
@import './utilities/custom.css';
```

---

## Cheat Sheet — Day 4

```
@APPLY
├── @layer components { .btn { @apply px-4 py-2 rounded; } }
├── Giữ được responsive/state modifiers
└── Dùng cho repeating patterns

CONFIG CUSTOMIZATION
├── theme.extend.colors.brand: { 500: '#...' }
├── theme.extend.fontFamily.sans: ['Inter', ...]
├── theme.extend.spacing: { '18': '4.5rem' }
└── plugins: [require('@tailwindcss/forms')]

LAYERS (priority low → high)
├── @layer base       → resets, html/body defaults
├── @layer components → .btn, .card, .input
└── @layer utilities  → .text-shadow, custom utils

PLUGINS
├── @tailwindcss/forms      → styled form elements
├── @tailwindcss/typography → prose content styling
└── Custom plugins          → addComponents, addUtilities
```

---

## Bài tập

1. **Button System**: Tạo complete button component với sizes (sm/md/lg), variants (primary/secondary/outline/ghost), states (loading/disabled)
2. **Form Components**: Tạo styled input, select, checkbox, radio với error states
3. **Card Variants**: Tạo card component với variants (default/bordered/elevated) và slots (header/body/footer)

---

## Navigation

- [← Day 3: Responsive + States](./day-3-responsive-states.md)
- [Day 5: Advanced →](./day-5-advanced.md)
