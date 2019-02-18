# Tips About Writing to Auth0

## Originality

Guest Authors have to produce **original articles**. That is, authors have to write content with their own words and **no plagiarism will be accepted**. It is normal and totally ok to use and quote external references. Actually, this is desired, as it makes the article more reliable from an SEO perspective (articles will rank better when linking trustworthy sources of information).

However, authors have to study, learn, and understand the technologies involved to a point where they can explain these topics with their own words. Authors also have to strive to produce explanations better than those found on the web. This is what makes an article valuable.

## Grammatical Person

Choose one [grammatical person](https://en.wikipedia.org/wiki/Grammatical_person) and stick with it. Usually, writers tend to stick with "you" (second-person singular) when writing articles. For example:

- "Now, you have to create a new file...";
- "Next, you will use this nice package to help you ...";

You can also choose the second-person plural when writing:

- "Now, we have to create a new file...";
- "Next, we will use this nice package to help our ...";

The thing is, it doesn't really matter which one you will choose. What is important is that you choose one and stick with it. Avoid using different grammatical person interchangeably.

## Punctuation

Writers often forget to give the proper attention to punctuation. Punctuation helps readers to understand what the author is trying to explain. So, please, **do use** commas, full stops, colons, parentheses, and so on where appropriate. Not using punctuation generally leads to [run-on sentences](http://grammar.ccc.commnet.edu/grammar/runons.htm) that are hard to understand.

## Transition Words and Prepositional Phrases

Use [transition words](http://www.smart-words.org/linking-words/transition-words.html) and prepositional phrases to make content more fluid and friendly. For example, when telling readers to update a file to add some code, instead of writing like this:

> Open the `index.js` file and update it as follows:

You can use a transition word (e.g. _next_ or _now_) to make it more fluid:

> Next, open the `index.js` file and ...

Another example is related to moving from one explanation to an action. In that case, a preposition phrase can make the content more friendly:

> With that covered, open a terminal and install the `create-react-app` package.

In this case, _"with that covered"_ is the prepositional phrase.

## Capitalization

Please, capitalize the words appropriately. There are two cases where you need to capitalize:

1. when starting new sentences;
2. when referring to proper nouns.

For example, _GitHub_ has two uppercase letters. So, it's important not to write _Github_ nor _github_. _JavaScript_ is another similar example. It is not _Javascript_ nor _javascript_.

**Note that** these are just two examples. Please, search how a proper noun is written and follow the convention.

## Formatting with Markdown

This section will give you some tips on what are the preferred approach when structuring/formatting you article. After reading the instructions here, please, refer to the following cheatsheets. They can help you learn more about formatting content on Markdown files:

- [_Markdown Cheatsheet_](https://github.com/auth0/blog/blob/master/post-cheat-sheet.markdown)
- [_Auth0 Markdown Cheatsheet_](https://github.com/auth0/blog/blob/master/post-cheat-sheet.markdown)

### Blank Lines

While writing your content, please, add a blank line before and after the following elements:

- Titles
- Paragraphs
- Lists
- Code Blocks
- Images
- etc

However, **don't add blank lines** between sentences in the same paragraph (actually, leave sentences in the same paragraph in the line) and between items on the same list. Also, **avoid** adding multiple blank lines. They don't add value to readability.

For example, adding blank lines like in this screenshot:

![Add blank lines when writing markdown files](https://cdn.auth0.com/blog/guest-writer/with-blank-lines.png)

Enhances readability when compared to this:

![Avoid cluttering everything together](https://cdn.auth0.com/blog/guest-writer/without-blank-lines.png)

### Formatting Inline Code

When writing a technical article, you will often refer to things like filenames, directories, library names, etc. If you need to refer to them inside a sentence, please, format them as inline code (like `this`). Formatting these elements as inline code will make it easier for readers to identify important things in your article.

So, to clarify, this is a quasi-complete list of elements that you have to format as inline code:

- Filenames (e.g. `app.js`)
- Directories (e.g. `./my-dir/whatever-dir/`)
- Library names (e.g. `create-react-app`)
- Function names (e.g. `sumTwoNumbers()`)
- Class names (e.g. `MyReactComponent`)
- Commands (e.g. `npm init`)
- etc

What is important to notice is that **you shouldn't use** inline code to highlight proper nouns. For example, if you are referring to Git, the version control system, don't use inline code. However, if you are referring to `git`, the command-line interface, then you have to format it as inline code.

### Formatting Code

Lastly, when dealing with code blocks, don't use four spaces or tabs to define these blocks. Please, use fenced code blocks that start and end with three consecutive backtick characters. Also, put the code block type after the opening line (`js`, `bash`, `typescript`, etc).

For example, prefer doing this:

![Markdown code block formatted with fences](https://cdn.auth0.com/blog/guest-writer/fenced-code-block.png)

Rather than this:

![Markdown code block formatted with tabs](https://cdn.auth0.com/blog/guest-writer/tabs-code-block.png)
