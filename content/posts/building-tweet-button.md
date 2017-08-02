+++
date = "2017-08-02T01:49:16-04:00"
title = "Building a better 'share' button"
categories = []
tags = ["Twitter", "Hugo", "Internet privacy", "Democracy", "Hacker culture", "CoCitizen"]
+++

"Tweet" and other social media sharing buttons are pretty much ubiquitous these days. That's assuming you aren't using something like the [Ghostery](https://ghostery.com) browser plugin. If you are, then those buttons are likely being blocked because they use techniques like [tracking cookies](https://www.tomsguide.com/us/-tracking-cookie-definition,news-17506.html) to record your browsing behaviour. Bear in mind, this isn't standard web analytics, looking at how you interact with an organization's own website. Instead, these "share" buttons allow social networking companies to track you across any site that has one of these buttons.

We at CoCitizen take [internet privacy](https://en.wikipedia.org/wiki/Privacy_concerns_with_social_networking_services#Privacy_concerns) seriously. Unfettered surveillance [undermines democracy](https://www.gnu.org/philosophy/surveillance-vs-democracy.en.html), and helping to build more resilient democracies is what we're all about.

Yesterday, when I set up this blog on [Hugo](https://gohugo.io/), I discovered both Twitter and Facebook buttons were a built-in feature when Ghostery blocked them. Now don't get me wrong; I fully appreciate the value of social media. So, while I don't want to help to compromise our visitors' privacy, I'd still like to make it easy for them to share our content with their followers/friends/contacts.

Being new to Hugo, and a [hacker](https://en.wikipedia.org/wiki/Hacker_culture) by nature, I figured this'd be a good chance to familiarize myself with how Hugo works. So, first, I unblocked my locally running instance of this blog in Ghostery, and observed how the "tweet" button behaved. Functionally, this turned out to be just a URL that encoded some variables in [query strings](https://en.wikipedia.org/wiki/Query_string). It looked something like this:

```
    https://twitter.com/intent/tweet?original_referer=http://localhost:1313/posts/hello-world/&ref_src=twsrc^tfw&text=hello world&tw_p=tweetbutton&url=http://localhost:1313/posts/hello-world/&via=kendo5731
```

This led to a pre-written tweet that included the title and URL of the current post along with "via @kendo5731". We're using the fun ["code-editor"](https://github.com/aubm/hugo-code-editor-theme) theme from Aurélien Baumann, and that's his Twitter handle. That obviously isn't what we want, but made tracking down the code that was generating it pretty easy:

``` bash
$ grep kendo5731 . -rn
./themes/code-editor/layouts/partials/share.html:2:<a href="https://twitter.com/share" class="twitter-share-button" data-via="kendo5731"></a>
```

So the next step was figuring out how to override that. It turns out that Hugo works pretty much as I expected: building pages from templates that are looked up on in a [particular order](http://gohugo.io/templates/lookup-order/). When Hugo creates a new site, it bootstraps the project with default directories to support this mechanism. So, it was a simple matter of copying that file into `./layouts/partials/`, and then editing it to alter its behaviour.

The relevant code to generate the "tweet" button looked like this:

``` html
<!-- Share on Twitter -->
<a href="https://twitter.com/share" class="twitter-share-button" data-via="kendo5731"></a>
<script>!function (d, s, id) {
    var js, fjs = d.getElementsByTagName(s)[0], p = /^http:/.test(d.location) ? 'http' : 'https';
    if (!d.getElementById(id)) {
        js = d.createElement(s);
        js.id = id;
        js.src = p + '://platform.twitter.com/widgets.js';
        fjs.parentNode.insertBefore(js, fjs);
    }
}(document, 'script', 'twitter-wjs');</script>
```

As we can see, most of the buttons behaviours are governed by the Twitter SDK's widget javascript. That, itself is the problem here. So what we want to do is reproduce the same behaviour, using just our own code. In this case, just HTML and CSS should be sufficient.

The HTML portion is pretty straight-forward. We can just replace our copied "partial" snippet with:

``` html
<!-- Share on Twitter -->
<div class="tweet"><a class="button" href="https://twitter.com/share?via=CoCitizenTech&text={{ .Title }}"><i></i><span class="label">Tweet</span></a></div>
```

After some experimentation, it turns out we can drop all but the "via" and "text" query strings. Here I've hardcoded our Twitter handle ([@CoCitizenTech](https://twitter.com/CoCitizenTech)), since we want it to remain consistent across all pages. The post's title text will have to change for each page though. So we use Hugo's [page template variables](https://gohugo.io/variables/page/) to inject the appropriate title.

So that get's us a functional "tweet" _link_, but what we wanted was the iconic button. For that we'll need to add some custom CSS. Again, Hugo makes this easy, by just dropping a file in the correct place:

``` bash
$ tree static/
static/
└── css
    └── cocitizen.css
```

While that'll make our CSS file available, it doesn't yet include the necessary `<link>` element in the page. For that, we need to alter another partial HTML snippet. There may be a more elegant way to do this, but for now I just overrode the `./themes/code-editor/layouts/partials/head.html` file by copying it to `layouts/partials/head.html`, and adding the following line:

``` html
<link href="{{ .Site.BaseURL }}/css/cocitizen.css" rel="stylesheet" type="text/css">
```

Bear in mind, I work mostly on backend programming, devops, etc. So forgive my weak CSS-fu. Anyway, after some experimentation, here's the CSS to match the default "tweet" button:

```
div.tweet {
  height: 20px;
  width: 61px;
  display: inline-block;
  white-space: nowrap;
  overflow: hidden;
  border-radius: 3px;
}

div.tweet a.button {
  padding: 2px 8px 2px 6px;
  background-color: #1b95e0;
  color: #fff;
  cursor: pointer;
  text-decoration: none;
}

div.tweet a:hover {
  background-color: #0081ce;
}

div.tweet span.label {
  margin-left: 3px;
  font: normal normal normal 11px/18px 'Helvetica Neue',Arial,sans-serif;
}
```

The final bit is to add the Twitter icon. Eventually, we'll probably add [Font Awesome](http://fontawesome.io/), which includes such an icon. For now though, I just inlined the SVG image directly. Note that the SVG is URL-encoded:

``` css
div.tweet i {
  position: relative;
  top: 3px;
  display: inline-block;
  width: 14px;
  height: 14px;
  background: transparent 0 0 no-repeat;
  background-image: url(data:image/svg+xml,%3Csvg%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%20viewBox%3D%220%200%2072%2072%22%3E%3Cpath%20fill%3D%22none%22%20d%3D%22M0%200h72v72H0z%22%2F%3E%3Cpath%20class%3D%22icon%22%20fill%3D%22%23fff%22%20d%3D%22M68.812%2015.14c-2.348%201.04-4.87%201.744-7.52%202.06%202.704-1.62%204.78-4.186%205.757-7.243-2.53%201.5-5.33%202.592-8.314%203.176C56.35%2010.59%2052.948%209%2049.182%209c-7.23%200-13.092%205.86-13.092%2013.093%200%201.026.118%202.02.338%202.98C25.543%2024.527%2015.9%2019.318%209.44%2011.396c-1.125%201.936-1.77%204.184-1.77%206.58%200%204.543%202.312%208.552%205.824%2010.9-2.146-.07-4.165-.658-5.93-1.64-.002.056-.002.11-.002.163%200%206.345%204.513%2011.638%2010.504%2012.84-1.1.298-2.256.457-3.45.457-.845%200-1.666-.078-2.464-.23%201.667%205.2%206.5%208.985%2012.23%209.09-4.482%203.51-10.13%205.605-16.26%205.605-1.055%200-2.096-.06-3.122-.184%205.794%203.717%2012.676%205.882%2020.067%205.882%2024.083%200%2037.25-19.95%2037.25-37.25%200-.565-.013-1.133-.038-1.693%202.558-1.847%204.778-4.15%206.532-6.774z%22%2F%3E%3C%2Fsvg%3E)
}
```

So, with that, we've built a working tweet button that respects our visitors' privacy, while still making it easy to share their thoughts on our content. You can see the result at the top of this page, and every other post. Feel free to use it ;)

