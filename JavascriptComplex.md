---
layout: default
---
{% include links %}
* TOC
{:toc}

# Optional parameters of functions #
Javascript doesn't allow a function to specify a default value for a parameter. When calling a function and not specifying one or several parameters, they are `undefined`. Use strict comparison operator === to compare with `undefined`. Do not compare for that by non-strict operator == (since `undefined` non-strictly equals to `false, null, 0` and "", i.e. `undefined==true`).

You could have an (optional) parameter, for which you set a meaningful default value, if it's not present. E.g.

```javascript
/** @param id 0-based ID of the sport. 0 by default.
*/
function sport( id=0 ) {
    return ['running', 'soccer', 'basketball'][id];
}
```

However, it gets complicated if you choose a special default value - i.e. other than `0/false/null/""`, or other than something clearly default for the purpose of the function. Programmers may remember that a parameter is optional, but it's easy to forget what the default value means. They would expect the behaviour to be similar or the same as if they passed 0 (or `false` or `null` or ""). E.g.

```javascript
/** @param id integer, 0-based ID of the sport. Optional; 2 by default (for which it returns 'basketball')
*/
function sport( id ) {
  id= id!==undefined
    ? id
    : 2;
  return ['running', 'soccer', 'basketball'][id];
}

sport(0); // this returns 'running'
sport();  // since id is optional and integer, the most obvious default value would be 0
          // or the lowest ID available, but not a fixed ID (i.e. 2)!
```

It becomes especially confusing if the parameter is a boolean, and you choose true as the default value. E.g.

```javascript
var cachedResult= undefined;

/** @param boolean cached Whether to return the cached result (if available). True by default.
*/
function process(cached) {
    cached= cached!==undefined
    ? cached
    : true;
    if( !cached || cachedResult===undefined ) {
        cachedResult= <calculate the result>;
    }
    return cachedResult;
}
process( true ) ; // This returns the cached result (if available)
process( false ); // This always returns a new result
process(); // Since cached is optional and boolean, the most obvious default value would be false, and not true!
```

