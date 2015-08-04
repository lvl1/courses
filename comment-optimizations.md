Title: Comment Optimizations
Date: 2015-08-04 9:11
Category: Tutorial
Id: 010109

#Comment Optimizations

This tutorial will cover the optimization of our comments system. Optimizations include making everything look prettier and making per-page comments.

Begin by editing the main CSS file for our pelican theme.
```
sudo vim /usr/lib/python2.7/dist-packages/pelican/themes/notmyidea/static/css/main.css
```
Add the following CSS to the bottom of the file:
```
.commentAuthor{
        float:left;
}
.time{
        float:right;
}
.text{
        clear:left;
        display:block;
}
.comment{
        background:rgba(0,0,0,0.05);
        padding:0.4em;
        margin-bottom:0.8em;
        border-radius:0.5em;
}
```
Here is a breakdown of the code:
- float:left ensures that the author of the comment is always to the left.
- float:right ensures that the time the comment was posted is displayed to the right.
- clear:left makes sure that there is no whitespace to the left of the actual comment.
- display:block turns the comment into its own paragraph, similar to an HTML <p> tag.
- background:rgba(0,0,0,0.05) sets the background color; in particular, we are setting the color intensity.
- padding:0.4em sets the border margin around the comment
- margin-bottom:0.8em sets the bottom spacing so that the comments don’t overlap each other
- border-radius:0.5em adds rounded borders to the comments.

Now that our CSS styling is in place, we need to edit the javascript code to put the styling into effect.
```
sudo vim /usr/lib/python2.7/dist-packages/pelican/themes/notmyidea/static/jsx/comments.jsx
```
```
var Comment = React.createClass({
  render: function() {
    var rawMarkup = marked(this.props.children.toString(), {sanitize: true});
    return (
      <div className="comment">
        <h2 className="commentAuthor">
          {this.props.author}
    </h2>
        <span className="time">
          {this.props.time}
        </span>
        <br />
    <span className="text" dangerouslySetInnerHTML={{__html: rawMarkup}} />
      </div>
    );
  }
});
```

