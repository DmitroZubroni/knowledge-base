# Angular — Компоненты, сервисы, DI

> **Теги:** #frontend #angular #components #di #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[Angular_Index]]

---

## 🔹 Компонент

```typescript
// user-card.component.ts
import { Component, Input, Output, EventEmitter, OnInit } from '@angular/core'
import { CommonModule } from '@angular/common'
import { UserService } from './user.service'

interface User { id: number; name: string; email: string }

@Component({
  selector: 'app-user-card',
  standalone: true,               // Standalone (Angular 15+, рекомендуется)
  imports: [CommonModule],
  template: `
    <div class="card" *ngIf="user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
      <button (click)="onDelete()">Удалить</button>
    </div>
  `,
  styles: [`
    .card { padding: 1rem; border: 1px solid #ddd; }
  `]
})
export class UserCardComponent implements OnInit {
  @Input({ required: true }) userId!: number
  @Output() deleted = new EventEmitter<number>()

  user: User | null = null

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    this.userService.getUser(this.userId).subscribe(user => this.user = user)
  }

  onDelete(): void { this.deleted.emit(this.userId) }
}
```

---

## 🔹 Сервис и Dependency Injection

```typescript
// user.service.ts
import { Injectable } from '@angular/core'
import { HttpClient } from '@angular/common/http'
import { Observable } from 'rxjs'
import { map, catchError } from 'rxjs/operators'

@Injectable({ providedIn: 'root' })  // singleton, доступен везде
export class UserService {
  private readonly apiUrl = '/api/users'

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl)
  }

  getUser(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`)
  }

  createUser(user: Partial<User>): Observable<User> {
    return this.http.post<User>(this.apiUrl, user)
  }

  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`)
  }
}
```

---

## 🔹 Template синтаксис

```html
<!-- Структурные директивы -->
<div *ngIf="user; else loading">{{ user.name }}</div>
<ng-template #loading><app-spinner /></ng-template>

<li *ngFor="let item of items; let i = index; trackBy: trackById">
  {{ i }}. {{ item.name }}
</li>

<div [ngSwitch]="status">
  <span *ngSwitchCase="'active'">Активен</span>
  <span *ngSwitchDefault>Неактивен</span>
</div>

<!-- Привязка данных -->
<p>{{ expression }}</p>                    <!-- интерполяция -->
<img [src]="imageUrl" [alt]="imageAlt">   <!-- property binding -->
<button (click)="handleClick($event)">    <!-- event binding -->
<input [(ngModel)]="value">               <!-- two-way binding -->
<div [class.active]="isActive">           <!-- class binding -->
<div [style.color]="textColor">           <!-- style binding -->

<!-- Pipes -->
{{ date | date:'dd.MM.yyyy' }}
{{ price | currency:'RUB' }}
{{ text | uppercase | slice:0:50 }}
{{ user$ | async }}    <!-- автоматическая подписка/отписка -->
```

---

## 🔹 Роутер и Guards

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', component: HomeComponent },
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [authGuard]    // Route Guard
  },
  {
    path: 'users',
    loadComponent: () => import('./users/users.component')
      .then(m => m.UsersComponent)  // lazy loading
  }
]

// auth.guard.ts
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService)
  const router = inject(Router)
  if (!auth.isLoggedIn()) {
    router.navigate(['/login'], { queryParams: { returnUrl: state.url } })
    return false
  }
  return true
}
```

---

## 🔗 Смотри также
> - [[Angular_RxJS]] — Observable, операторы RxJS
> - [[TypeScript]] — Angular полностью на TypeScript
