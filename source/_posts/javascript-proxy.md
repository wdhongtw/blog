---
title: The Backbone of Frontend Framework
date: 2025-02-04 22:10:00
categories: [notes]
tags: [javascript]
---

It's almost impossible to write a frontend without any UI framework nowadays.

All these frameworks provide some kind of mechanism that, whenever we change a value in JavaScript,
some element in UI will be changed automatically. For example, adding new row to a table
when we push a item into a list.

But how does this work under the surface?
That's the *Proxy Object*.

---

## Internal Method and Proxy

We need to talk about *internal method* in JS first.

A *internal method* is a special method/function that decide how a object behaves in JS runtime.
For example, `[[GET]]` internal method determines the result when we access some property `prop`
from an object `obj`, e.g. `obj.prop`. And `[[Call]]` internal method determines what to
proceed when we call the object `obj` as a function, e.g. `obj()`.

These internal methods can be intercepted and customized from some existing object,
with a *Proxy* instance.

For example, if we want to intercept the property-read action of some object. Here is the way to do.

```javascript
var person = {
    name: "Alice"
};

var agent = new Proxy(
    person,
    {
        get: function(target, property, receiver) {
            console.log(`access "${property}" as ${target[property]}`);
            return target[property];
        },
    },
);

var _ = agent.name; // prints: access "name" as Alice"
```

In the snippet above, `agent` is the proxy object instance.
The second argument to Proxy constructor is the *handler*, and the `get` function in that handler
is a *trap* to the `[[GET]]` internal method.

See 

## The `set` Trap for Binding

Just like we can intercept the property **read** action for some object, we can also intercept
the property **write** action. And that's what UI frameworks do underneath.

Assuming we have a paragraph in DOM tree, identified by id `greeting`.

```html
<p id="greeting">Hi</p>
```

We can ensure that the text is updated automatically when we are doing some
change to JavaScript object. 

```javascript
var element = document.querySelector('#greeting');
var handler = {
    set: function(target, property, value, receiver) {
        if (property !== 'text')
            return false;
        target[property] = value;
        element.textContent = value; // react to the change
        return true;
    }
};

var greeting = new Proxy({text: element.textContent}, handler);
console.log(greeting.text); // print 'Hi'
greeting.text = 'Hello';
// the text in the html page will be changed accordingly.
```

Once this proxy object is created, we make the view (HTML) reactive to
the model (JavaScript object), and the binding is established.

---

In the examples above, we may noticed that all the intercepted objects are an *object* literally.
That means we can not do similar thing to primitive type in JS, like `number` and `string`.
And that do have impact on the design of UI framework.
For example, Vue.js framework has a [`ref()`][vue-ref] wrapper for that.

[vue-ref]: https://vuejs.org/guide/essentials/reactivity-fundamentals#ref

However, Proxy is still a powerful feature in JS, and probably the most important feature
for any senior engineer who need to work with JS.
So just get used to it. :D

See Also: [Proxy - MDN][proxy-javascript-mdn]

[proxy-javascript-mdn]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy

## Appendix

Full HTML file for the view-model binding example.

```html
<!DOCTYPE html> 
<html>
<head>
    <title>Proxy Demo</title>
</head>
<body>
    <p id="greeting">Hi</p>
    <script>
        document.addEventListener('DOMContentLoaded', function() {
            var element = document.querySelector('#greeting');
            var handler = {
                set: function(target, property, value, receiver) {
                    if (property !== 'text')
                        return false;
                    target[property] = value;
                    element.textContent = value; // react to the change
                    return true;
                }
            };

            var greeting = new Proxy({text: element.textContent}, handler);
            console.log(greeting.text); // print 'Hi'
            greeting.text = 'Hello';
            // the text in the html page will change accordingly.
        });
    </script>
</body>
</html>
```
