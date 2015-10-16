To Do List Application
======================

This tutorial was based off the one found [here](http://survivejs.com/webpack_react/webpack_and_react/)

In part one, we created a hello world application using webpack and javascript. Now we will alter this project to allow us to create a react application using Redux.

Webpack With React
------------------

First we will need to install *babel-loader* to be able to tell Webpack we want to build jsx files.

```bash
npm i babel-loader --save-dev
```

Now inside the **webpack.config.js** file we need to tell our build system to include .jsx files, and we also need to tell it to use the babel loader.

#### webpack.config.js

```javascript
var path = require('path');
var HtmlwebpackPlugin = require('html-webpack-plugin');
var webpack = require('webpack');
var merge = require('webpack-merge');

var TARGET = process.env.npm_lifecycle_event;
var ROOT_PATH = path.resolve(__dirname);
var APP_PATH = path.resolve(ROOT_PATH, 'app');
var BUILD_PATH = path.resolve(ROOT_PATH, 'build');

process.env.BABEL_ENV = TARGET;

var common = {
  entry: APP_PATH,
  resolve: {
    extensions: ['', '.js', '.jsx']
  },
  output: {
    path: BUILD_PATH,
    filename: 'bundle.js'
  },
  module: {
    loaders: [
      {
        test: /\.jsx?$/,
        loaders: ['babel'],
        include: APP_PATH
      }
    ]
  },
  plugins: [
    new HtmlwebpackPlugin({
      title: 'To Do List'
    })
  ]
};

if(TARGET === 'start' || !TARGET) {
  module.exports = merge(common, {
    devtool: 'eval-source-map',
    devServer: {
      historyApiFallback: true,
      hot: true,
      inline: true,
      progress: true
    },
    plugins: [
      new webpack.HotModuleReplacementPlugin()
    ]
  });
}
```

Now that we've added the babel loader and included .jsx files in our build, we can start writing react components. But first we need to install react:

```bash
npm i react --save
```

> If you're wondering why we've used both --save and --save-dev: take a look at the package.json file. This will help show the difference. You'll notice that now 'react' is in the "dependencies" block. This is because it is good practice to separate application and development level dependencies.

Since we've already told Webpack that we want our "entry" (in the *webpack.config.js* file) to be the "app" directory in our project, we will have to re-write the source files there.

delete the **index.js** and **component.js** files in the "app" directory.

We will create a few new files so our app folder will look like this:

```
-app/
  -components/
    -app.jsx
    -list.jsx
  -index.jsx
```

We want to have the minimum amount of files for Webpack to build at our "entry" point, otherwise it could affect performance.

We will be using ES6 as the flux framework we will use is only available with ES6.

Now let's create our .jsx files to show how we can link them together with ES6:

#### todoform.jsx

```javascript
import React from 'react';

export default class TodoForm extends React.Component {

  render() {
    return (
      <div>
        <input type="text" />
        <button>Add Item</button>
      </div>
    );
  }
}
```

This **todoForm.jsx** file imports React and allows the TodoForm class to be imported by other files. This is how ES6 'requires' (We are used to seeing `var React = requires('react');`), and exports functions (instead of `module.exports = List;`\)

#### app.jsx

```javascript
import React from 'react';
import TodoForm from './todoform.jsx'

export default class App extends React.Component {
  render() {
    return (
      <h1>To Do List</h1>
      <TodoForm />
    );
  }
}
```

Nothing new here in **app.jsx** other than importing the file we created previously.

Now we'll write **index.jsx** to require **app.jsx** which will then call **todoform.jsx**

#### index.jsx

```javascript
import React from 'react';
import App from './components/app.jsx'

main();

function main(){
  const app = document.createElement('div');
  document.body.appendChild(app);
  React.render(<App />, app);
}
```

Here we've created a div element which will contain everything react will render.

Now run the following to build our app again:

```bash
node_modules/.bin/webpack
```

If that build was successful, go to the build directory of your project and open **index.html** in your browser. You should see the page's title **To Do List**, and a header "To Do List".

Enable Hot-Loading
------------------

We had enabled hot loading in part 1 however now we are using react which uses state. Every time we save our application when we edit, the state will be lost if we leave it the way it is now. We can use **react-transform-hmr** and **babel-plugin-react-transform**. This will allow changes to be made without forcing a full refresh, however changes to some things including class constructors, will still need to be refreshed.

```bash
npm i babel-plugin-react-transform react-transform-hmr --save-dev
```

We'll also need to configure babel and create a **.babelrc** file in the root of our project directory:

#### .babelrc

```javascript
{
  "stage": 1,
  "env": {
    "start": {
      "plugins": [
        "react-transform"
      ],
      "extra": {
        "react-transform": {
          "transforms": [
            {
              "transform": "react-transform-hmr",
              "imports": ["react"],
              "locals": ["module"]
            }
          ]
        }
      }
    }
  }
}
```

This was copied directly from the tutorial mentioned at the beginning of this article.

Redux
-----

To summarize, here is the idea behind redux:

```
  _________      _________       ___________  
 |         |    | Change  |     |   React   |
 |  Store  |----▶ events  |-----▶   Views   |
 |_________|    |_________|     |___________|

```

This model improves maintainability when we have more complex react applications, as the data can only flow one way, and the data is kept in the same place.

For example, let's say you have state in a component in your applications hierarchy. You pass this state down your application hierarchy as props to the child components. Now you would like to edit the overall state of your application from one of the child components. You would like your entire hierarchy to be updated with the new state. Redux will allow us to do this.

Redux is a framework for react that uses ES6. Some other nice things about Redux:

-	Everything is hot-reloadable
-	It preserves the benefits of Flux, but adds other nice properties thanks to its functional nature
-	It prevents some of the anti-patterns common in Flux code
-	It doesn't care how you store your data: you can use JS objects, arrays, Immutable JS, etc.
-	The API is minimal
-	The stores are stateless

If you're looking into a pure Flux framework check out the **Alt** framework.

For a quick intro to Redux check out this [tutorial](https://github.com/happypoulp/redux-tutorial), as well as the Redux documentation [here](https://rackt.github.io/redux/docs/introduction/index.html)

Let's install redux and the redux development tools:

```bash
npm install --save redux
npm install --save react-redux
npm install --save-dev redux-devtools
npm install --save-dev react-dom
```

Now let's start using it. Create another directory inside of your app/ directory called actions/ with a new js file inside:

```
-app/
  -components/
    -app.jsx
    -list.jsx
  **actions/**
    **list_actions.js**
```

Actions are quite simple in Redux. First we will create our most basic case which is to add a "todo" item. Calling this action will tell the store to update the state with a new todo item.

#### list_actions.js

```javascript
//Action Type
export const ADD_TODO = 'ADD_TODO';

//Action Creator
export function addTodo(text){
  return {
    type: ADD_TODO,
    text
  };
}
```

Actions are merely payloads of information that your React views can send to your store. All actions must have a **type** property that will indicate the action being performed. Types should typically be defined as string constants.

An action creator is merely a function that creates an action. All the action creator should do is return an action, with no side-effects.

Now let's create our reducer:

#### reducers.js

```javascript

import { ADD_TODO } from './actions/list_actions';
import addTodo from './actions/list_actions';

function todos(state = [], action) {
  switch (action.type) {
  case ADD_TODO:
    return [...state, {
      text: action.text
    }];
  default:
    return state;
  }
}
export default todos;
```

The reducer's job is to specify how the application's state will change depending on what action has happened. Since we only have one action so far that's all we've defined here in the reducer. In the "ADD_TODO" case within the `javascript switch` block, we return a *new* object containing the previous state array, plus a new object (`javascript text: action.text`). The `javascript default:`case is needed, otherwise our state may be lost if an unknown action occurs. Also, **do not mutate the state**, create a copy of it with `Object.assign()`.

#### Store

The Store does the following:

-	Holds application state
-	Allows access to state via `getState()`
-	Allows state to be updated `via dispatch(action)`
-	Registers listeners via `subscribe(listener)`

The Store connects the actions and the reducers.

#### index.jsx

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import App from './components/app.jsx';
import { Provider } from 'react-redux';
import { createStore } from 'redux';
import todos from './reducers';

main();

function main(){
  const app = document.createElement('div');
  document.body.appendChild(app);
  const store = createStore(todos);
  ReactDOM.render(
    <Provider store={store}>
    {<App />}
    </Provider>,
    app
  );
}
```

We want to have our store created at the top of our application's hierarchy. This way when the store is updated all the appropriate child branches props are updated as well.

Now we'll have to create new components and update our existing components to finish our Redux app, but before we get there we should understand the difference between "smart" and "dumb" components.

smart/dumb components http://stackoverflow.com/questions/29577977/react-ref-and-setstate-not-working-with-es6

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator
