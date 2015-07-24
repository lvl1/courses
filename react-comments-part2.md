Title: React Comments Part2
Date: 2015-07-24 12:36
Category: Tutorial

#React Comments

`vim /usr/share/nginx/html/comments.py`

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
            	db = MySQLdb.connect(host='127.0.0.1', user='root', passwd='iTel123', db='falcon')
            	cur = db.cursor()
            	cur.execute("SELECT * FROM comments;")
		comments = ''
            	for row in cur.fetchall():
                    	comments = (comments + '  {"author":"' + row[2] + '","text":"' + row[1] + '","time":" ' + str(row[3]) + ' PST"},\n')
            	resp.body = ('[\n' + comments[:-2] + '\n]')
            	cur.close()
    	def on_post(self, req, resp):
           	"""Handles POST requests"""
            	resp.status = falcon.HTTP_200  # This is the default status
		jsonread = json.loads(req.stream.read().decode("utf-8"))
            	author = jsonread["author"]
            	text = jsonread["text"]
            	db = MySQLdb.connect(host='127.0.0.1', user='root', passwd='iTel123', db='falcon')
            	cur = db.cursor()
            	cur.execute("INSERT INTO comments (comment,name) VALUES (%s,%s)", (text,author))
            	cur.close()
            	db.commit()
            	resp.body = ('Comment Posted')
app = falcon.API()
comments = CommentResource()
app.add_route('/sqlaccess', comments)
```


Edit /etc/init/supervisord.conf:
```
[program:uwsgi]
command=uwsgi --wsgi-file /usr/share/nginx/html/comments.py --callable app -s 127.0.0.1:8080
autostart=true
autorestart=true
startsecs=
```
`rm /usr/share/nginx/html/index.html`

`vim /usr/share/nginx/html/index.html`
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
