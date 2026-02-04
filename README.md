# AngularTopRxjsMistakes

This project was generated with [Angular CLI](https://github.com/angular/angular-cli) version 18.0.3.

# Development mistake:
1. Nested subscription:

  searchConfig$ = this.searchConfigForm.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    map((config) => {
      const trimmedConfig = {
        ...config,
        userName: config.userName?.trim() || '',
      };
      localStorage.setItem('searchConfig', JSON.stringify(trimmedConfig));
      return trimmedConfig;
    })
  );
  
  this.searchConfig$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe((searchConfig) => {
        this.usersService.findUsers(searchConfig).subscribe((users) => {
          this.users = users;
        });
      });

Should be:
  this.searchConfig$
      .pipe(
        switchMap((searchConfig) => this.usersService.findUsers(searchConfig)),
        takeUntilDestroyed(this.destroyRef))
      .subscribe((users) => this.users = users);

2.Wrong usage of takeUntil for unsub.
  this.searchConfig$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe((searchConfig) => {
        this.usersService.findUsers(searchConfig).subscribe((users) => {
          this.users = users;
        });
      });
      or
  this.searchConfig$
      .pipe(
        takeUntilDestroyed(this.destroyRef))
        switchMap((searchConfig) => this.usersService.findUsers(searchConfig)),
      .subscribe((users) => this.users = users);
      
takeUntil handles unsubscription only for the operators located before takeUntil - so if the component will be destroy if findUsers is panding state - it should not be canceled it does not unsubsctiptions downstream...

Should be:
  this.searchConfig$
      .pipe(
        takeUntilDestroyed(this.destroyRef))
        switchMap((searchConfig) => this.usersService.findUsers(searchConfig)),
      .subscribe((users) => this.users = users);

you can configure lint rule: no-unsafe-takeuntil

3. Manual subscription in components
In component there is better way to deliver data from observable to the component view - asyncPipe
  this.searchConfig$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe((searchConfig) => {
        this.usersService.findUsers(searchConfig).subscribe((users) => {
          this.users = users;
        });
      });

Should be:
    users = toSignal(this.searchConfig$.pipe(
    switchMap((searchConfig) => this.usersService.findUsers(searchConfig))))
or in older Angular version AsyncPipe:

users$ = this.searchConfig$.pipe(
    switchMap((searchConfig) => this.usersService.findUsers(searchConfig)),
    shareReplay(1)
  )

imports: [ReactiveFormsModule, AsyncPipe], 
  <p>
    Found <b>{{ (users$ | async)?.length }}</b> Users
  </p>
  <ul>
    @for (user of (users$ | async); track user.id) {
      <li class="card">{{ user.name }}</li>
    }
  </ul>

4. Executing Observable logic multiple times
new subscripotion on the view (async) trigers execute of the observable - cold observable
group rx operator - shareReplay(1) - how many result should be replay to new subscribers
 instead of:
users$ = this.searchConfig$.pipe(
    switchMap((searchConfig) => this.usersService.findUsers(searchConfig))
  )

Should be
users$ = this.searchConfig$.pipe(
    switchMap((searchConfig) => this.usersService.findUsers(searchConfig)),
    shareReplay(1)
  )  

5. Improper usage of distinctUntilChanged
distinctUntilChanged work proper for pure value not for the object - valueChanges is the {} so {} === {} return always false if new value can change to distinctUntilChanged( (prev, curr) => prev.userName !== curr.userName) or distinctUntilKeyChanged('userName')

searchConfig$ = this.searchConfigForm.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    map((config) => {
      const trimmedConfig = {
        ...config,
        userName: config.userName?.trim() || '',
      };
      localStorage.setItem('searchConfig', JSON.stringify(trimmedConfig));
      return trimmedConfig;
    })
  );

Should be:
   
searchConfig$ = this.searchConfigForm.valueChanges.pipe(
    debounceTime(300),
    distinctUntilKeyChanged('userName'),
    map((config) => {
      const trimmedConfig = {
        ...config,
        userName: config.userName?.trim() || '',
      };
      return trimmedConfig;
    }),
    tap((trimmedConfig) => localStorage.setItem('searchConfig', JSON.stringify(trimmedConfig)))
  );

6. Side effects in wrong place
when function modyfing outside our scope - in our example localStorage is outside - so side effect - it should be in tap region       

searchConfig$ = this.searchConfigForm.valueChanges.pipe(
    debounceTime(300),
    distinctUntilKeyChanged('userName'),
    map((config) => {
      const trimmedConfig = {
        ...config,
        userName: config.userName?.trim() || '',
      };
      localStorage.setItem('searchConfig', JSON.stringify(trimmedConfig));
      return trimmedConfig;
    })
  );

Should be:
   
searchConfig$ = this.searchConfigForm.valueChanges.pipe(
    debounceTime(300),
    distinctUntilKeyChanged('userName'),
    map((config) => {
      const trimmedConfig = {
        ...config,
        userName: config.userName?.trim() || '',
      };
      return trimmedConfig;
    }),
    tap((trimmedConfig) => localStorage.setItem('searchConfig', JSON.stringify(trimmedConfig)))
  );
## Development server

Run `ng serve` for a dev server. Navigate to `http://localhost:4200/`. The application will automatically reload if you change any of the source files.

## Code scaffolding

Run `ng generate component component-name` to generate a new component. You can also use `ng generate directive|pipe|service|class|guard|interface|enum|module`.

## Build

Run `ng build` to build the project. The build artifacts will be stored in the `dist/` directory.

## Running unit tests

Run `ng test` to execute the unit tests via [Karma](https://karma-runner.github.io).

## Running end-to-end tests

Run `ng e2e` to execute the end-to-end tests via a platform of your choice. To use this command, you need to first add a package that implements end-to-end testing capabilities.

## Further help

To get more help on the Angular CLI use `ng help` or go check out the [Angular CLI Overview and Command Reference](https://angular.dev/tools/cli) page.
