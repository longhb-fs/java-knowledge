# Day 5: Angular Integration

## Mục tiêu
- Hiểu cách Angular gọi Spring Boot API
- HTTP Client setup
- Service pattern
- Table với search/filter/pagination
- Download file (Export)

## 1. Angular HTTP Client Setup

### 1.1. Import HttpClientModule

```typescript
// app.config.ts (Angular 19+)
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, errorInterceptor])
    ),
  ]
};
```

> **Note:** Angular 19+ không còn dùng `NgModule` mặc định, sử dụng standalone components.

### 1.2. Environment Config

```typescript
// environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8080/api'
};

// environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: '/api'  // Relative path khi deploy cùng server
};
```

## 2. Models (DTOs)

### 2.1. Response Models

```typescript
// models/market-info.model.ts
export interface MarketInfoResponse {
  id: number;
  indexCode: string;
  value: number;
  tradingVolume: number;
  tradingValue: number;
  foreignBuyVolume: number;
  foreignBuyValue: number;
  foreignSellVolume: number;
  foreignSellValue: number;
  netBuySell: number;
  change5Sessions: number;
  change10Sessions: number;
  change30Sessions: number;
  stocksUp: number;
  stocksDown: number;
  stocksUnchanged: number;
  tradingDate: string;
}

// Page response từ Spring
export interface PageResponse<T> {
  items: T[];
  currentPage: number;
  totalPages: number;
  totalCount: number;
  pageSize: number;
  hasNext: boolean;
  hasPrevious: boolean;
}
```

### 2.2. Request Models

```typescript
// models/market-info-search.model.ts
export interface MarketInfoSearchRequest {
  keyword?: string;
  indexCode?: string;
  minValue?: number;
  maxValue?: number;
  page?: number;
  size?: number;
  sortBy?: string;
  sortDir?: 'asc' | 'desc';
}
```

## 3. HTTP Service

### 3.1. Base Service Pattern

```typescript
// services/base.service.ts
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { environment } from '../environments/environment';

export abstract class BaseService {
  protected baseUrl = environment.apiUrl;

  constructor(protected http: HttpClient) {}

  /**
   * Convert object sang HttpParams (bỏ qua null/undefined)
   */
  protected toHttpParams(obj: any): HttpParams {
    let params = new HttpParams();

    Object.keys(obj).forEach(key => {
      const value = obj[key];
      if (value !== null && value !== undefined && value !== '') {
        params = params.set(key, value.toString());
      }
    });

    return params;
  }
}
```

### 3.2. Market Info Service

```typescript
// services/market-info.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { BaseService } from './base.service';
import { MarketInfoResponse, PageResponse, MarketInfoSearchRequest } from '../models';

@Injectable({
  providedIn: 'root'
})
export class MarketInfoService extends BaseService {

  private endpoint = `${this.baseUrl}/market-info`;

  constructor(http: HttpClient) {
    super(http);
  }

  /**
   * Tra cứu thông tin thị trường
   */
  search(request: MarketInfoSearchRequest): Observable<PageResponse<MarketInfoResponse>> {
    const params = this.toHttpParams(request);
    return this.http.get<PageResponse<MarketInfoResponse>>(this.endpoint, { params });
  }

  /**
   * Lấy chi tiết theo ID
   */
  getById(id: number): Observable<MarketInfoResponse> {
    return this.http.get<MarketInfoResponse>(`${this.endpoint}/${id}`);
  }

  /**
   * Export Excel - trả về Blob
   */
  exportExcel(request: MarketInfoSearchRequest): Observable<Blob> {
    const params = this.toHttpParams(request);
    return this.http.get(`${this.endpoint}/export/excel`, {
      params,
      responseType: 'blob'
    });
  }

  /**
   * Export CSV - trả về Blob
   */
  exportCsv(request: MarketInfoSearchRequest): Observable<Blob> {
    const params = this.toHttpParams(request);
    return this.http.get(`${this.endpoint}/export/csv`, {
      params,
      responseType: 'blob'
    });
  }
}
```

## 4. Component Implementation

### 4.1. List Component

