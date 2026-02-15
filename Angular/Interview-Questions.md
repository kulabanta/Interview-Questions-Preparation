# 🧠 Memory Leaks in Angular
 A `memory leak happens` when objects are not released from memory even though they are no longer needed.

In Angular, this usually occurs when:<br>
- Subscriptions are not unsubscribed
- Event listeners are not removed
- Timers are not cleared
- Large objects remain referenced

## 🚨 Common Causes of Memory Leaks in Angular
### 1️⃣ Unsubscribed RxJS Observables (Most Common)
```ts
ngOnInit() {
  this.userService.getUsers().subscribe(users => {
    this.users = users;
  });
}
```
- If the component is destroyed, the subscription still lives → memory leak.

### 2️⃣ setInterval / setTimeout Not Cleared
```ts
ngOnInit() {
  setInterval(() => {
    console.log("Running...");
  }, 1000);
}
```
- This continues running even after component destruction.

### 3️⃣ Event Listeners Not Removed
```ts
ngOnInit() {
  window.addEventListener('resize', this.onResize);
}
```
- Listener stays alive unless manually removed.

### 4️⃣ Long-Lived Services Holding References
- Services provided in root live for the entire app lifecycle.
```ts
@Injectable({ providedIn: 'root' })
```
If they store component references → leak.

## ✅ How to Prevent Memory Leaks
### 1️⃣ Unsubscribe from Observables Properly
#### ✅ Method 1: Using ngOnDestroy
```ts
import { Subscription } from 'rxjs';

export class UserComponent implements OnInit, OnDestroy {
  private subscription!: Subscription;

  ngOnInit() {
    this.subscription = this.userService.getUsers()
      .subscribe(data => this.users = data);
  }

  ngOnDestroy() {
    this.subscription.unsubscribe();
  }
}
```
#### ✅ Method 2: Using takeUntil (Recommended)
```ts
import { Subject, takeUntil } from 'rxjs';

export class UserComponent implements OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.userService.getUsers()
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => this.users = data);
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```
✔ Cleaner<br>
✔ Scales for multiple subscriptions

#### ✅ Method 3: Using Async Pipe (Best Option)
```html
<div *ngFor="let user of users$ | async">
  {{ user.name }}
</div>
```
```ts
users$ = this.userService.getUsers();
```
✔ Automatically unsubscribes<br>
✔ No manual cleanup required

👉 Always prefer AsyncPipe in templates.
#### ✅ Method 4 (Angular 16+): takeUntilDestroyed() (Modern Way)
```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

constructor(private userService: UserService) {}

ngOnInit() {
  this.userService.getUsers()
    .pipe(takeUntilDestroyed())
    .subscribe(data => this.users = data);
}
```
✔ No Subject needed<br>
✔ Automatic cleanup<br>
✔ Recommended for Angular 16+

### 2️⃣ Clear Timers
```ts
intervalId!: any;

ngOnInit() {
  this.intervalId = setInterval(() => {
    console.log("Running...");
  }, 1000);
}

ngOnDestroy() {
  clearInterval(this.intervalId);
}
```
1. Method for clearing timeout : clearTimeout(timer)
2. Method for clearing interval : clearInterval(interval)
### 3️⃣ Remove Event Listeners
```ts
ngOnInit() {
  window.addEventListener('resize', this.onResize);
}

ngOnDestroy() {
  window.removeEventListener('resize', this.onResize);
}
```

### 4️⃣ Avoid Storing Component References in Services
```ts
@Injectable({ providedIn: 'root' })
export class DataService {
  componentRef: any;
}
```
✔ Instead, use Observables or Subjects.

## 🔍 How to Detect Memory Leaks
1️⃣ Chrome DevTools
1. Open DevTools → Memory tab
2. Take Heap Snapshot
3. Navigate between routes
4. Take another snapshot
5. Compare retained objects

If destroyed components still appear → memory leak.

## 🏆 Best Practices Checklist
| Practice                      | Recommended?  |
| ----------------------------- | ------------- |
| Manual unsubscribe everywhere | ⚠️ OK         |
| `takeUntil` pattern           | ✅ Good        |
| `AsyncPipe`                   | ⭐ Best        |
| `takeUntilDestroyed()`        | ⭐ Modern Best |
| Avoid global references       | ✅ Must        |
| Clear timers/listeners        | ✅ Must        |

