# Day 4: Components + Customization

> Mục tiêu: Hiểu khi nào dùng @apply, cách customize config, và build reusable components.

---

## 1. Vấn đề: Tailwind Class dài quá!

### Đây có phải vấn đề thật không?

Khi viết Tailwind, bạn sẽ thấy HTML như này:

```html
<button class="inline-flex items-center justify-center gap-2 px-4 py-2
               bg-blue-500 text-white font-medium rounded-lg
               hover:bg-blue-600 focus:outline-none focus:ring-2
               focus:ring-blue-500 focus:ring-offset-2
               transition-colors duration-200
               disabled:opacity-50 disabled:cursor-not-allowed">
  Click me
</button>
```

**Phản ứng thường gặp:** "Quá dài! Cần tạo class `.btn-primary` như CSS truyền thống!"

**Nhưng khoan, hãy nghĩ lại:**

```
┌─────────────────────────────────────────────────────────────┐
│  TRƯỚC KHI TẠO COMPONENT, HỎI:                              │
│                                                              │
│  1. Button này lặp lại bao nhiêu lần trong project?         │
│     ├── 1-2 lần → Copy/paste đủ rồi                         │
│     ├── 3-5 lần → Cân nhắc @apply                           │
│     └── 10+ lần → Tạo component                             │
│                                                              │
│  2. Có dùng framework (React/Vue/Angular) không?            │
│     ├── Có → Tạo JS/TS component, KHÔNG cần @apply          │
│     └── Không → Có thể dùng @apply                          │
│                                                              │
│  3. Style có thay đổi theo context không?                   │
│     ├── Có → Giữ utilities, truyền class từ parent          │
│     └── Không → Có thể extract                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Pattern 1: Copy/Paste (Đơn giản nhất)

### Khi nào dùng?

- Element xuất hiện 1-3 lần
- Style không cần maintain tập trung
- Prototyping nhanh

```html
<!-- Chỗ 1 -->
<button class="bg-blue-500 text-white px-4 py-2 rounded-lg hover:bg-blue-600">
  Submit
</button>

<!-- Chỗ 2 -->
<button class="bg-blue-500 text-white px-4 py-2 rounded-lg hover:bg-blue-600">
  Save
</button>
```

**Lợi ích:**
- Không tạo abstraction không cần thiết
- Dễ customize từng button riêng biệt
- Không có thêm layer phức tạp

---

## 3. Pattern 2: @apply (Extract to CSS)

### Khi nào dùng?

- **Không dùng** React/Vue/Angular
- Lặp lại nhiều lần trong pure HTML
- Muốn maintain ở 1 chỗ

### 🔄 So sánh CSS truyền thống vs @apply

**CSS truyền thống:**
```css
/* styles.css */
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  font-weight: 500;
  border-radius: 0.5rem;
  transition: all 0.2s;
}

.btn:focus {
  outline: none;
  box-shadow: 0 0 0 2px #fff, 0 0 0 4px #3b82f6;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background-color: #3b82f6;
  color: white;
}

