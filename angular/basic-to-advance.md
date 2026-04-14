
# Angular Basic to Advanced Interview Q&A

This guide is designed for Angular interview preparation from beginner to advanced level.
Each question includes:
- What interviewers expect
- A practical answer
- A code snippet
- Development-focused points you can discuss in real projects

## 1) What is Angular, and how is it different from a library like React?

**What interviewers expect:**
They want to know whether you understand Angular as a full framework, not just a UI layer.

**Answer:**
Angular is a TypeScript-based front-end framework maintained by Google. It provides routing, forms, dependency injection, HTTP communication, testing support, and reactive patterns out of the box.

React is mainly a UI library. In React, teams typically add extra libraries for routing, forms, and state management.

**Code snippet:**

```ts
@Component({
	selector: 'app-root',
	standalone: true,
	template: `<h1>{{ title }}</h1>`
})
export class AppComponent {
	title = 'Angular app';
}
```

**Development point:**
Angular is often chosen for enterprise applications because it gives teams a standard structure for large-scale development.

---

## 2) What are the main building blocks of an Angular application?

**Answer:**
The main building blocks are components, templates, directives, services, pipes, routing, and dependency injection.

**Code snippet:**

```ts
bootstrapApplication(AppComponent, {
	providers: [
		provideRouter(routes),
		provideHttpClient()
	]
});
```

**Development point:**
In modern Angular, standalone components reduce module complexity and simplify feature composition.

---

## 3) What is the difference between a component and a directive?

**Answer:**
A component has its own template and controls a section of UI. A directive changes the behavior or appearance of an existing DOM element.

**Code snippet:**

```ts
@Directive({
	selector: '[appAutoFocus]',
	standalone: true
})
export class AutoFocusDirective {
	constructor(private elementRef: ElementRef<HTMLInputElement>) {}

	ngAfterViewInit() {
		this.elementRef.nativeElement.focus();
	}
}
```

**Development point:**
Use a component for reusable UI blocks. Use a directive for behavior like auto-focus, permission-based hiding, or styling rules.

---

## 4) What is data binding in Angular?

**Answer:**
Angular supports interpolation, property binding, event binding, and two-way binding.

**Code snippet:**

```html
<h2>{{ title }}</h2>
<button [disabled]="isSaving" (click)="save()">Save</button>
<input [(ngModel)]="userName" />
```

**Development point:**
Reactive forms are usually preferred over `ngModel` for larger forms because they scale and test better.

---

## 5) What is the Angular component lifecycle?

**Answer:**
Important hooks include `ngOnChanges`, `ngOnInit`, `ngAfterViewInit`, and `ngOnDestroy`.

**Code snippet:**

```ts
export class DashboardComponent implements OnInit, OnDestroy {
	private timerId?: ReturnType<typeof setInterval>;

	ngOnInit() {
		this.timerId = setInterval(() => console.log('polling'), 5000);
	}

	ngOnDestroy() {
		if (this.timerId) {
			clearInterval(this.timerId);
		}
	}
}
```

**Development point:**
`ngOnDestroy` matters in production because subscriptions, timers, and event listeners can leak memory if they are not cleaned up.

---

## 6) What is dependency injection in Angular?

**Answer:**
Dependency Injection means Angular provides dependencies like services instead of the class creating them manually.

**Code snippet:**

```ts
@Injectable({ providedIn: 'root' })
export class UserService {
	constructor(private http: HttpClient) {}

	getUsers() {
		return this.http.get<User[]>('/api/users');
	}
}
```

**Development point:**
DI improves testability and separation of concerns. Components should stay thin while services handle business or integration logic.

---

## 7) What is the difference between constructor and `ngOnInit`?

**Answer:**
The constructor is for dependency injection and light initialization. `ngOnInit` is for business setup after Angular initializes inputs.

**Code snippet:**

```ts
export class ProfileComponent implements OnInit {
	profile?: UserProfile;

	constructor(private profileService: ProfileService) {}

	ngOnInit() {
		this.profileService.getProfile().subscribe((data) => {
			this.profile = data;
		});
	}
}
```

**Development point:**
Keep constructors lightweight. API calls and subscriptions belong in lifecycle hooks.

---

## 8) What are structural and attribute directives?

**Answer:**
Structural directives add or remove elements. Attribute directives change existing elements.

**Code snippet:**

```html
<div *ngIf="isAdmin">Admin Panel</div>
<li *ngFor="let item of items">{{ item }}</li>
<p [ngClass]="{ error: hasError }">Status message</p>
```

