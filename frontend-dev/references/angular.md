# Angular Reference

Covers Angular v21 (current stable as of early 2026; v22 expected around May 2026). Modern Angular means: Standalone Components, Signals, zoneless-by-default, functional APIs, Signal Forms (experimental).

## The Modern Angular Mindset

Angular has shed a lot of legacy baggage. In 2026:
- **NgModules are legacy** — standalone components are the default
- **Signals replace most RxJS** for local/component state
- **Zoneless is the default for new apps** in Angular 21+ — no provider needed
- **Functional guards/interceptors** replace class-based ones
- **Signal Forms (experimental)** are the direction of travel; Reactive Forms remain the stable choice for production
- **Vitest is the default test runner** — Karma is removed from the CLI defaults
- **Angular MCP Server** ships with the CLI for AI-assisted development

If you're working on a legacy codebase, migrate incrementally. New code should always follow modern patterns.

---

## Project Setup

```bash
ng new my-app --standalone --routing --style=scss
# Angular CLI now scaffolds standalone by default
```

`tsconfig.json` — Angular enforces strict TypeScript:
```json
{
  "compilerOptions": {
    "strict": true,
    "strictTemplates": true,
    "strictInjectionParameters": true
  }
}
```

`strictTemplates` is Angular-specific and catches type errors in HTML templates at compile time. Always enable it.

---

## Standalone Components (the default)

```typescript
import { Component, input, output } from '@angular/core'
import { CommonModule } from '@angular/common'

@Component({
  standalone: true,
  selector: 'app-user-card',
  imports: [CommonModule],   // Import dependencies directly — no NgModule
  template: `
    <div class="user-card">
      <h2>{{ user().name }}</h2>
      <p>{{ user().email }}</p>
      <button (click)="onEdit()">Edit</button>
    </div>
  `,
})
export class UserCardComponent {
  // Angular 17+ signal-based inputs
  user = input.required<User>()
  edit = output<string>()

  onEdit() {
    this.edit.emit(this.user().id)
  }
}
```

Key points:
- `standalone: true` — no NgModule required
- `input()` / `input.required()` — signal-based inputs (Angular 17+), replace `@Input()`
- `output()` — replaces `@Output() EventEmitter`
- Use `imports` array directly in the component for dependencies

---

## Signals

Signals are the recommended reactivity primitive for local and shared state. They replace most `BehaviorSubject` + async pipe patterns.

```typescript
import { signal, computed, effect } from '@angular/core'

@Component({
  standalone: true,
  template: `
    <p>Count: {{ count() }}</p>
    <p>Double: {{ double() }}</p>
    <button (click)="increment()">+</button>
  `
})
export class CounterComponent {
  // Writable signal
  count = signal(0)

  // Derived signal — recomputes automatically when count changes
  double = computed(() => this.count() * 2)

  constructor() {
    // Side effect that runs when dependencies change
    effect(() => {
      console.log('Count changed to', this.count())
    })
  }

  increment() {
    this.count.update(c => c + 1)
    // or: this.count.set(this.count() + 1)
  }
}
```

### When to use Signals vs RxJS

| Use case | Signals | RxJS |
|---|---|---|
| Local component state | ✅ | — |
| Derived/computed values | ✅ | — |
| HTTP requests | — | ✅ (HttpClient returns Observables) |
| Complex async streams, debounce, merge | — | ✅ |
| Global state (simple) | ✅ `@ngrx/signals` | — |
| Global state (complex, devtools) | — | ✅ NgRx |

Bridge between worlds: `toSignal()` converts Observables to Signals for template use.

```typescript
users = toSignal(this.userService.getAll(), { initialValue: [] })
// Now use users() in template — no async pipe needed
```

---

## Dependency Injection

Use `inject()` function (modern) instead of constructor injection:

```typescript
// Modern — inject() in field initializer
@Component({ ... })
export class UserListComponent {
  private userService = inject(UserService)
  private router = inject(Router)

  users = toSignal(this.userService.getAll(), { initialValue: [] })
}

// Legacy (still valid, avoid in new code)
constructor(private userService: UserService) {}
```