.btn-primary:hover {
  background-color: #2563eb;
}
```

**Tailwind @apply:**
```css
/* styles.css */
@layer components {
  .btn {
    @apply inline-flex items-center justify-center gap-2
           px-4 py-2 font-medium rounded-lg
           transition-all duration-200
           focus:outline-none focus:ring-2 focus:ring-offset-2
           disabled:opacity-50 disabled:cursor-not-allowed;
  }

  .btn-primary {
    @apply bg-blue-500 text-white
           hover:bg-blue-600
           focus:ring-blue-500;
  }
}
```

### 💡 Giải thích @layer components

```
┌─────────────────────────────────────────────────────────────┐
│  TẠI SAO CẦN @layer components?                              │
│                                                              │
│  Tailwind CSS có 3 layers (thứ tự priority):                │
│                                                              │
│  @layer base       ← Reset, HTML elements (h1, p, a...)     │
│       ↓                                                      │
│  @layer components ← .btn, .card, .input (component class)  │
│       ↓                                                      │
│  @layer utilities  ← .bg-blue-500, .p-4 (utilities)         │
│                                                              │
│  Priority: utilities > components > base                     │
│                                                              │
│  ➡️ Nếu bạn viết .btn trong @layer components,               │
│     thì utilities class vẫn có thể override nó:             │
│                                                              │
│     <button class="btn btn-primary bg-red-500">             │
│     → bg-red-500 sẽ override bg-blue-500 từ btn-primary     │
└─────────────────────────────────────────────────────────────┘
```

### Complete Button System với @apply

```css
/* components/buttons.css */
@layer components {
  /* ===== BASE BUTTON ===== */
  .btn {
    @apply inline-flex items-center justify-center gap-2
           font-medium rounded-lg
           transition-all duration-200
           focus:outline-none focus:ring-2 focus:ring-offset-2
           disabled:opacity-50 disabled:pointer-events-none;
  }

  /* ===== SIZES ===== */
  /*
   * Size chart:
   * ┌─────────┬────────────┬───────────┬───────────┐
   * │ Size    │ Height     │ Padding   │ Font      │
   * ├─────────┼────────────┼───────────┼───────────┤
   * │ xs      │ h-7 (28px) │ px-2      │ text-xs   │
   * │ sm      │ h-8 (32px) │ px-3      │ text-sm   │
   * │ md      │ h-10 (40px)│ px-4      │ text-sm   │
   * │ lg      │ h-12 (48px)│ px-6      │ text-base │
   * │ xl      │ h-14 (56px)│ px-8      │ text-lg   │
   * └─────────┴────────────┴───────────┴───────────┘
   */
  .btn-xs { @apply h-7 px-2 text-xs; }
  .btn-sm { @apply h-8 px-3 text-sm; }
  .btn-md { @apply h-10 px-4 text-sm; }
  .btn-lg { @apply h-12 px-6 text-base; }
  .btn-xl { @apply h-14 px-8 text-lg; }

  /* ===== SOLID VARIANTS ===== */
  /*
   * Solid button pattern:
   * - Background: color-500
   * - Hover: color-600 (darker)
   * - Active: color-700 (even darker)
   * - Focus ring: color-500
   */
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

  .btn-warning {
    @apply bg-yellow-500 text-white
           hover:bg-yellow-600 active:bg-yellow-700
           focus:ring-yellow-500;
  }

  /* ===== OUTLINE VARIANTS ===== */
  /*
   * Outline button pattern:
   * - No background (transparent)
   * - Border: color-500
   * - Text: color-500
   * - Hover: fill background, white text
   */
  .btn-outline-primary {
    @apply border-2 border-blue-500 text-blue-500 bg-transparent
           hover:bg-blue-500 hover:text-white
           focus:ring-blue-500;
  }

  .btn-outline-secondary {
    @apply border-2 border-gray-300 text-gray-700 bg-transparent
           hover:bg-gray-100
           focus:ring-gray-500;
  }

  .btn-outline-danger {
    @apply border-2 border-red-500 text-red-500 bg-transparent
           hover:bg-red-500 hover:text-white
           focus:ring-red-500;
  }

  /* ===== GHOST VARIANTS ===== */
  /*
   * Ghost button pattern:
   * - No background, no border
   * - Hover: light background
   */
  .btn-ghost {
    @apply text-gray-600 bg-transparent
           hover:bg-gray-100
           focus:ring-gray-500;
  }

  /* ===== SPECIAL MODIFIERS ===== */

  /* Icon-only button (square) */
  .btn-icon {
    @apply !p-0 aspect-square;
  }
  .btn-icon.btn-sm { @apply w-8; }
  .btn-icon.btn-md { @apply w-10; }
  .btn-icon.btn-lg { @apply w-12; }

  /* Full width */
  .btn-block {
    @apply w-full;
  }

  /* Loading state */
  .btn-loading {
    @apply relative text-transparent pointer-events-none;
  }
  .btn-loading::after {
    content: '';
    @apply absolute inset-0 m-auto
           w-4 h-4 border-2 border-current border-t-transparent
           rounded-full animate-spin;
  }
}
```

### Sử dụng:

```html
<!-- Combinations -->
<button class="btn btn-primary btn-md">Primary Medium</button>
<button class="btn btn-secondary btn-sm">Secondary Small</button>
<button class="btn btn-outline-primary btn-lg">Outline Large</button>
<button class="btn btn-danger btn-md btn-loading">Loading...</button>
<button class="btn btn-ghost btn-icon btn-md">
  <svg><!-- icon --></svg>
