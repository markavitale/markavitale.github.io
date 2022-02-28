---
layout: darkmode
permalink: index.html
---

I'm a software engineer who's worked on mostly-native apps for Apple platforms in some form for most of my career. I've also worked on CI/CD and custom build tooling for the same. If you're more interested in my work experience, feel free to check out my Linked In profile below.

I mostly have this domain for email purposes, but may write use it to drop some unorganized thoughts on something I find interesting. This is a personal website and all the thoughts should be treated as my own, as poorly thought out, and as probably wrong.

{% if site.posts.size > 0 %}
## Unorganized thoughts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
{% endif %}

Follow the [RSS feed]({{ site.url }}/feed.xml) to be notified of new posts.

## More organized things with some of my thoughts
- [Objective-C Coding Guidelines](https://microsoft.github.io/objc-guide/)
- [Swift Coding Guidelines](https://microsoft.github.io/swift-guide/)

## Apps
- [Rocket Calendar](/apps/RocketCalendar)

## Where to find me
- [LinkedIn](https://www.linkedin.com/in/mark-vitale-83139921/)
- [GitHub](https://github.com/markavitale/)
- [Twitter](https://twitter.com/MarkAVitale)

## Support me
To show your support for this site and encourage more articles you can [buy me a bourbon](https://www.buymeacoffee.com/markavitale).

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="markavitale" data-color="#FF5F5F" data-emoji="🥃"  data-font="Arial" data-text="Buy me a bourbon" data-outline-color="#000000" data-font-color="#ffffff" data-coffee-color="#FFDD00" ></script>
