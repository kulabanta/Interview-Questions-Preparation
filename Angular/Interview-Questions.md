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

# What is Change Detection in Angular
- **Change Detection** is the mechanism Angular uses to synchronize the UI (DOM) with the application state (component data).
- Whenever data in a component changes, Angular must detect the change and update the DOM.

## Role of Zone.js
Angular uses Zone.js to know when asynchronous operations complete.

Async operations include:
- setTimeout
- setInterval
- HTTP calls
- DOM events
- Promises
- Observables

Zone.js patches these APIs and notifies Angular when they complete.

## What is NgZone
NgZone is Angular’s wrapper around Zone.js.

It allows Angular to:
- Track async operations
- Trigger change detection automatically

Angular runs application code inside `NgZone`.
```
NgZone
   │
   ├── Async Task
   │       │
   │       └── Task completed
   │
   └── Angular triggers Change Detection
```

## Change Detection Flow with NgZone
Step-by-step flow
1. Angular runs inside NgZone
2. An async event occurs
3. Zone.js intercepts it
4. When the async task completes, Zone.js notifies Angular
5. Angular runs ApplicationRef.tick()
6. Angular performs change detection on the component tree

```
setTimeout triggered
      ↓
Zone.js intercepts
      ↓
callback executed
      ↓
Zone.js notifies Angular
      ↓
Angular runs change detection
      ↓
DOM updates
```

## Angular Change Detection Tree

Angular organizes components in a tree structure.

```
AppComponent
   │
   ├── HeaderComponent
   ├── DashboardComponent
   │        ├── ChartComponent
   │        └── StatsComponent
   └── FooterComponent
```

During change detection Angular:
1. Starts from root
2. Traverses top → bottom
3. Checks every component

## Change Detection Strategies
### Default Strategy
```
ChangeDetectionStrategy.Default
```
Angular checks all components every cycle.
Trigger events:
- DOM events
- HTTP responses
- timers
- promises
- observable emissions

### OnPush Strategy
```
ChangeDetectionStrategy.OnPush
```

Component updates only when:
1. Input reference changes
2. Event occurs inside component
3. Observable emits (async pipe)
4. `markForCheck()` is called

```Typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ExampleComponent {}
```

## NgZone APIs
### run()
- Execute code inside Angular zone.
```Typescript
this.ngZone.run(() => {
  this.data = 'updated';
});
```
Triggers change detection.

### runOutsideAngular()
Execute code outside Angular zone.
```Typescript
this.ngZone.runOutsideAngular(() => {
  setInterval(() => {
    console.log('running without change detection');
  }, 1000);
});
```
Angular does not trigger change detection.

Useful for:
1. animations
2. heavy loops
3. scroll events

## Problems with Zone-based Change Detection
1. Every async task triggers change detection.
2. Angular may check components even if no state changed.

## Angular 21 Changes (Zone-less + Signals)
Angular 16 introduced Signals, and Angular 21 moves further toward zone-less change detection.

Angular is shifting from:
```
Zone.js + global change detection
```
to
```
Signal-based reactive updates
```

Instead of Zone.js:
```
State Change
     ↓
Signal updates
     ↓
Angular knows exact dependencies
     ↓
Only affected components update
```
- Starting from Angular 16 and further in Angular 21, Angular introduced Signals and zone-less change detection. Instead of relying on Zone.js to detect async operations, Angular tracks reactive dependencies using signals. When a signal changes, Angular updates only the components that depend on that signal, leading to more predictable and performant UI updates.

## Comparison

| Feature                  | Old Angular           | Angular 21           |
| ------------------------ | --------------------- | -------------------- |
| Change detection trigger | Zone.js               | Signals / manual     |
| Scope                    | Entire component tree | Targeted updates     |
| Performance              | Moderate              | Much faster          |
| Control                  | Implicit              | Explicit             |
| Async tracking           | Automatic via Zone.js | Developer controlled |

# Angular Component Lifecycle
## What is Component Lifecycle in Angular
- The **Component Lifecycle** refers to the different stages a component goes through from creation to destruction.
- The Component Lifecycle refers to the different stages a component goes through from creation to destruction.

## The execution order of lifecycle hooks:
```
constructor()
↓
ngOnChanges()
↓
ngOnInit()
↓
ngDoCheck()
↓
ngAfterContentInit()
↓
ngAfterContentChecked()
↓
ngAfterViewInit()
↓
ngAfterViewChecked()
↓
ngOnDestroy()
```

## Lifecycle methods
### 1. constructor()
The constructor is called when the component class is instantiated.

Purpose:
- Dependency Injection
- Initial variable setup

```typescript
constructor(private service: DataService) {
  console.log('Constructor executed');
}
```
Important points:
- Angular has not initialized inputs yet
- Avoid heavy logic here

### 2. ngOnChanges()
Triggered when input properties change.

Used for:
- reacting to `@Input()` updates

```typescript
@Input() userName!: string;

ngOnChanges(changes: SimpleChanges) {
  console.log(changes);
}
```

Important:
- Runs before ngOnInit
- Runs every time input changes

### 3. ngOnInit()
Runs once after the first ngOnChanges.

Used for:
- component initialization
- API calls
- setting up subscriptions

```typescript
ngOnInit() {
  this.loadUserData();
}
```

Important:
- Inputs are fully initialized
- Runs only once

### 4. ngDoCheck()
Runs during every change detection cycle.
Used for:
- custom change detection logic

```typescript
ngDoCheck() {
  console.log('Change detection triggered');
}
```