</button>

<!-- Override với utilities vẫn hoạt động -->
<button class="btn btn-primary btn-md rounded-full">
  Pill shape (override rounded-lg)
</button>
```

---

## 4. Pattern 3: Framework Components (React/Vue/Angular)

### 💡 Tại sao nên dùng pattern này?

```
┌─────────────────────────────────────────────────────────────┐
│  FRAMEWORK COMPONENTS vs @APPLY                              │
│                                                              │
│  @apply:                                                    │
│  - Chỉ extract CSS                                          │
│  - Không có logic (props, variants)                         │
│  - Phải tạo nhiều class (.btn-sm, .btn-md, .btn-lg...)     │
│                                                              │
│  Framework Component:                                        │
│  - Extract CSS + Logic                                       │
│  - Props: size="sm", variant="primary"                      │
│  - TypeScript support, autocomplete                         │
│  - Conditional rendering, slots, children                   │
│  - Reusable across project                                  │
│                                                              │
│  ➡️ Nếu đã dùng React/Vue/Angular → Component > @apply      │
└─────────────────────────────────────────────────────────────┘
```

### React Component Example

```tsx
// components/Button.tsx
import { type ReactNode, type ButtonHTMLAttributes } from 'react';

// 1. Define types for props
type ButtonVariant = 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger';
type ButtonSize = 'xs' | 'sm' | 'md' | 'lg' | 'xl';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: ButtonVariant;
  size?: ButtonSize;
  isLoading?: boolean;
  leftIcon?: ReactNode;
  rightIcon?: ReactNode;
  children: ReactNode;
}

// 2. Define style mappings
const baseStyles = `
  inline-flex items-center justify-center gap-2
  font-medium rounded-lg
  transition-all duration-200
  focus:outline-none focus:ring-2 focus:ring-offset-2
  disabled:opacity-50 disabled:pointer-events-none
`;

const variantStyles: Record<ButtonVariant, string> = {
  primary: 'bg-blue-500 text-white hover:bg-blue-600 focus:ring-blue-500',
  secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200 focus:ring-gray-500',
  outline: 'border-2 border-blue-500 text-blue-500 hover:bg-blue-500 hover:text-white focus:ring-blue-500',
  ghost: 'text-gray-600 hover:bg-gray-100 focus:ring-gray-500',
  danger: 'bg-red-500 text-white hover:bg-red-600 focus:ring-red-500',
};

const sizeStyles: Record<ButtonSize, string> = {
  xs: 'h-7 px-2 text-xs',
  sm: 'h-8 px-3 text-sm',
  md: 'h-10 px-4 text-sm',
  lg: 'h-12 px-6 text-base',
  xl: 'h-14 px-8 text-lg',
};

// 3. Component implementation
export function Button({
  variant = 'primary',
  size = 'md',
  isLoading = false,
  leftIcon,
  rightIcon,
  children,
  className = '',
  disabled,
  ...props
}: ButtonProps) {
  return (
    <button
      className={`
        ${baseStyles}
        ${variantStyles[variant]}
        ${sizeStyles[size]}
        ${isLoading ? 'relative text-transparent pointer-events-none' : ''}
        ${className}
      `}
      disabled={disabled || isLoading}
      {...props}
    >
      {leftIcon && <span className="shrink-0">{leftIcon}</span>}
      {children}
      {rightIcon && <span className="shrink-0">{rightIcon}</span>}

      {/* Loading spinner */}
      {isLoading && (
        <span className="absolute inset-0 flex items-center justify-center">
          <span className="w-4 h-4 border-2 border-current border-t-transparent rounded-full animate-spin" />
        </span>
      )}
    </button>
  );
}
```

### Sử dụng React Component:

```tsx
// Có autocomplete, type checking
<Button variant="primary" size="md">
  Submit
</Button>

<Button variant="outline" size="lg" isLoading>
  Processing...
</Button>

<Button
  variant="secondary"
  leftIcon={<SearchIcon />}
  onClick={() => handleSearch()}
>
  Search
</Button>