```typescript
// components/market-info-list/market-info-list.component.ts
import { Component, OnInit, signal, computed, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { MarketInfoService } from '../../services/market-info.service';
import { MarketInfoResponse, MarketInfoSearchRequest, PageResponse } from '../../models';

@Component({
  selector: 'app-market-info-list',
  standalone: true,
  imports: [CommonModule, FormsModule],
  templateUrl: './market-info-list.component.html'
})
export class MarketInfoListComponent implements OnInit {

  // Inject với inject() function (Angular 19+ pattern)
  private marketInfoService = inject(MarketInfoService);

  // State với Signals (Angular 19+)
  data = signal<PageResponse<MarketInfoResponse> | null>(null);
  loading = signal(false);
  error = signal<string | null>(null);

  // Computed
  items = computed(() => this.data()?.items ?? []);
  totalCount = computed(() => this.data()?.totalCount ?? 0);

  // Search form
  searchRequest: MarketInfoSearchRequest = {
    keyword: '',
    indexCode: '',
    page: 0,
    size: 10,
    sortBy: 'indexCode',
    sortDir: 'asc'
  };

  // Filter panel visibility
  showAdvancedFilter = false;

  ngOnInit(): void {
    // Không load data mặc định theo SRS
    // "Mặc định: Không chọn tìm kiếm đầu vào. Không có dữ liệu"
  }

  /**
   * Search - gọi khi click "Tra cứu" hoặc Enter
   */
  search(): void {
    this.loading.set(true);
    this.error.set(null);

    // Reset page về 0 khi search mới
    this.searchRequest.page = 0;

    this.marketInfoService.search(this.searchRequest).subscribe({
      next: (response) => {
        this.data.set(response);
        this.loading.set(false);
      },
      error: (err) => {
        this.error.set('Có lỗi xảy ra khi tải dữ liệu');
        this.loading.set(false);
        console.error(err);
      }
    });
  }

  /**
   * Reset filter
   */
  resetFilter(): void {
    this.searchRequest = {
      keyword: '',
      indexCode: '',
      page: 0,
      size: 10,
      sortBy: 'indexCode',
      sortDir: 'asc'
    };
    this.data.set(null);  // Xóa kết quả
  }

  /**
   * Pagination
   */
  onPageChange(page: number): void {
    this.searchRequest.page = page;
    this.loadData();
  }

  onPageSizeChange(size: number): void {
    this.searchRequest.size = size;
    this.searchRequest.page = 0;
    this.loadData();
  }

  private loadData(): void {
    this.loading.set(true);
    this.marketInfoService.search(this.searchRequest).subscribe({
      next: (response) => {
        this.data.set(response);
        this.loading.set(false);
      },
      error: () => this.loading.set(false)
    });
  }

  /**
   * Export Excel
   */
  exportExcel(): void {
    this.marketInfoService.exportExcel(this.searchRequest).subscribe({
      next: (blob) => this.downloadFile(blob, 'ThongTinThiTruong.xlsx'),
      error: (err) => console.error('Export error', err)
    });
  }

  /**
   * Export CSV
   */
  exportCsv(): void {
    this.marketInfoService.exportCsv(this.searchRequest).subscribe({
      next: (blob) => this.downloadFile(blob, 'ThongTinThiTruong.csv'),
      error: (err) => console.error('Export error', err)
    });
  }

  /**
   * Download file từ Blob
   */
  private downloadFile(blob: Blob, filename: string): void {
    const url = window.URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = filename;
    link.click();
    window.URL.revokeObjectURL(url);
  }

  /**
   * Toggle advanced filter panel
   */
  toggleAdvancedFilter(): void {
    this.showAdvancedFilter = !this.showAdvancedFilter;
  }
}
```

### 4.2. Template