**Development point:**
A custom structural directive is useful for access control, feature flags, and tenant-specific UI rendering.

---

## 9) What is the difference between template-driven forms and reactive forms?

**Answer:**
Template-driven forms are simpler and defined in the template. Reactive forms are defined in TypeScript and are better for dynamic and complex forms.

**Code snippet:**

```ts
loginForm = new FormGroup({
	email: new FormControl('', { nonNullable: true }),
	password: new FormControl('', { nonNullable: true })
});
```

```html
<form [formGroup]="loginForm">
	<input formControlName="email" />
	<input type="password" formControlName="password" />
</form>
```

**Development point:**
For production business apps, reactive forms are usually the right choice because they are more maintainable and testable.

---

## 10) What is `FormControl`, `FormGroup`, and `FormArray`?

**Answer:**
`FormControl` is a single control, `FormGroup` is a group of controls, and `FormArray` is a dynamic collection of controls.

**Code snippet:**

```ts
profileForm = new FormGroup({
	name: new FormControl('', { nonNullable: true, validators: [Validators.required] }),
	email: new FormControl('', { nonNullable: true, validators: [Validators.email] }),
	skills: new FormArray<FormControl<string>>([])
});

addSkill() {
	this.profileForm.controls.skills.push(new FormControl('', { nonNullable: true }));
}
```

**Development point:**
Use `FormArray` for dynamic forms such as multiple addresses, education records, phone numbers, or order items.

---

## 11) How do you implement custom validation in Angular forms?

**Answer:**
Create a validator function that returns `ValidationErrors | null`.

**Code snippet:**

```ts
export function noSpaceValidator(control: AbstractControl): ValidationErrors | null {
	return String(control.value || '').includes(' ')
		? { noSpace: true }
		: null;
}

userName = new FormControl('', {
	nonNullable: true,
	validators: [Validators.required, noSpaceValidator]
});
```

**Development point:**
Custom validators are commonly used for business constraints like strong passwords, domain checks, or age validation.

---

## 12) What is `ControlValueAccessor`, and when do you use it?

**Answer:**
`ControlValueAccessor` allows a custom component to behave like a regular Angular form control.

**Code snippet:**

```ts
export class RatingInputComponent implements ControlValueAccessor {
	value = 0;
	onChange = (value: number) => {};
	onTouched = () => {};

	writeValue(value: number): void {
		this.value = value;
	}

	registerOnChange(fn: (value: number) => void): void {
		this.onChange = fn;
	}

	registerOnTouched(fn: () => void): void {
		this.onTouched = fn;
	}
}
```

**Development point:**
Use this when building reusable UI controls like date pickers, file uploads, auto complete wrappers, or custom selectors.

---

## 13) How would you build an auto complete feature in Angular?

**Answer:**
Use `FormControl.valueChanges` with `debounceTime`, `distinctUntilChanged`, and `switchMap`.

**Code snippet:**

```ts
searchControl = new FormControl('', { nonNullable: true });

results$ = this.searchControl.valueChanges.pipe(
	debounceTime(300),
	distinctUntilChanged(),
	filter((term) => term.trim().length >= 2),
	switchMap((term) => this.productService.search(term))
);
```

**Development point:**
`switchMap` prevents stale results by cancelling previous requests when the user types again.

---

## 14) What is an observable in Angular?

**Answer:**
An observable is a stream of values over time. Angular uses observables in HTTP, forms, route params, and event streams.

**Code snippet:**

```ts
users$ = this.http.get<User[]>('/api/users');

ngOnInit() {
	this.route.params.subscribe((params) => {
		console.log('Route id', params['id']);
	});
}
```

**Development point:**
Observables are useful because you can combine, cancel, transform, and retry asynchronous workflows.

---

## 15) What is the difference between Promise and Observable?

**Answer:**
A Promise emits one value and resolves once. An Observable can emit multiple values over time and can be cancelled.

**Code snippet:**

```ts
const userPromise = fetch('/api/user').then((res) => res.json());

const userObservable = this.http.get('/api/user');

userObservable.subscribe((user) => console.log(user));
```

**Development point:**
In Angular, observables fit better because Angular APIs and RxJS operators are built around them.

---

## 16) Explain `Subject`, `BehaviorSubject`, and `ReplaySubject`.

**Answer:**
`Subject` emits to current subscribers, `BehaviorSubject` keeps the latest value, and `ReplaySubject` replays previous values.

**Code snippet:**