// Override styles khi cần
<Button variant="primary" className="rounded-full w-full">
  Full width pill button
</Button>
```

### Angular Component Example

```typescript
// button.component.ts
import { Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';

type ButtonVariant = 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger';
type ButtonSize = 'xs' | 'sm' | 'md' | 'lg' | 'xl';

@Component({
  selector: 'app-button',
  standalone: true,
  imports: [CommonModule],
  template: `
    <button
      [ngClass]="buttonClasses"
      [disabled]="disabled || isLoading"
      [type]="type"
    >
      <!-- Left icon slot -->
      <ng-content select="[leftIcon]"></ng-content>

      <!-- Main content -->
      <span [class.invisible]="isLoading">
        <ng-content></ng-content>
      </span>

      <!-- Right icon slot -->
      <ng-content select="[rightIcon]"></ng-content>

      <!-- Loading spinner -->
      <span
        *ngIf="isLoading"
        class="absolute inset-0 flex items-center justify-center"
      >
        <span class="w-4 h-4 border-2 border-current border-t-transparent rounded-full animate-spin"></span>
      </span>
    </button>
  `,
})
export class ButtonComponent {
  @Input() variant: ButtonVariant = 'primary';
  @Input() size: ButtonSize = 'md';
  @Input() isLoading = false;
  @Input() disabled = false;
  @Input() type: 'button' | 'submit' | 'reset' = 'button';
  @Input() extraClass = '';

  private readonly baseStyles = `
    inline-flex items-center justify-center gap-2
    font-medium rounded-lg transition-all duration-200
    focus:outline-none focus:ring-2 focus:ring-offset-2
    disabled:opacity-50 disabled:pointer-events-none
    relative
  `;

  private readonly variantMap: Record<ButtonVariant, string> = {
    primary: 'bg-blue-500 text-white hover:bg-blue-600 focus:ring-blue-500',
    secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200 focus:ring-gray-500',
    outline: 'border-2 border-blue-500 text-blue-500 hover:bg-blue-500 hover:text-white',
    ghost: 'text-gray-600 hover:bg-gray-100 focus:ring-gray-500',
    danger: 'bg-red-500 text-white hover:bg-red-600 focus:ring-red-500',
  };

  private readonly sizeMap: Record<ButtonSize, string> = {
    xs: 'h-7 px-2 text-xs',
    sm: 'h-8 px-3 text-sm',
    md: 'h-10 px-4 text-sm',
    lg: 'h-12 px-6 text-base',
    xl: 'h-14 px-8 text-lg',
  };

  get buttonClasses(): string {
    return `
      ${this.baseStyles}
      ${this.variantMap[this.variant]}
      ${this.sizeMap[this.size]}
      ${this.extraClass}
    `.trim();
  }
}
```

### Sử dụng Angular Component:

```html
<app-button variant="primary" size="md">
  Submit
</app-button>

<app-button variant="outline" size="lg" [isLoading]="true">
  Processing...
</app-button>

<app-button variant="secondary" (click)="handleSearch()">
  <svg leftIcon><!-- icon --></svg>
  Search
</app-button>
```

---

## 5. Common UI Components với @apply

### Input System

```css
/* components/forms.css */
@layer components {
  /* ===== LABEL ===== */
  .form-label {
    @apply block text-sm font-medium text-gray-700 mb-1;
  }

  .form-label-required::after {
    content: ' *';
    @apply text-red-500;
  }

  /* ===== INPUT ===== */
  /*
   * Input states diagram:
   *
   * ┌─────────────────────────────────────────┐
   * │  DEFAULT                                │
   * │  ┌─────────────────────────────────┐   │
   * │  │ border-gray-300                 │   │
   * │  └─────────────────────────────────┘   │
   * │                                         │
   * │  FOCUS                                  │
   * │  ┌─────────────────────────────────┐   │
   * │  │ border-blue-500                 │   │
   * │  │ ╔═══════════════════════════╗   │   │  ← ring-2 ring-blue-500/20
   * │  │ ║                           ║   │   │
   * │  │ ╚═══════════════════════════╝   │   │
   * │  └─────────────────────────────────┘   │
   * │                                         │
   * │  ERROR                                  │
   * │  ┌─────────────────────────────────┐   │
   * │  │ border-red-500                  │   │
   * │  │ ring-red-500/20                 │   │
   * │  └─────────────────────────────────┘   │
   * │  ⚠️ Error message here                 │
   * │                                         │
   * │  DISABLED                               │
   * │  ┌─────────────────────────────────┐   │
   * │  │ bg-gray-100, cursor-not-allowed │   │
   * │  └─────────────────────────────────┘   │
   * └─────────────────────────────────────────┘
   */
  .form-input {
    @apply w-full px-4 py-2.5 rounded-lg
           border border-gray-300
           bg-white text-gray-900
           placeholder-gray-400
           focus:border-blue-500 focus:ring-2 focus:ring-blue-500/20
           transition-colors duration-200
           disabled:bg-gray-100 disabled:text-gray-500 disabled:cursor-not-allowed;
  }

  .form-input-error {
    @apply border-red-500
           focus:border-red-500 focus:ring-red-500/20;
  }

  /* ===== SELECT ===== */
  .form-select {
    @apply form-input pr-10
           bg-no-repeat bg-right
           appearance-none;
    background-image: url("data:image/svg+xml,%3csvg xmlns='http://www.w3.org/2000/svg' fill='none' viewBox='0 0 20 20'%3e%3cpath stroke='%236b7280' stroke-linecap='round' stroke-linejoin='round' stroke-width='1.5' d='M6 8l4 4 4-4'/%3e%3c/svg%3e");
    background-position: right 0.75rem center;
    background-size: 1.25rem;
  }

  /* ===== TEXTAREA ===== */
  .form-textarea {
    @apply form-input resize-y min-h-[100px];
  }

  /* ===== CHECKBOX & RADIO ===== */
  .form-checkbox {
    @apply w-4 h-4 rounded
           border-gray-300 text-blue-500
           focus:ring-2 focus:ring-blue-500/20 focus:ring-offset-0
           transition-colors;
  }

  .form-radio {
    @apply w-4 h-4 rounded-full
           border-gray-300 text-blue-500
           focus:ring-2 focus:ring-blue-500/20 focus:ring-offset-0
           transition-colors;
  }

  /* ===== HELPER TEXT ===== */
  .form-hint {
    @apply text-sm text-gray-500 mt-1;
  }

  .form-error {
    @apply text-sm text-red-500 mt-1;
  }
}
```

### Card System

```css
/* components/cards.css */
@layer components {
  /*
   * Card anatomy:
   *
   * ┌─────────────────────────────────────────┐
   * │ ┌─────────────────────────────────────┐ │
   * │ │           CARD HEADER               │ │  ← .card-header
   * │ │         border-bottom               │ │
   * │ └─────────────────────────────────────┘ │
   * │ ┌─────────────────────────────────────┐ │
   * │ │                                     │ │
   * │ │           CARD BODY                 │ │  ← .card-body
   * │ │                                     │ │
   * │ └─────────────────────────────────────┘ │
   * │ ┌─────────────────────────────────────┐ │
   * │ │           CARD FOOTER               │ │  ← .card-footer
   * │ │       bg-gray-50, border-top        │ │
   * │ └─────────────────────────────────────┘ │
   * └─────────────────────────────────────────┘
   *   ↑ .card (bg-white, rounded-xl, shadow)
   */

  .card {
    @apply bg-white rounded-xl shadow-md overflow-hidden;
  }

  /* Variants */
  .card-bordered {
    @apply border border-gray-200 shadow-none;
  }

  .card-elevated {
    @apply shadow-xl;
  }

  .card-interactive {
    @apply transition-all duration-200
           hover:-translate-y-1 hover:shadow-xl
           cursor-pointer;
  }

  /* Parts */
  .card-header {
    @apply px-6 py-4 border-b border-gray-200;
  }

  .card-body {
    @apply p-6;
  }

  .card-footer {
    @apply px-6 py-4 bg-gray-50 border-t border-gray-200;
  }

  /* Image on top */
  .card-image {
    @apply w-full h-48 object-cover;
  }
}
```

### Badge System

```css
/* components/badges.css */
@layer components {
  .badge {
    @apply inline-flex items-center px-2.5 py-0.5
           rounded-full text-xs font-medium;
  }

  /* Color variants */
  .badge-gray   { @apply bg-gray-100 text-gray-800; }
  .badge-blue   { @apply bg-blue-100 text-blue-800; }
  .badge-green  { @apply bg-green-100 text-green-800; }
  .badge-red    { @apply bg-red-100 text-red-800; }
  .badge-yellow { @apply bg-yellow-100 text-yellow-800; }
  .badge-purple { @apply bg-purple-100 text-purple-800; }

  /* With dot indicator */
  .badge-dot::before {
    content: '';
    @apply w-1.5 h-1.5 rounded-full mr-1.5;
  }
  .badge-dot-green::before { @apply bg-green-500; }
  .badge-dot-red::before { @apply bg-red-500; }
  .badge-dot-yellow::before { @apply bg-yellow-500; }

  /* Outline variant */
  .badge-outline {
    @apply bg-transparent border;
  }
  .badge-outline-blue { @apply border-blue-500 text-blue-600; }
  .badge-outline-red { @apply border-red-500 text-red-600; }
}
```

### Alert System

```css
/* components/alerts.css */
@layer components {
  /*
   * Alert structure:
   *
   * ┌──────────────────────────────────────────────────┐
   * │  [Icon]  Title (optional)                        │
   * │          Description message here that can       │
   * │          wrap to multiple lines if needed.       │
   * │                                        [Close X] │
   * └──────────────────────────────────────────────────┘
   */

  .alert {
    @apply flex gap-3 p-4 rounded-lg;
  }

  .alert-icon {
    @apply shrink-0 w-5 h-5;
  }

  .alert-content {
    @apply flex-1;
  }

  .alert-title {
    @apply font-medium mb-1;
  }

  .alert-description {
    @apply text-sm opacity-90;
  }

  .alert-close {
    @apply shrink-0 p-1 rounded hover:bg-black/10 transition-colors;
  }

  /* Variants */
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

## 6. Tailwind Config Customization

### 💡 Hiểu cấu trúc config

```js
// tailwind.config.js
module.exports = {
  // 1. CONTENT: Nơi Tailwind scan để tìm class names
  content: [
    './src/**/*.{html,js,ts,jsx,tsx}',
    './components/**/*.{js,ts,jsx,tsx}',
  ],

  // 2. THEME: Customize design tokens
  theme: {
    // 2a. EXTEND: Thêm vào defaults (RECOMMENDED)
    extend: {
      colors: { ... },      // Thêm màu mới
      spacing: { ... },     // Thêm spacing mới
      fontFamily: { ... },  // Thêm font mới
    },

    // 2b. OVERRIDE: Thay thế defaults (CAUTION!)
    // colors: { ... },  // Xóa hết màu mặc định!
  },

  // 3. PLUGINS: Thêm functionality
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
}
```

### Extending Colors

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        // Brand colors với đầy đủ shades
        brand: {
          50:  '#eff6ff',
          100: '#dbeafe',
          200: '#bfdbfe',
          300: '#93c5fd',
          400: '#60a5fa',
          500: '#3b82f6',  // Primary shade
          600: '#2563eb',
          700: '#1d4ed8',
          800: '#1e40af',
          900: '#1e3a8a',
          950: '#172554',
        },

        // Single semantic colors
        primary: '#3b82f6',
        secondary: '#64748b',
        accent: '#f59e0b',

        // Surface colors cho dark mode
        surface: {
          DEFAULT: '#ffffff',
          secondary: '#f9fafb',
          tertiary: '#f3f4f6',
          dark: '#1f2937',
          'dark-secondary': '#111827',
        },

        // Content (text) colors
        content: {
          DEFAULT: '#111827',
          secondary: '#6b7280',
          tertiary: '#9ca3af',
          inverse: '#ffffff',
        },
      },
    },
  },
}
```

**Sử dụng:**

```html
<div class="bg-brand-500 text-white">Brand color</div>
<div class="bg-surface text-content">Surface with content</div>
<div class="dark:bg-surface-dark dark:text-content-inverse">Dark mode</div>
```

### Extending Typography

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      fontFamily: {
        // Override default sans
        sans: ['Inter', 'system-ui', '-apple-system', 'sans-serif'],

        // Add new font families
        display: ['Poppins', 'sans-serif'],
        mono: ['Fira Code', 'JetBrains Mono', 'monospace'],
      },

      fontSize: {
        // Custom sizes: [size, { lineHeight, letterSpacing }]
        '2xs': ['0.625rem', { lineHeight: '0.875rem' }],  // 10px
        '3xl': ['1.875rem', { lineHeight: '2.25rem', letterSpacing: '-0.02em' }],
        '4xl': ['2.25rem', { lineHeight: '2.5rem', letterSpacing: '-0.02em' }],
      },
    },
  },
}
```

