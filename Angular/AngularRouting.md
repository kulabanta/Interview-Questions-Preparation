# NavigationExtras properties
- You pass a NavigationExtras object as the second argument to `router.navigate()` or `router.navigateByUrl()` .

## Usecases
1. Using state (Transient Data):<br>
Useful for passing complex objects between components without showing them in the URL.

```typescript
// Component A
this.router.navigate(['/target'], { state: { user: { id: 1, name: 'John' } } });

```
**Retrieving state:** Access it in the target component’s constructor using router.getCurrentNavigation().

```typescript
// Component B
constructor(private router: Router) {
  const navigation = this.router.getCurrentNavigation();
  const state = navigation?.extras.state as { user: any };
  console.log(state?.user);
}

```
2. Using queryParams:<br>
Pass data that remains visible in the URL (e.g., ?id=123).
```typescript
this.router.navigate(['/search'], { queryParams: { id: 123, filter: 'active' } });

```
3. Using relativeTo:<br>

Navigates relative to the current ActivatedRoute rather than from the root.

```typescript
this.router.navigate(['../details'], { relativeTo: this.route });

```

## Key Properties of NavigationExtras
1. **replaceUrl**: If true, replaces the current state in the browser history instead of creating a new entry.
2. **skipLocationChange**: If true, navigates to the new route without changing the URL in the browser address bar.
3. **fragment**: Sets a hash fragment (e.g., #section1).
4. **queryParamsHandling**: Determines how to handle existing query parameters (options: 'merge', 'preserve', or null). 

# Common Router Events
Navigation follows a specific sequence of events:
1. **NavigationStart**: Triggered when a new navigation begins.
2. **RouteConfigLoadStart / End**: Triggered before and after the router lazy-loads a route configuration.
3. **RoutesRecognized**: Emitted once the router matches the URL to a specific route in the configuration.
4. **GuardsCheckStart / End**: Occurs when the router starts and finishes running Route Guards (e.g., canActivate).
5. **ResolveStart / End**: Emitted when the router starts and finishes pre-fetching data via Resolvers.
6. **ActivationStart / End**: Fired when the router begins and ends activating a component for a route.
7. **NavigationEnd**: The final event when navigation completes successfully and the URL is updated.
8. **NavigationCancel / Error**: Triggered if navigation is stopped by a guard or fails due to an unexpected error. 

```typescript
import { Component } from '@angular/core';
import { Router, NavigationEnd, Event } from '@angular/router';
import { filter } from 'rxjs/operators';

@Component({...})
export class AppComponent {
  constructor(private router: Router) {
    this.router.events.pipe(
      // Filter for only NavigationEnd events
      filter((event: Event): event is NavigationEnd => event instanceof NavigationEnd)
    ).subscribe((event: NavigationEnd) => {
      console.log('Navigation ended to:', event.url);
    });
  }
}

```

# What is RouteReuseStrategy
**RouteReuseStrategy** is an Angular interface that allows developers to cache, detach, and reattach component instances during navigation instead of destroying them. By implementing a custom strategy, you can preserve component states (like scroll positions or form data), boost performance by avoiding re-renders, and prevent unnecessary API re-calls. 

## Key Aspects of RouteReuseStrategy:
1. **Default Behavior**: By default, Angular destroys components when navigating away and recreates them upon return.
2. **Purpose**: It is primarily used for optimizing complex UIs, such as dashboards with multiple tabs, ensuring a smoother user experience.
3. **Five Key Methods**: Implementing the strategy requires defining five methods: shouldDetach, store, shouldAttach, retrieve, and shouldReuseRoute.
4. **Implementation**: You can create a custom class that implements RouteReuseStrategy and provide it in your application module to override default behavior. 