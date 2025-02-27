# Rules.JS

[![Build](https://github.com/valtech-sd/rules-js/actions/workflows/npm-publish-github-packages.yml/badge.svg)](https://github.com/valtech-sd/rules-js)
[![npm](https://img.shields.io/npm/v/@valtech-sd/rules-js.svg)](https://npmjs.org/package/@valtech-sd/rules-js)
[![npm](https://img.shields.io/npm/dt/@valtech-sd/rules-js.svg)](https://npmjs.org/package/@valtech-sd/rules-js)

## Overview

This is an implementation of a very lightweight pure javascript rule engine.
It's very loosely inspired on drools, but keeping an extra effort in keeping complexity to the bare minimum.

- [Rules.JS](#rulesjs)
  * [Overview](#overview)
    + [Install](#install)
    + [Usage Example](#usage-example)
  * [Engine](#engine)
  * [Fact](#fact)
  * [Closures](#closures)
    + [Provided closures](#provided-closures)
      - [Parameterizable closures](#parameterizable-closures)
      - [Parameterless closures (syntax sugar)](#parameterless-closures-syntax-sugar)
    + [Rules](#rules)
    + [Closure arrays (reduce)](#closure-arrays-reduce)
    + [Rules flow](#rules-flow)
		
### Install

```
npm install @valtech-sd/rules-js
```


### Usage Example
This very naive example on how to create an small rule engine to process online orders. This isn't meant to imitate a full business process, neither to shown the full potential of Rules.JS

We start by defining a rules file in JSON.

```json
{
	"name": "process-orders",
	"rules": [
		 {
			 "when": "always",
			 "then": "calculateTotalPrice"
		 },
		 {
			 "when": { "closure": "checkStockLocation", "location": "localDeposit" },
			 "then": [
				 { "closure": "calculateTaxes", "salesTax": 0.08 },
				 "createDispatchOrder"
			 ]
		 },
		 {
			 "when": { "closure": "checkStockLocation", "location": "foreignDeposit" },
			 "then": [
				 "calculateShipping",
				 "createDispatchOrder"
			 ]
		 },
		 {
			 "when": { "closure": "checkStockLocation", "location": "none" },
			 "then": { "closure": "error", "message": "There is availability of such product"}
		 }
	]
}
```

Now we can evaluate any order using such rules file. First we create the engine.  We can do it inside a js module:

```javascript
const engine = new Engine();

// We are intentionally missing something here:
// We need first to define what the verbs 'checkStockLocation', 'calculateTaxes'
// 'calculateShipping' and 'createDispatchOrder' actually mean

const definitions = require("./process-orders.rules.json");
engine.add(definitions);

module.exports = function (order) {
	return engine.process("process-orders", order);
}
```

Then we just use that module to evaluate orders.  Each order is a **fact** that
will be provided to our rule engine.

```javascript
const orderProcessorEngine = require("./order-processor-engine");

const order = {
	books: [
		{ name: "Good Omens", icbn: "0060853980", price: 12.25 }
	],
	client: "mr-goodbuyer"
};

//result is a Promise since rules might evaluate asynchronically.
orderProcessorEngine(order).then(result => {
	const resultingDispatchOrder = result.fact;
	// handle the result in any way
})
```

Of course, we intentionally omit defining what this verbs mean: *calculateTotalPrice*,
*checkStockLocation*, *calculateTaxes*, *calculateShippingAndHandling* and *createDispatchOrder*.  
Those reference to provided closures and are the place were implementor should provide
their own business code.

This is where we drift slightly away from the drools-like frameworks take on defining
how a rule engine should actually be configured.  We believe that's a good idea
to define separately the implementation of the business actions that can be executed
and the logic that tells us when and how they are triggered.

There are two ways to do define the missing verbs, through functions or objects. Both
of them are rather similar:


```javascript
engine.closures.add("calculateTotalPrice", (fact, context) => {
	fact.totalPrice = fact.books.reduce((total, book) => total + book.price, 0);
	return fact;
});
```

or 

```javascript
class CalculateTotalPrize extends Closure {
	process(fact, context) {
		fact.totalPrice = fact.books.reduce((total, book) => total + book.price, 0);
		return fact;
	} 
}

engine.closures.add("calculateTotalPrice", new CalculateTotalPrize());
```

## Engine
This is the main entry point for Rules.js.  The typically lifecycle of a rule engine implies three steps:

1. Instantiate the rule engine.  This instance will be kept alive during the whole life of the application.
2. Configure the engine provided closures.
3. Configure the engine with high-level closures (rules, ruleflows). This is usually done by requiring a rule-flow definition file 
4. Evaluate multiple facts using the engine and obtain a result for each one of them.

```javascript
const Engine = require("rules-js");

// 1. instantiate
const engine = new Engine();

// 2. configure
engine.closures.add("calculateTotalPrice", (fact, context)) => {
	fact.totalPrice = fact.books.reduce((total, book) => total + book.price, 0);
	return fact;
});

engine.closures.add("calculateTaxes", (fact, context)) => {
	fact.taxes = fact.totalPrice * context.parameters.salesTax;
	return fact;
}, { required: ["salesTax"] });

const definitions = require("./process-orders.rules.json");
engine.closures.create(definitions);

// 3. at some time later, evaluate facts using the engine
module.exports = function (fact) {
	return engine.process("process-orders", fact);
}

```

## Fact
A fact is an object that is feeded to the rule engine in order to produce a
computational result. Any object can be a fact.

## Closures
One of the core concepts of Rules.JS are closures. We defined **closure** as any
computation bound to a certain context that can act over a **fact** and return a
value.

### Provided closures

In Rules.JS we have a mechanism to tie either a plain old javascript function or
an object that extends the `Closure` class to a certain
name. These are provided closures can be later referenced by any other piece of
the rule engine (*rules*, *ruleFlows*) hence becoming the foundational stones of
the library.

```javascript
// a simple closure implementation function
function (fact, context) {
	return fact.totalPrice * context.parameters.salesTax;
}

//the same thing implemented through a class
class TaxCalculator extends Closure {
	process(fact, context) {
		return fact.totalPrice * context.parameters.salesTax;
	}
}
```


Note that any closure will receive two parameters:
```
@param      {Object}  fact - the fact is the object that is current being evaluated by the closure.
@param      {Context} context - the fact's execution context
@param      {Object}  context.parameters - the execution parameters
@param      {Engine}  context.engine - the rule engine

@return     {Object}  the result of the computation (this can be a Promise too!)
```
The main parameter is of course the **fact**, closures need to derive their result
from each different fact that is provided. **context.parameters** hash is introduced 
to allow the reuse of closure implementations through parameterization. 

Closures will often enhance the current provided fact by adding extra information
to it. Of course, a closure can always alter the fact.

```javascript
function (fact, context) {
	fact.taxes = fact.totalPrice * 0.8;
	return fact;
}
```

*Note*: It's a good idea to keep closures stateless and idempotent, however this
is not a limitation imposed by Rules.JS

We can register provided closures into a rule engine by invoking the following:

```javascript
engine.closures.add("calculateTaxes", (fact, context) => {
	fact.taxes = fact.totalPrice * 0.08
	return fact;
});
```

Notice that in a simplest form the `add` method receive the name
that we want the closure to have and the closure implementation function.

We can later reference to any provided closure (actually, any *named* closure)
in the JSON rule file through a json object like the following:

```json
{ "closure": "calculateTaxes" }
```

#### Parameterizable closures

We can add parameters to our closures implementation, that way the same closure
code can be reused in different contexts. We can change the former `calculateTaxes`
to receive the tax percentage.

```javascript
engine.closures.add("calculateTaxes", (fact, context) => {
	fact.taxes = fact.totalPrice * context.parameters.salesTax;
	return fact;
}, { required: ["salesTax"] });
```

Now every time that a closure is referenced in a rules file we will need to provide
a value for the `salesTax` parameter (otherwise we will get an error while parsing
it!).

```json
{ "closure": "calculateTaxes", "salesTax": 0.08 }
```

#### Parameterless closures (syntax sugar)
When using closures that don't receive any parameters we can, instead of writing
the whole closure object `{ "closure": calculateShipping" }` we can simply
reference it by its name: `"calculateShipping"`.

### Rules
Rules an special kind of closures that are composed by two component closures (of
any kind!). One of the closures will act as a condition (the *when*), conditionating
the execution of the second closure (the *then*) to the result of its evaluation.

```json
{
	"when": { "closure": "hasStockLocally" },
	"then": { "closure": "calculateTaxes", "salesTax": 0.08 }
}
```

... which is the same than writing ...

```json
{
	"when": "hasStockLocally",
	"then": { "closure": "calculateTaxes", "salesTax": 0.08 }
}
```

### Closure arrays (reduce)
Closures can also be expressed as an array of closures.  When evaluating an
array of closures Rule.JS will perform a reduction, meaning that the resulting
object of each closure will become the fact of the next one.

```json
[
	{ "closure": "calculateTaxes", "salesTax": 0.08 },
	{ "closure": "makeCreditCardCharge" },
	{ "closure": "createDispatchOrder" }
]
```

You can also mix syntaxes inside the array

```json
[
	{ "closure": "calculateTaxes", "salesTax": 0.08 },
	"makeCreditCardCharge",
	"createDispatchOrder"
]
```

Or even using completely different types of closures (i.e. regular provided closures,
rules, nested arrays of closures)

```json
[
	{
		"when": "isTaxAccountable",
		"then": { "closure": "calculateTaxes", "salesTax": 0.08 }
	},
	"makeCreditCardCharge",
	"createDispatchOrder"
]
```

#### Arrays as conditions

You can also use closure arrays as conditions. By default they will work with "and" (`&&`) logic

```json
{
	"when": ["isFoo", "isBar"],
	"then": "executeOrder66"
},
```

You can also define an "and" or "or" strategies to apply them.

```json
{
	"when": ["isFoo", "isBar"],
	"conditionStrategy": "or",
	"then": "executeOrder66"
},
```

There is also a "last" strategy, which makes it work like a regular reducer closure array.

```json
{
	"when": ["transformForFoo", "isFoo"],
	"conditionStrategy": "last",
	"then": "executeOrder66"
},
```

### Rules flow
A rule flow is a definition of a chain of rules that will be evaluated (and applied)
in order. Typically this is the higher order construction that is registered into rules js.

```json
{
	"name": "process-orders",
	"rules": [
		 {
			 "when": "always",
			 "then": "calculateTotalPrice"
		 },
		 {
			 "when": { "closure": "checkStockLocation", "location": "localDeposit" },
			 "then": [
				 { "closure": "calculateTaxes", "salesTax": 0.08 },
				 "createDispatchOrder"
			 ]
		 },
		 {
			 "when": { "closure": "checkStockLocation", "location": "foreignDeposit" },
			 "then": [
				 "calculateShipping",
				 "createDispatchOrder"
			 ]
		 },
		 {
			 "when": { "closure": "checkStockLocation", "location": "none" },
			 "then": { "closure": "error", "message": "There is availability of such product"}
		 }
	]
}
```

 
