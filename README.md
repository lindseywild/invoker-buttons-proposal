# Invoker Buttons

## Summary

Adding `invokertarget` and `invokeraction` attributes to `<button>` and
`<input type="button">` / `<input type="reset">` elements would allow authors to
assign behaviour to buttons in a more accessible and declarative way, while
reducing bugs and simplifying the amount of JavaScript pages are required to
ship for interactivity. Buttons with `invokertarget` will - when clicked,
touched, or enacted via keypress - dispatch an `InvokeEvent` on the element
referenced by `invokertarget`, with some default behaviours.

In addition, adding an `interesttarget` attribute to `<button>`, `<a>`,
`<input>` elements would allow disclosure of high fidelity tooltips in a more
accessible and declaritive way. Elements with `interesttarget` will - when
hovered, long pressed, or focussed - dispatch an `InterestEvent` on the element
referenced by `interesttarget`, with some default behaviours.

## Background

All elements within the DOM are capable of having interactions added to them. A
long while ago this took the form of adding inline JavaScript to an event
attribute, such as `<button onclick="other.open()"></button>`. Inline JavaScript
has (rightly so) fallen out of favour due to the security and maintainability
concerns. Newer pages may instead introduce _more_ JavaScript to imperatively
discover elements and call `addEventListener('click', ...)` to invoke the same
behaviour. These patterns reduce developer experience and introduce more
boilerplate and friction, while remediating security and maintainability
concerns. Some frameworks attempt to reintroduce the developer experience of
inline handlers by introducing new JavaScript or HTML shorthands, such as
React's `onclick={...}`, Vue's `@click=".."` or HTMX's `hx-trigger="click"`.

There has also been a rise in the desire to customise controls for components.
Many sites, for example, introduce custom controls for File Uploads or dropdown
menus. These often require a large amount of work to _reintroduce_ the built in
functionality of those controls, and often unintentionally sacrifice
accessibility in doing so.

With the new `popover` attribute, we saw a straightforward declarative way to
tell the DOM that a button was interested in being a participant of the popover
interaction. `popovertarget` would indicate to a browser that if that button was
interacted with, it should open the element the `popovertarget` value pointed
to. This allows for popovers to be created and interacted with - in an
accessible and reliable way - without writing any additional JavaScript, which
is a very welcome addition. While `popovertarget` sufficiently captured the use
case for popovers, it fell short of providing the same developer & user
experience for other interactive elements such as `<dialog>`, `<details>`,
`<video>`, `<input type="file">`, and so on. This proposal attempts to redress
the balance.

### Terms

- Invoke/Invoked/Invoking: The action of _Invoking_ refers to a complete press
  action of a button, using a Human Input Device (HID). If the HID is a mouse,
  this would be a `click` event. If the HID is a touch screen, this would be a
  press using a stylus or finger. If the HID is a keyboard this would be the
  `Space` or `Enter` (Carriage Return) key on the keyboard. For other HIDs such
  as eye tracking or pedals or game controllers, the equivalent expected "click"
  action should be used to invoke the element.
- Interest/Shows Interest: The action of _Interest_ refers to the user "landing"
  on an element without invoking it, using a HID. If the HID is a mouse, this
  would be a `hover` event. If the HID is a touch screen, this would be a long
  press on the element using a stylus or finger, if the HID is a keyboard, this
  would be when the element is focussed. For other HIDs such as eye tracking or
  pedals or game controllers, the equivalent "focus or hover" action is used to
  take interest on the element.
- Loses/Lost Interest: The action of _Loses Interest_ refers to the user "moving
  away" from an element, or its _interstee_, using a HID; in other words
  interest must be on a different element that is neither. Elements can only
  _Lose Interest_ if they are in the state of Showing Interest. If the HID is a
  mouse, this would be a `mouseout` event. If the HID is a touch screen, this
  would be long pressing outside of the elements bounds. If the HID is a
  keyboard, this would be moving focus away from the element. For other HIDs
  such as eye tracking or game controllers, the equivalent "focusout or
  mouseout" action is used to _Lose Interest_ on the element.