### Extending Spacing

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      spacing: {
        // Tailwind default: number × 4px
        // 4 = 16px, 8 = 32px, etc.

        // Custom values
        '18': '4.5rem',   // 72px
        '22': '5.5rem',   // 88px
        '88': '22rem',    // 352px
        '128': '32rem',   // 512px

        // Percentage values
        'full': '100%',
        '1/2': '50%',
        '1/3': '33.333%',
      },
    },
  },
}
```

### Custom Animations

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      animation: {
        // name: 'keyframeName duration timingFunction iterationCount'
        'fade-in': 'fadeIn 0.3s ease-out',
        'fade-in-up': 'fadeInUp 0.4s ease-out',
        'slide-in-right': 'slideInRight 0.3s ease-out',
        'scale-in': 'scaleIn 0.2s ease-out',
        'spin-slow': 'spin 3s linear infinite',
        'wiggle': 'wiggle 1s ease-in-out infinite',
        'pulse-slow': 'pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite',
      },

      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        fadeInUp: {
          '0%': {
            opacity: '0',
            transform: 'translateY(10px)'
          },
          '100%': {
            opacity: '1',
            transform: 'translateY(0)'
          },
        },
        slideInRight: {
          '0%': { transform: 'translateX(100%)' },
          '100%': { transform: 'translateX(0)' },
        },
        scaleIn: {
          '0%': {
            transform: 'scale(0.9)',
            opacity: '0'
          },
          '100%': {
            transform: 'scale(1)',
            opacity: '1'
          },
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

**Sử dụng:**

```html
<div class="animate-fade-in">Fade in</div>
<div class="animate-fade-in-up">Fade in up</div>
<div class="animate-wiggle">Wiggling element</div>
```

### Custom Shadows

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      boxShadow: {
        'soft': '0 2px 15px -3px rgba(0, 0, 0, 0.07), 0 10px 20px -2px rgba(0, 0, 0, 0.04)',
        'hard': '0 4px 6px -1px rgba(0, 0, 0, 0.2)',
        'inner-lg': 'inset 0 4px 6px -1px rgba(0, 0, 0, 0.1)',
        'glow': '0 0 15px rgba(59, 130, 246, 0.5)',
        'glow-lg': '0 0 30px rgba(59, 130, 246, 0.6)',
      },
    },
  },
}
```

---

## 7. Plugins

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

Tự động style form elements với look đẹp hơn:

```html
<!-- Without plugin: ugly browser default -->
<!-- With plugin: modern, consistent styling -->

<input type="text" class="form-input rounded-md" />
<select class="form-select rounded-md">
  <option>Option 1</option>
</select>
<input type="checkbox" class="form-checkbox rounded" />
<input type="radio" class="form-radio" />
<textarea class="form-textarea rounded-md"></textarea>
```

### @tailwindcss/typography

Style cho content từ CMS, Markdown:

```html
<!-- prose class styles all HTML elements inside -->
<article class="prose lg:prose-xl dark:prose-invert max-w-none">
  <h1>Article Title</h1>
  <p>First paragraph with <strong>bold</strong> and <em>italic</em>.</p>

  <h2>Section Heading</h2>
  <ul>
    <li>List item one</li>
    <li>List item two</li>
  </ul>

  <pre><code>Code blocks are styled too</code></pre>

  <blockquote>
    Quotes look great as well.
  </blockquote>
</article>
```

### Custom Plugin

```js
// tailwind.config.js
const plugin = require('tailwindcss/plugin');

module.exports = {
  plugins: [
    plugin(function({ addComponents, addUtilities, addVariant, theme }) {

      // Add custom components
      addComponents({
        '.card-glass': {
          backgroundColor: 'rgba(255, 255, 255, 0.8)',
          backdropFilter: 'blur(10px)',
          borderRadius: theme('borderRadius.xl'),
          border: '1px solid rgba(255, 255, 255, 0.3)',
          padding: theme('spacing.6'),
        },
      });

      // Add custom utilities
      addUtilities({
        '.text-shadow': {
          textShadow: '2px 2px 4px rgba(0, 0, 0, 0.1)',
        },
        '.text-shadow-lg': {
          textShadow: '4px 4px 8px rgba(0, 0, 0, 0.2)',
        },
        '.text-shadow-none': {
          textShadow: 'none',
        },
      });

      // Add custom variants
      addVariant('hocus', ['&:hover', '&:focus']);
      addVariant('not-first', '&:not(:first-child)');
      addVariant('not-last', '&:not(:last-child)');
    }),
  ],
}
```

