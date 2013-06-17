# Domain specific languages

The combination of first class environments and lexical scoping gives us a powerful toolkit for creating domain specific languages in R. There's much to be said about domain specific languages, and most of it is said very well by Martin Fowler in his book [Domain Specific Languages](http://amzn.com/0321712943). In this chapter we'll explore how you can new languages that use R's syntax but have different behaviours. Creating new DSLs in R combines many of techniques you have learned about from lexical scoping, to first class environments to computing on the language.  Before you begin, make sure you're familiar with:

* scoping rules
* creating and manipulating functions
* computing on the language
* S3 basics

This chapter will develop two simple, but useful, DSLs, one for generating HTML, and one for turning R mathematical expressions into a form suitable for inclusion in latex. They will build up in complexity, and show you how you can adapt R to generate very different kinds of output using rather different semantics to usual. The combination of a small amount of computing on the language, and constructing special evaluation environments is very powerful, and quite efficient.

One package that makes extensive use of these ideas is dplyr, which provides `to_sql()` which converts R expressions into SQL:

```R
library(dplyr)
to_sql(sin(x) + tan(y))
to_sql(x < 5 & !(y >= 5))
to_sql(first %like% "Had*")
to_sql(first %in% c("John", "Roger", "Robert"))
to_sql(like == 7)
```

Once you have read this chapter, you might want to study the source code for dplyr. An important part of the overall structure of the package is `partial_eval()` which helps manage expressions where some of the components refer to variables in the database and some refer to local R objects. You could use very similar ideas if you needed to translate small R expressions into other languages, like javascript or python. Converting complete R programs would be extremely difficult, but often being able to communicate a simple description of computation between languages is very useful.

## HTML

HTML is the language that underlies the majority of the web. It is a special case of SGML, and similar (but not identical) to XML. HTML looks like this:

```html
<h1>This is a heading.</h1>
<p>This is some text. <b>This sentence is bold</b></p>
```

Even if you've never looked at raw HTML before, hopefully you can see the key component of the structure: HTML is composed of tags that look like `<tag></tag>`. Tags can be contained inside other tags and intermingled with text. Generally, html ignores whitespace: an sequence of whitespace is equivalent to a single space. You could put the previous example all on online and it would still display the same in the browser:

```html
<h1>This is a heading.</h1> <p>This is some text. <b>This sentence is bold</b></p>
```

There are over 100 html tags, but in case you're not familiar with html, we're going to focus on just a few:

* `<h1>`: creates a heading-1, the top level heading
* `<p>`: creates a paragraph
* `<b>`: emboldens text

(you probably guessed what these did already!)

Tags can also have named attributes that look like `<tag a = "a" b = "b"></tags>`. Tag value should always be enclosed in either single or double quotes. Two important attributes used on just about every tag are `id` and `class`. These are used in conjunction with CSS (cascading style sheets) in order to control the style of the document.

Finally, some tags, like `<img>`, can't have any content and are known as void tags. Instead of writing `<img></img>`, for tags that don't have any content you write `<img />`. Of course, these tags are usually used for their attributes, and `img` has three important attributes `src` (where the image lives), `width` and `height`, so it will often look like `<img src='myimg.png' width='100' height='100' />`

Because `<` and `>` have special meanings in html, you can't write them directly. Instead you have to use the html escapes `&gt;` and `&lt;` respectively. And since `&` has a special meaning too, you'll need to use `&amp;`

### Goal

Our goal is to make it possible to generate html easily in R using R functions. To give a concrete example, we want to generate the following html:

```R
<h1 id='first'>A heading</h1>
<p>Some text &amp; <b>some bold text.</b>
<img src="myimg.png" width="100" height = "100"
```

And we want to generate using code that looks as similar to the html as possible. We will work our way up to the following DSL:

```R
with_html(
  h1("A heading", id = "first"),
  p("Some text &", b("some bold text."))
  img(src = "myimg.png", width = 100, height = 100)
)
```

Note that the nesting of function calls is the same as the nesting of html tags, unnamed arguments become the content of the tag, and named arguments become the attributes.  Because tags and text are clearly distinct in this api, we can automatically escape `&` and other special characters.

### Escaping

