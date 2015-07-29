Title: Pelican and React Comments
Date: 2015-07-29 11:03
Category: Tutorial

#Pelican and React Comments

The previous tutorials “React Comments” Part One and Two covered the creation of a dynamic comment system. This comment system is not very useful on its own; it would be much better if it were used with the Pelican blog that we created in our previous Salt tutorials! 

> NOTE -- There are two possible scenarios for continuing with this tutorial. The first is if you completed the “React Comments” tutorials on a standalone machine with no previous configuration. You will need to install Pelican and Markdown and make a sample article for your comments to be displayed on. 

> The second option is to continue with the previous configuration from the “Salt” tutorials. This requires that NGINX, uWSGI, Falcon, Supervisor, and React be configured on the minion the same way the “React Comments” tutorials specify. 

> This tutorial is written from the perspective of the “Salt” way.

We will start by editing the CSS (Cascading Style Sheets) for the main theme. This will allow the comments system to fit nicely on any Pelican article.
```
sudo vim /usr/lib/python2.7/dist-packages/pelican/themes/notmyidea/static/css/main.css
```
> TIP -- Within vim, you can add line numbers by entering `:set number`

On line 53, remove the line `padding: 0 1px;`
This fixes some spacing issues in the comments.

On line 421, replace `#comments-list blockquote {` with `#comments-list .comment {`.
In our React, we use the comment class for our comment text instead of the HTML5 blockquote tag. This increases compatibility with older browsers

On line 434, replace `#comments-list li:nth-child(2n) blockquote {background: #F5f5f5;}` with `#comments-list li:nth-child(2n) .comment {background: #F5f5f5;}`

On line 439, replace `#add-comment input[type='email'],` with `#add-comment input[type='author'],`
In our React, we name the input field for username ‘author’. This corrects the template looking for ‘email’.

That is it for the main.css file. Now we need to add our comment system to the article configuration file.
```
sudo vim /usr/lib/python2.7/dist-packages/pelican/themes/notmyidea/templates/article.html
```
Add the following lines starting at line 17 (just after .entry-content -->):
```
<div id="react-comments"></div>
<script type="text/jsx" src="/theme/jsx/comments.jsx"></script>
```
Remove all code starting at `{% if DISQUS_SITENAME` and ending at `{% endif %}`

There is one last file that needs to be edited. This is the base configuration file that all articles use when they are loaded.
```
sudo vim /usr/lib/python2.7/dist-packages/pelican/themes/notmyidea/templates/base.html
```
Add the following starting on line 18 (after `<![endif]-->`). These are our script tags that allow React, Superagent, and Markdown to work together.
```
<script src="https://cdnjs.cloudflare.com/ajax/libs/superagent/1.2.0/superagent.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/0.13.3/react.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/0.13.3/JSXTransformer.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/marked/0.3.2/marked.min.js"></script>
```

There is one final step to take. We need to add our React code!
```
sudo mkdir /usr/lib/python2.7/dist-packages/pelican/themes/notmyidea/static/jsx
sudo vim /usr/lib/python2.7/dist-packages/pelican/themes/notmyidea/static/jsx/comments.jsx
```
This file contains the same javascript code as our index.html from the “React Comments” tutorial but is stripped of any html. It should look like the following:
```
var CommentBox = React.createClass({
  loadCommentsFromServer: function() {
    superagent
    .get(this.props.url)
    .type('json')
    .end(function(err,res){
    this.setState({data: JSON.parse(res.text)});
    }.bind(this));
  },
  handleCommentSubmit: function(comment) {
    var comments = this.state.data;
    var newComments = comments.concat([comment]);
    this.setState({data: newComments});
    superagent
    .post(this.props.url)
    .send(comment)
    .end()
  },
  getInitialState: function() {
    return {data: []};
  },
  componentDidMount: function() {
    this.loadCommentsFromServer();
    setInterval(this.loadCommentsFromServer, this.props.pollInterval);
  },
  render: function() {
    return (
      <div className="commentBox">
        <h1>Comments</h1>
        <CommentList data={this.state.data} />
    <CommentForm onCommentSubmit={this.handleCommentSubmit} />
      </div>
    );
  }
});

var CommentList = React.createClass({
  render: function() {
    var commentNodes = this.props.data.map(function (comment) {
      return (
        <Comment author={comment.author} time={comment.time}>
          {comment.text}
        </Comment>
      );
    });
    return (
      <div className="commentList">
    {commentNodes}
      </div>
    );
  }
});

var CommentForm = React.createClass({
   handleSubmit: function(e) {
    e.preventDefault();
    var author = React.findDOMNode(this.refs.author).value.trim();
    var text = React.findDOMNode(this.refs.text).value.trim();
    if (!text || !author) {
      return;
    }
    this.props.onCommentSubmit({author: author, text: text});
    React.findDOMNode(this.refs.author).value = '';
    React.findDOMNode(this.refs.text).value = '';
    return;
  },
  render: function() {
    return (
      <form className="commentForm" onSubmit={this.handleSubmit}>
        <input type="text" placeholder=" Your Name" ref="author" />
        <input type="text" placeholder=" Comment" ref="text" />
        <input type="submit" value="Post" />
      </form>
    );
  }
});

var Comment = React.createClass({
  render: function() {
    var rawMarkup = marked(this.props.children.toString(), {sanitize: true});
    return (
      <div className="comment">
        <h2 className="commentAuthor">
          {this.props.author}
          {this.props.time}
    </h2>
    <span dangerouslySetInnerHTML={{__html: rawMarkup}} />
      </div>
    );
  }
});

React.render(
  <CommentBox url="/sqlaccess" pollInterval={2000} />,
  document.getElementById('react-comments')
);
```

That is it for this tutorial. Now, when you browse to a Pelican article, you should see a neat (if not basic) comments box on the bottom.