Sometimes you want some functionality to be used by default, and to turn it off by specifying an optional parameter. Then you need to pick a name for such parameter that represents negation of the default behaviour. It may complicate documentation wording a tiny bit, but it makes the API clearer and more error-proof. See [SeLiteSettings source](https://github.com/SeLite/SeLite/tree/master/settings/src/chrome/content/SeLiteSettings.js) and its function `getFieldsDownToFolder( folderPath, dontCache )` in class `SeLiteSettings.Module`. It demonstrates two approaches for optional parameters

  * `folderPath` defaults to the folder of [suite] currently open in Selenium IDE
  * `dontCache` defaults to false, i.e. the function caches the results by default. (Note that the meaning of `dontCache` is not exactly reverse of `cached` parameter of function `process(cached)` above, because `process(false)` returns a new result every time, but it still updates it in `cachedResult`, which will be returned by `process(true)`).

So, use default parameter values that are 'logically equivalent' to one of `0, false, null` or `""`. Deviate from this only if a different default value (possibly dynamic: based on other parameters) really makes sense for the use of the function. Document it clearly.

# Operator `instanceof` and standard classes #
Operator `instanceof` doesn't work for objects of most standard classes (`Array, Boolean, Number` etc.) if such objects are passed between [Javascript code modules](#javascript-code-modules). That's because such classes (i.e. their constructors) are 'global objects' and they are separate for each 'global scope'. For `Array` you can use `Array.isArray(object)`. For all those classes you can use `SeLiteMisc.isInstance(object)`. See [SeLiteMisc source](https://github.com/SeLite/SeLite/tree/master/misc/src/chrome/content/selite-misc.js).

# Loading Javascript files

## Files loaded through mozIJSSubScriptLoader
Such Javascript files don't need any special structures. SeLite ([Bootstrap](Bootstrap) and [Extension Sequencer]) uses this to load Javascript files into [Core scope]. It's intended for files that extend Selenium or add functonality that should be available in parameters to Selenese commands. This uses [mozIJSSubScriptLoader](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Reference/Interface/mozIJSSubScriptLoader)'s `loadSubScript()`. See also {{navCoreExtensionsLoadedTwice}}.

## Javascript code modules
[Javascript code modules](https://developer.mozilla.org/en-US/docs/Mozilla/JavaScript_code_modules/Using) are loaded via [Components.utils.import(url, scope)](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Language_Bindings/Components.utils.import). They have separate scopes, with any exported classes/functions/objects listed in array `EXPORTED_SYMBOLS`. Code modules don't have access to the outer scope (e.g. to `window` object and to Selenese objects). They are suitable for non-GUI operations and also for functionality not specific to Selenium.

### Functions in Javascript code modules ###
In Javascript code modules you can define functions in [JavascriptEssential](JavascriptEssential) > [the classic way](JavascriptEssential#the-classic-way). You don't need an anonymous function to protect the global scope, because the global scope will contain exported symbols only. So you can use

```javascript
"use strict";

function myFunction( param, anotherParam... ) {
    ...
}
var EXPORTED_SYMBOLS= [myFunction, ...];
```

### Javascript code modules and NetBeans ###
When you have functionality in Javascript code modules (for Firefox) loaded via

```
var XxxYyy= Components.utils.import( 'chrome://my-extension/content/some/path/some-file.js', {} );
```

then NetBeans can't navigate to functionality exported from of such a code module. For that you need to

  * encapsulate the functionality within one plain object (serving as an associative array) in the code module
  * make `EXPORTED_SYMBOLS` contain just the name of that plain object
  * load sources of SeLite or 3rd party code modules as a part of your NetBeans project (see [DevelopmentTools](DevelopmentTools))
  * load the code module into the global scope of the client code using e.g.<br/>`Components.utils.import( "chrome://selite-misc/content/SeLiteMisc.js" );`<br/>instead of e.g.<br/>`var SeLiteMisc= Components.utils.import( "chrome://selite-misc/content/SeLiteMisc.js", {} ).SeLiteMisc;`<br/>because with the second, if you `Ctrl+click` at `SeLiteMisc.abcDef`, NetBeans takes you to the `Components.utils.import()` line, rather than to the source of the functionality.

That makes the navigation and naming same, whether in the code module source itself, or in code where you use the functionality. It means that the code module source itself is a bit more wordy, repeating the name of the exported variable (e.g. `SeLiteMisc`) a lot, but that's OK.

Another option (that facilitates NetBeans navigation) would be to have all classes/functions listed in `EXPORTED_SYMBOLS` instead of having them encapsulated in an object. Again, load the code module into the global scope of the client code using e.g. `Components.utils.import( "chrome://selite-misc/content/SeLiteMisc.js" );`

However, that would flood the client's namespace with no obvious benefit. It would also go against [JavascriptEssential](JavascriptEssential) > [Prevent name conflicts](JavascriptEssential#prevent-name-conflicts).

### Dependent Javascript code modules ###
A Javascript code module can depend on one or more other code modules. If the dependent functionality is optional, wrap any respective code, e.g. calls to `Components.utils.import()`, in a `try{..}` statement, so that clients can still use the essential functionality even without the optional module.

Also, two or more Javascript code modules can depend on each other cyclically. That can be required (for essential functionality), each one requiring the other(s), or optional (for some functionality only). Make sure a code module defines `EXPORTED_SYMBOLS` and any symbols required by its cyclic dependent code module(s) before it loads those dependants via `Components.utils.import()`. See sources of `SeLiteSettings` and `SeLiteData.Storage`. <!--TODO links.-->

# Class inheritance #
There are many ways of class inheritance in Javascript. Follow [Mozilla way](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Introduction_to_Object-Oriented_JavaScript#Inheritance) with `Object.create()`. E.g.:

```javascript
"use strict";
function Parent( name ) {
    this.name= name;
}

function Child( name ) {
    Parent.call( this, name );
}

Child.prototype= Object.create(Parent.prototype);
Child.prototype.constructor= Child;

// Or within a block/function:
if( true ) {
    var LocalParent= function LocalParent( name ) {
        this.name= name;
    };

    var LocalChild= function LocalChild( name ) {
        LocalParent.call( this, name );
    };

    LocalChild.prototype= Object.create(LocalParent.prototype);
    LocalChild.prototype.constructor= LocalChild;
}
```

That ensures integrity of `SeLiteMisc.isInstance()` and operator `instanceof`, which recognise such object as an instance of its class and of any of its parent classes. See also [JavascriptComplex](JavascriptComplex) > [Operator instanceof and standard classes](JavascriptComplex#operator-instanceof-and-standard-classes).

Don't use `objectExtend()` from Selenium Core, since it doesn't work with operator `instanceof`.

# ECMAScript 6 and 7
Use [MDN: Arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions), [MDN: Default parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Default_parameters), [MDN: Rest parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters) and [MDN: Destructuring assignment](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment).

See 

 * [Mozilla support for ECMAScript 6](https://developer.mozilla.org/en/docs/Web/JavaScript/New_in_JavaScript/ECMAScript_6_support_in_Mozilla)
 * [ECMAScript 6 specification](http://www.ecma-international.org/ecma-262/6.0/#sec-function-definitions)
 * [ECMAScript 7 draft](https://tc39.github.io/ecma262/#sec-function-definitions)
 * [new features in ECMAScript 6](http://es6-features.org/) - especially
   *  [Enhanced Object Properties &gt; Computed Property Names](http://es6-features.org/#ComputedPropertyNames)

# See also #
  * [JavascriptEssential](JavascriptEssential)
  * [Google Style Guide](https://google.github.io/styleguide/javascriptguide.xml), especially [Closures](https://google.github.io/styleguide/javascriptguide.xml#Closures)
  * [MDN: Guide to iterators and generators](https://developer.mozilla.org/en/JavaScript/Guide/Iterators_and_Generators)