Escaping is so fundamental we're going to start with it. We first start by creating a way of escaping the characters that have special meaning for html, while making sure we don't end up double-escaping at any point. The easiest way to do this is to create an S3 class that allows us to distinguish between regular text (that needs escaping) and html (that doesn't).

We then write an escape method that leaves html unchanged and escapes the special characters (`&`, `<`, `>`) in ordinary text. We also add a method for lists for convenience

```r
html <- function(x) structure(x, class = "html")
escape <- function(x) UseMethod("escape")
escape.html <- function(x) x
escape.character <- function(x) {
  x <- gsub("&", "&amp;", x)
  x <- gsub("<", "&lt;", x)
  x <- gsub(">", "&gt;", x)
  
  html(x)
}
escape.list <- function(x) {
  lapply(x, escape)
}

# Now we check that it works
escape("This is some text.")
escape("x > 1 & y < 2")
escape(escape("x > 1 & y < 2"))

# Double escaping is not a problem
escape(escape("This is some text. 1 > 2"))

# And text we know is html doesn't get escaped.
escape(html("<hr />"))
```

### Basic tag functions

Next we'll write a few simple tag functions and then figure out how to generalise for all possible html tags.  Let's start with a paragraph tag since that's probably the most commonly used.

HTML tags can have both attributes (e.g. id, or class) and children (like `<b>` or `<i>`). We need some way of separating these in the function call: since attributes are named values and children don't have names, it seems natural to separate using named vs. unnamed arguments. Then a call to `p()` might look like:

```R
p("Some text.", b("Some bold text"), i("Some italic text"), 
  class = "mypara")
```

We could list all the possible attributes of the p tag in the function definition, but that's hard because there are so many, and it's possible to use [custom attributes](http://html5doctor.com/html5-custom-data-attributes/) Instead we'll just use ... and separate the components based on whether or they are named. To do this correctly, we need to be aware of a "feature" of `names()`:

```R
names(c(a = 1, b = 2))
names(c(a = 1, 2))
names(c(1, 2))
```

With this in mind we create two helper functions to extract the named and unnamed components of a vector:

```r
named <- function(x) {
  if (is.null(names(x))) return(NULL)
  x[names(x) != ""]
}
unnamed <- function(x) {
  if (is.null(names(x))) return(x)
  x[names(x) == ""]
}
```

With this in hand, we can create our `p()` function. There's one new function here: `html_attributes()`. This takes a list of name-value pairs and creates the correct html attributes specification from them. It's a little complicated, not that important and doesn't introduce any important new ideas, so I won't discuss it here, but it's described at the end of the chapter.

```r
p <- function(...) {
  args <- list(...)
  attribs <- html_attributes(named(args))
  children <- unlist(escape(unnamed(args)))
  
  html(paste0("<p", attribs, ">", paste(children, collapse = ""), "</p>"))
}

p("Some text")
p("Some text", id = "myid")
p("Some text", image = NULL)
p("Some text", class = "important", "data-value" = 10)
```

### Tag functions

With this definition of `p()` it's pretty easy to see what will change for different tags: we just need to replace `p` with a variable.  We'll use a closure to make it easy to generate a tag function given a tag name:

```r
tag <- function(tag) {
  force(tag)
  function(...) {
    args <- list(...)
    attribs <- html_attributes(named(args))
    children <- unlist(escape(unnamed(args)))
    
    html(paste0("<", tag, attribs, ">", 
      paste(children, collapse = ""), 
      "</", tag, ">"))
  }
}
```

(We're forcing the evaluation `tag` with the expectation we'll be calling this function from a loop later on - that avoids potential bugs caused by lazy evaluation.)

Now we can run our earlier example:

```R
p <- tag("p")
b <- tag("b")
i <- tag("i")
p("Some text.", b("Some bold text"), i("Some italic text"), 
  class = "mypara")
```

Before we continue to generate functions for every possible html tag, we need a variant of `tag()` for void tags. It can be very similar to `tag()`, but needs to throw an error is there are any unnamed tags, and the tag itself also looks slightly different.

```r
void_tag <- function(tag) {
  force(tag)
  function(...) {
    args <- list(...)
    if (length(unnamed(args)) > 0) {
      stop("Tag ", tag, " can not have children", call. = FALSE)
    }
    attribs <- html_attributes(named(args))
    
    html(paste0("<", tag, attribs, " />"))
  }
}

img <- void_tag("img")
img(src = "myimage.png", width = 100, height = 100)
```

Next we need a list of all the html tags:

```r
tags <- c("a", "abbr", "address", "article", "aside", "audio", "b", 
  "bdi", "bdo", "blockquote", "body", "button", "canvas", "caption", 
  "cite", "code", "colgroup", "data", "datalist", "dd", "del", 
  "details", "dfn", "div", "dl", "dt", "em", "eventsource", 
  "fieldset", "figcaption", "figure", "footer", "form", "h1", "h2", 
  "h3", "h4", "h5", "h6", "head", "header", "hgroup", "html", "i", 
  "iframe", "ins", "kbd", "label", "legend", "li", "mark", "map", 
  "menu", "meter", "nav", "noscript", "object", "ol", "optgroup", 
  "option", "output", "p", "pre", "progress", "q", "ruby", "rp", 
  "rt", "s", "samp", "script", "section", "select", "small", "span", 
  "strong", "style", "sub", "summary", "sup", "table", "tbody", 
  "td", "textarea", "tfoot", "th", "thead", "time", "title", "tr", 
  "u", "ul", "var", "video") 

void_tags <- c("area", "base", "br", "col", "command", "embed", 
  "hr", "img", "input", "keygen", "link", "meta", "param", "source", 
  "track", "wbr")
```

### `with_html`

If you look at this list carefully, you'll see there are quite a few tags that have the same name as base R functions (`body`, `col`, `q`, `source`, `sub`, `summary`, `table`), and others that clash with popular packages (e.g. `map`). So we don't want to make all the functions available (in either the global environment or a package environment) by default.  So what we'll do is put them in a list, and add some additional code to make it easy to use them when desired.

```r
tag_fs <- c(
  setNames(lapply(tags, tag), tags), 
  setNames(lapply(void_tags, void_tag), void_tags)
)
```

This gives us a way to call tag functions explicitly, but is a little
verbose:

```R
tags$p("Some text.", tags$b("Some bold text"), 
  tags$i("Some italic text"))
```

We finish off our HTML DSL by creating a function that allows us to evaluate code in the context of that list:

```r
with_html <- function(code) {
  eval(substitute(code), tag_fs)
}
```

This gives us a succinct API which allows us to write html when we need it without cluttering up the namespace when we don't. Inside `with_html` if you want to access the R function overridden by an html tag of the same name, you can use the full `package::function` specification.

```R
with_html(p("Some text", b("Some bold text"), i("Some italic text")))
```

### Code to make html attributes

```r
html_attributes <- function(list) {
  if (length(list) == 0) return("")

  attr <- Map(html_attribute, names(list), list)
  paste0(" ", unlist(attr), collapse = "")
}
html_attribute <- function(name, value = NULL) {
  if (length(value) == 0) return(name)
  if (length(value) != 1) stop("value must be NULL or of length 1")

  if (is.logical(value)) {
  value <- tolower(value)
  } else {
  value <- escape_attr(value)
  }
  paste0(name, " = '", value, "'")
}
escape_attr <- function(x) {
  x <- escape.character(x)
  x <- gsub("\'", '&#39;', x)
  x <- gsub("\"", '&quot;', x)
  x <- gsub("\r", '&#13;', x)
  x <- gsub("\n", '&#10;', x)
  x
}
```

### Exercises

* The escaping rules for `<script>` and `<style>` tags are different: you don't want to escape angle brackets or ampersands, but you do want to escape `</`.  Adapt the code above to follow these rules.

* The use of ... for all functions has some big downsides: there's no input validation and there will be little information in the documentation or autocomplete about how to use the function. Create a new function that when given a named list of tags and their attribute names (like below), creates functions with those signatures.

  ```R
  list(
    a = c("href"),
    img = c("src", "width", "height")
  )
  ```

  All tags should get `class` and `id` attributes.

## Latex

The next DSL we're going to tackle will convert R expression into their latex math equivalents. (This is a bit like `?plotmath`, but for text instead of plots.) Latex is the lingua franca of mathematicians and statisticians: whenever you want to describe an equation in text (e.g. in an email) you write it as a latex equation. Many reports are produced from R using latex, so it might be useful to facilitate the automate conversion from mathematical expressions from one language to the other.

This math expression DSL will be more complicated than the HTML dsl, because not only do we need to convert functions, but we also need to convert symbols.  We'll also create a "default" conversion, so that functions we don't know how to convert get a standard fallback. Like the HTML dsl, we'll also write functionals to make it easier to generate the translators.

Before we begin, let's quickly cover how formulas are expressed in latex.

### Latex mathematics

Latex mathematics are complex, and [well documented](http://en.wikibooks.org/wiki/LaTeX/Mathematics). They have a fairly simple structure:

* Most simple mathematical equations are represented in the way you'd type them into R: `x * y`, `z ^ 5`.  Subscripts are written using `_`, e.g. `x_1`.

* Special characters start with a `\`: `\pi` = π, `\pm` = ±, and so on. There are a huge number of symbols available in latex. Googling for `latex math symbols` finds many [lists](http://www.sunilpatel.co.uk/latex-type/latex-math-symbols/), and there's even [a service](http://detexify.kirelabs.org/classify.html) where you can sketch a symbol in the browser and it will look it up for you.

* More complicated functions look like `\name{arg1}{arg2}`.  For example to represent a fraction you use `\frac{a}{b}`, and a sqrt looks like `\sqrt{a}`.

* To group elements together use `{}`: i.e. `x ^ a + b` vs. `x ^ {a + b}`.

* In good math typesetting, a distinction is made between variables and functions, but without extra information, latex doesn't know whether `f(a * b)` represents calling the function `f` with argument `a * b`, or is shorthand for `f * a * b`. If `f` is a function, you can tell latex to typeset it using an upright font with `\textrm{f}(a * b)`

### Goal

Our goal is to use these rules to automatically convert from an R expression to a latex representation of that expression. We will tackle it in four stages:

* Convert known symbols: `pi` -> `\pi`
* Leave other symbols unchanged: `x` -> `x`, `y` -> `y`
* Convert known functions: `x * pi` -> `x * \pi`, `sqrt(frac(a, b))` -> `\sqrt{\frac{a, b}}`
* Wrap unknown functions with `\textrm`: `f(a)` -> `\textrm{f}(a)`

Compared to the HTML dsl, we'll work in the opposite direction: we'll start with the infrastructure and work our way down to generate all the functions we need

### `to_math`

To begin, we need a wrapper function that we'll use to convert R expressions into latex math expressions. This works the same way as `to_html`: we capture the unevaluated expression and evaluate it in a special environment. However, the special environment is no longer fixed, and will vary depending on the expression. We need this in order to be able to deal with symbols and functions that we don't know about a priori.

```r
to_math <- function(x) {
  expr <- substitute(x)
  eval(expr, latex_env(expr))
}
```

### Known symbols

Our first step is to create an environment that allows us to convert the special latex symbols used for Greek, e.g. `pi` to `\pi`. This is the same basic trick used in `subset` to make it possible to select column ranges by name (`subset(mtcars, , cyl:wt)`): we just bind a name to a string in a special environment.

First we create than environment by creating a named vector, converting that vector into a list, and then turn that list into an environment.

```r
greek <- c(
  "alpha", "theta", "tau", "beta", "vartheta", "pi", "upsilon", 
  "gamma", "gamma", "varpi", "phi", "delta", "kappa", "rho", 
  "varphi", "epsilon", "lambda", "varrho", "chi", "varepsilon", 
  "mu", "sigma", "psi", "zeta", "nu", "varsigma", "omega", "eta", 
  "xi", "Gamma", "Lambda", "Sigma", "Psi", "Delta", "Xi", "Upsilon", 
  "Omega", "Theta", "Pi", "Phi")
greek_list <- setNames(paste0("\\", greek), greek)
greek_env <- list2env(as.list(greek_list), parent = emptyenv())
```

We can then check it:

```R
latex_env <- function(expr) {
  greek_env
}

to_math(pi)
to_math(beta)
```

### Unknown symbols

If a symbol isn't greek, we want to leave it as is. This is trickier because we don't know in advance what symbols will be used, and we can't possibly generate them all. So we'll use a little bit of computing on the language to find out what symbols are present in an expression. The `all_names` function takes an expression: if it's a name, it converts it to a string; if it's a call, it recurses down through its arguments.

```r
all_names <- function(x) {
  # Base cases
  if (is.name(x)) return(as.character(x))
  if (!is.call(x)) return(NULL)

  # Recursive case
  children <- lapply(x[-1], all_names)
  unique(unlist(children))
}

all_names(quote(x + y + f(a, b, c, 10)))
# [1] "x" "y" "a" "b" "c"
```

We now want to take that list of symbols, and convert it to an environment so that each symbol is mapped to a string representing itself (e.g. so `eval(quote(x), env)` yields `"x"`). We again use the pattern of converting a named character vector to a list, then an environment.

```r
latex_env <- function(expr) {
  names <- all_names(expr)
  symbol_list <- setNames(as.list(names), names)
  symbol_env <- list2env(symbol_list)

  symbol_env
}

to_math(x)
to_math(longvariablename)
to_math(pi)
```

This works, but we need to combine it with the enviroment of the Greek symbols. Since we want to prefer Greek to the defaults (e.g. `to_math(pi)` should give `"\\pi"`, not `"pi"`), `symbol_env` needs to be the parent of `greek_env`, and thus we need to make a copy of `greek_env` with a new parent.  Strangely R doesn't come with a function for cloning environments, but we can easily create one by combining two existing functions:

```r
clone_env <- function(env, parent = parent.env(env)) {
  list2env(as.list(env), parent = parent)
}
```

This gives us a function that can convert both known (Greek) and unknown symbols.

```r
latex_env <- function(expr) {
  # Unknown symbols
  names <- all_names(expr)
  symbol_list <- setNames(as.list(names), names)
  symbol_env <- list2env(symbol_list)

  # Known symbols
  clone_env(greek_env, symbol_env)
}

to_math(x)
to_math(longvariablename)
to_math(pi)
```

### Known functions

Next we'll add functions to our DSL.  We'll start with a couple of helper closures that make it easy to add new unary and binary operators. These functions are very simple since they only have to assemble strings. (Again we use `force` to make sure the arguments are evaluated at the right time.)

```r
unary_op <- function(left, right) {
  force(left)
  force(right)
  function(e1) {
    paste0(left, e1, right)
  }
}

binary_op <- function(sep) {
  force(sep)
  function(e1, e2) {
    paste0(e1, sep, e2)
  }
}
```

Using these helpers, we can map a few illustrative examples from R to latex. Note how the lexical scoping rules of R help us: we can easily provide new meanings for standard functions like `+`, `-` and `*`, and even `(` and `{`. 

```r
# Binary operators
fenv <- new.env(parent = emptyenv())
fenv$"+" <- binary_op(" + ")
fenv$"-" <- binary_op(" - ")
fenv$"*" <- binary_op(" * ")
fenv$"/" <- binary_op(" / ")
fenv$"^" <- binary_op("^")
fenv$"[" <- binary_op("_")

# Grouping
fenv$"{" <- unary_op("\\left{ ", " \\right}")
fenv$"(" <- unary_op("\\left( ", " \\right)")
fenv$paste <- paste

# Other math functions
fenv$sqrt <- unary_op("\\sqrt{", "}")
fenv$sin <- unary_op("\\sin(", ")")
fenv$log <- unary_op("\\log(", ")")
fenv$abs <- unary_op("\\left| ", "\\right| ")
fenv$frac <- function(a, b) {
  paste0("\\frac{", a, "}{", b, "}")
}

# Labelling
fenv$hat <- unary_op("\\hat{", "}")
fenv$tilde <- unary_op("\\tilde{", "}")
```

We again modify `latex_env()` to include this environment. It should be the first environment in which names are looked for, so that `sin(sin)` works. (because of R's matching rules wrt functions vs. other objects)

```r
latex_env <- function(expr) {
  # Default symbols
  names <- all_names(expr)
  symbol_list <- setNames(as.list(names), names)
  symbol_env <- list2env(symbol_list)

  # Known symbols
  greek_env <- clone_env(greek_env, parent = symbol_env)

  # Known functions
  clone_env(f_env, greek_env)
}

to_math(sin(x + pi))
to_math(log(x_i ^ 2))
to_math(sin(sin))
```

### Unknown functions

Finally, we'll add a default for functions that we don't know about. Like the unknown names, we can't know in advance what these will be, so we again use a little computing on the language to figure them out:

```r
all_calls <- function(x) {
  # Base name
  if (!is.call(x)) return(NULL)

  # Recursive case
  fname <- as.character(x[[1]])
  children <- lapply(x[-1], all_calls)
  unique(c(fname, unlist(children, use.names = FALSE)))
}

all_calls(quote(f(g + b, c, d(a))))
```

And we need a closure that will generate the functions for each unknown call

```r
unknown_op <- function(op) {
  force(op)
  function(...) {
    contents <- paste(..., collapse = ", ")
    paste0("\\mathrm{", op, "}(", contents, ")")
  }
}
```

### Exercises

* Add automatic escaping. Special symbols that should be escaped by adding a backslash in front of them are `\`, `$` and `%`.  Like for sql, you'll need to make sure you don't end up double-escaping, so you'll need to create a small s3 class and then use that in function operators.  That will also allow you to embed arbitrary latex if needed.

* Complete the DSL to support all the functions that `plotmath` supports

* There's a repeating pattern in `latex_env()`: we take a character vector, do something to each piece, then convert it to a list, and then an environment. Write a function to automate this task, and then rewrite `latex_env()`