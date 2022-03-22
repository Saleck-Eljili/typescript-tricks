# Typescript Tricks

## Summary


## Top 5 techniques in TypeScript to bring your code to the next level

> **Author**: Oleh Baranovskyi
> 
> **Url**: https://obaranovskyi.medium.com/top-5-techniques-in-typescript-to-bring-your-code-to-the-next-level-6f20be543b39
> 
> **Rewriter**: Renaud Racinet

With time and practice, we all deliver code with better quality. However, there is always room for improvement. Whenever I look at the code that I wrote half of the year ago, and I don’t know how to improve it, I think about two things. Either I didn’t grow, or it is already in good shape. If you’re like me, and the code quality is essential to you, then that makes two of us. Most likely, this article will discover at least a few new techniques you didn’t know and might use on a regular basis.

### Topics to cover:

* Better validations with `includes` and selector function
* Use callbacks to encapsulate code that changes
* Consider using predicate combinators
* Even better predicate combinators with factories
* Encapsulate algorithm in the new class

### Better validations with `includes` and selector function

**Problem**:
We have a function that compares the same object property with multiple values of the same type and returns `true` in case if at least one statement is positive.

```typescript
enum UserStatus {
  Administrator = 1,
  Author = 2,
  Contributor = 3,
  Editor = 4,
  Subscriber = 5,
}

interface User {
  firstName: string;
  lastName: string;
  status: UserStatus;
}

function isEditActionAvailable(user: User): boolean {
  return (
    user.status === UserStatus.Administrator ||
    user.status === UserStatus.Author ||
    user.status === UserStatus.Editor
  );
}
```

**Solution**:

We can use an array method `includes`. With this approach, an if statement will look cleaner than before.

```typescript
const EDIT_ROLES = [
  UserStatus.Administrator,
  UserStatus.Author,
  UserStatus.Editor,
];

function isEditActionAvailable(user: User): boolean {
  return EDIT_ROLES.includes(user.status);
}
```

But there is a catch.

First, we have hard-coded data inside the function, or to put it in another way we have an implicit input (`EDIT_ROLES`).

Second, what if we want to make another role guard function that will be checking different action roles?

We can provide a factory function that will take a selector function and data responsible for describing a user role. The function itself returns another function that takes a user and compares his status with the previously passed role list.

```typescript
function roleCheck<D, T>(selector: (data: D) => T, roles: T[]): (value: D) => boolean {
    return (value: D) => roles.includes(selector(value));
}

const isEditActionAvailable = roleCheck((user: User) => user.status, EDIT_ROLES);
```

In this way, we’ve split the data from the function, what is good from the functional standpoint, and made it reusable.

Here is an example of how easy we can add another role guard function:

```typescript
const ADD_ROLES = [
  UserStatus.Administrator,
  UserStatus.Author
];

const isAddActionAvailable = roleCheck((user: User) => user.status, ADD_ROLES);
```

But wait a minute. You probably think about the selector function. Why do we need it?

With the help of the selector function, it’s possible to select different fields.

Suppose a user has a team status role field, and we have to check whether he is a team lead or manager. It’s effortless to develop a guard function that follows mentioned requirements by using an already implemented function.

```typescript
// ...

enum TeamStatus {
    Lead = 1,
    Manager = 2,
    Developer = 3
}

interface User {
  firstName: string;
  lastName: string;
  status: UserStatus;
  teamStatus: TeamStatus;
}


function roleCheck<D, T>(selector: (data: D) => T, roles: T[]): (value: D) => boolean {
    return (value: D) => roles.includes(selector(value));
}

const MANAGER_OR_LEAD = [
    TeamStatus.Lead,
    TeamStatus.Manager
]

const isManagerOrLead = roleCheck((user: User) => user.teamStatus, MANAGER_OR_LEAD);
```

### Use callbacks to encapsulate code that changes

**Problem**:
We have multiple functions that are pretty similar, with a minor difference. So it would be great to get rid of the duplicated code.

```typescript
async function createUser(user: User): Promise<void> {
  LoadingService.startLoading();
  await userHttpClient.createUser(user);
  LoadingService.stopLoading();
  UserGrid.reloadData();
}

async function updateUser(user: User): Promise<void> {
  LoadingService.startLoading();
  await userHttpClient.updateUser(user);
  LoadingService.stopLoading();
  UserGrid.reloadData();
}
```

**Solution**:

We can extract the code that changes and pass it through the callback. In such a manner, we’ll remove duplicate code.

```typescript
async function makeUserAction(fn: Function): Promise<void> {
  LoadingService.startLoading();
  await fn();
  LoadingService.stopLoading();
  UserGrid.reloadData();
}

async function createUser2(user: User): Promise<void> {
  makeUserAction(() => userHttpClient.createUser(user));
}

async function updateUser2(user: User): Promise<void> {
  makeUserAction(() => userHttpClient.updateUser(user));
}
```

## Consider using predicate combinators

**Problem**:
Our predicate functions are checking too much and are in charge of more than should.

```typescript
enum UserRole {
  Administrator = 1,
  Editor = 2,
  Subscriber = 3,
  Writer = 4,
}

interface User {
  username: string;
  age: number;
  role: UserRole;
}

const users = [
  { username: "John", age: 25, role: UserRole.Administrator },
  { username: "Jane", age: 7, role: UserRole.Subscriber },
  { username: "Liza", age: 18, role: UserRole.Writer },
  { username: "Jim", age: 16, role: UserRole.Editor },
  { username: "Bill", age: 32, role: UserRole.Editor },
];

const greaterThen17AndWriterOrEditor = users.filter((user: User) => {
  return (
    user.age > 17 &&
    (user.role === UserRole.Writer || user.role === UserRole.Editor)
  );
});

const greaterThen5AndSubscriberOrWriter = users.filter((user: User) => {
    return user.age > 5 && user.role === UserRole.Writer;
});
```

