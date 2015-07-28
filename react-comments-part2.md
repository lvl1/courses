Title: React Comments Part2
Date: 2015-07-28 12:08
Category: Tutorial

#React Comments

Welcome to Tutorial Part Two for configuring React comments. Part One is a prerequisite for this article.

This tutorial will complete the configuration of a dynamic comments system using React. The majority of this configuration mirrors the tutorial made by Facebook [here](https://facebook.github.io/react/docs/tutorial.html). 


Before we begin working with react, we need to create a falcon file that will handle JSON posts and will communicate with our sql database.

>NOTE -- JSON (JavaScript Object Notation) is a lightweight data format that is easy for both machines and humans to read. We will be using JSON to communicate between our comments system and the sql database.
Proper raw JSON data looks like the following: {“author”:"Me","text":"Hello"...etc}


Create a new comments.py file:
```
vim /usr/share/nginx/html/comments.py
```
```python
import json
# Let's get this party started
import falcon
## my comments will have TWO shift-threes.
## get our SQL on:
import MySQLdb

class CommentResource:
    	def on_get(self, req, resp):
            	"""Handles GET requests"""
            	resp.status = falcon.HTTP_200  # This is the default status
            	db = MySQLdb.connect(host='127.0.0.1', user='root', passwd='password', db='comments_db')
            	cur = db.cursor()
            	cur.execute("SELECT * FROM comments_table;")
		commentslist = ''
            	for row in cur.fetchall():
                    	commentslist = (commentslist + '  {"author":"' + row[2] + '","text":"' + row[1] + '","time":" ' + str(row[3]) + ' PST"},\n')
            	resp.body = ('[\n' + commentslist[:-2] + '\n]')
            	cur.close()
    	def on_post(self, req, resp):
           	"""Handles POST requests"""
            	resp.status = falcon.HTTP_200  # This is the default status
		jsonread = json.loads(req.stream.read().decode("utf-8"))
            	author = jsonread["author"]
            	text = jsonread["text"]
            	db = MySQLdb.connect(host='127.0.0.1', user='root', passwd='password', db='comments_db')
            	cur = db.cursor()
            	cur.execute("INSERT INTO comments_table (comment,name) VALUES (%s,%s)", (text,author))
            	cur.close()
            	db.commit()
            	resp.body = ('Comment Posted')
app = falcon.API()
comments = CommentResource()
app.add_route('/sqlaccess', comments)
```
For anyone with little Python experience, this file can appear a little intimidating. However, when broken down into pieces, it can be more easily understood.

Similar to the comments.py file we made earlier in /opt/comments/, our new comments.py contains a CommentsResource class. Within this class is two functions - one for HTTP GET requests, the other for HTTP POST requests.

Within the on_get function, the db variable connects to the MySQL database (“password” must be replaced with the appropriate password). cur.execute runs the “SELECT * FROM comments_table;" command and the for loop saves the output to the variable commentslist in the correct json form. resp.body then returns the list with a success code of 200.

The on_post function uses the db variable to connect to the MySQL database (again, “password” must be replaced with the appropriate password). Our comments system will eventually pass a JSON stream to Falcon whenever a new comment is made. jsonread loads this stream and puts the “author” and “text” information into new variables. cur.execute then inserts the author and text values into the sql table “comments_table”.

The bottom three lines then add a route to the Falcon application at /sqlaccess in the same way that our previous comments.py did. 

Because we created a new Falcon application at a new file location, we need to edit the supervisor configuration so that Falcon is started automatically.
```
sudo vim /etc/init/supervisord.conf:
```
```
[program:uwsgi]
command=uwsgi --wsgi-file /usr/share/nginx/html/comments.py --callable app -s 127.0.0.1:8080
autostart=true
autorestart=true
startsecs=
```


We are ready to test our new falcon file. The simplest way to do this is to POST raw json data to it. Curl, a command line tool for browsing the internet, is perfect for this. This command should work from any machine with internet connection.
```
curl -X POST "http://your-server-ip/sqlaccess" -H "Content-Type: application/json" -d'{"author":"Me","text":"Hello"}'
```

If all went well, you should see a “comment posted” message. You can log into MySQL to verify that the comment was actually posted.

Your react file should be stored in /usr/share/nginx/html/index.html. Because nginx comes with a default index.html, you should first remove it:
```
rm /usr/share/nginx/html/index.html
```


```
vim /usr/share/nginx/html/index.html
```
```
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>Hello React</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/0.13.3/react.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/0.13.3/JSXTransformer.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/marked/0.3.2/marked.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/superagent/1.2.0/superagent.js"></script>
  </head>
  <body>
    <div id="content"></div>
    <script type="text/jsx">

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
  <CommentBox url="http://yourip/sqlaccess" pollInterval={2000} />,
  document.getElementById('content')
);
    </script>
  </body>
</html>
```

Now browse to http://yourip
