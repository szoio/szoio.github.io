---
layout: post
slug: react-and-higher-order-components
title: React, redux and higher order components
date: 2018-06-25
---

In this post I discuss my journey as a backend developer getting to grips with front end develoment using the React framework, using Redux for state management, and [higher order components](https://reactjs.org/docs/higher-order-components.html). The outcome is some principles for doing front end development that lead to a good separation of concerns, and make React easier write and understand and more enjoyable to work with.

Working for a small startup, it happens at times that backend developers are forced to go full stack and get to grips with the frontend. Our front end projects are in React. I found this quite difficult to begin with, having to get up to date with JavaScript and learn quite a few new (for me) frameworks and paradigms. Conceptually it made quite good sense, and the ideas around a virtual dom are really appealing, but there are some key concepts and implementation details I struggled to work with. I wouldn't be surprised if just about every developer new to React has this experience. Among these are:
* The distinction between props and state
* Stateful class components and pure functional components
* How to effectively separate out presentation and data-only components
* How to use with Redux and get the most out of it

For me the distinction between props and state is a messy abstraction, full of conceptual clutter, and the cause of [endless cumulative confusion](https://github.com/uberVU/react-guide/blob/master/props-vs-state.md). What if we could just do away with this so called "state" and have just props? 

The key epiphany, or rather discovery, was [higher order components](https://reactjs.org/docs/higher-order-components.html) (HOCs), and the [recompose](https://github.com/acdlite/recompose) library, and this transformed my experience. Suddenly eveything fell into place and some principles around a sensible way of working with React emerged. These principles are of course guided by my own opinion on aesthetics. As functional programming enthusiast, I favour immutable data structures; I have a strong prefences for pure functional presentation components, and for the use of props over state. As for the data flow, I like to think of the redux store as the fundamental source of truth, projection of that truth through the filter of the user's navigational choices is what the user sees rendered as a view. User interaction and backend data changes modify the single source of truth via state transition functions (action handlers in Redux terminology). The combination of using higher order components and using Redux in the right way allows the implementation to match this conceptual model of view as a function of state.

For those unfamiliar with [higher order components](https://reactjs.org/docs/higher-order-components.html) (HOCs) and recompose, the basic idea is that a HOC is a function that modifies a React component. These modifications include but are not limited to:
* Adding to or modifying the props that a component receives.
* Adding state to a component by passing in this state, as well as a function to modify it, as props.
* Adding handlers to handle component lifecycle events.
* Adding user event handlers.
* Passing in data from the redux store to a component. In fact the usual redux `connect` function is itself a HOC.
* Controlling when a component will be rerendered.

Using HOCs, there is no need for data-only components. All react components can be presentation components, and the work otherwise done by data-only components is performed by a HOC. State goes away too. A component that needs statefulness can be simply transformed into a component that receives this state. State is immediately turned into props, dispensing with the distinction between them.
As a result react class components are not necessary. All react components can and should be pure functional components of the form `props => <JSX>` (i.e. [dumb as f*ck components](https://github.com/uberVU/react-guide/blob/master/props-vs-state.md#component-types)). Of course there are some exceptions - some stateful components may be helpful for certain use cases like animation. But they are by far the exception.

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

```javascript
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

This makes it both very easy to write them, and also easy to understand them if they've been written by others. `compose` HOCs can be written by working backwards, where we write the view component first, and then work backwards to fill in the props that the view requires, or by working forwards where we start with the input props (in the above case there are none) work forwards until we have assembled all the props the view will need.

At the implementation level, each line in the compose leads to a layer in the react component heirarchy, by creating a new component with the new props or behaviours that wraps the input component as a child. This is something to be aware of for efficiency reasons, but not something we generally need to pay too much attention to - I prefer to think of each line as a step in a sequence of prop transformations, in which we add data (e.g. via redux `connect`), transform data, or add state or event handlers.

For a slightly more involved example, consider and contrast the two following components (also taken from the [react tutorial](https://reactjs.org/docs/lifting-state-up.html) ), and the same example refactored into HOC style.

* [Origial react](https://codepen.io/gaearon/pen/WZpxpz?editors=0010)
* [HOC style](https://codepen.io/szoio/pen/ERdxrz?editors=0010)

The benefits of bering about to reason sequentially about your dynamic components is perhaps not totally evident from the toy examples given above, but it becomes drastically apparent for larger, more complex components, as the length of traversals your eyes have to scroll through scales linearly rather than quadratically with the complextiy of a component.

The examples given are using standard recompose API functions like `withState`, `withProps`, but you will soon start rolling your own HOCs to do increasingly complex things. And you'll find that these generic HOCs are much more reusable than other mechanisms for creating shared functionality.

### Redux and data flow

The use of redux promotes the concept of circular data flow. The use of redux within a react component has two modes of operation, and these are reflected in the `connect` function. The first is reading from the store, represented by the `mapStateToProps` parameter. The second is writing, i.e. dispatching actions that through action handlers, modify store state. This is done using the `mapDispatchToProps` parameter of the `connect` function.

Adding a `connect` function to a component introduces a dependency on the redux store, and this should be done sparingly. Rather than have `connect` dependencies sprayed through the component stack, my preference is to have a single dependency at the top level component (typically the screen component), and pass in data dependencies explicitly into child component as props. So data trickles down the component stack. Any component that needs to capture user events will probably also want to dispatch actions to the store. We make that dependency explicit by passing in the `dispatch` callback function as a prop.

### Key principles

Summary of the key ideas of React development in a functional way, with Redux store state representing the primary source of truth, with view a function of state, and with circular data flow in the application.

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

How to start building React components in this way? Here is one way of going about it. 

We start with steps 1 - 3 in https://reactjs.org/docs/thinking-in-react.html

Step 1 - Break The UI Into A Component Hierarchy

Step 2 - Build A Static Version in React

This can be done by creating a functional component that takes exactly the props it needs and renders then in JSX, and wrapping it with a stub `withProps` function that pushes in a static version of these props.

Step 3 - Identify The Minimal (but complete) Representation Of UI State

Step 4 - Implement the higher order component to add data and behaviours

Working this way, we end up with a view centric architecture, as the component hierarchy defined in Step 2 follows from the UI itself.

HOCs are pitched as an advanced topic, but beginners should not be put off - it's the easiest way to do React for noobs like me. Take the trouble to get to grips with it from the start and save yourself the trouble of having to grapple with the same issues I did.

In a following blog post I may drill deeper into the Redux side of things, and discuss some ideas for how to get the most out of it, and in particular how to get flicker-free UIs that support streaming data updates.
