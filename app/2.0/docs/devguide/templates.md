---
title: Data binding helper elements
---

<!-- toc -->

Polymer provides a set of custom elements to help with common
data binding use cases:

-   Template repeater (`dom-repeat`). Creates an instance of the template's contents for each item
    in an array.
-   Array selector. Manages selection state for an array of structured data.
-   Conditional template (`dom-if`). Stamps its contents if a given condition is true.
-   Auto-binding template (`dom-bind`). Allows data binding outside of a Polymer element.

**2.0 tip.** The data binding helper elements are bundled in to the backward-compatible,
`polymer.html` import. If you aren't using the legacy import, you'll need to import the
helper elements you're using.
{.alert .alert-info}

## Template repeater (dom-repeat) {#dom-repeat}

The template repeater is a specialized template that binds to an array.
It creates one instance of the template's contents for each item in the array.
For each instance, it creates a new [data binding scope](data-system#data-binding-scope)
that includes the following properties:

*   `item`. The array item used to create this instance.
*   `index`. The index of `item` in the array. (The `index` value changes if
    the array is sorted or filtered)

There are two ways to use a template repeater:

*   **Inside a Polymer element or another Polymer-managed template.** Use the shorthand form
    `<template is="dom-repeat>`.

        <template is="dom-repeat" items="{{items}}">
          ...
        </template>

*   **Outside of a Polymer-managed template.** Use the `<dom-repeat>` wrapper element:

        <dom-repeat>
          <template>
            ...
          </template>
        </dom-repeat>

    In this form, you typically set the `items` property imperatively:

        var repeater = document.querySelector('dom-repeat');
        repeater.items = someArray;

A Polymer-managed template includes a Polymer element's template, or the template belonging
to a `dom-bind`, `dom-if`, or `dom-repeat` template, or a template managed by the `Templatize`
library.

In most cases, you'll use the first (shorthand) form for `dom-repeat`.

The template repeater is included in the legacy (`polymer.html`) import for backwards compatibility.
If you're not importing `polymer.html`, import `dom-repeat.html` as shown in the code below.

Example: { .caption }

```html
<link rel="import" href="components/polymer/polymer-element.html">
<! -- import template repeater -->
<link rel="import" href="components/polymer/src/elements/dom-repeat.html">

<dom-module id="x-custom">
  <template>

    <div> Employee list: </div>
    <template is="dom-repeat" items="{{employees}}">
        <div># [[index]]</div>
        <div>First name: [[item.first]]</span></div>
        <div>Last name: [[item.last]]</span></div>
    </template>

  </template>

  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          employees: {
            type: Array,
            value() {
              return [
                {first: 'Bob', last: 'Smith'},
                {first: 'Sally', last: 'Johnson'},
              ];
            }
          }
        }
      }

    }

    customElements.define(XCustom.is, XCustom);
  </script>

</dom-module>
```

Notifications for changes to item sub-properties are forwarded to the template
instances, which update using the normal [change notification events](data-system#change-events).
If the `items` array is bound using two-way binding delimiters, changes to individual items can also
flow upward.

For the template repeater to reflect changes, you need to update the `items` array
observably. For example:

```js
// Use Polymer array mutation methods:
this.push('employees', {first: 'Diana', last: 'Villiers'});

// Use Polymer set method:
this.set('employees.2.last', 'Maturin');

// Use native methods followed by notifyPath
this.employees.push({first: 'Barret', last: 'Bonden'});
this.notifyPath('employees');
```

For more information, see [Mutating objects and arrays observably](data-system#make-observable-changes).


### Handling events in `dom-repeat` templates {#handling-events}

When handling events generated by a `dom-repeat` template instance, you
frequently want to map the element firing the event to the model data that
generated that item.

When you add a declarative event handler **inside** the `<dom-repeat>` template,
the repeater adds a `model` property to each event sent to the listener. The `model`
object contains the scope data used to generate the template instance, so the item
data is `model.item`:

```html
<link rel="import" href="polymer/polymer-element.html">
<link rel="import" href="polymer/src/elements/dom-repeat.html">

<dom-module id="x-custom">

  <template>
    <template is="dom-repeat" id="menu" items="{{menuItems}}">
        <div>
          <span>{{item.name}}</span>
          <span>{{item.ordered}}</span>
          <button on-click="order">Order</button>
        </div>
    </template>
  </template>

  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          menuItems: {
            type: Array,
            value() {
              return [
                {name: 'Pizza', ordered: 0},
                {name: 'Pasta', ordered: 0},
                {name: 'Toast', ordered: 0}
              ];
            }
          }
        }
      }

      order(e) {
        e.model.set('item.ordered', e.model.item.ordered+1);
      }
    }

    customElements.define(XCustom.is, XCustom);
  </script>

</dom-module>
```

The `model` is an instance of `TemplateInstance`, which provides the Polymer
data APIs: `get`, `set`, `setProperties`, `notifyPath` and the array manipulation methods.
You can use these to manipulate the model, using paths _relative to template instance._

For example, in the code above, if the user clicks the button next to **Pizza**, the handler
runs this code:

```js
e.model.set('item.ordered', e.model.item.ordered+1);
```

This increments the order count for the item (in this case, Pizza).

**Only bound data is available on the model object.** Only the properties that are actually data
bound inside the `dom-repeat` are added to the model object. So in some cases, if you need to
access a property from an event handler, it might be necessary to bind it to a property in the
template. For example, if your handler needs to access a `productId` property, simply bind it to
a property where it doesn't affect the display:

```html
  <template is="dom-repeat" items="{{products}}" as="product">
    <div product-id="[[product.productId]]">[[product.name]]</div>
  </template>
```

#### Handling events outside the `dom-repeat` template.

The `model` property is **not** added for event listeners registered
imperatively (using `addEventListener`), or listeners added to one of the
`dom-repeat` template's parent nodes. In these cases, you can use
the `dom-repeat` `modelForElement` method to retrieve the
model data that generated a given element. (There are also corresponding
`itemForElement` and `indexForElement` methods.)
{ .alert .alert-info }


### Filtering and sorting lists

To filter or sort the _displayed_ items in your list, specify a `filter` or
`sort` property on the `dom-repeat` (or both):

*   `filter`. Specifies a filter callback function, that takes a single argument
    (the item) and returns true to display the item, false to omit it.
    Note that this is **similar** to the standard
    `Array` [`filter`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter) API, but the callback only takes a single argument, the array item. For performance reasons, it doesn't include the `index` argument. See [Filtering on array index](#filtering-on-index) for more information.
*   `sort`. Specifies a comparison function following the standard `Array`
    [`sort`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort) API.

In both cases, the value can be either a function object, or a string identifying a
function defined on the host element.

By default, the `filter` and `sort` functions only run when one of the
following occurs:

*   An [observable change](data-system#observable-changes) is made to the array
    (for example, by adding or removing items).
*   The `filter` or `sort` function is changed.

To re-run the `filter` or `sort` when an unrelated piece of data changes,
call [`render`](#synchronous-renders). For example, if your element has a
`sortOrder` property that changes how the `sort` function works, you can
call `render` when `sortOrder` changes.

To re-run the `filter` or `sort` functions when certain sub-fields
of `items` change, set the `observe` property to a space-separated list of
`item` sub-fields that should cause the list to be re-filtered or re-sorted.

For example, for a `dom-repeat` with a filter of the following:

```
isEngineer: function(item) {
    return item.type == 'engineer' || item.manager.type == 'engineer';
}
```

Then the `observe` property should be configured as follows:

```
<template is="dom-repeat" items="{{employees}}"
    filter="isEngineer" observe="type manager.type">
```

Changing a `manager.type` field should now cause the list to be re-sorted:

```
this.set('employees.0.manager.type', 'engineer');
```

#### Dynamic sort and filter changes

The `observe` property lets you specify item sub-properties to
observe for filtering and sorting purposes. However, sometimes you want to
dynamically change the sort or filter based on another unrelated value. In
this case, you can use a computed binding to _return_ a dynamic filter or
sort function when one or more dependent properties changes.

```html
<dom-module id="x-custom">

  <template>
    <input value="{{searchString::input}}">

    <!-- computeFilter returns a new filter function whenever searchString changes -->
    <template is="dom-repeat" items="{{employees}}" as="employee"
        filter="{{computeFilter(searchString)}}">
        <div>{{employee.lastname}}, {{employee.firstname}}</div>
    </template>
  </template>

  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          employees: {
            type: Array,
            value() {
              return [
                { firstname: "Jack", lastname: "Aubrey" },
                { firstname: "Anne", lastname: "Elliot" },
                { firstname: "Stephen", lastname: "Maturin" },
                { firstname: "Emma", lastname: "Woodhouse" }
              ]
            }
          }
        }
      }

      computeFilter(string) {
        if (!string) {
          // set filter to null to disable filtering
          return null;
        } else {
          // return a filter function for the current search string
          string = string.toLowerCase();
          return function(employee) {
            var first = employee.firstname.toLowerCase();
            var last = employee.lastname.toLowerCase();
            return (first.indexOf(string) != -1 ||
                last.indexOf(string) != -1);
          };
        }
      }
    }

    customElements.define(XCustom.is, XCustom);
  </script>
</dom-module>
```

In this example, whenever the value of the `searchString` property changes,
`computeFilter` is called to compute a new value for the `filter` property.

#### Filtering on array index {#filtering-on-index}

Because of the way Polymer tracks arrays internally, the array
index isn't passed to the filter function. Looking up the array index for an
item is an O(n) operation. Doing so  in a filter function could have
**significant performance impact**.

If you need to look up the array index and are willing to pay the performance penalty,
you can use code like the following:

```js
filter: function(item) {
  var index = this.items.indexOf(item);
  ...
}
```

The filter function is called with the `dom-repeat` as the `this` value, so
you can access the original array as `this.items` and use it to look up the index.

This lookup returns the items index in the **original** array, which may not match
the index of the array as displayed (filtered and sorted).

### Nesting dom-repeat templates {#nesting-templates}

When nesting multiple `dom-repeat` templates, you may want to access data
from a parent scope. Inside a `dom-repeat`, you can access any properties available
to the parent scope unless they're hidden by a property in the current scope.

For example, the default `item` and `index` properties added by `dom-repeat`
hide any similarly-named properties in a parent scope.

To access properties from nested `dom-repeat` templates, use the `as` attribute to
assign a different name for the item property. Use the `index-as` attribute to assign a
different name for the index property.

```html
<div> Employee list: </div>
<template is="dom-repeat" items="{{employees}}" as="employee">
    <div>First name: <span>{{employee.first}}</span></div>
    <div>Last name: <span>{{employee.last}}</span></div>

    <div>Direct reports:</div>

    <template is="dom-repeat" items="{{employee.reports}}" as="report" index-as="report_no">
      <div><span>{{report_no}}</span>.
           <span>{{report.first}}</span> <span>{{report.last}}</span>
      </div>
    </template>
</template>
```

### Forcing synchronous renders {#synchronous-renders}

Call [`render`](/{{{polymer_version_dir}}}/docs/api/dom-repeat#method-render)
to force a `dom-repeat` template to synchronously render any changes to its
data. Normally changes are batched and rendered asynchronously. Synchronous
rendering has a performance cost, but can be useful in a few scenarios:

*   For unit testing, to ensure items have rendered before checking the
    generated DOM.
*   To ensure a list of items have rendered before scrolling to a
    specific item.
*   To re-run the `sort` or `filter` functions when a piece of data changes
    *outside* the array (sort order or filter criteria, for example).

`render` **only** picks up [observable changes](data-system#observable-changes)
such as those made with Polymer's [array mutation methods](model-data#array-mutation).
If you or a third-party library mutate the array **without** using Polymer's methods, you can do
one of the following:

*   If you know the _exact set of changes made to your array_, use
    [`notifySplices`](model-data#notifysplices) to ensure that any elements watching the
    array are properly notified.
*   If you don't have an exact set of changes, you can [Override dirty
    checking](model-data#override-dirty-check) to force the data system to reevaluate the entire
    array.

For more information on working with arrays and the Polymer data system, see [Work with
arrays](model-data#work-with-arrays).

### Improve performance for large lists {#large-list-perf}

By default, `dom-repeat` tries to render all of the list items at once. If
you try to use `dom-repeat` to render a very large list of items, the UI may
freeze while it's rendering the list. If you encounter this problem, enable
"chunked" rendering by setting
[`initialCount`](/{{{polymer_version_dir}}}/docs/api/dom-repeat#property-initialCount).
In chunked mode,
`dom-repeat` renders `initialCount` items at first, then renders the rest of
the items incrementally one chunk per animation frame. This lets the UI thread
handle user input between chunks. You can keep track of how many items have
been rendered with the
[`renderedItemCount`](/{{{polymer_version_dir}}}/docs/api/dom-repeat#property-renderedItemCount)
read-only property.

`dom-repeat` adjusts the number of items rendered in each chunk to try and
maintain a target framerate. You can further tune rendering by setting
[`targetFramerate`](/{{{polymer_version_dir}}}/docs/api/dom-repeat#property-targetFramerate).

You can also set a debounce time that must pass before a `filter` or `sort`
function is re-run by setting the
[`delay`](/{{{polymer_version_dir}}}/docs/api/dom-repeat#property-delay)
property.

## Data bind an array selection (array-selector) {#array-selector}

Keeping structured data in sync requires that Polymer understand the path
associations of data being bound.  The `array-selector` element ensures path
linkage when selecting specific items from an array.

The `items` property accepts an array of user data. Call `select(item)`
and `deselect(item)` to update the `selected` property, which may be bound to
other parts of the application. Any changes to sub-fields of the selected
item(s) are kept in sync with items in the `items` array.

The array selector supports either single or multiple selection.
When `multi` is false, `selected` is a property representing the last selected
item.  When `multi` is true, `selected` is an array of selected items.

The array selector is included in the legacy (`polymer.html`) import for backwards compatibility.
If you're not importing `polymer.html`, import `array-selector.html` as shown in the code below.

```html
<link rel="import" href="components/polymer/polymer-element.html">
<! -- import template repeater -->
<link rel="import" href="components/polymer/src/elements/dom-repeat.html">
<!-- import array selector -->
<link rel="import" href="components/polymer/src/elements/array-selector.html">

<dom-module id="x-custom">

  <template>

    <div> Employee list: </div>
    <template is="dom-repeat" id="employeeList" items="{{employees}}">
        <div>First name: <span>{{item.first}}</span></div>
        <div>Last name: <span>{{item.last}}</span></div>
        <button on-click="toggleSelection">Select</button>
    </template>

    <array-selector id="selector" items="{{employees}}" selected="{{selected}}" multi toggle></array-selector>

    <div> Selected employees: </div>
    <template is="dom-repeat" items="{{selected}}">
        <div>First name: <span>{{item.first}}</span></div>
        <div>Last name: <span>{{item.last}}</span></div>
    </template>

  </template>

  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          employees: {
            type: Array,
            value() {
              return [
                {first: 'Bob', last: 'Smith'},
                {first: 'Sally', last: 'Johnson'},
                // ...
              ];
            }
          }
        }
      }

      toggleSelection(e) {
        var item = this.$.employeeList.itemForElement(e.target);
        this.$.selector.select(item);
      }
    }

    customElements.define(XCustom.is, XCustom);
  </script>

</dom-module>
```

## Conditional templates (dom-if) {#dom-if}

Elements can be conditionally stamped based on a boolean property by wrapping
them in a custom `HTMLTemplateElement` type extension called `dom-if`.  The
`dom-if` template stamps its contents into the DOM only when its `if` property becomes
truthy.

If the `if` property becomes falsy again, by default all stamped elements are hidden
(but remain in the DOM tree). This provides faster performance should the `if`
property become truthy again.  To disable this behavior, set the
`restamp` property to `true`. This results in slower `if` switching behavior as the
elements are destroyed and re-stamped each time.

There are two ways to use a conditional template:

*   **Inside a Polymer element or another Polymer-managed template.** Use the shorthand form
    `<template is="dom-repeat>`.

        <template is="dom-if" if="{{condition}}">
          ...
        </template>

*   **Outside of a Polymer-managed template.** Use the `<dom-if>` wrapper element:

        <dom-if>
          <template>
            ...
          </template>
        </dom-if>

    In this form, you typically set the `items` property imperatively:

        var conditional = document.querySelector('dom-if');
        conditional.if = true;

A Polymer-managed template includes a Polymer element's template, a `dom-bind`, `dom-if`, or
`dom-repeat` template, or a template managed by the `Templatizer` class.

In most cases, you'll use the first (shorthand) form for `dom-if`.

The conditional template is included in the legacy (`polymer.html`) import for backwards
compatibility.  If you're not importing `polymer.html`, import `array-selector.html` as shown in
the code below.

The following is a simple example to show how conditional templates work. Read below for
guidance on recommended usage of conditional templates.

Example: { .caption }

```
<link rel="import" href="components/polymer/polymer-element.html">
<! -- import conditional template -->
<link rel="import" href="components/polymer/src/elements/dom-if.html">

<dom-module id="x-custom">

  <template>

    <!-- All users will see this -->
    <my-user-profile user="{{user}}"></my-user-profile>


    <template is="dom-if" if="{{user.isAdmin}}">
      <!-- Only admins will see this. -->
      <my-admin-panel user="{{user}}"></my-admin-panel>
    </template>

  </template>

  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          user: Object
        }
      }

    }

    customElements.define(XCustom.is, XCustom);
  </script>

</dom-module>
```

Conditional templates introduce some overhead, so they shouldn't be used for small UI elements
that could be easily shown and hidden using CSS.

Instead, use conditional templates to improve loading time or reduce your page's memory footprint.
For example:

-   Lazy-loading sections of your page. If some elements of your page aren't required on first
    paint, you can use a `dom-if` to hide them until their definitions have loaded. This use
    of conditional templates is described in [Case study: the Shop app](https://www.polymer-project.org/1.0/toolbox/case-study#views).

-   Reducing the memory footprint of a large or complex site. For a single-page application
    with multiple complex views, it may be beneficial to put each view inside a `dom-if`
    with the `restamp` property set. This improves memory use at the cost of some latency
    each time the user switches view (to recreate the DOM for that section).

There's no one-size-fits-all guidance for when to use a conditional template. Profiling your site
should help you understand where conditional templates are helpful.

## Auto-binding templates (dom-bind) {#dom-bind}

Polymer data binding is only available in templates that are managed
by Polymer. So data binding works inside an element's DOM
template (or inside a `dom-repeat` or `dom-if` template),
but not for elements placed in the main document.

To use Polymer bindings **without** defining a new custom element,
use the `<dom-bind>` element.  This template immediately stamps the contents of
its child templateinto the main document. Data bindings in an auto-binding template use
the `<dom-bind>` element itself as the binding scope.

```
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <script src="components/webcomponentsjs/webcomponents-lite.js"></script>
  <link rel="import" href="polymer/src/elements/dom-bind.html">
  <link rel="import" href="polymer/src/elements/dom-repeat.html">

</head>
<body>
  <!-- Wrap elements with auto-binding template to -->
  <!-- allow use of Polymer bindings in main document -->
  <dom-bind>
    <template>

      <template is="dom-repeat" items="{{data}}">
        <div>{{item.name}}: {{item.price}}</div>
      </template>

    </template>
  </dom-bind>
  <script>
    var autobind = document.querySelector('dom-bind');

    // The dom-change event signifies when the template has stamped its DOM.
    autobind.addEventListener('dom-change', function() {
      console.log('template is ready.')
    });

    // set data property on dom-bind
    autobind.data = [
      { name: 'book', price: '$5.00'},
      { name: 'pencil', price: '$1.00'},
      { name: 'flux capacitor', price: '$8,000,000.00'}
    ];
  </script>
</body>
</html>
```

All of the features in `dom-bind` are already available _inside_ a Polymer
element. **Auto-binding templates should only be used _outside_ of a Polymer element.**

## dom-change event {#dom-change}

When one of the template helper elements updates the DOM tree, it fires a `dom-change` event.

In most cases, you should interact with the created DOM by changing the _model data_, not by
interacting directly with the created nodes. For those cases where you need to access the
nodes directly, you can use the `dom-change` event.