```ts
const authState$ = new BehaviorSubject<boolean>(false);

authState$.next(true);

authState$.subscribe((isLoggedIn) => {
	console.log('Current auth state', isLoggedIn);
});
```

**Development point:**
`BehaviorSubject` is commonly used for current user or session state because new subscribers immediately receive the last value.

---

## 17) What are common RxJS operators used in Angular applications?

**Answer:**
Frequently used operators include transformation, filtering, flattening, combination, error handling, and utility operators.

**Code snippet:**

```ts
import {
	BehaviorSubject,
	combineLatest,
	forkJoin,
	of,
	timer
} from 'rxjs';
import {
	catchError,
	concatMap,
	debounceTime,
	distinctUntilChanged,
	exhaustMap,
	filter,
	finalize,
	map,
	mergeMap,
	retry,
	scan,
	shareReplay,
	startWith,
	switchMap,
	take,
	takeUntil,
	tap,
	withLatestFrom
} from 'rxjs/operators';

// Sources
const searchTerm$ = this.searchControl.valueChanges;
const saveClick$ = this.saveClicked$;
const destroy$ = this.destroy$;
const selectedOrg$ = new BehaviorSubject<string>('default-org');

// map: transform response to VM shape
const usersVm$ = this.http.get<User[]>('/api/users').pipe(
	map((users) => users.map((user) => ({ id: user.id, name: user.name.toUpperCase() })))
);

// filter: allow only active items
const activeUsers$ = usersVm$.pipe(
	map((users) => users.filter((user) => user.isActive))
);

// tap: side effects (log, metrics)
const monitoredUsers$ = activeUsers$.pipe(
	tap((users) => this.logger.info('Active users count', users.length))
);

// debounceTime + distinctUntilChanged: optimize input-driven API calls
const debouncedSearch$ = searchTerm$.pipe(
	filter((term) => !!term && term.trim().length >= 2),
	debounceTime(300),
	distinctUntilChanged()
);

// switchMap: cancel previous request (best for search)
const switchMappedSearch$ = debouncedSearch$.pipe(
	switchMap((term) => this.http.get<Product[]>('/api/products', { params: { q: term } }))
);

// mergeMap: run requests in parallel
const parallelSaves$ = saveClick$.pipe(
	mergeMap((payload) => this.http.post('/api/audit', payload))
);

// concatMap: preserve order (queue requests)
const queuedSaves$ = saveClick$.pipe(
	concatMap((payload) => this.http.post('/api/orders', payload))
);

// exhaustMap: ignore new triggers until current request completes
const singleSubmit$ = saveClick$.pipe(
	exhaustMap((payload) => this.http.post('/api/checkout', payload))
);

// combineLatest: combine latest values from multiple streams
const vm$ = combineLatest([
	this.userService.getUser(),
	this.settingsService.getSettings()
]).pipe(
	map(([user, settings]) => ({ user, settings }))
);

// forkJoin: wait for all observables to complete (parallel one-time load)
const startupData$ = forkJoin({
	profile: this.http.get('/api/profile'),
	permissions: this.http.get('/api/permissions'),
	features: this.http.get('/api/features')
});

// withLatestFrom: combine event stream with latest state
const saveWithOrg$ = saveClick$.pipe(
	withLatestFrom(selectedOrg$),
	mergeMap(([payload, orgId]) => this.http.post('/api/save', { ...payload, orgId }))
);

// startWith: seed initial value (useful for first render)
const state$ = this.http.get('/api/state').pipe(
	startWith({ loading: true, data: null })
);

// scan: accumulate values over time
const runningTotal$ = of(10, 20, 15).pipe(
	scan((total, value) => total + value, 0)
);

// take: take first N emissions
const firstTwoTicks$ = timer(0, 1000).pipe(
	take(2)
);

// takeUntil: unsubscribe on destroy
const safeStream$ = this.notifications$.pipe(
	takeUntil(destroy$)
);

// retry + catchError + finalize: resilient API handling
const resilientRequest$ = this.http.get('/api/report').pipe(
	retry(2),
	catchError((error) => {
		this.logger.error('Report API failed', error);
		return of({ items: [] });
	}),
	finalize(() => this.loader.hide())
);

// shareReplay: cache latest result for multiple subscribers
const cachedConfig$ = this.http.get<AppConfig>('/api/config').pipe(
	shareReplay(1)
);
```

**Development point:**
In interviews, explain why each operator was chosen. For example: `switchMap` for search cancellation, `concatMap` for ordered writes, `exhaustMap` for double-submit prevention, and `shareReplay(1)` for caching.

