---
published: true
layout: post
author: marco
categories:
  - fun
  - joke
  - humour
  - humor
  - javascript
  - js
featured: true
---
Javascript is nonsense. Period. Often, I never found enough motivate to looking for way to express the frustration or a ways to mock it. Then I came across this [video](https://www.youtube.com/watch?v=b7WxO4ipnh0&t=2s) which talks about Python and some JS.

So, fire up you Firefox/Chrome console. Here's some guessing game from the video:

What's the results of:

- `[] + []`

>! "" // empty string !<

- `[] + {}`

>! `[object Object] // an object` !<

- `{} + []`

>! `0 // a number` !<

- `{} + {}`

>! `NaN // Not-a-number` !<

- `Array(16).join("wat" + 1)`

>! `wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1` !<

- `Array(16).join("wat" - 1) + " Batman!"`

>! `NaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaN Batmant!` !<
