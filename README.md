# Improve readability with Typescript when working with large numbers.

```javascript
class AmountInput {
  // To create a more readable number instead of 99999999 which is hard to read, we can use _ to separate the numbers 
  // and make it more readable with less brain effort.
  private static MAX_ALLOWED = 99_999_999;
  
  amount: number = 0;
  
  formatMillion() {
    return `${this.amount / 1_000_000}M`;
  }
}
```

# Class usar safe with strict property initializacion

```javascript
class Library {
  titles: string[];
}

// Later on
const library = new Library();
const shortTitles = library.titles.filter(title => title.length < 5); // This will throw since library has not being initialized.
```

We should enable `"strictPropertyInitialization": true` in our tsconfig.json, remember that you also need strictNullChecks enabled.

This will warn on about this types of problems at compile time.

```javascript
class Library {
  titles: string[]; // By enabling the rule, now tsc will complain here.
}

// Later on
const library = new Library();
const shortTitles = library.titles.filter(title => title.length < 5); // This will throw since library has not being initialized.
```

Now we can do something like
```javascript
class Library {
  titles: string[] | undefined; 
}

// Later on
const library = new Library();
const shortTitles = library.titles.filter(title => title.length < 5); // Now TSC will complain here, since library might be undefined.
```

If you do that, it means that now you will have to safe guard your code like this:
```javascript
class Library {
  titles: string[] | undefined; 
}

// Later on
const library = new Library();
const shortTitles = library.titles?.filter(title => title.length < 5);
```
Now if library is undefined, the compiler will be happy and so your code.

If you don't want to protect the error everywhere you try to use `library` you can do:
```javascript
class Library {
  titles: string[] = ["Mocha Vanilla"]; 
}

// Later on
const library = new Library();
const shortTitles = library.titles.filter(title => title.length < 5);
```

or in the constructor to make the compiler happy
```javascript
class Library {
  titles: string[]; 
  
  constructor() {
    this.titles = ["Latte Vanilla"]
  }
}

// Later on
const library = new Library();
const shortTitles = library.titles.filter(title => title.length < 5);
```

Both are valid options and will make the error go away.

If you initialize in the constructor, you need to make sure every path is covered, let's look at the example:

```javascript
class Library {
  titles: string[]; 
  
  constructor(launchDay: boolean) {
    if (launchDay) {
      this.titles = ["Latte Vanilla"]
    } else {
      this.titles = []; // If you don't cover this path, typescript will also complain.
    }
  }
}

// Later on
const library = new Library();
const shortTitles = library.titles.filter(title => title.length < 5);
```

