---
published: false
---
## Javascript nonsense

Javascript is nonsense. Period. Often, I never found enough motivate to looking for way to express the frustration look for ways to mock about it. Then I came across this [video](https://www.youtube.com/watch?v=b7WxO4ipnh0&t=2s) which talks about Python but mocking JS.

So, fire up you Firefox/Chrome console. Here's some guessing game from the video:

What the result of:

1. `[] + []`
>! `"" // empty string`

2. `[] + {}`
>! `[object Object] // an object`

3. `{} + []`
>! `0 // a number`

4. `{} + {}`
>! `NaN // Not-a-number`

5. `Array(16).join("wat" + 1)`
>! `wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1wat1`

6. `Array(16).join("wat" - 1) + " Batman!"`
>! `NaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaNNaN Batmant!`