Important:
- Runs very frequently
- Avoid heavy operations

### 5. ngAfterContentInit()
Triggered after external content is projected into the component.

```html
<app-card>
  <p>Projected content</p>
</app-card>
```

```typescript
ngAfterContentInit() {
  console.log('Content initialized');
}
```
Runs **once**.

### 6. ngAfterContentChecked()
Runs after every change detection of projected content.
```typescript
ngAfterContentChecked() {
  console.log('Content checked');
}
```
Runs **multiple times**.

### 7. ngAfterViewInit()
Triggered after component view and child views are initialized.

Used when accessing:
- `@ViewChild`
- `@ViewChildren`

```typescript
@ViewChild('input') input!: ElementRef;

ngAfterViewInit() {
  console.log(this.input);
}
```
Runs **once**.
### 8. ngAfterViewChecked()
Runs after every change detection of the component’s view.
```typescript
ngAfterViewChecked() {
  console.log('View checked');
}
```
Runs **multiple times**.

### 9. ngOnDestroy()
Triggered when Angular destroys the component.

Used for cleanup tasks.

```typescript
ngOnDestroy() {
  this.subscription.unsubscribe();
  clearInterval(this.timer);
}
```

Common cleanup tasks:
- unsubscribe observables
- clear timers
- remove event listeners

## Lifecycle Hooks Summary Table
| Hook                  | When it Runs                     | Frequency |
| --------------------- | -------------------------------- | --------- |
| constructor           | Component instance creation      | Once      |
| ngOnChanges           | Input property changes           | Multiple  |
| ngOnInit              | After first input initialization | Once      |
| ngDoCheck             | Every change detection cycle     | Multiple  |
| ngAfterContentInit    | After projected content init     | Once      |
| ngAfterContentChecked | After projected content check    | Multiple  |
| ngAfterViewInit       | After view initialization        | Once      |
| ngAfterViewChecked    | After view check                 | Multiple  |
| ngOnDestroy           | Before component destruction     | Once      |

*Note :*
When using **signals**: Change detection becomes more targeted, but lifecycle hooks still execute in the same order.

# Dependency Injection in Angular (Angular 21)

Dependency Injection (DI) is a design pattern where a class receives its dependencies from an external source instead of creating them itself.

A dependency is any service or object that a class needs to perform its task.

## Problem Without Dependency Injection
Problems:
- Tight coupling
- Hard to test
- Hard to replace implementations
- Hard to maintain

## Key Parts of Angular DI
Angular DI system has three main parts.

| Part       | Description                                               |
| ---------- | --------------------------------------------------------- |
| Dependency | The service needed                                        |
| Provider   | Instructions on how to create dependency                  |
| Injector   | Object responsible for creating and delivering dependency |

## How Dependency Injection Works in Angular
```
Component requests dependency
        ↓
Angular Injector checks provider registry
        ↓
If instance exists → return it
        ↓
If not → create new instance
        ↓
Return instance to component
```
```typescript
@Injectable({
  providedIn: 'root'
})
export class UserService {}
```
Angular registers the service in the root injector.

```typescript
//Component
constructor(private userService: UserService) {}
```
Angular resolves it from the injector.

## What is a Provider
A provider tells Angular how to create a dependency.

```typescript
providers: [UserService]
```
equivalent to

```typescript
providers: [
  { provide: UserService, useClass: UserService }
]
```

| Provider Type | Purpose                |
| ------------- | ---------------------- |
| useClass      | Use a class            |
| useValue      | Use a constant value   |
| useFactory    | Use factory function   |
| useExisting   | Alias existing service |

## What is an Injector

An Injector is responsible for:
- Creating service instances
- Managing service lifecycle
- Providing dependencies

Every Angular application has multiple injectors arranged in a hierarchy.

## Hierarchical Dependency Injection
Angular uses Hierarchical DI where injectors form a tree structure.

This allows different instances of a service at different levels.

```
Root Injector
      ↓
Module Injector
      ↓
Component Injector
      ↓
Child Component Injector
```
Each level can:
- Use parent dependency
- Override parent dependency
- Create its own instance

Component tree
```
AppComponent
   │
   ├── DashboardComponent
   │        └── ChartComponent
   │
   └── ProfileComponent
```

Injector Hierarchy

```
Root Injector
      │
      └── AppComponent Injector
            │
            ├── DashboardComponent Injector
            │        └── ChartComponent Injector
            │
            └── ProfileComponent Injector
```

## Dependency Resolution Algorithm
When Angular resolves a dependency:

```
Component requests dependency
        ↓
Check component injector
        ↓
If not found → check parent injector
        ↓
Continue upward
        ↓
Reach root injector
```

If dependency is found at any level, Angular returns that instance.

## Types of Injectors in Angular

Angular mainly has two injector hierarchies.

| Injector             | Description              |
| -------------------- | ------------------------ |
| Environment Injector | Root-level services      |
| Element Injector     | Component-level services |

### root level services
Angular:
- Registers it in Root Injector
- Creates singleton instance

```typescript
@Injectable({
  providedIn: 'root'
})
export class UserService {}
```
### Component-Level Providers
A component can define its own providers.

```typescript
@Component({
  selector: 'app-dashboard',
  providers: [UserService]
})
export class DashboardComponent {}
```

## Benefits of Hierarchical DI
1. **Memory Optimization** : Services created only where needed.
2. **Scoped Services** : Different components can have different service instances.
3. **Encapsulation** : Child components can use private services.
4. **Better Testing** : Dependencies can be easily mocked or replaced.