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
sudo vim /etc/supervisor/supervisor.conf:
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

If all went well, you should see a “comment posted” message. You can log into MySQL to verify that the comment was actually posted. You can also go to http://your-server-ip/sqlaccess in your browser to see the comment.

It’s time for our actual comments system, React style! Your react file should be stored in /usr/share/nginx/html/index.html. Because NGINX comes with a default index.html, you should first remove it:
```
rm /usr/share/nginx/html/index.html
```

Create a new index.html file. This will hold all the React code.
```
vim /usr/share/nginx/html/index.html
```
The following code is explained in detail on [Facebook’s Tutorial](https://facebook.github.io/react/docs/tutorial.html)

This code accomplishes five main goals:
- A simple GUI of two text boxes and a submit button is created
- The comments are updated in real time (every two seconds) from our MySQL server.
- Comments are posted to our MySQL server whenever the ‘submit’ button is clicked. 
- Comments are shown to the user before they are saved on the server to make it feel “fast”.
- Markdown formatting support is added so users can write comments with markdown

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
  <CommentBox url="http://your-server-ip/sqlaccess" pollInterval={2000} />,
  document.getElementById('content')
);
    </script>
  </body>
</html>
```

>NOTE -- The above example must have ‘your-server-ip’ in the sixth last line replaced with your server’s ip.

Our code differs from Facebook’s example in only one way. Facebook’s example uses the jQuery library to send POST requests and accept GET requests between the React code and the SQL server. For our use, jQuery is outdated so we used Superagent as a replacement. The Superagent section of code (the first two functions of CommentBox), was also only half as the original jQuery code.


Now browse to http://your-server-ip. You should be able to submit comments and see them update in realtime. Timestamps should also automatically be added to your comments as they are posted!

##Troubleshooting

Parts One and Two of the React comments tutorials contain multiple layers of complexity. If you followed our examples word for word, your configuration should work as expected. However, problems may still arise because of unintentional (or intentional for fun!) mistakes. Here are a few troubleshooting tips that will help you diagnose your problem quicker.

- If you receive a “502 Bad Gateway” error, this means that NGINX cannot find and/or communicate with uWSGI. You should check your NGINX and uWSGI configuration to verify that they are setup to communicate with each other correctly.

- An “internal server error” usually indicates that NGINX and uWSGI are configured correctly, but there is a communication problem with MySQL. Check that MySQL authentication is working correctly and verify that the commands are being entered with proper syntax. 

- React errors can be determined by using the “inspect element” feature in your browser. The console and network views will often give you direct error messages about your code.

- If all else fails, stop uWSGI with `sudo supervisorctl stop uwsgi`. If you run `uwsgi --wsgi-file /usr/share/nginx/html/comments.py --callable app -s 127.0.0.1:8080` in your server’s command prompt, you will get verbose output as to what is happening in the background. Often, problems within your Falcon python code will pop up when using this method to troubleshoot. You can restart uWSGI with `sudo supervisorctl start uwsgi`.