### Services

```typescript
@Injectable({ providedIn: 'root' })  // Singleton, tree-shakable
export class UserService {
  private http = inject(HttpClient)
  private apiUrl = inject(API_URL_TOKEN)  // Use injection tokens for config

  getAll(): Observable<User[]> {
    return this.http.get<User[]>(`${this.apiUrl}/users`).pipe(
      map(users => users.map(u => UserSchema.parse(u)))  // Validate at boundary
    )
  }

  update(id: string, data: Partial<User>): Observable<User> {
    return this.http.patch<User>(`${this.apiUrl}/users/${id}`, data)
  }
}
```

---

## Routing

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'users',
    loadComponent: () => import('./features/users/user-list.component')
      .then(m => m.UserListComponent),  // Lazy load by default
  },
  {
    path: 'users/:id',
    loadComponent: () => import('./features/users/user-detail.component')
      .then(m => m.UserDetailComponent),
    resolve: {
      user: userResolver,  // Functional resolver
    },
    canActivate: [authGuard],  // Functional guard
  },
]

// Functional guard (replaces class-based CanActivate)
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService)
  const router = inject(Router)
  return authService.isAuthenticated()
    ? true
    : router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } })
}

// Functional resolver
export const userResolver: ResolveFn<User> = (route) => {
  const userService = inject(UserService)
  return userService.getById(route.paramMap.get('id')!)
}
```

---

## Forms

Angular has three forms APIs available in 2026:

| API | Status | When to use |
|---|---|---|
| Reactive Forms (`FormGroup`, `FormControl`) | Stable | Default for production. Use for anything where stability guarantees matter. |
| Template-driven Forms (`ngModel`) | Stable | Simple forms only. Most teams skip this in favor of Reactive. |
| Signal Forms (`@angular/forms/signals`) | **Experimental** in v21 | New projects where you're already signal-first and can tolerate API changes. Expected to stabilize in a future release. |

### Reactive Forms (the stable default)

Still the right choice for most production apps. Typed, well-understood, no API churn. Pair with `FormBuilder` + `inject()`:

```typescript
@Component({
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
      <input formControlName="email" type="email" />
      <input formControlName="password" type="password" />
      <button [disabled]="loginForm.invalid">Login</button>
    </form>
  `
})
export class LoginComponent {
  private fb = inject(FormBuilder)

  loginForm = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]],
  })

  onSubmit() {
    if (this.loginForm.valid) {
      // this.loginForm.value is fully typed
    }
  }
}
```

### Signal Forms (experimental, new in v21)

Signal Forms redefine forms as reactive state built on signals. The form model IS a signal, validation is declared against the model shape via a schema function, and the template uses a single `[formField]` directive instead of `formGroupName`/`formControlName`. Imports come from `@angular/forms/signals` (part of `@angular/forms`, no extra install).

```typescript
import { Component, signal } from '@angular/core'
import { form, required, email, FormField } from '@angular/forms/signals'

interface LoginModel {
  email: string
  password: string
}

@Component({
  standalone: true,
  imports: [FormField],
  template: `
    <form (submit)="onSubmit($event)">
      <input type="email" [formField]="loginForm.email" />
      @if (loginForm.email().errors().length) {
        <span>{{ loginForm.email().errors()[0].message }}</span>
      }

      <input type="password" [formField]="loginForm.password" />
      <button [disabled]="!loginForm().valid()">Login</button>
    </form>
  `
})
export class LoginComponent {
  loginModel = signal<LoginModel>({ email: '', password: '' })

  loginForm = form(this.loginModel, (path) => {
    required(path.email, { message: 'Email is required' })
    email(path.email, { message: 'Enter a valid email' })
    required(path.password, { message: 'Password is required' })
  })

  onSubmit(event: Event) {
    event.preventDefault()
    if (this.loginForm().valid()) {
      const credentials = this.loginModel()
      // ...
    }
  }
}
```

Key differences from Reactive Forms:
- The model signal is the source of truth — mutating it updates the form, and form edits update it
- Validation lives in a schema function attached to the data structure, not scattered across controls
- No `FormGroup`/`FormControl`/`FormArray` — the form mirrors your model type exactly, with full inference
- Conditional logic (`disabled`, `hidden`, `readonly`) is declared reactively in the schema, no manual `enable()`/`disable()` calls
- Composable via `schema()` + `apply()` for reusable field validation rules
- Integrates naturally with zoneless change detection

**Recommendation**: Use Reactive Forms for production apps where you need stability. Reach for Signal Forms in new greenfield work where you're comfortable tracking an experimental API, and where the form model-as-state mental shift genuinely simplifies things (complex forms with conditional logic and cross-field validation benefit most).

---

## Change Detection

New Angular 21 apps are zoneless by default — **you don't need to add any provider**. The CLI scaffolds `ng new` with no `zone.js` in polyfills and no change-detection provider in `app.config.ts`:

```typescript
// main.ts — Angular 21 default (zoneless, no provider needed)
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    provideBrowserGlobalErrorListeners(), // recommended with zoneless
  ]
})
```

If you're migrating an existing app, Angular 20+ stabilized the API as `provideZonelessChangeDetection()` — add it explicitly and remove `zone.js` from polyfills in `angular.json`:

```typescript
// Explicit opt-in (Angular 20+ migration path)
import { provideZonelessChangeDetection } from '@angular/core'

bootstrapApplication(AppComponent, {
  providers: [
    provideZonelessChangeDetection(),
    provideRouter(routes),
  ]
})
```

With zoneless + Signals, Angular only re-renders components when their signals change. In zoneless apps, change detection runs on: signal updates used in templates, template-bound events (like `(click)`), the `async` pipe, and explicit `markForCheck()` / `ComponentRef.setInput()` calls. No more blanket "everything async triggers CD" behavior.

If you need to keep Zone.js (legacy third-party libraries that depend on it), add `provideZoneChangeDetection()` instead. For components not yet on Signals in a zone-based app, use `ChangeDetectionStrategy.OnPush` as the minimum:

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  // ...
})
```

---

## Deferrable Views

Lazy load parts of a template with `@defer`:

```typescript
@Component({
  template: `
    <app-header />

    @defer (on viewport) {
      <app-heavy-chart [data]="chartData()" />
    } @placeholder {
      <div class="chart-placeholder">Chart loading...</div>
    } @loading (minimum 200ms) {
      <app-skeleton />
    } @error {
      <p>Failed to load chart</p>
    }
  `
})
```

`@defer` conditions:
- `on idle` — when browser is idle
- `on viewport` — when element enters viewport
- `on interaction` — on user interaction
- `on timer(2s)` — after a delay
- `when condition` — signal/boolean condition

Use `@defer` for: heavy third-party widgets, charts, below-the-fold content, non-critical UI.

---

## State Management

| App size | Solution |
|---|---|
| Small / medium | Signals + services |
| Large, complex, needs devtools | `@ngrx/signals` store |
| Legacy / very complex | NgRx (actions/reducers/effects) |

### Signal Store (`@ngrx/signals`)

```typescript
import { signalStore, withState, withMethods, withComputed } from '@ngrx/signals'

