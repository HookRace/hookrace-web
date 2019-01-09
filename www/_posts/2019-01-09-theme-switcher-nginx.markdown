---
layout: post
title: "Theme Switcher in Nginx"
tags: Programming DDNet
permalink: /blog/theme-switcher-nginx/
---

Some people really liked the dark [DDNet](https://ddnet.tw/) theme for Halloween by [Soreu](), so we decided to keep it possible to use the default bright or the dark theme.

Thanks to xse we got a [JavaScript based theme switcher](https://github.com/ddnet/ddnet-web/pull/69). After some improvements I finally I switched it away from JavaScript entirely and finally am also using it on this blog with an OLED friendly dark theme.

<!--more-->
The final result is quite simple, and only requires static HTML, a cookie for the theme and Nginx for setting and returning the correct CSS file for it. You can try it out by clicking on the [Switch Theme](/switch-theme/) button at the top of this page, which just redirects you to `/switch-theme/`.

Doing the CSS decision server-side instead of in JavaScript has the advantage that you don't get any flicker on rendering, no matter what theme you chose.

The entire nginx logic is:

{% highlight nginx %}
location /switch-theme/ {
  if ($http_cookie ~* "user_theme=/public/css-dark.css") {
    add_header Set-Cookie "user_theme=/public/css-empty.css;path=/;domain=hookrace.net";
    return 302 $http_referer;
  }
  add_header Set-Cookie "user_theme=/public/css-dark.css;path=/;domain=hookrace.net";
  return 302 $http_referer;
}

location /public/css-dark.css {
  if ($http_cookie ~* "user_theme=/public/css-dark.css") {
    return 302 /public/css/css-dark.css;
  }
  return 302 /public/css/css-empty.css;
}
{% endhighlight %}

Note that the `/switch-theme/` location sends the user back to the `$http_referer`. `/public/css-dark.css` is statically included on each page, but what it does depends on the cookie value.

The `302 Moved Temporarily` response makes the client send another request, adding another roundtrip to the page rendering. If the CSS is small enough you can just pull it directly into the Nginx config:

{% highlight nginx %}
location /public/css-dark.css {
  if ($http_cookie ~* "user_theme=/public/css-dark.css") {
    return 200 "body {color: #fff; background-color: #000;} .page-title, .post-title, .post-title a, h1, h2, h3, h4, h5, h6 {color: #ccc;} .masthead-title a {color: #bbb;}";
  }
  return 200 ""; 
}
{% endhighlight %}

One side effect of this entire approach is that the CSS file can't be cached anymore though, otherwise we'd end up showing the old theme when it gets switched. I'm sure this could be optimized somehow to tell the browser to only fetch the CSS file again after the theme was switched. But for my site the current performance is good enough.

Also note that [If Is Evil](https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/) in Nginx, so if you abuse features like this you might run into trouble.
