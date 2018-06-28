---
layout: post
slug: f2018-06-25-backend-programmers-journey-into-react-and-redux
title: A Backend developers journer into React and Redux
date: 2018-06-25
---

In this post I describe my journer as a backend developer getting to grips with front end develoment using the React framework, using Redux for state management, and what I learned along the way. The outcome is some principles for doing front end development that lead to a good separation of concerns, and make React easier write and understand and more enjoyable to work with.

Working for a small startup, it happens at times that backend developers are forced to go full stack and get to grips with the frontend. Our front end projects are in React. I found this quite difficult to begin with, having to get up to date with JavaScript and learn quite a few new (for me) frameworks and paradigms. Conceptually it made quite good sense, and the ideas around a virtual dom are really appealing, but there are some key concepts and implementation details I struggled to work with. I wouldn't be surprised if just about every developer new to React has this experience. Among these are:
* The distinction between props and state
* Stateful class components and pure functional components.
* How to effectively separate out presentation and data components
* How to use with Redux and get the most out of it

The key epiphany, or rather discovery, was [higher order components](https://reactjs.org/docs/higher-order-components.html) (HOCs), and the recompose library, and this transformed my experience. Suddenly eveything just fell into place and some principles around a sensible way of working with React emerged. These principles are of course guided by my own opinion on aesthetics. As functional programming enthusiast, I favour immutable data structures. For that reason, I have a strong prefences for pure functional presentation components, and for the use of props over state. As for the data flow, I like to think of the redux store as the fundamental source of truth, projection of that truth through the filter of the user's navigational choices is what the user sees rendered as a view. User interaction and server data modify the single source of truth via state transition functions (action handlers in Redux terminology). The combination of using higher order components and using Redux in the right way allows the implementation to match this conceptual model.

For those unfamiliar with higher order components (HOCs) and recompose, the basic idea is that a HOC is a function that modifies a components. these modifications include but are not limited to:
* Adding to or modifying the props that a component receives.
* Adding state to a component by passing this state, as well as a function to modify it, as props.
* Adding handlers to handle component lifecycle events.
* Adding handlers to handle user events.
* Passing in data from the redux store to a component. In fact the usual redux `connect` function is itself a HOC.
* Controlling when a component will be rerendered.

Using HOCs, there is no such thing as data components. All react components are presentation components. A component that needs statefulness can be simply transformed into a component that receives this state. State is immediately turned into props, dispensing with the distinction between them.
As a result react class components are not necessary. All presentation components can be pure functional components of the form `props => <JSX>`. Of course there are exceptions - some stateful components may be helpful for certain use cases like animation. But they are by far the exception.

As an example consider the following simple example, taken from the [react tutorial - handling events](https://reactjs.org/docs/handling-events.html) page.

```javascript
class Toggle extends React.Component {
  constructor(props) {
    super(props);
    this.state = {isToggleOn: true};

    // This binding is necessary to make `this` work in the callback
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    this.setState(prevState => ({
      isToggleOn: !prevState.isToggleOn
    }));
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        {this.state.isToggleOn ? 'ON' : 'OFF'}
      </button>
    );
  }
}

ReactDOM.render(
  <Toggle />,
  document.getElementById('root')
);
```

The `Toggle` class represents a toggle button in a on or off state. This state is maintained in the class `state` property. It also handles the rendering, and defines event handlers within the class.

Here is the same example, refactored into HOC style:

```
const ToggleView = ( { isToggleOn, handleClick } ) => (
  <button onClick={handleClick}>{isToggleOn ? 'ON' : 'OFF'}</button> 
)

const Toggle = Recompose.compose(
  Recompose.withState( 'isToggleOn', 'setToggleOn', true ),
  Recompose.withHandlers( {
    handleClick: ( { isToggleOn, setToggleOn } ) => () => setToggleOn( !isToggleOn )
  } )
)(ToggleView)
 
ReactDOM.render(
  <Toggle />,
  document.getElementById('root')
);
```

Writing a component in HOC style is typically involves creating a view as a functions `( props ) => ( <JSX> )`, and a `compose` HOC with a sequence of HOCs that define and transform the props that are passed in to the view.

Apart from being more succinct, this code results in a better separation of concerns. The rendering code has been contained to a single stand-alone function, and the data and logic is all within the `compose` HOC. However perhaps the biggest benefit of this approach is that when encountering a `compose` function, we can typically reason about it in a very linear, sequential way. 

This makes it both very easy to write them, and also easy to understand them if they've been written by others. `compose` HOCs can be written by working backwards, where we writ the view component first, and then work backwards to fill in the props that the view requires, or by working forwards where we start with the input props (in the above case there are none) work forwards until we have assembled all the props the view will need.

At the implementation level, each line in the compose leads to a layer in the react component heirarchy. This is something to be aware of, but not something we need to pay too much attention to - I find it better to think of each line as a step in a sequence of prop transformations, in which we add data (e.g. via redux `connect`), transform data, or add state or event handlers.

For a slightly more involved example, consider and contrast the two following components (also taken from the [react tutorial](https://reactjs.org/docs/lifting-state-up.html) ), and the same example refactored into HOC style.

* [Origial react](https://codepen.io/gaearon/pen/WZpxpz?editors=0010)
* [HOC style](https://codepen.io/szoio/pen/ERdxrz?editors=0010)

The benefits of bering about to reason sequentially about your dynamic components is perhaps not totally evident from the toy examples given above, but it becomes drastically apparent for larger, more complex components, as the length of traversals your eyes have to scroll through scales linearly rather than quadratically with the complextiy of a component.

The examples given are using standard recompose API functions like `withState`, `withProps`, but you will soon start rolling your own HOCs to do increasingly complex things. And you'll find that these generic HOCs are much more reusable than other mechanisms for creating shared functionality.

### Redux and data flow

The use of redux promotes the concept of circular data flow. The use of redux within a react component has two modes of operation, and these are reflected in the `connect` function. The first is reading from the store, represented by the `mapStateToProps` parameter. The second is writing, i.e. dispatching actions that through action handlers, modify store state. This is done using the `mapDispatchToProps` parameter of the `connect` function.

Adding a `connect` function to a component introduces a dependency on the redux store, and this should be done sparingly. Rather than have `connect` dependencies sprayed through the component stack, my preference is to have a single dependency at the top level component (typically the screen component), and pass in data dependencies explicitly into child component as props. So data trickles down the component stack. Any component that needs to capture user events will probably also want to dispatch actions to the store. We make that dependency explicit by passing in the `dispatch` callback function as a prop.

Key principles:

1. Components, particularly presentation components, refer to their dependencies explicitly, unless there's a good reason why they shouldn't. Use of dependency injection should be kept to a minimum.
1. Presentation components are pure functions `props => <JSX>`, wherever possible.
1. We can avoid `state`, or rather localise it, by immediately transforming it into props via an HOC (e.g. `withState`).
1. The responsibility of getting state from the redux store (*read*) and dispatching to the redux store (*write*) is separated. Bundling them together means we are giving redux store write access to components that don't need them.
1. Only the top level component needs a `connect` function to read from the redux store. All it's descendents receive the redux store data they need as props, and don't need to connect to the store for this. Parents also should pass `dispatch` as props to a child if the child or any of its descendants need to publish actions to the redux store.
1. Presentation components do not need to do any manipulation / transformation of their props. They are given the props they need exactly as they need them.
1. Parent components are not responsible for these transformation either. These transformations are handled by HOCs.
1. Components ***can*** dispatch actions to the redux store directly (via event handlers). This is why the data flow is circular. They don't need to bubble the events up to the top level. 
1. Event handlers are injected into presentation components via higher order components. 
1. These event handler HOCs can live close to the presentation component unless it needs to be shared (e.g. if makes sense to define an event handler for a form near the form, maybe in the same source file, if nobody else needs it). Shared HOCs can be factored out.
1. Remove the references to React completely from all but presentation code. So everything else is just pure Javascript.

How to go about building in this way? I think we should take advantage of [HOCs](https://reactjs.org/docs/higher-order-components.html), and in particular use recompose (https://github.com/acdlite/recompose), that provides useful common abstractions, and a mechanism to compose them. 

We start with steps 1 - 3 in https://reactjs.org/docs/thinking-in-react.html

Step 1 - Break The UI Into A Component Hierarchy

Step 2 - Build A Static Version in React

Step 3 - Identify The Minimal (but complete) Representation Of UI State

Here is where we diverge, as the Redux store holds the application state:

Step 4 - Transform presentation only components into components with circular data flow.
* Transform properties as they come in into properties required by child component.
* Add the event handlers in a HOC that creates functions that dispatch to the redux store.
* Add lifecycle events.

With recompose, and other hocs like (https://github.com/deepsweet/hocs) for with-lifecycle, all the above logic can be expressed as pure functions.

So we end up with the following:
* A pure `props => <JSX>` rendering layer.
* All controllers for event handlers and state transformation are pure Javascript functions.
* All lifecycling is done as pure functions that simply receive React lifecycle events and transform them to props.

Because this is all done with only functions, anything can shared or unit tested as required. Dependency injection can be added at this point if necessary.

This can be done is a composable way, where the confusing component heirarchy that results from interleaving data and presentation components (as in some of our older developments) is simplified. 

All that remains is a single hierarchy of presentation components, where each is appropriately transformed through HOCs to give them the data and behaviour they need.

An example of compose steps might be:
1. Map input props to props required to generate event handlers / do lifecycling
2. Add event handlers with the `mapDispatchToProps` part of `connect`
3. Add lifecycling handlers
4. Add a final transformation step to transform / filter the properties to just those required by the presentation layer.

It's so ridiculously simple doing things this way that even I can do it.

So how does this fit in with current proposals?

Not really all that different, but has some elements of both! 

Like Hugo's proposed architecture is a view centric architecture. Presentation components directly reference each other. 