```html
<!-- market-info-list.component.html -->
<div class="container-fluid">
  <!-- Title -->
  <h4>Tra cứu thông tin Thông tin thị trường</h4>

  <!-- Search Bar -->
  <div class="row mb-3">
    <div class="col-md-4">
      <input
        type="text"
        class="form-control"
        placeholder="Nhập Chỉ số, Giá trị"
        [(ngModel)]="searchRequest.keyword"
        (keyup.enter)="search()"
      />
    </div>
    <div class="col-md-auto">
      <button class="btn btn-outline-secondary" (click)="toggleAdvancedFilter()">
        <i class="fa fa-filter"></i> Lọc nâng cao
        <span class="badge bg-primary" *ngIf="showAdvancedFilter">●</span>
      </button>
    </div>
    <div class="col-md-auto">
      <div class="dropdown">
        <button class="btn btn-outline-secondary dropdown-toggle" data-bs-toggle="dropdown">
          <i class="fa fa-download"></i> Xuất file
        </button>
        <ul class="dropdown-menu">
          <li><a class="dropdown-item" (click)="exportExcel()">Excel</a></li>
          <li><a class="dropdown-item" (click)="exportCsv()">CSV</a></li>
        </ul>
      </div>
    </div>
  </div>

  <!-- Advanced Filter Panel -->
  <div class="card mb-3" *ngIf="showAdvancedFilter">
    <div class="card-body">
      <div class="row">
        <div class="col-md-3">
          <label>Chỉ số</label>
          <select class="form-select" [(ngModel)]="searchRequest.indexCode">
            <option value="">Chọn</option>
            <option value="HNX30">HNX30</option>
            <option value="VN30">VN30</option>
          </select>
        </div>
        <div class="col-md-3">
          <label>Giá trị từ</label>
          <input type="number" class="form-control" [(ngModel)]="searchRequest.minValue" />
        </div>
        <div class="col-md-3">
          <label>Giá trị đến</label>
          <input type="number" class="form-control" [(ngModel)]="searchRequest.maxValue" />
        </div>
        <div class="col-md-3 d-flex align-items-end">
          <button class="btn btn-secondary me-2" (click)="resetFilter()">Đặt lại</button>
          <button class="btn btn-primary" (click)="search()">Tra cứu</button>
        </div>
      </div>
    </div>
  </div>

  <!-- Loading -->
  <div *ngIf="loading()" class="text-center py-5">
    <div class="spinner-border text-primary"></div>
  </div>

  <!-- No Data Message -->
  <div *ngIf="!loading() && items().length === 0" class="text-center py-5 text-muted">
    <img src="assets/no-data.svg" alt="No data" width="100" />
    <p>Vui lòng nhập điều kiện tìm kiếm để hiển thị dữ liệu!</p>
  </div>

  <!-- Data Table -->
  <div *ngIf="items().length > 0" class="table-responsive">
    <table class="table table-striped table-bordered">
      <thead class="table-dark">
        <tr>
          <th>STT</th>
          <th>Chỉ số</th>
          <th>Giá trị</th>
          <th>KL giao dịch</th>
          <th>GT giao dịch</th>
          <th>KL NĐTNN mua</th>
          <th>GT NĐTNN mua</th>
          <th>KL NĐTNN bán</th>
          <th>GT NĐTNN bán</th>
          <th>Mua/bán ròng</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let item of items(); let i = index">
          <td>{{ searchRequest.page! * searchRequest.size! + i + 1 }}</td>
          <td>
            <a href="javascript:void(0)" (click)="viewDetail(item.id)">
              {{ item.indexCode }}
            </a>
          </td>
          <td class="text-end">{{ item.value | number:'1.3-3' }}</td>
          <td class="text-end">{{ item.tradingVolume | number }}</td>
          <td class="text-end">{{ item.tradingValue | number:'1.3-3' }}</td>
          <td class="text-end">{{ item.foreignBuyVolume | number }}</td>
          <td class="text-end">{{ item.foreignBuyValue | number:'1.3-3' }}</td>
          <td class="text-end">{{ item.foreignSellVolume | number }}</td>
          <td class="text-end">{{ item.foreignSellValue | number:'1.3-3' }}</td>
          <td class="text-end">{{ item.netBuySell | number:'1.3-3' }}</td>
        </tr>
      </tbody>
    </table>
  </div>

  <!-- Pagination -->
  <div *ngIf="data()" class="d-flex justify-content-between align-items-center">
    <div>
      Tất cả danh sách: {{ totalCount() }}
    </div>
    <div class="d-flex align-items-center">
      <span class="me-2">Hiển thị</span>
      <select class="form-select form-select-sm" style="width: auto"
              [(ngModel)]="searchRequest.size"
              (change)="onPageSizeChange(searchRequest.size!)">
        <option [value]="10">10 bản ghi</option>
        <option [value]="20">20 bản ghi</option>
        <option [value]="50">50 bản ghi</option>
      </select>

      <nav class="ms-3">
        <ul class="pagination pagination-sm mb-0">
          <li class="page-item" [class.disabled]="!data()?.hasPrevious">
            <a class="page-link" (click)="onPageChange(0)">««</a>
          </li>
          <li class="page-item" [class.disabled]="!data()?.hasPrevious">
            <a class="page-link" (click)="onPageChange(searchRequest.page! - 1)">«</a>
          </li>
          <li class="page-item active">
            <span class="page-link">{{ (searchRequest.page ?? 0) + 1 }}</span>
          </li>
          <li class="page-item" [class.disabled]="!data()?.hasNext">
            <a class="page-link" (click)="onPageChange(searchRequest.page! + 1)">»</a>
          </li>
          <li class="page-item" [class.disabled]="!data()?.hasNext">
            <a class="page-link" (click)="onPageChange(data()!.totalPages - 1)">»»</a>
          </li>
        </ul>
      </nav>
    </div>
  </div>
</div>
```