---

## 18) What is the difference between `switchMap`, `mergeMap`, and `concatMap`?

**Answer:**
`switchMap` cancels previous work, `mergeMap` runs in parallel, and `concatMap` preserves order by running one at a time.

**Code snippet:**

```ts
saveClicks$.pipe(
	concatMap((payload) => this.orderService.save(payload))
).subscribe();

searchTerms$.pipe(
	switchMap((term) => this.searchService.search(term))
).subscribe();
```

**Development point:**
Operator choice affects performance and correctness. Wrong choices can create duplicate calls or out-of-order updates.

---

## 19) How do you prevent memory leaks in Angular?

**Answer:**
Use `async` pipe where possible. If you subscribe manually, clean up with `takeUntil` or similar lifecycle-aware patterns.

**Code snippet:**

```ts
private destroy$ = new Subject<void>();

ngOnInit() {
	this.userService.getUsers()
		.pipe(takeUntil(this.destroy$))
		.subscribe();
}

ngOnDestroy() {
	this.destroy$.next();
	this.destroy$.complete();
}
```

**Development point:**
Leaks often appear on dashboard pages, router navigation flows, or components with repeated subscriptions.

---

## 20) How does change detection work in Angular?

**Answer:**
Angular checks bindings and updates the DOM when data changes. Change detection runs during events, HTTP responses, timers, and async operations.

**Code snippet:**

```ts
count = 0;

increment() {
	this.count += 1;
}
```

```html
<button (click)="increment()">Increment</button>
<p>{{ count }}</p>
```

**Development point:**
Understanding change detection helps when optimizing slow screens with many nested components.

---

## 21) What is `OnPush` change detection, and when should you use it?

**Answer:**
`OnPush` reduces unnecessary checks and works best when data is treated immutably.

**Code snippet:**

```ts
@Component({
	selector: 'app-user-list',
	template: `
		<li *ngFor="let user of users; trackBy: trackById">{{ user.name }}</li>
	`,
	changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListComponent {
	@Input() users: User[] = [];

	trackById(index: number, user: User) {
		return user.id;
	}
}
```

**Development point:**
Use `OnPush` on reusable presentational components, data-heavy lists, and dashboard widgets.

---

## 22) How do you optimize Angular application performance?

**Answer:**
Use lazy loading, `OnPush`, `trackBy`, debounced search, cached API responses, smaller bundles, and minimal template logic.

**Code snippet:**

```ts
products$ = this.productService.getProducts().pipe(
	shareReplay(1)
);
```

```html
<input [formControl]="searchControl" />
<app-product-card
	*ngFor="let product of products; trackBy: trackById"
	[product]="product">
</app-product-card>
```

**Development point:**
Talk about rendering optimization and network optimization together. Both matter in real apps.

---

## 23) Why is `trackBy` important in `*ngFor`?

**Answer:**
It helps Angular reuse DOM elements instead of recreating them when collections change.

**Code snippet:**

```html
<tr *ngFor="let user of users; trackBy: trackByUserId">
	<td>{{ user.id }}</td>
	<td>{{ user.name }}</td>
</tr>
```

```ts
trackByUserId(index: number, user: User) {
	return user.id;
}
```

**Development point:**
This matters a lot for grids, tables, and large lists where re-rendering is expensive.

---

## 24) What is lazy loading in Angular, and why is it useful?

**Answer:**
Lazy loading loads routes only when needed, reducing the initial bundle size.

**Code snippet:**

```ts
export const routes: Routes = [
	{
		path: 'admin',
		loadChildren: () => import('./admin/admin.routes').then((m) => m.ADMIN_ROUTES)
	}
];
```

**Development point:**
Admin, reports, analytics, and settings screens are strong candidates for lazy loading.

---

## 25) What is routing in Angular?

**Answer:**
Routing handles navigation between views without a full page reload.

**Code snippet:**

```ts
export const routes: Routes = [
	{ path: '', component: HomeComponent },
	{ path: 'users/:id', component: UserDetailComponent },
	{ path: '**', redirectTo: '' }
];
```

**Development point:**
Well-structured routes improve maintainability, security, and feature isolation.

---

## 26) What are route guards in Angular?

**Answer:**
Route guards decide whether navigation is allowed.

**Code snippet:**

```ts
export const authGuard: CanActivateFn = () => {
	const authService = inject(AuthService);
	const router = inject(Router);

	return authService.isLoggedIn() ? true : router.createUrlTree(['/login']);
};
```

