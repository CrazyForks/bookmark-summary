Title: Fizz Buzz in CSS - Susam Pal

URL Source: https://susam.net/fizz-buzz-in-css.html

Published Time: Sat, 06 Dec 2025 00:42:53 GMT

Markdown Content:
By **Susam Pal** on 06 Dec 2025

What is the smallest CSS we can write to produce the Fizz Buzz sequence? One could of course do this with no CSS at all, simply by placing the entire sequence as plain text in the HTML body. So to make the problem precise and keep it interesting, we require that every number and word that appears in the output must come directly from the CSS. Placing any part of the output numbers or words outside the stylesheet or using JavaScript is not allowed. With this constraint, I think it can be done in four lines of CSS as shown below:

```
li { counter-increment: n }
li:not(:nth-child(5n))::before { content: counter(n) }
li:nth-child(3n)::before { content: "Fizz" }
li:nth-child(5n)::after { content: "Buzz" }
```

Here is a complete working example: [css-fizz-buzz.html](https://susam.net/css-fizz-buzz.html).

I am neither a web developer nor a code-golfer. Seasoned code-golfers looking for a challenge can probably shrink this solution further. However, such wizards are also likely to scoff at any mention of counting lines of code, since CSS can be collapsed into a single line. The number of characters is probably more meaningful. The code can also be minified slightly by removing all whitespace:

```
$ curl -sS https://susam.net/css-fizz-buzz.html | sed -n '/counter/,/after/p' | tr -d '[:space:]'
li{counter-increment:n}li:not(:nth-child(5n))::before{content:counter(n)}li:nth-child(3n)::before{content:"Fizz"}li:nth-child(5n)::after{content:"Buzz"}
```

This minified version is composed of 152 characters:

```
$ curl -sS https://susam.net/css-fizz-buzz.html | sed -n '/counter/,/after/p' | tr -d '[:space:]' | wc -c
152
```

If you manage to create a shorter solution, please do leave a comment.

See also: [Fizz Buzz with Cosines](https://susam.net/fizz-buzz-with-cosines.html).