export const UserStore = signalStore(
  { providedIn: 'root' },
  withState({ users: [] as User[], isLoading: false, error: null as string | null }),
  withComputed(({ users }) => ({
    userCount: computed(() => users().length),
  })),
  withMethods((store, userService = inject(UserService)) => ({
    async loadUsers() {
      patchState(store, { isLoading: true })
      try {
        const users = await firstValueFrom(userService.getAll())
        patchState(store, { users, isLoading: false })
      } catch (error) {
        patchState(store, { error: 'Failed to load users', isLoading: false })
      }
    }
  }))
)
```

---

## File Structure

```
src/
  app/
    app.config.ts          # Bootstrap providers
    app.routes.ts          # Root routes
    features/
      users/
        components/
          user-card.component.ts
          user-card.component.spec.ts
        pages/
          user-list.page.ts
          user-detail.page.ts
        services/
          user.service.ts
          user.service.spec.ts
        store/
          user.store.ts    # @ngrx/signals store (if needed)
        user.routes.ts     # Feature-level routes (lazy loaded)
        index.ts           # Public barrel export
    shared/
      components/          # Shared UI primitives
      directives/
      pipes/
      services/
    core/
      guards/
      interceptors/
      tokens/              # Injection tokens
```

---

## HTTP Interceptors (functional)

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService)
  const token = authService.getToken()

  if (!token) return next(req)

  const authReq = req.clone({
    headers: req.headers.set('Authorization', `Bearer ${token}`)
  })
  return next(authReq)
}

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) inject(Router).navigate(['/login'])
      return throwError(() => error)
    })
  )
}

// Register in app.config.ts
provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
```