**Development point:**
Use guards for authentication, unsaved forms, and route-level permission checks.

---

## 27) How do you implement authentication in Angular?

**Answer:**
The Angular side usually includes login UI, auth service, token or session state handling, route protection, and request interception.

**Code snippet:**

```ts
login(credentials: LoginRequest) {
	return this.http.post<AuthResponse>('/api/login', credentials).pipe(
		tap((response) => {
			this.tokenStore.set(response.accessToken);
			this.authState$.next(true);
		})
	);
}
```

**Development point:**
The backend must validate credentials and protect APIs. Angular only handles the front-end workflow.

---

## 28) How do you implement authorization in Angular?

**Answer:**
Authorization decides what an authenticated user can do, often using roles or permissions.

**Code snippet:**

```ts
canEditUser(user: CurrentUser) {
	return user.permissions.includes('user.edit');
}
```

```html
<button *ngIf="canEditUser(currentUser)" (click)="edit()">Edit</button>
```

**Development point:**
Hide UI actions for usability, but always enforce authorization again on the backend.

---

## 29) What is an HTTP interceptor in Angular?

**Answer:**
An interceptor modifies or inspects requests and responses globally.

**Code snippet:**

```ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
	const token = inject(TokenStoreService).get();

	const authReq = token
		? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
		: req;

	return next(authReq);
};
```

**Development point:**
Interceptors are ideal for auth headers, request IDs, centralized error handling, and logging.

---

## 30) How do you handle token refresh in Angular?

**Answer:**
The interceptor catches `401`, refreshes the token once, then retries pending requests.

**Code snippet:**

```ts
refreshToken() {
	if (!this.refreshInFlight$) {
		this.refreshInFlight$ = this.http.post<AuthResponse>('/api/refresh', {}).pipe(
			tap((response) => this.tokenStore.set(response.accessToken)),
			shareReplay(1),
			finalize(() => {
				this.refreshInFlight$ = undefined;
			})
		);
	}

	return this.refreshInFlight$;
}
```

**Development point:**
Without a shared refresh flow, parallel `401` responses can trigger duplicate refresh calls and race conditions.

---

## 31) How do you manage session sharing in Angular applications?

**Answer:**
Keep session state in a shared service and rehydrate it on refresh. For multi-tab sync, use `storage` events or `BroadcastChannel`.

**Code snippet:**

```ts
private sessionSubject = new BehaviorSubject<Session | null>(null);
session$ = this.sessionSubject.asObservable();

restoreSession() {
	const raw = sessionStorage.getItem('session');
	this.sessionSubject.next(raw ? JSON.parse(raw) as Session : null);
}
```

**Development point:**
Avoid storing sensitive data directly in browser storage unless the architecture explicitly allows it. Secure cookies are safer when supported.

---

## 32) How do components communicate in Angular?

**Answer:**
Common options are `@Input`, `@Output`, shared services, router params, or a state store.

**Code snippet:**

```ts
@Component({
	selector: 'app-parent',
	template: `<app-child [user]="selectedUser" (saved)="reload()"></app-child>`
})
export class ParentComponent {
	selectedUser?: User;

	reload() {
		console.log('reload parent data');
	}
}
```

**Development point:**
Use `@Input` and `@Output` for close component relationships. Use a service or store for cross-feature state.

---

## 33) What is `EventEmitter`, and when should you use it?

**Answer:**
`EventEmitter` is used with `@Output` to emit events from child to parent.

**Code snippet:**

```ts
@Output() saved = new EventEmitter<User>();

submit() {
	this.saved.emit(this.form.getRawValue() as User);
}
```

**Development point:**
Use it for component output events only. For app-wide events, use RxJS in a shared service.

---

## 34) How would you share data between unrelated components?

**Answer:**
Use a shared service with `BehaviorSubject` or another observable-based state stream.

**Code snippet:**

```ts
@Injectable({ providedIn: 'root' })
export class SharedStateService {
	private selectedUserSubject = new BehaviorSubject<User | null>(null);
	selectedUser$ = this.selectedUserSubject.asObservable();

	setSelectedUser(user: User | null) {
		this.selectedUserSubject.next(user);
	}
}
```

**Development point:**
This works well for session state, filters, selected row state, and page-level coordination.

---

## 35) What is the difference between `setValue` and `patchValue` in reactive forms?

**Answer:**
`setValue` requires the full structure. `patchValue` updates only the supplied fields.

**Code snippet:**