- Invoker: An invoker is a button element (that is a `<button>`,
  `<input type="button>`, or `<input type="reset">`) that has an `invokertarget`
  attribute set.
- Invokee: An element which is referenced to by an Invoker, via the
  `invokertarget` attribute.
- Interestee: An element which is referenced to by an Interest element, via the
  `interesttarget` attribute.

## Proposed Plan

In the style of `popovertarget`, this document proposes we add `invokertarget`,
and `invokeraction` as available attributes to `<button>`,
`<input type="button">` and `<input type="reset">` elements, as well as an
`interesttarget` attribute to `<button>`, `<a>`, and `<input>` elements.

```webidl
interface mixin InvokerElement {
  [CEReactions] attribute Element? invokerTargetElement;
  [CEReactions] attribute DOMString invokerAction;
};
interface mixing InterestElement {
  [CEReactions] attribute Element? interestTargetElement;
}
```

The `invokertarget` value should be an IDREF pointing to an element within the
document. `.invokerTargetElement` also exists on the element to imperatively
assign a node to be the invoker target, allowing for cross-root invokers (in
some cases, see
[the popovertarget attr-asociated element steps for
more](https://html.spec.whatwg.org/multipage/common-dom-interfaces.html#attr-associated-element).

The `invokeraction` (and the `.invokerAction` reflected property) is a freeform
hint to the Invokee. If `invokeraction` is a falsey value (`''`, `null`, etc.)
then it will default to `'auto'`. Values which are not recognised should be
passed verbatim, and should not be assumed to be `auto`. This allows for custom
actions. Built-in interactive elements have built-in behaviours (detailed below)
which are determined by the `invokeraction` but also Invokees will dispatch
events when Invoked, allowing custom code to take control of invocations without
having to manually wire up DOM nodes for the variety of invocation patterns.

The `interesttarget` value should be an IDREF pointing to an element within the
document. `.interestTargetElement` also exists on the element to imperatively
assign a node to be the invoker target, allowing for cross-root invokers (in
some cases, see
[the popovertarget attr-asociated element steps for
more](https://html.spec.whatwg.org/multipage/common-dom-interfaces.html#attr-associated-element).

Elements with `invokertarget` set will dispatch an `InvokeEvent` on the
_Invokee_ (the element referenced by `invokertarget`) when the element is
_Invoked_. The `InvokeEvent`'s `type` is always `invoke`. The event also
contains a `relatedTarget` property that will reference the _Invoker_ element.
`InvokeEvents` are always non-bubbling, cancellable events.

```webidl
[Exposed=Window]
interface InvokeEvent : Event {
  constructor(InvokeEventInit invokeEventInit);
  readonly attribute Element relatedTarget;
  readonly attribute DOMString type = "invoke";
  readonly attribute DOMString action;
};
dictionary InvokeEventInit : EventInit {
  DOMString action = "auto";
  Element relatedTarget;
};
```

Elements with `interesttarget` set will dispatch an `InterestEvent` on the
_Interestee_ (the element referenced by `interesttarget`) when the element
_Shows Interest_ or _Loses Interest_. When the element _Shows Interest_ the
event type will be `'interest'`. If the element has _Shown Interest_, and
interest moves away from both the _Interest Element_ and the _Interestee_, then
the element _Loses Interest_ and an `InterestEvent` with the type of
`'loseinterest'` will be dispatched on the _Interestee_. The event also contains
a `relatedTarget` property that will reference the _Interest_ element.
`InterestEvents` are always non-bubbling, cancellable events.

```webidl
[Exposed=Window]
interface InterestEvent : Event {
  constructor(DOMString type, InterestEventInit interestEventInit);
  readonly attribute Element relatedTarget;
};
dictionary InterestEventInit : EventInit {
  Element relatedTarget;
};
```

Both `interesttarget` and `invokertarget` can exist on the same element at the
same time, and both should be respected.

If an element also has both a `popovertarget` and `invokertarget` attribute,
then `popovertarget` _must_ be ignored: `invokertarget` takes precedence. An
element with both `interesttarget` and `popovertarget` is valid and both actions
will work.

If an `<button>` is a form participant, or has `type=submit`, then
`invokertarget` _must_ be ignored. `interesttarget` is still valid in these
scenarios.

If an `<input>` is a form participant, or has a `type` other than `reset` or
`button`, then `invokertarget` _must_ be ignored. `interesttarget` is still
valid in these scenarios.

### Example Code

#### Popovers

When pointing to a `popover`, `invokertarget` acts like much like
`popovertarget`, allowing the toggling of popovers.

```html
<button invokertarget="my-popover">Open Popover</button>
<!-- Effectively the same as popovertarget="my-popover" -->

<div id="my-popover" popover="auto">Hello world</div>
```

An `interesttarget` allows for tooltips using `popover`:

```html
<button interesttarget="my-popover">Open Popover</button>

<div id="my-popover" popover="hint">Hello world</div>
```

#### Dialogs

When pointing to a `<dialog>`, `invokertarget` can toggle a `<dialog>`'s
openness.

```html
<button invokertarget="my-dialog">Open Dialog</button>

<dialog id="my-dialog">
  Hello world!

  <button invokertarget="my-dialog" invokeraction="close">Close</button>
</dialog>
```

#### Details

When pointing to a `<details>`, `invokertarget` can toggle a `<details>`'
openness.

```html
<button invokertarget="my-details">Open Details</button>
<!-- Can be used to replicate the `<summary>` interaction -->

<details id="my-details">
  <summary>Summary...</summary>
  Hello world!
</details>
```

#### Customizing `input type=file`

Pointing an `invokertarget` to an `<input type=file>` acts the same as the
rendered button _within_ the input; and can be used to declare a customised
alternative button to the input's button.

```html
<button invokertarget="my-file">Pick a file...</button>

<input id="my-file" type="file" />
```

#### Customizing video/audio controls

The `<video>` and `<audio>` tags have many interactions, here `invokeraction`
shines, allowing multiple buttons to handle different interactions with the
video player.

```html
<button invokertarget="my-video">Play/Pause</button>
<button invokertarget="my-video" invokeraction="mute">Mute</button>

<video id="my-video"></video>
```

#### Custom behaviour

_Invokers_ will dispatch events on the _Invokee_ element, allowing for custom
JavaScript to be triggered without having to wire up manual event handlers to
the Invokers.

```html
<button invokertarget="my-custom">Invoke a div... to do something?</button>
<button invokertarget="my-custom" invokeraction="frobulate">Frobulate</button>

<div id="my-custom"></div>

<script>
  document.getElementById("my-custom").addEventListener("invoke", (e) => {
    if (e.action === "auto") {
      console.log("invoked an auto action!");
    } else if (e.action === "frobulate") {
      alert("Successfully frobulated the div");
    }
  });
</script>
```

Elements with _Interest_ will dispatch events on the _Interestee_ element,
allowing for custom JavaScript to be triggered without having to wire up manual
event handlers to the Interest elements.

```html
<button interesttarget="my-custom">
  When this button shows interest, the below div will display.
</button>

<div id="my-custom" hidden>Supplementary information</div>

<script>
  const custom = document.getElementById("my-custom");
  custom.addEventListener("interest", (e) => {
    custom.hidden = false;
  });
  custom.addEventListener("loseinterest", (e) => {
    custom.hidden = true;
  });
</script>
```

### Accessibility

> [!WARNING]\
> This section is incomplete PRs welcome.

The _Invoker_ implicitly receives `aria-controls=IDREF` or `aria-details=IDREF`
(tbd) to expose the _Invoker_ controls another element (the _Invokee_) for
instances where the invokee is not a sibling to the invoker element.

If the _Invokee_ has the `popover` attribute, the _Invoker_ implicitly receives
an `aria-expanded` state, as well as an `aria-details` association (for
instances where the elements are not DOM siblings) which will match the state of
the popover's openness. It will be `aria-expanded=true` when the `popover` is
`:popover-open` and `aria-expanded=false` otherwise.

If the _Invokee_ is a `<details>` element the _Invoker_ implicitly receives an
`aria-expanded` state which will match the state of the _Invokee_'s openness. It
will be `aria-expanded=true` when the _Invokee_ is open and
`aria-expanded=false` otherwise.

If the _Invokee_ is a `<dialog>` element the _Invoker_ implicitly receives an
`aria-expanded` state which will match the state of the _Invokee_'s openness. It
will be `aria-expanded=true` when the _Invokee_ is open and
`aria-expanded=false` otherwise.

TBD: Accessibility attributes for `interesttarget`.

### Defaults

The `InvokeEvent` has a default behaviour depending on the element. Non-trusted
events are ignored, but can be useful for implementers. Trusted events do the
following. Note that this list is ordered and higher rules take precedence:

| Invokee Element       | `action` hint   | Behaviour                                                                            |
| :-------------------- | :-------------- | :----------------------------------------------------------------------------------- |
| `<* popover>`         | `'auto'`        | Call `.togglePopover()` on the invokee                                               |
| `<* popover>`         | `'hidePopover'` | Call `.hidePopover()` on the invokee                                                 |
| `<* popover>`         | `'showPopover'` | Call `.showPopover()` on the invokee                                                 |
| `<dialog>`            | `'auto'`        | If the `<dialog>` is not `open`, call `showModal()`, otherwise cancel the dialog     |
| `<dialog>`            | `'showModal'`   | If the `<dialog>` is not `open`, call `showModal()`                                  |
| `<dialog>`            | `'close'`       | If the `<dialog>` is `open`, cancel the dialog                                       |
| `<details>`           | `'auto'`        | If the `<details>` is `open`, then close it, otherwise open it                       |
| `<details>`           | `'open'`        | If the `<details>` is not `open`, then open it                                       |
| `<details>`           | `'close'`       | If the `<details>` is `open`, then close it                                          |
| `<input type="file">` | `'auto'`        | Open the OS file picker, in other words act as if the input itself had been clicked  |
| `<video>`             | `'auto'`        | Toggle the `.playing` value                                                          |
| `<video>`             | `'pause'`       | If `.playing` is `true`, set it to `false`                                           |
| `<video>`             | `'play'`        | If `.playing` is `false`, set it to `true`                                           |
| `<video>`             | `'mute'`        | Toggle the `.muted` value                                                            |
| `<video>`             | `'fullscreen'`  | Request the element to enter into 'fullscreen' mode                                  |
| `<audio>`             | `'auto'`        | Toggle the `.playing` value                                                          |
| `<audio>`             | `'pause'`       | If `.playing` is `true`, set it to `false`                                           |
| `<audio>`             | `'play'`        | If `.playing` is `false`, set it to `true`                                           |
| `<audio>`             | `'mute'`        | Toggle the `.muted` value                                                            |
| `<canvas>`            | `'clear'`       | Remove all image data on the canvas (effectively (`.clearRect(0, 0, width, height)`) |

> [!NOTE] The above table is an attempt at wide coverage, but ideas are welcome.
> Please submit a PR if you have one!

The `InterestEvent` has a default behaviour depending on the element.
Non-trusted events are ignored, but can be useful for implementers. Trusted
events do the following. Note that this list is ordered and higher rules take
precedence:

| Interestee Element | Event Type       | Behaviour                            |
| :----------------- | :--------------- | :----------------------------------- |
| `<* popover=hint>` | `'interest'`     | Call `.showPopover()` on the invokee |
| `<* popover=hint>` | `'loseinterest'` | Call `.hidePopover()` on the invokee |

> [!NOTE] The above table is an attempt at wide coverage, but ideas are welcome.
> Please submit a PR if you have one!

### Invoke/Interest and Custom Elements

As the underlying system for invoke/interest elements uses event dispatch,
Custom Elements can make use of `InvokeEvent`/`InterestEvent`s for their own
behaviours. Consider the following:

```html
<button
  interesttarget="my-element"
  invokertarget="my-element"
  invokeraction="spin"
>
  Spin the widget
</button>

<spin-widget id="my-element"></spin-widget>
<script>
  customElements.define(
    "spin-widget",
    class extends HTMLElement {
      connectedCallback() {
        this.addEventListener("invoke", (e) => {
          if (e.action === "spin") {
            this.spin();
          }
        });
        this.addEventListener("interest", (e) => {
          this.style.transform = "rotate(1deg)";
        });
        this.addEventListener("loseinterest", (e) => {
          this.style.transform = "rotate(0)";
        });
      }
    },
  );
</script>
```

### PAQ (Potentially Asked Questions)

#### Why the name `invoker`? Why not `click`?

While `click` is a fairly well established name in the world of the web, it is
quite specific to certain types of HID and is not a term which encompasses all
viable methods of interaction. In addition a `clickaction` attribute is deemed
to be a little too ambiguous as it conflates existing concepts. Given the
opportunity to supply a new name, `invoke` was settled on.

#### Why the name `interest`? Why not `hover` or `focus`?

Much like `click`, `hover` or `focus` are specific to certain types of HID, and
are not terms which encompass all viable methods of interaction. Lots of
[alternatives were discussed](https://github.com/openui/open-ui/issues/767) and
it was deemed that `interest` is the best name to explain the concept of a
"hover or focus or equivalent".

#### What about adding defaults for `<form>`?

Defaults for `<form>` are intentionally omitted as this proposal does not aim to
replace Reset or Submit buttons. If you want to control forms, use those.

#### What about adding defaults for `<a>`?

Defaults for `<a>` are intentionally omitted as this proposal does not aim to
replace anchors. If you intend to produce a page navigation, use an `<a>` tag.

### Why is `invokertarget` limited to buttons?

This is by design, to allow for a "pit of success"; invoking actions on
non-button elements such as `<div>`s or `<a>`s creates many problems, especially
for non-interactive elements. While `<a>`s _are_ interactive, they should _only_
be used for page navigation and not for invoking other behaviours, and so
`invokertarget` should not be allowed.

#### Why isn't `input[type=submit]` included?

This is not added by design. Submit inputs already have a default action:
submitting forms. If you want a button to control the submission of a form, use
`input[type=submit]`, if you want a button to control invocation of something
_other_ than a form then you should use `input[type=button]`.

#### Why _is_ `input[type=reset]` included?

It may stand to reason that if `input[type=submit]` is _excluded_ then so should
`input[type=reset]`, however, there are valid use cases to resetting a form at
the same time as some other action, for example closing the dialog that contains
a form:

```html
<dialog id="my-dialog">
  <form>
    <input type="text" />
    <!-- This button closes the dialog _and_ resets the form -->
    <input type="reset" invokertarget="my-dialog" value="Cancel" />
  </form>
</dialog>
```

#### Why is `interesttarget` less limited?

While _invocation_ should only be limited to buttons, disclosure of
supplementary information can be expanded to _all_ interactive elements. There
are many useful use cases for offering a tooltip on anchors, such as signalling
that they are external, or that they will open in a new window, or to show
preview information (think: preview windows on iOS Safari or the hovercards that
display on GitHub over a user's handle).

#### Why is `interesttarget` not unlimited, like `title` is?

It could be considered a mistake to allow `title` on all elements; as adding
interactivity to non-interactive elements creates many problems. Limiting where
`interesttarget` is allowed aims to create a "pit of success", guiding
developers to use it only on interactive elements, where it makes sense.

#### What does this mean for `popovertarget`?

Whilst `invokertarget` _does_ replicate `popovertarget`'s functionality, it does
not necessarily mean `popovertarget` gets removed from the spec.

#### InvokerTarget seems limited, what if I wanted to add arguments?

`invokeraction` is a freeform text hint to your own elements. If you feel it
necessary you can invent your own DSLs for passing extra data using this hint.
For example:

```html
<button invokertarget="my-counter" invokeraction="add:1">Add 1</button>
<button invokertarget="my-counter" invokeraction="add:2">Add 2</button>
<button invokertarget="my-counter" invokeraction="add:10">Add 10</button>

<input readonly id="my-counter" value="0" />

<script>
  const counter = document.getElementById("my-counter");
  counter.addEventListener("invoke", (e) => {
    let addMatch = /^add:(\d+)$/.match(e.action);
    if (addMatch) {
      counter.value = Number(counter.value) + Number(addMatch[1]);
    }
  });
</script>
```
