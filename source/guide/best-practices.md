title: Tips & Best Practices
type: guide
order: 15
is_new: true
---

## Data Initialization

Vue's data observation model favors deterministic data models. It is recommended to initialize all the data properties that needs to be reactive upfront in the `data` option. For example, given the following template:

``` html
<div id="demo">
  <p v-class="green: validation.valid">{{message}}</p>
  <input v-model="message">
</div>
```

It's recommended to initialize your data like this instead of an empty object:

``` js
new Vue({
  el: '#demo',
  data: {
    message: '',
    validation: {
      valid: false
    }
  }
})
```

The reason for this is that Vue observes data changes by recursively walking the data object and converting existing properties into reactive getters and setters using `Object.defineProperty`. If a property is not present when the instance is created, Vue will not be able to track it.

You don't have to set every single nested property in your data though. It is ok to initialize a field as an empty object, and set it to a new object with nested structures later, because Vue will be able to walk the nested properties of this new object and observe them.

## Adding and Deleting Properties

As mentioned above, Vue observes data by converting properties with `Object.defineProperty`. However, in ECMAScript 5 there is no way to detect when a new property is added to an Object, or when a property is deleted from an Object. To deal with this constraint, observed Objects are augmented with three methods:

- `obj.$add(key, value)`
- `obj.$set(key, value)`
- `obj.$delete(key)`

These methods can be used to add / delete properties from observed objects while triggering the desired DOM updates. The difference between `$add` and `$set` is that `$add` will return early if the key already exists on the object, so just calling `obj.$add(key)` won’t overwrite the existing value with `undefined`.

A related note is that when you change a data-bound Array directly by setting indices (e.g. `arr[1] = value`), Vue will not be able to pick up the change. Again, you can use augmented methods to notify Vue.js about those changes. Observerd Arrays are augmented with two methods:

- `arr.$set(index, value)`
- `arr.$remove(index | value)`

Vue component instances also have corresponding instance methods:

- `vm.$get(path)`
- `vm.$set(path, value)`
- `vm.$add(key, value)`
- `vm.$delete(key, value)`

Note that `vm.$get` and `vm.$set` both accept paths.

<p class="tip">Despite the existence of these methods, make sure to only add observed fields when necessary. It's helpful to think of the `data` option as the schema for your component state. Explicitly listing possiblely-present properties in the component definition makes it easy to understand what a component may contain when you look at it later.</p>

## Understanding Async Updates

It is important to know that by default Vue performs view updates **asynchronously**. Whenever a data change is observed, Vue will open a queue and buffer all the data changes that happens in the same event loop. If the same watcher is triggered multiple times, it will be pushed into the queue only once. Then, in the next event loop "tick", Vue flushes the queue and performs only the necessary DOM updates. Internally Vue uses `MutationObserver` if available for the asynchronous queueing and falls back to `setTimeout(fn, 0)`.

For example, when you set `vm.someData = 'new value'`, the DOM will not update immediately. It will update in the next "tick", when the queue is flushed. This behavior can be tricky when you want to do something that depends on the updated DOM state. Although Vue.js generally encourages developers to think in a "data-driven" way and avoid touching the DOM directly, sometimes you might just want to use that handy jQuery plugin you've always been using. In order to wait until Vue.js has finished updating the DOM after a data change, you can use `Vue.nextTick(callback)` immediately after the data is changed - when the callback is called, the DOM would have been updated. For example:

``` html
<div id="example">{{msg}}</div>
```

``` js
var vm = new Vue({
  el: '#example',
  data: {
    msg: '123'
  }
})
vm.msg = 'new message' // change data
vm.$el.textContent === 'new message' // false
Vue.nextTick(function () {
  vm.$el.textContent === 'new message' // true
})
```

There is also the `vm.$nextTick()` instance method, which is especially handy inside components, because it doesn't need global `Vue` and its callback's `this` context will be automatically bound to the current Vue instance:

``` js
Vue.component('example', {
  template: '<span>{{msg}}</span>',
  data: function () {
    return {
      msg: 'not updated'
    }
  },
  methods: {
    updateMessage: function () {
      this.msg = 'updated'
      console.log(this.$el.textContent) // => 'not updated'
      this.$nextTick(function () {
        console.log(this.$el.textContent) // => 'updated'
      })
    }
  }
})
```

## Component Scope

Every Vue.js component is a separate Vue instance with its own scope. It's important to understand how scopes work when using components. The rule of thumb is:

> If something appears in the parent template, it will be compiled in parent scope; if it appears in child template, it will be compiled in child scope.

A common mistake is trying to bind a directive to a child property/method in the parent template:

``` html
<div id="demo">
  <!-- does NOT work -->
  <child-component v-on="click: childMethod"></child-component>
</div>
```

