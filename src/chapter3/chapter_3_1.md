# React and state management

For this application we will be using react and redux for state management on the front end side of
things at least in the beginning.  Once we start getting into tensorflow, we will start using rust
data structures to capture some of the data, however the state of the application itself will be
handled with redux.

## Why react

One could ask why we are using react here.  There are other popular and not so popular front end
frameworks out there including but not limited to angular, vue, 

## Why redux

## Hooking a component into redux

- Add a new Action Creator with some new state to pass along
- Add a reducer that that does something with the action type and new state
- Add the new reducer to the combineReducers function
- Add a mapPropsToState on component and only pass the data needed (that the reducer takes)
- Add a mapDispatchToState if needed
  - Used if component needs to change state that another component needs to react to