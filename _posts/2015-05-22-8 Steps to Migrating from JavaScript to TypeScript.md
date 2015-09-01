---
title: 8 Steps to Migrating from JavaScript to TypeScript
---

We have been moving our Browser Agent to TypeScript recently.
It’s fun to learn a new language and to see how it can benefit us.

Let me share with you how we have been doing it!

# Why TypeScript
Before moving to TypeScript, our browser agent has thousands lines of code but in just two JavaScript files.  
We feel obliged to refactor it so as to make our life easier when adding more features.

Having been experiencing the pain of developing large scale app in JavaScript, we decided to take a shot at  
its friend languages that have better support for large scale development.

After looking into languages such as TypeScript, CoffeeScript and PureScript, etc., we decided to go for  
TypeScript for these reasons:

1. Static Typing
2. Module and Classes
3. Superset of JavaScript, easier to learn for JavaScript developers
4. Success story from our front-end team

# Effort it Takes

## 1. Prepare Knowledge
* [TypeScript Official Web Site](http://www.typescriptlang.org/) is the best start
* [TypeScript Succinctly](https://www.stevefenton.co.uk/Content/TypeScript-Succinctly/) is a good book for free
If you are alraedy a JavaScript developer, it feels smooth to pick the knowledge up.

## 2. Rename File
We rename all the js files to ts files and as TypeScript is just a superset of JavaScript, you can just start
compiling your new ts files with the TypeScript compiler.

## 3. Fix Compiling Errors
There were quite a few compiling errors due to the static type checking by the compiler.  
For instance, the compiler will complains about js code below:

{% highlight ts %}
// Example One
// typescript compiler declares the types of all the common JavaScript object
// in a lib.d.ts file
// access to a browser specific property that is not in lib.d.ts gives
// “error TS2339: Property 'chrome' does not exist on type 'Window’.”
var xdr = window.XDomainRequest;

// Example Two
function foo(a: number, b: number) {
    return;
}

// Optinal function args need to be marked explicitly in typescript, or it gives
// “error TS2346: Supplied parameters do not match any signature of call target.”
foo(1);

// Example Three
var myObj = {};

// creating new object property by dot syntax gives
// “error TS2339: Property 'name' does not exist on type ‘{}’”
// beacause the implicit type of myObj is an empty object without ‘name’ property
myObj.name = "myObj";
{% endhighlight %}

The solutions are:

{% highlight ts %}
// Solution One
// declare the specific property on our own
interface Window {
    XDomainRequest?: any;
}

// Solution Two
// question mark the optional arg explicitly
function foo(a: number, b?: number) {
    return;
}

// Solution Three
// use bracket to creat the new property
myObj['name'] = 'myObj';
// or define an interface for the myObj
interface MyObj {
    name?: string
}

var myObj: MyObj = {};
myObj.name = 'myObj';
{% endhighlight %}

It’s kind of fun to fix these errors and you learn about the language and how the compiler can help.

## 4. Fix Test Cases
After successfully getting a JavaScript file from those ts files, we ran the tests against the new JavaScript files and fix all the failures.

One example of the test failures caused by moving to TypeScript is about the difference between these two ways of exporting a function:

{% highlight ts %}
export function foo() {}
export var foo = function() {}
{% endhighlight %}

Assuming your original JavaScript code is like:

{% highlight ts %}
var A = {
	foo: function() {},
	bar: function() {foo();}
}
{% endhighlight %}

test case is like

{% highlight ts %}
var origFoo = A.foo;
var fooCalled = false;
A.foo = function(){fooCalled = true;};
A.bar();
assertTrue(fooCalled);
A.foo = origFoo;
{% endhighlight %}

If the TypeScript rewrite the JavaScript like

{% highlight ts %}
module A {
	export function foo() {}
	export function bar() {foo();}
}
{% endhighlight %}

The test case will fail.  
Can you tell why?  
Looking at the generated JavaScript code will let you know why.

{% highlight ts %}
// generated from export function foo() {}
var A;
(function (A) {
    function foo() { }
    A.foo = foo;
    function bar() { foo(); }
    A.bar = bar;
})(A || (A = {}));
{% endhighlight %}

In the test case, when the A.foo is replaced, you are just replacing the “foo” property of A  
but not the foo function, bar function still calls the same foo function.

**export var foo = function(){}** can help.

TypeScript

{% highlight ts %}
module A {
	export var foo = function () { };
	export var bar = function () { foo(); };
}
{% endhighlight %}


generates

{% highlight ts %}
// generated from expot var foo = function() {}
var A;
(function (A) {
    A.foo = function () { };
    A.bar = function () { A.foo(); };
})(A || (A = {}));
{% endhighlight %}

Now we can replace the foo function called by A.bar.

## 5. Refactor Code

TypeScript **Modules** and **Classes** help to organize the code in a modularized and object oriented way.

Dependenies are referenced in the file header.

{% highlight ts %}
///<reference path=“moduleA.ts” />
///<reference path=“moduleB.ts” />
module ADRUM.moduleC.moduleD {
	...
}
{% endhighlight %}
One thing I like is that when compiling one ts file, ts compiler has the “--out” option to concatenate all the directly or indirectly referenced ts files, so that I needn’t to use requirejs or browserify to for the same purpose.

By TypeScript, we can define classes in a classical inheritenace way rather than the prototypal inheritance way which is more familar to Java and C++ programmers.

But you lose the flexibility JavaScript provides too.
For example, if you are seeking a way to hide a function in the class scope, save your time, [it isn’t supported](https://typescript.codeplex.com/discussions/448054).
The workaround is to define the function in the module and use it in the class.

TypeScript allows you to define modules and classes in an easy way and generates the idiomatic JavaScript for you.  
As a result, I feel like you may also have less opportunites to learn more advanced JavaScript knowledge than programming in pure JavaScript.
But just like moving from assebmly to C/C++, by and large, it’s still a good thing.

We did not bother adding all the type information in the existing code but do it when changing or adding code.

It is also worth moving the test cases to TypeScript, as the test cases could be auto updated when refactoring the code in the IDE.

## 6. Fix Minification
Don’t be surprised if the minifictaion is broken especially when you use Google Closure Compiler with advanced optimization.

**Problem One Dead Code Mistakenly Removed**  
The advanced optimization has a “dead code removal” feature that removes the code which recognized as unused by the compiler.
Some early version closure compilier like version 20121212 mistakenly recognizes some code in some TypeScript modules as unused  
and removes them.
Fortunately, it’s been fixed in the latest version compiler.

**Problem Two Export Symbols in Modules**  

To tell the compiler not to rename the symbols in your code, you need to [export the symbols](https://developers.google.com/closure/compiler/docs/api-tutorial3#export) by the quote notation. It means you need to export you API as below to allow keep the API name so that other libraries can call them even with the minified js file.

{% highlight ts %}
module A {
	export function fooAPI() { }
	A['fooAPI’] = fooAPI;
}
{% endhighlight %}

transpiled to

{% highlight ts %}
var A;
(function (A) {
    function foo() { }
    A.foo = foo;
    A['foo'] = foo;
})(A || (A = {}));
{% endhighlight %}
It’s a little bit tedious.

Another option is to use the deprecated @expose annotation.
{% highlight ts %}
module A {
	/**
    * @expose
    */
	export function fooAPI() { }
}
{% endhighlight %}

But looks it’s planned to be removed in future, and hopefully you might be able to use @export while it’s removed.(Refer to the discussion at [@expose annotation causes JSC_UNSAFE_NAMESPACE warning](https://github.com/google/closure-compiler/issues/636).

**Problem Three Export Symbols in Interfaces**  
Say, you define a BeaconJsonData interface which will be passed to other libraries, so you want to keep its key names.

{% highlight ts %}
interface BeaconJsonData {
	url: string,
	metrics?: any
}
{% endhighlight %}

@expose does not help as the interface definition transpile to nothing.
{% highlight ts %}
interface BeaconData {
	/**
    * @expose
    */
	url: string,
	/**
    * @expose
    */
	metrics?: any
}
{% endhighlight %}

You can reserve the key names by quote notation, 
var beaconData: BeaconData = {
	“url”: ‘www.example.com’
	“metrics: {...}
};

But what if you want to assign the optional key later?
var beaconData: BeaconData = {
	“url”: ‘www.example.com’
};

// ‘metrics’ will not be renamed but you lose the type checking by ts compiler
// because you can create any new properties with quote notation
beaconData[‘metrics’] = {...};
beaconData[‘metricsTypo’] = {...}; // no compiling error

// ‘metrics’ will be renamed but dot notation is protected by type checking
beaconData.metrics = {...};
beaconData.metricsTypo = {...}; // compiling error

// expose in the interface file
/** @expose */ export var metrics;


## 7. Auto Generate Google Closure Compiler Externs Files
For Closure Compiler, if your js code calls external js lib’s APIs, you need to declare these APIs in an externs file to tell the compiler not to rename the symbols of these APIs. Refer to [Do Not Use Externs Instead of Exports!](https://developers.google.com/closure/compiler/docs/api-tutorial3#no)

We used to manually create the externs files and any time use a new API, we have to manually update its externs file.

After using TypeScript, we found that TypeScript .d.ts and the externs file have the similar information.
They both contain the external API declarations(.d.ts files just have more typing information), so we can try to get rid of one of them..

The first idea came into my mind is to check if typescript compiler support minification.
As the ts compiler understand the .d.ts file, it won’t need the externs files.
Unfortunately it doesn’t support it, so we have to stay with google closure compiler.

Then we thought the right thing is to generate the externs files from the .d.ts files.
Thanks to the open source ts compiler, we use it to parse the .d.ts files and convert them to externs file (see my solution at https://goo.gl/l0o6qX).
Now, each time we add a new external API declaration in our .d.ts file, the API symbols automatically appears in the externs file when build our project.

## 8. Wrap the ts code in one function
Ts compiler generates code for modules like below:

{% highlight ts %}
// typescript
module A {
    export var a: number;
}

module A.B {
    export var b: number;
}

// converted to ==> 

// javascript
var A;
(function (A) {
    A.a;
})(A || (A = {}));
var A;
(function (A) {
    var B;
    (function (B) {
        B.b;
    })(B = A.B || (A.B = {}));
})(A || (A = {}));

{% endhighlight %}

For each module, there is a variable created and a function called.
The function creates properties in the module variable for exported symbols.

However, sometimes you want to stop execution for some condition like your libraries
has be defined or it has been disabled, you need to wrap all the js code in a function by yourself, like

{% highlight ts %}
(function(){
    if (global.ADRUM || global.ADRUM_DISABLED) {
    	return;
    } 
    
    // typescript generated javascript
}(global);
{% endhighlight %}

# TypeScript Main Benefits to Us
1. Class and Module support  
2. ES6 features support

{% highlight ts %}
// class
class B extends A {
	private a: number;
	constructor(b) {
		super(b);
		this.a = 0;
	}
	
	method(): void {
		console.log(this.a);
	}
	
	static method() {
		console.log(‘I am a static method’);
	}
}

// for..of loops
var arr = [‘a’, ‘b’, ‘c’];
for (let item of arr) {
	console.log(item);
}
{% endhighlight %}

3. More smooth to switch back toand forth between frontend and backend coding as in contrary to JavaScript, TypeScript syntax is more similar to Java than JavaScript.
4. APIs are clearly declared in .d.ts
5. Static type checks to surfface more bugs at compiling time  
   For example if we use the wrong data type in our browser beacon, we now can get compiling errors while before using typescript, can only be found by testing against backend.
Refer to [TypeScript ES6 Compatibility Table](http://kangax.github.io/compat-table/es6/#typescript) for more ES6 features you can use.
6. Easily to tailor the js library into multiple versions. For example, we from the same coe base, we can generate specific verions for desktop browser and mobile browser with specific features on difference devices. We just need to create a main ts file and reference the modules to include for each version.
7. Extra bonus when we start using Scala as they have [similar syntax](http://www.slideshare.net/razvanc/quick-typescript-vs-scala-sample)

# Wanted Features

1. Can we merge the same module in the same function rather than multiple functions?


{% highlight ts %}
module A {
	function foo() { }
}

module A {
	function bar() {
		foo();
	}
}
{% endhighlight %}

generates below code with compiling error “cannot find name ‘foo’”.

{% highlight ts %}
var A;
(function (A) {
    function foo() { }
})(A || (A = {}));
var A;
(function (A) {
    function bar() {
        foo();
    }
})(A || (A = {}));
{% endhighlight %}

foo function defined within the first anonymous function call for module A is not visible in the second anonymous function call, you have to export it as

{% highlight ts %}
module A {
	export function foo() { }
}

module A {
	function bar() {
		foo();
	}
}
{% endhighlight %}

generates below code without error

{% highlight ts %}
var A;
(function (A) {
    function foo() { }
    A.foo = foo;
})(A || (A = {}));
var A;
(function (A) {
    function bar() {
        A.foo();
    }
})(A || (A = {}));
{% endhighlight %}

The problem here is now A.foo is not only visible to module A, anyone can call it and change it now.

There is no module level visible concept which should be similar to Java’s “package-private” when there is no modifier for Java classes or members.

This could be solved by generating

{% highlight ts %}
module A {
	export function foo() { }
}

module A {
	function bar() {
		foo();
	}
}
{% endhighlight %}

to

{% highlight ts %}
var A;
(function (A) {
    function foo() { }

    function bar() {
        foo();
    }
})(A || (A = {}));
{% endhighlight %}

The problem of merging into one function is there could be name conflicts between the same module in two files. But the compiler can report error in this case, and if it’s the same that two people are working independently on the same module in two files, isn’t it better to create a two different sub modules?

So I think merging into one function could be a feasible way support module level visibility.

When I am writing this article, I find the **/* @internal */** annotation in the ts compiler source code and it’s an experimental option released with typescript 1.5.0-alpha to [strip the declarations marked as @internal](https://github.com/Microsoft/TypeScript/pull/1913).

It helps to only include the declarations without @internal(which serves as your external APIs) when generating the .d.ts file from your code.

And if your consumer are using TypeScript too, this prevents if from using your interal members.

Generating the .d.ts file for

{% highlight ts %}
module A {
	/* @internal */ export function internal() {}
	export function external() {}
}
{% endhighlight %}

by

{% highlight bash %}
tsc -d --stripInternal A.ts
{% endhighlight %}

will output

{% highlight ts %}
declare module A {
    function external(): void;
}
{% endhighlight %}

However, if your consumer uses JavaScript, they can still use the internal function.

# Summary
By and large, it’s a pleasant and rewarding experience to move to TypeScript.
Thought it adds limitations on your JavaScript implementation, you can either find a good workaround or the benefits outweigh it.

It’s now an active open source project (about 300 commits to master in last month) with well documentation to help you start easily.

And just two month ago, Google has also announced to stop AtScript and replaced it with TypeScript.
Angular 2 is now built with TypeScript too.

So far, we are happy that we moved to TypeScript.


	