---

## Testing

Angular 21 ships **Vitest as the default test runner**. Karma has been removed from CLI defaults; Jest and Web Test Runner are deprecated and scheduled for removal in v22. New projects get Vitest out of the box. Existing projects on Karma/Jasmine still work — migrate when convenient (Angular provides an experimental migration schematic).

```typescript
// user.service.spec.ts
describe('UserService', () => {
  let service: UserService
  let httpMock: HttpTestingController

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [UserService, provideHttpClientTesting()]
    })
    service = TestBed.inject(UserService)
    httpMock = TestBed.inject(HttpTestingController)
  })

  afterEach(() => httpMock.verify())

  it('fetches users', () => {
    const mockUsers: User[] = [{ id: '1', name: 'Alice', email: 'alice@test.com' }]
    service.getAll().subscribe(users => {
      expect(users).toEqual(mockUsers)
    })
    const req = httpMock.expectOne('/api/users')
    expect(req.request.method).toBe('GET')
    req.flush(mockUsers)
  })
})

// Component spec — use Testing Library (@testing-library/angular)
it('displays user list', async () => {
  const { getByText } = await render(UserListComponent, {
    providers: [{ provide: UserService, useValue: mockUserService }]
  })
  expect(getByText('Alice')).toBeInTheDocument()
})
```

For zoneless components, prefer `await fixture.whenStable()` over `fixture.detectChanges()` in tests — it better matches production behavior. If your test suite still relies on `fakeAsync`/`tick` patterns, note that those need updating for Vitest's native timer APIs.

Prefer `@testing-library/angular` for component tests — same philosophy as React Testing Library, tests behavior not implementation.

---

## Accessibility: `@angular/aria`

Angular 21 introduced `@angular/aria`, a new package providing headless primitives for common WAI-ARIA patterns (combobox, listbox, menu, tabs, etc.) with built-in keyboard interactions and focus management. It complements Angular Material and the CDK for teams that need ARIA-correct building blocks without Material Design visuals. Currently in developer preview.

```bash
ng add @angular/aria
```

If you're building a custom design system on top of Angular, prefer `@angular/aria` primitives over hand-rolling accessibility logic.

---

## AI Tooling: Angular MCP Server

Angular 21 ships a stable Model Context Protocol server built into the CLI, giving AI coding agents structured access to Angular-aware tools. Relevant when working with Claude Code, Cursor, or any MCP-compatible agent on an Angular codebase.

Notable tools exposed by the server:
- `get_best_practices` — returns the current Angular best-practices guide
- `search_documentation` — queries official docs scoped to your Angular version
- `find_examples` — retrieves up-to-date code examples
- `onpush_zoneless_migration` — analyzes the codebase and helps migrate components to OnPush and zoneless

This is the most reliable way to keep an AI agent from generating outdated (NgModule-based, zone-dependent) Angular code. Configure your agent to use the Angular MCP server whenever working in an Angular workspace.
