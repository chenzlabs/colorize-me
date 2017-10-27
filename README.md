# `colorize-me`
# Or, An Example Using `networked-aframe` to Update Custom Components, and How I Debugged It 
# (A Dramatic Interpretation)

This brief example, 
which represents current best practice
for applying properties like `color`
to components within children of a networked entity, 
stems from [a recent conversation](https://aframevr.slack.com/archives/C65MC3NSF/p1508822525000083) in the [A-Frame Slack](https://aframevr.slack.com).

## Which One of You Is Which?

When authoring networked multiplayer experiences, one of the first questions asked is typically, "How do I distinguish which player is which?"  One common and easily understood solution is to assign each player a color, so that the player (and any associated objects) can be distinguished from others at a glance.

## Great, How Do I Do That With A-Frame?

Using A-Frame and its Entity-Component-System relationships, it is extremely common for the player representation to be comprised of multiple child entities that would subsequently need to share the same color as the player.  That color is generally applied by utilizing the `color` property of the `material` component.

## Fine, But How Do Others See It Too?  Enter `networked-aframe`

`networked-aframe` provides a framework to easily synchronize entities across networked clients, and also to define which components and properties should be synchronized.  Briefly summarizing, after declaring that the `a-scene` is networked by attaching the `networked-scene` component, each entity that should be synchronized is noted using the `networked` component, using a `template` value that defines its basic structure, and an optional associated schema that declares which components and properties of that structure should be synchronized across the network.

## Thanks for the Hammer.  Which Nail Should I Use?

But how best to synchronize across nested entities, such as a parent player rig entity and all of the children that comprise its visual appearance, is somewhat less obvious.

Conceptually, if each player is assigned a color, then that color relationship needs to be synchronized only once, rather than once for each child entity or component that may have that same color, and each client can take responsibility for applying that color to the player's representation(s) as appropriate.  This is also more efficient from a network standpoint, since it prevents redundant copies of identical information from requiring transmission and reception.

## Clever! ... So How Do I Do That?

A simple way to apply a property to an entity and its children is to use a component that holds the property value to apply, and to modify its update method to apply that property to itself and then each child, in recursive fashion.  That component code looks something like this, from the A-Frame Slack:

```
...
{
  schema: {color: {type: 'color', default: 'white'}},
  ...
  update: function () {
    this.setColor(this.el);
    this.updateChildren(this.el);
  },
  ...
  updateChildren: function (el) {
    for(var i = 0; i < el.children.length; i++){
      this.setColor(el.children[i]);
      if(el.children[i].children.length > 0){
        this.updateChildren(el.children[i]);
      }
    }  
  },
  ...
  setColor: function (el) {
    el.setAttribute('material', 'color', this.data.color);
  }
}
```

## Refactoring for Reusability

Assuming we will want to create other experiences besides this one, it is common practice to reuse code where reasonable, and to minimize assumptions and dependencies to maximize the number of situations that qualify as reuse opportunities.

Looking a little more closely at the above code, a few opportunities for improvement are apparent:

- `updateChildren` is specifically tied to `setColor` -- so if we wanted to add a text label later to apply where appropriate, we can't reuse it directly.

- `update` has to call `setColor` for its own element, and then separately call `updateChildren` to apply the same logic to each of its children.

And a slightly less obvious opportunity to simplify:

- `updateChildren` is an object method, but the neither the function to apply nor the recursion actually relies on the object, so we can make this a simple function instead.

So let's refactor with those opportunities in mind.
Going in reverse order, we can make a simple function that applies a given function to an element and its children:

```
  function applyToElementAndChildren (el, elFunction) {
    // Apply the function to the element.
    elFunction(el);

    // Recursively apply the function to the element's children.
    for(var i = 0; i < el.children.length; i++){
      applyToElementAndChildren(el.children[i], elFunction);
    }
  }
```

And then we can use that function to separate the invocation in `update` from what it actually does to each, with a new method `updateElement`.

```
  ...
  update: function () {
      applyToElementAndChildren(this.el, this.updateElement);
    },
    
    updateElement: function (el) {
      this.setColor(el);
    },

    setColor: function (el) {
      ...
```

## Nice! ... Wait, Why Doesn't That Work?

Nice trick, but as-is, that refactoring doesn't work.  Can you spot why?
`(cue Jeopardy music...)`
OK, time's up!

Remember how we turned `applyToElementAndChildren` into a simple function?
Looking more closely, `elFunction` is also supposed to be a simple function.
But we're passing in `this.updateElement` which is an object method!
`(facepalm)`

Armed with that realization, we can use the standard trick to turn an object method (that receives its object as `this`) into a simple function using `bind`.  Rather than doing it every time, let's do it once in the `init` method, which we need to add:

```
  ...
  init: function () {
    this.updateElement = this.updateElement.bind(this);
  },
  
  update: function () {
    ...
```

## Good Catch!  (grumble stupid refactoring grumble)  Wait, Something's Still Weird!

Now that we have correct code, we can add the code to set a random color for each player, then fire things up in a few browser windows to test the networked synchronization and ... rapidly become disappointed that it doesn't quite work as expected; something's still weird.

* NOTE: The above code snippets were a simpliflication for educational purposes.  The real example had installed asynchronous event handlers for `componentinitialized` and `child-attached` which complicated matters.

More specifically, sometimes the color doesn't get applied correctly -- maybe even intermittently, the worst kind of bug -- and getting the color right cross the network is never working at all (but at least the behavior is consistent).

What to do? *

* NOTE: This is a dramatic re-imagination of the original scenario, no actual hairs were pulled out during the real-life event.  Or at (least by me.)

## When In Doubt, Log It Out

Especially for intermittent problems that aren't always reproducible, it's very important to provide some logging and/or telemetry to see what is actually happening, and hopefully in what order.  (The latter can get quite difficult in networked scenarios, but that's a tale for a different time.)

The smoking gun came from logging the update color changes and seeing a completely unexpected sequence from setting the player color to red:
```
Updated: red
Updated: white
```

What do you mean `white` -- is there a ghost in the machine?

* NOTE: At this point in the real-world exercise, hairs were precariously close to being forcibly extracted from their natural habitat.

## The Cold Harsh Reality of the Facepalm

Quickly searching the code for where the `white` value was coming from:

```
  schema: {color: {type: 'color', default: 'white'}},
```

Since `networked-aframe` was instantiating the local player template with `colorize` attached, it was sending an initial sync message with the default value.  Which is `white`.

But we were setting it to `red` before that sync message was received and replied -- and then we'd get the news from with the initial default, and faithfully apply it, thereby changing from our lovely `red` to a cold harsh `white`.

Facepalm.

## Wrapping Up, If You're Still Reading

There are at least three methods of solution, with accompanying levels of effort and risk:

1. Don't use `white` as a default; instead, use something that indicates no colorization should be performed, and wait for a real value to arrive.  This is a self-contained fix, but doesn't prevent similar issues if we forget to do this next time.

2. Don't put `colorize` on the template, so there hopefully won't be an initial value that only shows up later.  There is a little bit of risk here because `networked-aframe` requires that it be in the schema for synchronization, and so it might send over some initial value even if nothing is there -- and more importantly, that behavior could change between versions of `networked-aframe`, whether intentional or unintentional.

3. Try and deal with the out-of-order execution problem.  There is a significant amount of level of effort and risk here, with some reasons including: 
- variability: variations in network bandwidth and latency between clients
- inconsistency: variations in network bandwidth and latency at any given point in time
- complexity: as an example, one might naively choose timestamping as a method, without realizing that networked clients never exactly agree on what time it really is... computer science has devised solutions to this particular problem, but they are nowhere near as isolated and trival as either 1 or 2 above.

Homework: which fixes does the Glitch example use, and will they work for you?

Anyway, thanks for taking the time to get this far!  I hope you enjoyed reading this, and I hope that you find it helpful as you embark on your own creative coding adventures.

Special thanks also to: 
- @dmarcos and @ngokevin, for making this lovely A-Frame framework
- @haydenlee, for making `networked-aframe`
- @michael.andrew, for bringing it up and leading me on this adventure
- the Glitch creators and maintainers, without which this coding would be much less immediately gratifying

Best
M
