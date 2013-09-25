---
layout: default
title: "Communication & Message Passing"
subtitle: Custom Elements

#draft: true

article:
  author: ebidel
  published: 2013-08-12
  #updated: 2013-07-09
  polymer_version: 0.0.20130808
  description: Techniques for passing messages to, from, and inbetween custom elements.
tags:
- signaling
- messaging
- communication
---

{% include authorship.html %}

{% include toc.html %}

You're off creating new elements in HTML! There will come a time
when you need to have one element send a message to another element (or set of elements).
I'm using "message passing" as an overloaded term. What I really mean is **relaying
information between elements** or to the world outside of a custom element.

In {{site.project_title}}, there a slew of reasons why you might need such a
communication channel:

- _Element B_'s internal data model changes. _Element A_ needs to be notified.
- _Element A_ updates its UI based on a selection in _Element B_.
- A third sibling, _Element C_, fires an event that _A_ and _B_ to react to.

In this article, I outline several techniques for sending information to other elements.
It's worth pointing out that **_most_ of these techniques are not specific to {{site.project_title}}**. They're standard ways to make DOM elements interact with each other. The only
difference is that now we're in complete control of HTML! We implement the
control hooks that users tap in to.

## Methods

We'll cover the following techniques:

1. [Data binding](#binding)
1. [Changed watchers](#changedwatchers)
1. [Custom events](#events)
1. [Using an element's API](#api) 

### 1. Data binding {#binding}

<table class="table">
  <tr>
    <th>Pros</th><th>Limitations</th>
  </tr>
  <tr>
    <td>
      <ol>
        <li>No code required</li>
        <li>"Messaging" is two-way between elements</li>
      </ol>
    </td>
    <td>
      <ol>
        <li>Only works inside a {{site.project_title}} element</li>
        <li>Only works between {{site.project_title}} elements</li>
      </ol>
    </td>
  </tr>
</table>

The first (and most {{site.project_title}}ic) way for elements to relay information
to one another is to use data binding. {{site.project_title}} implements
two-way data binding. Binding to a common property
is useful if you're working inside a {{site.project_title}} element and want to
"link" elements together via their [published properties](/polymer.html#published-properties).

Here's an example:

{% raw %}
    <!-- Publish .items so others can use it for attribute binding. -->
    <polymer-element name="td-model" attributes="items">
      <script>
        Polymer('td-model', {
          ready: function() {
            this.items = [1, 2, 3];
          }
        });
      </script>
    </polymer-element>

    <polymer-element name="my-app">
      <template>
        <td-model items="{{list}}"></td-model>
        <polymer-localstorage name="myapplist" value="{{list}}"></polymer-localstorage>
      </template>
      <script>
        Polymer('my-app', {
          ready: function() {
            // Initialize the instance's "list" property to empty array.
            this.list = this.list || [];
          }
        });
      </script>
    </polymer-element>
{% endraw %}

When a {{site.project_title}} element [publishes](/polymer.html#published-properties) one of its properties, you can bind to that property using an HTML attribute of the same name. In the example,
`<td-model>.items` and `<polymer-localstorage>.value` are bound together with "list":

{% raw %}
    <td-model items="{{list}}"></td-model> 
    <polymer-localstorage name="myapplist" value="{{list}}"></polymer-localstorage>
{% endraw %}

What's neat about this? Whenever `<td-model>` updates its `items` array internally,
elements that bind to `list` on the outside see the changes. In this
example, `<polymer-localstorage>`. You can think of "list" as an
internal bus within `<my-app>`. Pop some data on it; any elements that care about
`items` are magically **kept in sync by data-binding**. This means there is one source
of truth. Data changes are simultaneously reflected in all contexts. There is no no dirty check.

**Remember:** Property bindings are two-way. If `<polymer-localstorage>`
changes `list`, `<td-model>`'s items will also change.
{: .alert .alert-info }

#### Property serialization

Wait a sec..."list" is an array. How can it be property bound as an HTML string attribute?

{{site.project_title}} is smart enough to serialize primitive types (numbers, booleans, arrays, objects) when they're used in attribute bindings. As seen in the `ready()` callback
of `<my-app>`, be sure to initialize and/or [hint the type](/polymer.html#hinting-an-attributes-type) of your properties.

### 2. Changed watchers {#changedwatchers}

<table class="table">
  <tr>
    <th>Pros</th><th>Limitations</th>
  </tr>
  <tr>
    <td>
      <ol>
        <li>Useful for elements that haven't published a property for attribute binding.</li>
        <li>Allows an element to observe its own data, regardless of how it's modified.</li>
      </ol>
    </td>
    <td>
      <ol>
        <li>Only works inside a {{site.project_title}} element.</li>
        <li>Requires some knowledge of an element's API</li>
        <li>Small amount of code hookup</li>
      </ol>
    </td>
  </tr>
</table>

Let's say someone creates `<polymer-localstorage2>` and you want to use its new
hotness. The element still defines a `.value` property but it doesn't publish the
property in `attributes=""`. Something like:

    <polymer-element name="polymer-localstorage2" attributes="name useRaw">
    <template>...</template>
    <script>
      Polymer('polymer-localstorage2', {
        value: null,
        valueChanged: function() {
          this.save();
        },
        ...
      });
    </script>
    </polymer-element>

When it comes time to use this element, we're left without a (`value`) HTML attribute
to bind to:

    <polymer-localstorage2 name="myapplist" id="storage"></polymer-localstorage2>

A desperate time like this calls for a [changed watcher](/polymer.html#change-watchers) and
a sprinkle of data-binding. We can exploit the fact that `<polymer-localstorage2>` defines
a `valueChanged()` watcher. By setting up our own watcher for `list`, we can
automatically persist data to `localStorage` whenever `list` changes!

{% raw %}
    <polymer-element name="my-app">
    <template>
      <td-model items="{{list}}"></td-model>
      <polymer-localstorage2 name="myapplist" id="storage"></polymer-localstorage2>
    </template>
    <script>
      Polymer('my-app', {
        ready: function() {
          this.list = this.list || [];
        },
        listChanged: function() {
          this.$.storage.value = this.list;
        }
      });
    </script>
    </polymer-element>
{% endraw %}

When `list` changes, the chain reaction is set in motion:

1. {{site.project_title}} calls `<my-app>.listChanged()`
2. Inside `listChanged()`, `<polymer-localstorage2>.value` is set
3. This calls `<polymer-localstorage2>.valueChanged()`
4. `valueChanged()` calls `save()` which persists data to `localStorage`

**Tip:** I'm using a {{site.project_title}} feature called [automatic node finding](/polymer.html#automatic-node-finding) to reference `<polymer-localstorage>` by its `id` (e.g. `this.$.storage === this.querySelector('#storage')`).
{: .alert .alert-success }

### 3. Custom events {#events}

<table class="table">
  <tr>
    <th>Pros</th><th>Limitations</th>
  </tr>
  <tr>
    <td>
      <ol>
        <li>Works with elements inside and outside a {{site.project_title}} element</li>
        <li>Easy way to pass arbitrary data to other elements</li>
        <li>Inside a {{site.project_title}} element, using <code>on-*</code> declarative handlers reduces code</li>
        <li>Can use bubbling for internal event delegation in an element.</li>
      </ol>
    </td>
    <td>
      <ol>
        <li>Small amount of code hookup</li>
        <li>Requires some knowledge of an element's API</li>
      </ol>
    </td>
  </tr>
</table>

A third technique is to **emit custom events from within your element**. Other elements
can listen for said events and respond accordingly. {{site.project_title}} has two nice helpers for sending events, [fire() and asyncFire()](/polymer.html#fire). They're essentially wrappers around `node.dispatchEvent(new CustomEvent(...))`. Use the asynchronous version for when you need to fire an event after [microtasks](http://www.whatwg.org/specs/web-apps/current-work/#perform-a-microtask-checkpoint) have completed.

Let's walk through an example:

{% raw %}
    <polymer-element name="say-hello" attributes="name">
      <template>Hello {{name}}!</template>
      <script>
        Polymer('say-hello', {
          sayHi: function() {
            this.fire('said-hello');
          }
        });
      </script>
    </polymer-element>
{% endraw %}

Calling `sayHi()` fires an event named `'said-hello'`. And since **custom events bubble**,
a user of `<say-hello>` can setup a handler for the event:

    <say-hello name="Larry"></say-hello>
    <script>
      var sayHello = document.querySelector('say-hello');
      sayHello.addEventListener('said-hello', function(e) {
        ...
      });
      sayHello.sayHi();
    </script>

As with normal DOM events outside of {{site.project_title}}, you can attach additional data to a custom event. This makes **events an ideal way to distribute arbitrary information to other elements**.

**Example:** include the `name` property as part of the payload: 

    sayHi: function() {
      this.fire('said-hello', {name: this.name});
    }

And someone listening could use that information:

    sayHello.addEventListener('said-hello', function(e) {
      alert('Said hi to ' + e.detail.name + ' from ' + e.target.localName);
    });

#### Using declarative event mappings {#declartivemappings}

The {{site.project_title}}ic approach to events is combine event bubbling
and [`on-*` declarative event mapping](/polymer.html#declarative-event-mapping).
Combining the two gives you a declarative way to listen for events and requires very little code.

**Example:** Defining an `on-click` that calls `sayHi()` whenever the element is clicked:

{% raw %}
    <polymer-element name="say-hello" attributes="name" on-click="sayHi">
      <template>Hello {{name}}!</template>
      <script>
        Polymer('say-hello', {
          sayHi: function() {
            this.fire('said-hello', {name: this.name});
          }
        });
      </script>
    </polymer-element>
{% endraw %}

##### Without {{site.project_title}}'s sugaring

The same can be done by adding a click listener in the element's `ready` callback:

{% raw %}
    <polymer-element name="say-hello" attributes="name">
      <template>Hello {{name}}!</template>
      <script>
        Polymer('say-hello', {
          ready: function() {
            this.addEventListener('click', this.sayHi);
          },
          sayHi: function() {...}
        });
      </script>
    </polymer-element>
{% endraw %}

#### Utilizing event delegation

You can setup internal event delegation for your element by declaring an `on-*`
handler on the `<polymer-element>` definition. Use it to catch events that bubble
up from children.

Things become come very interesting when several elements need to respond to an event.

{% raw %}
    <polymer-element name="my-app" on-said-hello="third">
      <template>
        <div on-said-hello="second">
          <say-hello name="Eric" on-said-hello="first"></say-hello>
        </div>
      </template>
      <script>
        (function() {
          function logger(prefix, detail, sender) {
            alert(prefix + ' Said hi to ' + detail.name +
                  ' from ' + sender.localName);
          }

          Polymer('my-app', {
            first: function(e, detail, sender) {
              logger('first():', detail, sender);
            },
            second: function(e, detail, sender) { 
              logger('second():', detail, sender);
            },
            third: function(e, detail, sender) {
              logger('third():', detail, sender);
            }
          });
        })();
      </script>
    </polymer-element>

    <my-app></my-app>

    <script>
      var myApp = document.querySelector('my-app');
      myApp.addEventListener('said-hello', function(e) {
        alert('outside: Said hi to ' + e.detail.name + ' from ' + e.target.localName);
      });
    </script>
{% endraw %}

{% raw %}
<polymer-element name="say-hello" attributes="name" on-click="sayHi">
  <template>Hello {{name}}! (click me)</template>
  <script>
    Polymer('say-hello', {
      sayHi: function() {
        this.fire('said-hello', {name: this.name});
      }
    });
  </script>
</polymer-element>
    
<polymer-element name="my-app-demo" on-said-hello="third">
  <template>
    <style>
      @host { :scope {
        display: inline-block;
        padding: 10px;
        border: 1px dotted black;
        border-radius: 3px;
        cursor: pointer;
      }}
    </style>
    <div on-said-hello="second">
      <say-hello name="Eric" on-said-hello="first"></say-hello>
    </div>
  </template>
  <script>
    (function() {
      function logger(prefix, detail, sender) {
        alert(prefix + ' Said hi to ' + detail.name +' from ' + sender.localName);
      }

      Polymer('my-app-demo', {
        first: function(e, detail, sender) {
          logger('first():', detail, sender);
        },
        second: function(e, detail, sender) { 
          logger('second():', detail, sender);
        },
        third: function(e, detail, sender) {
          logger('third():', detail, sender);
        }
      });
    })();
  </script>
</polymer-element>
{% endraw %}

**Try it:** <my-app-demo></my-app-demo>

Clicking `<say-hello>` alerts the following (remember it defined
a click handler on itself):

    first(): Said hi to Eric from say-hello
    second(): Said hi to Eric from div
    outside: Said hi to Eric from my-app
    third(): Said hi to Eric from my-app

#### Sending messages to siblings/children

Say you wanted an event that bubbles up from one element to also fire on
sibling or child elements. That is:

    <polymer-element name="my-app" on-said-hello="sayHi">
      <template>
        <say-hello name="Bob"></say-hello>
        <say-bye></say-bye> <!-- Defines an internal listener for 'said-hello' -->
        ...

When `<say-hello>` fires `said-hello`, it bubbles and `sayHi()` handles it.
However, suppose `<say-bye>` has setup an internal listener for the same event.
It wants in on the action! Unfortunately, this means we can no longer exploit
the benefits of event bubbling...by itself. 

This particular problem isn't new to the web but you can easily handle it in
{{site.project_title}}. Just use event delegation and manually fire the event
on `<say-bye>`.

    Polymer('my-app', {
      sayHi: function(e, details, sender) {
        // Fire 'said-hello' on all <say-bye> in the element.
        [].forEach.call(this.querySelectorAll('say-bye'), function(el, i) {
          el.fire('said-hello', details);
        });
      }
    });

#### Using &lt;polymer-signals&gt;

`<polymer-signals>`is a [utility element](/docs/elements/#polymer-signals) that
makes the pubsub pattern a bit easier, It also **works outside of {{site.projed}}
elements**.

Your element fires `polymer-signal` and names the signal in its payload:

    this.fire('polymer-signal', {name: "foo", data: "Foo!"});

This event bubbles up to `document` where a handler constructs and dispatches
a new event, <code>polymer-signal<b>-foo</b></code>, to *all instances* of `<polymer-signals>`.
Parts of your app or other {{site.project_title}} elements can declare a `<polymer-signals>`
element to catch the named signal:

    <polymer-signals on-polymer-signal-foo="fooSignal"></polymer-signals>

Here's a full example:

    <polymer-element name="sender-element">
      <template>Hello</template>
      <script>
        Polymer('sender-element', {
          ready: function() {
            this.asyncFire('polymer-signal', {name: "foo", data: "Foo!"});
          }
        });
      </script>
    </polymer-element>

    <link rel="import" href="polymer-signals.html">

    <polymer-element name="my-app">
      <template>
        <polymer-signals on-polymer-signal-foo="fooSignal"></polymer-signals>
        <content></content>
      </template>
      <script>
        Polymer('my-app', {
          fooSignal: function(e, detail, sender) {
            this.innerHTML += '<br>[my-app] got a [' + detail + '] signal<br>';
          }
        });
      </script>
    </polymer-element>

    <!-- Note: polymer-signals works outside of {{site.project_title}}.
         Here, sender-element is outside of a {{site.project_title}} element. -->
    <sender-element></sender-element>
    <my-app></my-app>

<!-- {% raw %}
<polymer-element name="signal-sender-element">
  <template>Hello</template>
  <script>
    Polymer('signal-sender-element', {
      ready: function() {
        this.fire('polymer-signal', {name: "foo", data: "Foo!"});
      }
    });
  </script>
</polymer-element>

<link rel="import" href="/polymer-all/polymer-elements/polymer-signals/polymer-signals.html">

<polymer-element name="my-app-signals">
  <template>
    <polymer-signals on-polymer-signal-foo="fooSignal" on-polymer-signal-bar="barSignal"></polymer-signals>
    <content></content>
  </template>
  <script>
    Polymer('my-app-signals', {
      fooSignal: function(e, detail, sender) {
        this.innerHTML += '[my-app] got a [' + detail + '] signal<br>';
      }
    });
  </script>
</polymer-element>
{% endraw %}

<signal-sender-element></signal-sender-element>
**Demo**: <my-app-signals></my-app-signals>
 -->

### 4. Use an element's API {#api}

Lastly, don't forget you can always **orchestrate elements by using their public
methods (API)**. This may seem silly to mention but it's not immediately obvious
to most people.

**Example:** instruct `<polymer-localstorage>` to save its data by call
it's `save()` method (code outside a {{site.project_title}} element):

    <polymer-localstorage name="myname" id="storage"></polymer-localstorage>
    <script>
      var storage = document.querySelector('#storage');
      storage.useRaw = true;
      storage.value = 'data data data!';
      storage.save();
    </script>

## Conclusion

The unique "messaging" feature that {{site.project_title}} brings to the table two-way
data-binding and changed watchers. However, data binding has been a part of
other frameworks for a long time, so technically it's not a new concept.

Whether you're inside or outside a `<polymer-element>`, there are plenty of
ways to send instructions/messages/data to other web components. Hopefully,
you're seeing that nothing has changed in the world of custom elements. That's
the point :) It's the same web we've always known...just more powerful! 

<script>
  var myApp = document.querySelector('my-app-demo');
  myApp.addEventListener('said-hello', function(e) {
    alert('outside: Said hi to ' + e.detail.name + ' from ' + e.target.localName);
  });
</script>

{% include disqus.html %}
