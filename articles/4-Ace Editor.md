I've recently discovered [Ace Editor](http://ace.ajax.org), which is apparently *the* text editor for web browsers. It's all done with Javascript, and performs very well. It gets used for all sorts of things, including [dillinger.io](http://dillinger.io/), which I use to write these blog entries. I like how it works, and I like how it looks, so what can a programmer do but to try to put it to use?

In a project for work, we have a page where we try to display a bunch of code, and of course that code has to have syntax highlighting. For quite a while, we were satisfied with the job that [this syntax highlighter](http://alexgorbatchev.com/SyntaxHighlighter/) did, but one thing really bugged us; the scrollbars.

Having extra scrollbars on a web page sucks. You've already got a vertical scrollbar at the side of almost any web page. The aforementioned syntax highlighter library *always* adds an extra vertical scrollbar to the highlighted segment. You can get rid of it easily enough by adding `padding: 1px` to the container's style, but that's not the big problem. When the code is long (horizontally as well as vertically), you get a horizontal scrollbar *all the way* at the bottom of the highlighted code. When that code is a few hundred lines long, it becomes a huge chore to see what's at the end of line 3.

So now my quest is to change the code display page so that the code itself doesn't introduce any extra scrollbars. My current attempt uses Ace, and the rest of this post explains just how it works.

Ace likes to be told how big to be. The demo shows you how to embed ace in a page so that it takes up the whole page. The gist of the demo gives you an `editor` element, some style information, and the setup script, along the lines of...

    <style>
    #editor {
        position: absolute;
        top: 0;
        right: 0;
        bottom: 0;
        left: 0;
    }
    </style>
    
    <div id="editor">A whole bunch of code can go here</div>
    
    <script>
        var editor = ace.edit('editor');
    </script>
    
(excerpt from [http://ace.ajax.org/#nav=embedding](http://ace.ajax.org/#nav=embedding))

Okay, great - Ace is on my page. But I don't want it to take up the *whole* page. I decide to make an `editor-container` to put the `editor` in.

    #editor-container {
        position: relative;
        height: 10000px;
    }
    
By using `position: relative;` on the container, the absolute positioning of the editor itself becomes relative to that container. I picked an arbitrarily high number for the height, just because if you leave it alone, the editor doesn't even show up. At least with this much extra vertical space, the editor should be able to display every single line without scrolling. 

As for horizontal scrolling, I can eliminate that by turning on word wrap.

    editor.getSession().setUseWrapMode(true);

And now the only scroll bars should be the one(s) that normally show up. Now the issue is that there's too much vertical space taken by the editor.

To solve this problem, I wrap the container in a "masking" element:

    #editor-mask {
        height: 1000px;
        overflow-y: hidden;
    }

The 1000 is arbitrary: I plan to change it using javascript later on. For now, what it accomplishes is that the stupidly-tall container gets tucked under the mask, without a vertical scrollbar. Now the task is to dynamically adjust the mask so that it is always exactly the height I want it to be.

I can find that exact number by finding the bottom position of the last line number in Ace's gutter (obviously line numbers need to be enabled for this). I can use [jQuery](http://jquery.com/) to select the last line number element...

    var lastLine = $("#editor .ace_gutter-cell:last")

Since the cells have `positioned:absolute;`, and are relative to the container `<div>`, you can use the `top` and `outerHeight` to get the needed height for the masking `<div>`.

    var neededHeight = lastLine.position().top + lastLine.outerHeight();
    $("#editor-mask").height(neededHeight);
    
Now I can see all of the lines of code; no more, no less! One caveat here: Ace hides lines that are "out of sight" in the editor. So if the `editor-container` element isn't tall enough, the editor will introduce a vertical scroll, and the selector that I used to get the `lastLine` will return whatever the last *visible* line is. That's why the `height` of the `editor-container` needs to be extra tall. And just to reiterate, the mask's job is to hide any extra blank space that gets introduced this way.

The last little bit of work is to make sure that the height calculation happens whenever the window resizes. Since I decided to use word wrapping to eliminate the need for a horizontal scroll bar, I have to make sure that when the lines get taller due to wrapping, the mask continues to be the right size.

I wrap the height calculation into a handly little function...

    function setEditorHeight(){
    	var lastLine = $("#editor .ace_gutter-cell:last");
		var neededHeight = lastLine.position().top + lastLine.outerHeight();
		$("#editor-mask").height(neededHeight);
	}
    
...and make sure that it gets called when the window resizes (and also when the page loads).

    $(window).on('resize', setEditorHeight);
    setEditorHeight();
    
And that's it! Check out a working demo [here](/demos/ace-space-filler)