```ts
this.profileForm.setValue({
	name: 'Asha',
	email: 'asha@company.com',
	skills: []
});

this.profileForm.patchValue({
	email: 'new@company.com'
});
```

**Development point:**
Use `patchValue` for partial API responses. Use `setValue` when you want structure validation.

---

## 36) How do you handle API errors in Angular applications?

**Answer:**
Handle errors at component, service, and interceptor levels depending on the concern.

**Code snippet:**

```ts
loadUsers() {
	this.userService.getUsers().pipe(
		catchError((error) => {
			this.errorMessage = 'Unable to load users';
			return of([]);
		})
	).subscribe((users) => {
		this.users = users;
	});
}
```

**Development point:**
Show safe user-facing messages, and keep raw backend details out of the UI.

---

## 37) What is the purpose of pipes in Angular?

**Answer:**
Pipes transform values for display in templates.

**Code snippet:**

```html
<p>{{ today | date:'dd/MM/yyyy' }}</p>
<p>{{ price | currency:'USD' }}</p>
<p>{{ user.name | uppercase }}</p>
```

**Development point:**
Pipes should stay focused on presentation. Avoid embedding heavy business rules inside them.

---

## 38) What is a pure pipe vs an impure pipe?

**Answer:**
Pure pipes run when input references change. Impure pipes can run on every change detection cycle.

**Code snippet:**

```ts
@Pipe({
	name: 'activeUsers',
	standalone: true,
	pure: true
})
export class ActiveUsersPipe implements PipeTransform {
	transform(users: User[]) {
		return users.filter((user) => user.isActive);
	}
}
```

**Development point:**
Prefer pure pipes for performance. Use impure pipes only when you fully understand the cost.

---

## 39) How do you organize an Angular application for scalability?

**Answer:**
Organize by feature, with shared and core layers for reusable pieces and app-wide services.

**Code snippet:**

```text
src/app/
	core/
	shared/
	features/
		users/
		orders/
		reports/
```

**Development point:**
Feature-based structure scales better than grouping only by file type because related code stays together.

---

## 40) When would you use NgRx or another state management library?

**Answer:**
Use it when state is complex, shared widely, or needs predictable transitions and debugging.

**Code snippet:**

```ts
export const loadUsers = createAction('[Users] Load Users');

export const usersReducer = createReducer(
	initialState,
	on(loadUsersSuccess, (state, { users }) => ({ ...state, users }))
);
```

**Development point:**
Do not add NgRx by default. For smaller apps, services with RxJS are often enough.

---

## 41) How do you secure Angular applications beyond login?

**Answer:**
Use route guards, permission checks, secure transport, sanitized rendering, secure token handling, and strict backend enforcement.

**Code snippet:**

```ts
if (!currentUser.permissions.includes('invoice.approve')) {
	this.router.navigate(['/forbidden']);
}
```

**Development point:**
Angular improves user experience, but backend security enforcement is the real protection layer.

---

## 42) What are common Angular interview scenarios based on real development work?

**Answer:**
Typical scenarios include dynamic forms, auto complete, route protection, session reuse, search optimization, and token refresh handling.

**Code snippet:**

```ts
orderSummary$ = combineLatest([
	this.cartService.items$,
	this.discountService.discount$
]).pipe(
	map(([items, discount]) => calculateSummary(items, discount))
);
```

**Development point:**
Interviewers usually care more about practical tradeoffs than memorized theory. Explain why you chose a pattern and what issue it solved.

---

## 43) What are the most common mistakes Angular developers make?

**Answer:**
Common mistakes include leaking subscriptions, mixing too much logic into components, misusing RxJS operators, and insecure session handling.

**Code snippet:**

```ts
// Bad: nested subscribe and no cleanup
this.authService.user$.subscribe((user) => {
	this.orderService.getOrders(user.id).subscribe();
});

// Better
this.orders$ = this.authService.user$.pipe(
	switchMap((user) => this.orderService.getOrders(user.id))
);
```

**Development point:**
A strong Angular developer understands not only features, but also the production failure modes.

---

## 44) How do you answer Angular questions in senior-level interviews?

**Answer:**
Senior answers should combine definition, real use case, risk, and best practice.

**Code snippet:**

```ts
// Example senior-level answer reference point:
// Use OnPush + trackBy + cached API response for large tables
users$ = this.userService.getUsers().pipe(shareReplay(1));
```

**Development point:**
For senior interviews, explain architecture, tradeoffs, performance impact, maintainability, and security implications, not just syntax.