If you need to bind child-scope directives on a component root node, you should do so in the child component's own template:

``` js
Vue.component('child-component', {
  // this does work, because we are in the right scope
  template: '<div v-on="click: childMethod">Child</div>',
  methods: {
    childMethod: function () {
      console.log('child method invoked!')
    }
  }
})
```

Note this pattern also applies to `$index` when using a component with `v-repeat`.

Similarly, HTML content inside a component container are considered "transclusion content". They will not be inserted anywhere unless the child template contains at least one `<content></content>` outlet. The inserted contents are also compiled in parent scope:

``` html
<div>
  <child-component>
    <!-- compiled in parent scope -->
    <p>{{msg}}</p>
  </child-component>
</div>
```

You can use the `inline-template` attribute to indicate you want the content to be compiled in the child scope as the child's template:

``` html
<div>
  <child-component inline-template>
    <!-- compiled in child scope -->
    <p>{{msg}}</p>
  </child-component>
</div>
```

For more details, see [Content Insertion](/guide/components.html#Content_Insertion).

## Communication Between Instances

A common pattern for parent-child communication in Vue is passing down a parent method as a callback to the child using `props`. This allows the communication to be defined inside the template (where composition happens) while keeping the JavaScript implementation details decoupled:

``` html
<div id="demo">
  <p>Child says: {{msg}}</p>
  <child-component send-message="{{onChildMsg}}"></child-component>
</div>
```

``` js
new Vue({
  el: '#demo',
  data: {
    msg: ''
  },
  methods: {
    onChildMsg: function(msg) {
      this.msg = msg
      return 'Got it!'
    }
  },
  components: {
    'child-component': {
      props: [
        // you can use prop assertions to ensure the
        // callback prop is indeed a function.
        {
          name: 'send-message',
          type: Function,
          required: true
        }
      ],
      // props with hyphens are auto-camelized
      template:
        '<button v-on="click:onClick">Say Yeah!</button>' +
        '<p>Parent responds: {{response}}</p>',
      // component `data` option must be a function
      data: function () {
        return {
          response: ''
        }
      },
      methods: {
        onClick: function () {
          this.response = this.sendMessage('Yeah!')
        }
      }
    }
  }
})
```

**Result:**

<div id="demo"><p>Child says: {&#123;msg&#125;}</p><child-component send-message="{&#123;onChildMsg&#125;}"></child-component></div>

<script>
new Vue({
  el: '#demo',
  data: {
    msg: ''
  },
  methods: {
    onChildMsg: function(msg) {
      this.msg = msg
      return 'Got it!'
    }
  },
  components: {
    'child-component': {
      props: ['send-message'],
      data: function () {
        return {
          fromParent: ''
        }
      },
      template:
        '<button v-on="click:onClick">Say Yeah!</button>' +
        '<p>Parent responds: <span v-text="fromParent"></span></p>',
      methods: {
        onClick: function () {
          this.fromParent = this.sendMessage('Yeah!')
        }
      }
    }
  }
})
</script>

When you need to communicate across multiple nested components, you can use the [Event System](/api/instance-methods.html#Events). In addition, it is also quite feasible to implement a [Flux](https://facebook.github.io/flux/docs/overview.html)-like architecture with Vue, which you may want to consider for larger-scale applications.

## Fragment Instance

Starting in 0.12.2, the `replace` option now defaults to `true`. This basically means:

> Whatever you put in the template will be what ends up rendered in the DOM.

So, if your template contains more than one top-level element:

``` js
Vue.component('example', {
  template:
    '<div>A</div>' +
    '<div>B</div>'
})
```

Or, if the template contains only text:

``` js
Vue.component('example', {
  template: 'Hello world'
})
```

In both cases, the instance will become a **fragment instance** which doesn't have a root element. A fragment instance's `$el` will point to an "anchor node", which is an empty Text node (or a Comment node in debug mode). What's probably more important though, is that directives, transitions and attributes (except for props) on the component element will not take any effect - because there is no root element to bind them to:

``` html
<!-- doesn't work due to no root element -->
<example v-show="ok" v-transition="fade"></example>

<!-- props work as intended -->
<example prop="{{someData}}"></example>
```

There are, of course, valid use cases for fragment instances, but it is in general a good idea to give your component template a single root element. It ensures directives and attributes on the component element to be properly transferred, and also results in slightly better performance.

## Changing Default Options

It is possible to change the default value of an option by setting it on the global `Vue.options` object. For example, you can set `Vue.options.replace = false` to give all Vue instances the behavior of `replace: false`. Use this feature carefully, and use it only when you are starting a new project, because it affects the behavior of every instance.

Next: [Common FAQs](/guide/faq.html).