**Solution**:

We have to start using the predicate combinators. It will increase code readability and reusability.

```typescript
type PredicateFn = (value: any, index?: number) => boolean;
type ProjectionFn = (value: any, index?: number) => any;

function or(...predicates: PredicateFn[]): PredicateFn {
  return (value) => predicates.some((predicate) => predicate(value));
}

function and(...predicates: PredicateFn[]): PredicateFn {
  return (value) => predicates.every((predicate) => predicate(value));
}

function not(...predicates: PredicateFn[]): PredicateFn {
  return (value) => predicates.every((predicate) => !predicate(value));
}
```

Let’s take a look at the combinator predicates in action:

```typescript
const isWriter = (user: User) => user.role === UserRole.Writer;
const isEditor = (user: User) => user.role === UserRole.Editor;
const isGreaterThan17 = (user: User) => user.age > 17;
const isGreaterThan5 = (user: User) => user.age > 5;

const greaterThan17AndWriterOrEditor = users.filter(
    and(isGreaterThan17, or(isWriter, isEditor))
);

const greaterThan5AndSubscriberOrWriter = users.filter(
    and(isGreaterThan5, isWriter)
);
```

### Even better predicate combinators with factories

**Problem**:
Predicate combinators create too many variables, so it’s easy to get lost between these functions. If we use the combinator predicate function only once, then it’s better to have something more generic.

```typescript
const isWriter = (user: User) => user.role === UserRole.Writer;
const isEditor = (user: User) => user.role === UserRole.Editor;
const isGreaterThan17 = (user: User) => user.age > 17;
const isGreaterThan5 = (user: User) => user.age > 5;

const greaterThan17AndWriterOrEditor = users.filter(
    and(isGreaterThan17, or(isWriter, isEditor))
);

const greaterThan5AndSubscriberOrWriter = users.filter(
    and(isGreaterThan5, isWriter)
);
```

**Solution**:
We have to start using the combinator predicate factories. Let’s add a few:

```typescript
const isRole = (role: UserRole) => 
    (user: User) => user.role === role;

const isGreaterThan = (age: number) =>
    (user: User) => user.role === age;


const greaterThan17AndWriterOrEditor = users.filter(
    and(isGreaterThan(17), or(isRole(UserRole.Writer), isRole(UserRole.Editor)))
);

const greaterThan5AndSubscriberOrWriter = users.filter(
    and(isGreaterThan(5), isRole(UserRole.Writer))
);
```

You’ve probably noticed that some function invocation is repeated with the same arguments. The best option will be to mix predicate factories with combinator predicates together. Thus we’ll have the best of both worlds.

```typescript
const isRole = (role: UserRole) => 
    (user: User) => user.role === role;

const isGreaterThan = (age: number) =>
    (user: User) => user.role === age;

const isWriter = isRole(UserRole.Writer)

const greaterThan17AndWriterOrEditor = users.filter(
    and(isGreaterThan(17), or(isWriter, isRole(UserRole.Editor)))
);

const greaterThan5AndSubscriberOrWriter = users.filter(
    and(isGreaterThan(5), isWriter)
);
```

In this way, we do fewer repeats and keep the code clean and neat.

### Encapsulate algorithm in the new class

**Problem**:

We have a class responsible for too many things. It’s bound with algorithm logic, but it shouldn’t.

```typescript
class User {
  constructor(
    public firstName: string,
    public lastName: string,
    public signUpDate: Date
  ) {}

  getFormattedUserDetails(): string {
    const formattedSignUpDate = `${this.signUpDate.getFullYear()}-${this.signUpDate.getMonth() + 1}-${this.signUpDate.getDate()}`;
    const username = `${this.firstName.charAt(0)}${this.lastName}`.toLowerCase();

    return `
        First name: ${this.firstName},
        Last name: ${this.lastName},
        Sign up date: ${formattedSignUpDate},
        Username: ${username}
    `;
  }
}

const user = new User("John", "Doe", new Date());
console.log(user.getFormattedUserDetails());
```

**Solution**:

It’s fair to say this method shouldn’t reside in the user data model. Therefore, our mission is to split the responsibility. We have to extract the algorithm and encapsulate it in the new class to do this.

```typescript
interface User {
    firstName: string,
    lastName: string,
    signUpDate: Date
}

class UserDetailsFormatter {
  constructor(private user: User) {}

  format(): string {
    const { firstName, lastName } = this.user;

    return `
        First name: ${firstName},
        Last name: ${lastName},
        Sign up date: ${this.getFormattedSignUpDate()},
        Username: ${this.getUsername()}
    `;
  }

  private getUsername(): string {
    const { firstName, lastName } = this.user;

    return `${firstName.charAt(0)}${lastName}`.toLowerCase();
  }

  private getFormattedSignUpDate(): string {
    const signUpDate = this.user.signUpDate;

    return [
      signUpDate.getFullYear(),
      signUpDate.getMonth() + 1,
      signUpDate.getDate(),
    ].join("-");
  }
}

const user = { firstName: "John", lastName: "Doe", signUpDate: new Date() };
const userFormatter = new UserDetailsFormatter(user);
console.log(userFormatter.format());
```