## 5. HTTP Interceptors

### 5.1. Auth Interceptor

```typescript
// interceptors/auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('access_token');

  if (token) {
    req = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
  }

  return next(req);
};
```

### 5.2. Error Interceptor

```typescript
// interceptors/error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';
import { inject } from '@angular/core';
import { Router } from '@angular/router';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        // Unauthorized - redirect to login
        localStorage.removeItem('access_token');
        router.navigate(['/login']);
      } else if (error.status === 403) {
        // Forbidden
        console.error('Access denied');
      } else if (error.status === 500) {
        // Server error
        console.error('Server error');
      }

      return throwError(() => error);
    })
  );
};
```

## 6. Detail Modal Component

### 6.1. Component

```typescript
// components/market-info-detail/market-info-detail.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CommonModule } from '@angular/common';
import { MarketInfoResponse } from '../../models';

@Component({
  selector: 'app-market-info-detail',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="modal" [class.show]="isVisible" [style.display]="isVisible ? 'block' : 'none'">
      <div class="modal-dialog modal-lg">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">THÔNG TIN CHI TIẾT THÔNG TIN THỊ TRƯỜNG</h5>
            <button type="button" class="btn-close" (click)="close()"></button>
          </div>
          <div class="modal-body" *ngIf="data">
            <div class="row">
              <div class="col-md-4 mb-3">
                <label class="text-muted">Chỉ số</label>
                <div class="fw-bold">{{ data.indexCode }}</div>
              </div>
              <div class="col-md-4 mb-3">
                <label class="text-muted">Giá trị</label>
                <div class="fw-bold">{{ data.value | number:'1.3-3' }}</div>
              </div>
              <div class="col-md-4 mb-3">
                <label class="text-muted">Khối lượng giao dịch</label>
                <div class="fw-bold">{{ data.tradingVolume | number }}</div>
              </div>
              <!-- More fields... -->
            </div>
          </div>
          <div class="modal-footer">
            <button class="btn btn-secondary" (click)="close()">
              <i class="fa fa-times"></i> Đóng
            </button>
          </div>
        </div>
      </div>
    </div>
    <div class="modal-backdrop fade show" *ngIf="isVisible" (click)="close()"></div>
  `
})
export class MarketInfoDetailComponent {
  @Input() isVisible = false;
  @Input() data: MarketInfoResponse | null = null;
  @Output() closeModal = new EventEmitter<void>();

  close(): void {
    this.closeModal.emit();
  }
}
```

## 7. Tổng kết Day 5

### Angular - Spring Integration Summary

| Angular | Spring Boot | Mô tả |
|---------|-------------|-------|
| `HttpClient.get()` | `@GetMapping` | GET request |
| `HttpClient.post()` | `@PostMapping` | POST request |
| `HttpParams` | `@RequestParam` | Query parameters |
| `responseType: 'blob'` | `ResponseEntity<Resource>` | File download |
| `catchError()` | `@ExceptionHandler` | Error handling |

### Key Takeaways
1. **Service pattern** - Tách logic HTTP ra service
2. **Model/Interface** - Type-safe với TypeScript
3. **Interceptors** - Auth token, error handling
4. **Blob download** - Export file qua API
5. **Signals** - State management (Angular 19+)
6. **inject()** - Function-based DI thay constructor injection
7. **Standalone Components** - Không còn NgModule

### Cấu trúc thư mục Angular

```
src/app/
├── models/
│   ├── market-info.model.ts
│   └── index.ts
├── services/
│   ├── base.service.ts
│   ├── market-info.service.ts
│   └── index.ts
├── interceptors/
│   ├── auth.interceptor.ts
│   └── error.interceptor.ts
├── components/
│   ├── market-info-list/
│   │   ├── market-info-list.component.ts
│   │   └── market-info-list.component.html
│   └── market-info-detail/
│       └── market-info-detail.component.ts
└── app.config.ts
```

## Navigation

- [← Day 4: Export Excel CSV](./day-04-export-excel-csv.md)
- [Back to Overview](./00-overview.md)

---

**Hoàn thành khóa học 5 ngày!** 🎉

Bạn đã học:
- Spring Boot 4.0 setup & DI (với Virtual Threads)
- JPA + Specification (dynamic query) với Hibernate 7
- REST API + Pagination
- Export Excel/CSV
- Angular 19+ integration (Signals, Standalone Components)

**Tech Stack đã sử dụng:**
- Java 25 LTS
- Spring Boot 4.0+
- Angular 19+
- MySQL 8.0+