**Sử dụng:**

```html
<div class="card-glass">Glassmorphism card</div>
<h1 class="text-shadow-lg">Text with shadow</h1>
<button class="hocus:bg-blue-600">Hover or Focus</button>
<li class="not-first:border-t">List item with border except first</li>
```

---

## 8. Tổ chức CSS Files

### Recommended Structure

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
│   │   ├── badges.css
│   │   └── alerts.css
│   └── utilities/
│       └── custom.css        # Custom utilities
```

### main.css

```css
/* Tailwind directives */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Base layer */
@import './base/reset.css';
@import './base/typography.css';

/* Components layer */
@import './components/buttons.css';
@import './components/forms.css';
@import './components/cards.css';
@import './components/badges.css';
@import './components/alerts.css';

/* Custom utilities */
@import './utilities/custom.css';
```

### base/reset.css

```css
@layer base {
  /* Smooth scrolling */
  html {
    @apply scroll-smooth;
  }

  /* Default body styles */
  body {
    @apply bg-gray-50 text-gray-900 antialiased;
  }

  /* Remove default button styles */
  button {
    @apply bg-transparent border-none cursor-pointer;
  }

  /* Remove default list styles */
  ul, ol {
    @apply list-none p-0 m-0;
  }

  /* Images */
  img {
    @apply max-w-full h-auto block;
  }
}
```

---

## Cheat Sheet — Day 4

```
WHEN TO USE EACH PATTERN
├── Copy/Paste     → 1-3 times, simple styles
├── @apply         → 10+ times, no framework, DRY
└── Component      → Framework (React/Vue/Angular), with logic

