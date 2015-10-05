Building Our First Webpack Project
==================================

This tutorial is mainly based off of the one found [here](http://survivejs.com/webpack_react/developing_with_webpack/)

Webpack is a **module bundler**, which means it takes a bunch of files you have defined separately and bundles them together to automatically build your project. Webpack takes all the messy details and deals with them for you.

We will build a simple project to get the gist of how Webpack works. We'll start by laying out the objectives and requirements:

1.	We want to be able to generate a web page from a few different files to avoid huge files that are difficult to read and follow.
2.	We want to eliminate the need to create an index.html file manually
3.	We want to enable hot-loading.
4.	End with a "Hello World" javascript project, that we can eventually turn into a bigger "todolist" project with React and Flux.

Basic Setup
-----------

If you haven't already, download **npm**. Npm is a package manager for JavaScript. Create a directory for your new project and initialize your javascript project using npm:

```bash
mkdir todolist
cd todolist
npm init
```

The last command should prompt you for questions regarding your project. It also asks if you want to set up a repository.

Now install Webpack with npm inside the todolist directory:

```bash
npm i webpack node-libs-browser --save-dev
```

This should create a directory called **node_modules** which contains webpack. >Some ubuntu users may run into a problem finding node. This can be solved by >adding the location of node to your PATH variable, or by creating a symbolic >link from where your computer thinks node lives, to where it actually lives.

Structure
---------

This is the general directory structure of our starting application:

```
-app
  -component.js
  -index.js
-node_modules (do not create, it will be created automatically with npm)
-package.json
-webpack.config.js
```

Creating Our Initial Components
-------------------------------

In other tutorials we created index.html files. Now we will get Webpack to do it for us. After defining components in javascript. We will tie this into react later.

#### component.js

```javascript
module.exports = function () {
  var element = document.createElement('h1');

  element.innerHTML = 'Hello world';

  return element;
};
```

The above code creates a module that can be imported by other js files. This is what module.exports does. Now the file component.js can be imported by another file.

#### index.js

```javascript
var component = require('./component');
var app = document.createElement('div');

document.body.appendChild(app);

app.appendChild(component());
```

The above code imports the component.js file we created in line 1. As you can see this makes it a lot easier to read and follow. It also makes a project more maintainable as it grows when the files are split up like this.

Webpack Configuration
---------------------

Now that we have a document for Webpack to build, we can configure it in **webpack.config.js**. First, download html-webpack-plugin in the project root directory:

```bash
npm i html-webpack-plugin --save-dev
```

#### webpack.config.js

```javascript
var path = require('path');
var HtmlwebpackPlugin = require('html-webpack-plugin');

var ROOT_PATH = path.resolve(__dirname);

module.exports = {
  entry: path.resolve(ROOT_PATH, 'app'),
  output: {
    path: path.resolve(ROOT_PATH, 'build'),
    filename: 'bundle.js'
  },
  plugins: [
    new HtmlwebpackPlugin({
      title: 'To Do List'
    })
  ]
};
```

Now enter the following in the your todolist directory:

```bash
node_modules/.bin/webpack
```

This should build your project. You should now see a **build** directory that contains a **bundle.js** file and an **index.html** file. Open index.html in your browser.

Let's look at how that worked:

-	In webpack.config.js we found the path of our project with the path module.
-	Then, we exported info about our project so that webpack could deal with the details for us
-	Finally, we included our HtmlwebpackPlugin with the title of our page

Look at the following:

```javascript
entry: path.resolve(ROOT_PATH, 'app'),
output: {
  path: path.resolve(ROOT_PATH, 'build'),
  filename: 'bundle.js'
```

Here we say take all files from the app directory and bundle them together and put the output into the build directory to be viewed by a user.

Hot-Loading
-----------

Now we will enable hot-loading so that everytime we make a change to our components, we do not have to refresh the page. Once we save our project the changes will be applied to the build.

First, install the webpack-dev-server in your project's root directory:

```bash
npm i webpack-dev-server --save-dev
```

Also, for this to work we'll need to add something to our **package.json** file:

#### package.json

```javascript
...
"scripts": {
  "start": "webpack-dev-server"
},
...
```

In addition, we'll also need to update our **webpack.config.js** file:

```javascript
...
var webpack = require('webpack');

var ROOT_PATH = path.resolve(__dirname);

module.exports = {
  ...
  devServer: {
    historyApiFallback: true,
    hot: true,
    inline: true,
    progress: true
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    ...
  ]
};
```

Now in your project's root directory:

```bash
npm start
```

In your browser enter **127.0.0.1:8080**, then make changes to the "Hello World" string in **component.js**. It should update the string in your browser as you edit the string and save it in **component.js**.
