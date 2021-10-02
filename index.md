## Welcome to BucketOfCode.com

Hi 👋. My name is Dan Schaefer and this is my personal instance of Stack Overflow and GitHub's Gist.

I have been a software developer for over 16 years and there have been countless times where I have found answers to software problems that get me 80% of the answer I'm looking for. I want to capture the whole solution to the very specific problems that I have solved. I hope to help at least one of you to get you to your final 20%.

Here's a few articles to get you started.

<ul>
{% for post in site.posts %}
  <li><a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>