@LAYER PRIORITY (low → high)
├── @layer base       → html/body resets
├── @layer components → .btn, .card, .input
└── @layer utilities  → Tailwind classes (always win)

CONFIG STRUCTURE
├── content: ['./src/**/*.{html,js}']  → Where to scan
├── theme.extend.colors: {...}         → Add colors
├── theme.extend.fontFamily: {...}     → Add fonts
├── theme.extend.spacing: {...}        → Add spacing
├── theme.extend.animation: {...}      → Add animations
└── plugins: [...]                     → Add plugins

OFFICIAL PLUGINS
├── @tailwindcss/forms      → Styled form elements
├── @tailwindcss/typography → Prose content styling
└── @tailwindcss/aspect-ratio → Aspect ratio utilities

CUSTOM PLUGIN API
├── addComponents({ '.card': {...} })
├── addUtilities({ '.text-shadow': {...} })
└── addVariant('hocus', ['&:hover', '&:focus'])
```

---

## Bài tập

1. **Button System**: Tạo complete button component với:
   - Sizes: xs, sm, md, lg, xl
   - Variants: primary, secondary, outline, ghost, danger
   - States: loading, disabled
   - Icon button support

2. **Form Components**: Tạo form system với:
   - Input, Select, Textarea
   - Checkbox, Radio
   - Error states
   - Helper text

3. **Custom Config**: Mở rộng tailwind.config.js với:
   - Brand colors (50-950)
   - Custom fonts
   - Custom animations (fade-in, slide-up)

---

## Navigation

- [← Day 3: Responsive + States](./day-3-responsive-states.md)
- [Day 5: Advanced →](./day-5-advanced.md)