All of this is great, but what if this the class properties will be initialized at runtime? like fetching the titles from the server?
In that case you can use [definite assignment assertion modifiers](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-7.html#:~:text=Definite%20Assignment%20Assertions,TypeScript's%20analyses%20cannot%20detect%20so.) this will tell typescrypt that the property will be definetely initialized.

```javascript
class Library {
  titles!: string[]; // Notice this '!' modifier.
  
  constructor() {
    initialize();
  }
}
```

The importance of this is that you as developer will be aware of the potential error that you might have.

# Use "in" operator for automatic type inference
```javascript
 interface Admin {
  id: string;
  role: string;
 }
 
 interface User {
  email: string;
 }
 
 function redirect(usr: Admin | User) {
  if (/* check if is admin or not*/) {
    routeAdmin(usr.role);
  } else {
    routeUser(usr.email);
  }
 }
```

Now how can we figure it out if user is an admin or a user? You guess it, typescript will tell us!
```javascript
 function redirect(usr: Admin | User) {
  if ((<Admin>usr).role !== undefined) {
    routeAdmin(usr.role); // typescript complains here cause it might still be an user!
  } else {
    routeUser(usr.email);  // typescript complains here cause it might still be an admin!
  }
 }
```

We can also create a custom type guard and typescript will imply the type correctly in the `redirect` function
```javascript
function isAdmin(usr: Admin | User): usr is Admin {
  return (<Admin>usr).role !== undefined;
}

 function redirect(usr: Admin | User) {
  if (isAdmin(usr)) {
    routeAdmin(usr.role); 
  } else {
    routeUser(usr.email); 
  }
 }
```

The problem with this solution is that you are going to be adding to much extra code everytime you need to tell typescript to figure the type correctly.

There is a better way, the [in](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/in) operator. 
```javascript
 function redirect(usr: Admin | User) {
  if ("role" in usr) { // checks if the property role exists IN usr, since it only exists for Admin, typescript will infer correctly that it is an Admin.
    routeAdmin(usr.role); 
  } else {
    routeUser(usr.email); 
  }
 }
```

# Infer types in switch statements.

```javascript
interface Action {
  type: string;
}

export class Add implements Action {
  readonly type: string = "Add";
  constructor(public payload: string) {}
}


export class RemoveAll implements Action {
  readonly type: string = "Remove All";
}
```

```javascript
interface ITodoState {
  todos: string[];
}

function todoReducer(
  action: Action,
  state: ITodoState = { todos: [] }
): ITodoState {
  switch(action.type) {
    case "Add": {
      return {
        todos: [...state.todos, action.payload] // Since action only has a type defined, typescript will complain
      }
    }
    
    case "Remove All": {
      return {
        todos: []
      }
    }
  }
}
```

We can add an optional attribute to the Action interface
```javascript
interface Action {
  type: string;
  payload?: any; // but in here we not actually checking for the type of the payload.
}
```

Another thing we can do is to cast inside the Add action as follows:
```javascript
case "Add": {
  const payload = (<Add>action).payload; // If payload is other than a string, is going to fail.
  return {
    todos: [...state.todos, payload];
  }
}
```

Again the problem here is that you will be casting for every action that you have.
Since we know if action has a unique type attribute, we can define them as follows so that typescript can differenciate one from another,
we do this by removing the `:string` definition in the type object to tell the compiler that the type is "Add" and not any other generic string.
It is important to notice that the property should be `readonly` so that typescript knows that it won't change to another type in the future.

```javascript
interface Action {
  type: string;
}

export class Add implements Action {
  readonly type = "Add";
  constructor(public payload: string) {}
}


export class RemoveAll implements Action {
  readonly type = "Remove All";
}
```

Now we need to create a finite list of all the types typescript needs to look at when searching for a class based on a string;

```javascript
export type TodoActions = Add | RemoveAll;
```

We can change now in our reducer that instead of receiving any type of actions, it will get a specific set of Actions `TodoActions`
```javascript
interface ITodoState {
  todos: string[];
}

function todoReducer(
  action: TodoActions, // Here
  state: ITodoState = { todos: [] }
): ITodoState {
  switch(action.type) {
    case "Add": {
      return {
        todos: [...state.todos, action.payload]
      }
    }
    
    case "Remove All": {
      const t = action; // If you hoover over `t` you will see that typescript now knows the type, based on the value
      return {
        todos: []
      }
    }
  }
}
```
What we are doing here is called "Discriminated Unions" you can read more about it [here](https://basarat.gitbook.io/typescript/type-system/discriminated-unions)

If we would like to add a new action
```javascript
export class RemoveOne implements Action {
  readonly type = "Remove One"
  constructor(public payload: number) {}
}

export type TodoActions = Add | RemoveAll | RemoveOne;
```
And you try to use it in the reducer like this:
```javascript 

function todoReducer(
  action: TodoActions, // Here
  state: ITodoState = { todos: [] }
): ITodoState {
  switch(action.type) {
    case "Add": {
      return {
        todos: [...state.todos, action.payload]
      }
    }
    
    case "Remove All": {
      const t = action; 
      return {
        todos: []
      }
    }
    
    case "Remove One": {
      return {
        todos: [...state.todos, action.payload] // wrong
      }
    }
  }
}
```

if we try to use it's payload which is of type number to the todo list that it is of type string typescripe will notice and throw an error, this is great because we are all protected.

The right way to do so is by:
```javascript
case "Remove One": {
  return {
    todos: state.todos.slice().splice(action.payload, 1)
  }
}
```

Finally, something very interesting, if you have defined all the actions, but you didn't implement of all them in the reducer, we can add an extra layer of protection by defining a default case for it.
```javascript
function todoReducer(
  action: TodoActions, // Here
  state: ITodoState = { todos: [] }
): ITodoState {
  switch(action.type) {
    case "Add": {
      return {
        todos: [...state.todos, action.payload]
      }
    }
    
    case "Remove All": {
      const t = action; 
      return {
        todos: []
      }
    }
    
    case "Remove One": {
      return {
        todos: [...state.todos, action.payload] // wrong
      }
    }
    default: {
      const x: never = action;
      // If you remove any case, you will see an error over the `x`
    }
  }
}
```

# Mapped Typed Modifiers

```javascript
interace IPet {
  name: string;
  age: number;
}

type ReadonlyPet = {
  readonly [K in keyof IPet]: IPet[K]; // Apply bulk changes to all of the properties.
}
```

A piece of code like this is extremely useful if you want to guarante the inmutability state of the object.

```javascript
const pet: IPet = { name: 'bobby', age: 32 }; // mutable
const readonlyPet: ReadonlyPet = { name: 'bark', age 14 }; // inmutable

// try to apply mutation
pet.age = 6; // this is fine
readonlyPet.age = 5; // typescript will complain, age is readonly
```

If you want to make all of the pet properties optinal, you can do so with the same syntaxis.
```javascript
type ReadonlyPet = {
  readonly [K in keyof IPet]?: IPet[K]; // Apply bulk changes to all of the properties.
}
```

or make them all strings:
```javascript
type ReadonlyPet = {
  readonly [K in keyof IPet]?: string; // Apply bulk changes to all of the properties.
}
```

This feature also allows you to overwrite some properties
```javascript
interace IPet {
  name: string;
  age: number;
  favoritePark?: string; // added as optional
}

type ReadonlyPet = {
  readonly [K in keyof IPet]-?: string; // Remove any optional properties
}

// Since it now required, you need also to add the property to your objects.
const pet: IPet = { name: 'bobby', age: 32, favoritePark: 'NY' }; 
const readonlyPet: ReadonlyPet = { name: 'bark', age 14, favoritePark: 'SJ' };
```

Along with the `-` the `+` has been added as well, if you want to be explicit you can change it to:
```javascript
type ReadonlyPet = {
  +readonly [K in keyof IPet]-?: string;
}
```
This can be read as we are adding (+) a readonly flag and removing (-) optional properties.

# Types vs interfaces

## Types

Are used to alias a more complex type, like a union of other types to a reusable name

## Interfaces

More used as object oriented programming where you use the interface for a contact of the object you try to implement.

Even though types and interfaces they kind of behave the same way, for example 
```javascript
interface IAnimal {
  age: number;
  eat(): void;
  speak(): string;
}
```
is the equivalent of 
```javascript
type AnimalTypeAlias = {
  age: number;
  eat(): void;
  speak(): string;
}
```

it is even possible to assign one to the other, since typescript uses structural checks.
```javascript
 let animalInterface: IAnimal;
 let animalTypeAlias: AnimalTypeAlias;
 
 animalInterface = AnimalTypeAlias; // typescript will not complain as long as they have the same structure.
 ```
 
 With type aliases we can express a merge of interfaces by intersecting them
 ```javascript
 type Cat = IPet & IFeline; // cat is both a pet and a feline.
 ```
 
 If you will like to do the same with an interface, even though it is possible, you will have to create a totally new interface for it
 ```javascript
 interface ICat extends IPet, IFeline {}
 ```
 *Functionally, types and interfaces are kind of the same, so far the differences are more lexical.
 
 Interfaces can also extend types
 ```javascript
 type Pet = {
  pose(): void;
 }
 
 interface IFeline {
  nightvision: boolean;
 }
 
 interface ICat extends IFeline, Pet {}
```
Classes can also implement both
```javascript
class HouseCat implements IFeline, Pet {}
```

Something you cannot do is to extend or implement union types, this is because for interfaces or classes, you are creating a contract that's lock down into it, 
it cannot be one thing or the other
```javascript
type PetType = IDog | ICat;

interface IPet extends PetType {} // Error
class Pet implements PetType {} // Error
```

If you have 2 interfaces with the same name they get merged
```javascript
interface Foo {
  a: string;
}

interface Foo {
  b: string;
}

let foo: Foo;
foo.a // works
foo.b // works
```

This is not possible with types, you can't have 2 or more separate definitions of a type of name X.

*Merging interfaces is very important when you extend dependencies, this way you can add your own code into existing libraries.

# Self-referencing type aliases & Iterators

In typescript we declare types that reference themselves 
```javascript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T>;
  right: TreeNode<T>;
}

interface LinkedListNode<T> {
  value: T;
  next: LinkedListNode<T>;
}

let node: LinkedListNode<string>;
node.next.next.next.next.next.value;
```

In redux, you can time travel through your execution, if we would like to do something like this, we can do it by implementing a linked list.

```javascript
interface Action {
  type: string;
}

interface ListNode<T> {
  value: T;
  next: ListNode<T>;
  prev: ListNode<T>;
}

let action: { type: "LOGIN" }
let actioN2: { type: "LOAD" }

let actionNode1: ListNode<Action> = {
  value: action1,
  next: null,
  prev: null
}

let actionNode2: ListNode<Action> = {
  value: action2,
  next: null,
  prev: actionNode1
}

actionNode2.prev = actionNode2;
```

Since the interface allows us to do this:
```javascript
actionNode1.next.next.next.next.value; // Not safe, of course.
```

We might have some unexpected behavior in our app, in order to traverse the list we can have a null check on the current node and stop when it is null:
```javascript
// Go backwards
let currentNode = actionNode2;

do {
  console.log(currentNode.value);
  currentNode = currentNode.prev;
} while(currentNode);

// Go forward
let currentNode = actionNode1;

do {
  console.log(currentNode.value);
  currentNode = currentNode.next;
} while(currentNode);
```

As you might notice, this is messy, cause it leaves the responsability to the developer to do all the validations for prev and next nodes,
we can abstract all of this implementation by using iterators.

```javascript
class BackwardsActionIterator implements IterableIterator<Action> {
  constructor(private _currentActionNode: ListNode<Action> {

  }
  // Required method for Iterable
  [Symbol.iterator](): IterableIterator<Action> {
    return this;
  }
  
  // Require for Iterator
  next(): IteratorResult<Action> {
     const curr = this._currentActionNode;
     if (!curr?.value) {
      return { value: null, done: true }; // Required by the protocol to specify that you have reach the end of the iterations.
     }
     
     this._currentActionNode = curr.prev;
     return { value: curr.value, done: false };
  }
}

const backwardsActionList = new BackwardsActionIterator(actionNode2);

// Here we just have an abstract way to loop through our list, now the dev doesn't have to
// worry about how to traverse the linked list, to check for null values, or to go next or prev.
for(let action of backwardsActionList) {
  console.log(action.type);
}
```
The iterator and Iterable protocols are part of the JS spec, you can find more info [here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols)

# Unknown types

```javascript
const colors = ['yellow', 'blue', 'black', 'white'];

colors.indexOf('blue'); // valid
colors() // invalid
colors.a.b.c // invalid
```

In the previous example, typescript infers the type of `colors` based on the initialization values that we provide, so that typescript knows
that all color is an `array` of `string`.

But, if we specify colors as any, now all the incorrect ones are now valid.
```javascript
const colors: any = ['yellow', 'blue', 'black', 'white'];

colors.indexOf('blue'); // valid
colors() // valid
colors.a.b.c // valid
```
*Even though there are valid use cases for `any`, for most of the time it is not, it will get you in trouble.

A better aproach is to use the keyword `unknown` which will tell the developer that it is really not safe to access without proper previous checking
```javascript

interface ISky {
  message: string;
}

const colors: unknown;

if (typeof colors === 'string') {
  console.log(colors.toUpperCase()); // now it is safe, cause we are checking for the type
}

if (isLikeSky(colors)) {
  console.log(colors.message)
}

// Custom define type guard check.
function isLikeSky(type: any) type is ISky {
  return (<ISky>type).message !== undefined
}
```
*`any` is the less restricted type, vs `unknown` is the most restricted type.

# Conditional Types

By passing the generic T we can tell the compiler to analize a ternary expression
 ```javascript
 interface StringContainer {
  value: string;
  format(): string;
  split(): string[];
}

interface NumberContainer {
  value: number;
  round();
}
 
 type Item<T> = {
  id: T,
  container: T extends string ? StringContainer : NumberContainer;
 }
```

Now if we create an item, depending on the type, intellisense will figure it out the type correctly
```javascript
let item: Item<string> = {
  id: 'myid',
  container: null
}
```

Another example
```javascript
interface Book {
  id: string;
  tableOfContents: string[]l
}

interface Tv {
  id: number;
  diagonal: number;
}

interface IItemService {
  getItem<T>(id: T): T extends string ? Book : Tv;
}

let itemService: IItemServicel;
const book = itemService.getItem("10") // Since the id is a string, it will use the Book interface
const tv = itemService.getItem(10) // Since the id is a number, it will use the tv interface
```

Now, there is one more catch with this solution, and it is that you are able to do
```javascript
const tv = itemService.getItem(true); // Typescript will allow this to happen
```

We know that we only support strings or numbers for the id, that's why we need to lock in the generic
```javascript
interface IItemService {
  getItem<T extends string | number>(id: T): T extends string ? Book : Tv;
}
```

Now if you try to use a boolean parameter for the id, typescript will complain.

# Reusable flatten type

```javascript
const numbers = [2, 1]

const someObj = {
  id: '23',
  name: 'Doe'
}

const someBoolean = true;
```

If we like to find what the type of the value in the array is we can create a type:
```javascript
type FlattenArray<T extends number>: T[number]

type NumberArrayFlattened = FlattenArray<typeof numbers>;
```
We use number because the indexes of all arrays are numbers, so it will return the matching type for the value.

For the object we will do something like
```javascript
type FlattenObject<T extends object>: T[keyof T];

type ObjectFlattened = FlattenObject<typeof someObj>;
```
Since it is an object, keyof will return all the keys of T --> "id" | "name"
And hence someObj has id as number, and name as a string, the FlattenObject will say that the returned types are `string | number`

Problem with this is that we are again creating different types for each case, instead we can use conditional types to create a reusable interface
```javascript
types Flatten<T>: T extends any[] ? T[number] :
  T extends object ? T[keyof T] : 
  T;
```

Since we have built a generic and reusable interface we can now use it
```javascript
type NumberArrayFlattened = Flatten<typeof numbers>;
type ObjectFlattene d = Flatten<typeof someObj>;
type BooleanFlattened = Flatten<typeof someBoolean>;
```
# Infer

Will allow typescript to figure the type based on the content.

Typescript is very powerfull, you don't need to define the return type for all of the functions, cause typescript will know.
```javascript
function generateId(num: number) {
  return 5 + num;
}

const n: number = generateId(5); // This will work since typescript will infer the return type.
const s: string = generateId(5); // Here compilation will fail 
```

If the we don't really know the type of the id that's gonna be returned, cause it can be either a string or a number or something else,
we will like to  link both the lookup function with the generateId
```javascript
function lookup(id: string) { // won't work
  // to stuff
}
```

To fix this, we can create a return type as follows
```javascript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : any
```
When using conditional types you also get access to the infer keyword, so  
if T is a function with any number of arguments, infer it's return type and store into R, and then return R or T doesn't extend the function just return any as a placeholder.

Now we can create the Id type that matches the corresponding return value of generateId
```javascript
type Id = ReturnType<typeof generateId>
```

and now, we can link it to the lookup function 
```javascript
function lookup(id: Id) { ... }
```
*You don't need to implement the [ReturnType](https://www.typescriptlang.org/docs/handbook/utility-types.html#returntypetype), it is already in typescript 2.8+. 


For example
```javascript
type UnpackPromise<T> = T extends Promise<infer K>[] ? K : any; // extract the type of the result of the array of promises
const arr = [Promise.resolve(true)]

type ExpectedBooelan = UnpackPromise<typeof arr>;
```

# Optional chaining

## For objects
```javascript
type FormattingOptions = {
  formatting?: {
    indent?: number
  }
}

function format(value: any, options?: FormattingOptions) {
  const indent = options?.formatting?.indent; 
  return JSON.stringify(value, null, indent);
}

const user = { name: 'john' }
format(user);
format(user, {});
format(user, { formatting: {} });
format(user, { formatting: { indent: 2 } });
```

## For index-like access
```javascript
type FormattingOptions = {
  formatting?: {
    "index-line"?: number
  }
}

function format(value: any, options?: FormattingOptions) {
  const indent = options?.formatting?.["index-line"]; 
  return JSON.stringify(value, null, indent);
}

const user = { name: 'john' }
format(user);
format(user, {});
format(user, { formatting: {} });
format(user, { formatting: { "index-line": 2 } });
```

## For functions
```javascript
type FormattingOptions = {
  formatting?: {
    getIndent: () => number;
  }
}

function format(value: any, options?: FormattingOptions) {
  const indent = options?.formatting?.getIndent?.(); 
  return JSON.stringify(value, null, indent);
}

const user = { name: 'john' }
format(user);
format(user, {});
format(user, { formatting: {} });
format(user, { formatting: { getIndent: () => 2 } });
```
