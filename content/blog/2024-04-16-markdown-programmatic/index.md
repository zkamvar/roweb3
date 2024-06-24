---
slug: "markdown-programmatic-parsing.edits"
title: All the ways to programmatically edit R Markdown / Quarto documents
author:
  - Maëlle Salmon
  - Christophe Dervieux
  - Zhian N. Kamvar
# Set the date below to the publication date of your post
date: 2024-04-16
# Minimal tags for a post about a community-contributed package 
# that has passed software peer review are listed below
# Consult the Technical Guidelines for information on choosing tags
tags:
  - pandoc
  - rmarkdown
  - tinkr
  - quarto
  - markdown
  - tech notes
description: ""
output: hugodown::md_document
---

If life gives you a bunch of Markdown files to analyse or edit, do you warm up your regex muscles and get going?
How about using more specific parsing tools instead?
In this post, we shall give an overview of programmatic ways to parse and edit Markdown files: Markdown, R Markdown, Quarto, Hugo files, you name it.

## What is Markdown?

Markdown is a (punny, eh) markup language created by John Gruber and Aaron Swartz.
Here is an example:

```md

# My first header

Some content, with parts in **bold** or *italic*.
Let me add a [link](https://ropensci.org).

```

Different Markdown files can lead to the same output, for instance this is equivalent to our first example:

```md

My first header
===============

Some content, with parts in __bold__ or _italic_. Let me add a [link](https://ropensci.org).

```

Furthermore there are different _flavors_ of Markdown[^md-flavours], which add some [extended syntax], like emoji written so: `:grin:`.

[^md-flavours]: As of 2024-06-20, there are [76 programs that parse markdown](https://github.com/markdown/markdown.github.com/wiki/Implementations), some with their own unique flavour.

[extended syntax]: https://www.markdownguide.org/extended-syntax/


Markdown formats that R users will commonly interact with include: R Markdown (uses Pandoc under the hood), Quarto (uses Pandoc under the hood... see any trend here?), GitHub, Hugo (for blogdown or hugodown websites).

Many tools using Markdown also accept metadata at the top of Markdown files, either YAML or TOML.
Here is an example with YAML:

```md
---
title: My cool thing
author: Myself
---

Some content, *nice* content.
```

Most often R users will write Markdown manually, or with the help of an editor such as the RStudio IDE visual editor.
But sometimes, one will have to create or edit a bunch of Markdown files at once, and editing all those files by hand is a huge waste of time. 
This blog post will give you resources in R that you can use to create, parse, and edit markdown documents, so that you can become the markdown wizard you have always dreamed of becoming :mage:!


## Templating Tools for Boilerplate Documents

Imagine you need to create a bunch of different R Markdown files, for instance for students to use as personalized exercises.
In that case, you can create a boilerplate document as a template, and create its different output versions using a templating tool.

Templating tools include:

- [`knitr::knit_expand()`](https://cran.r-project.org/web/packages/knitr/vignettes/knit_expand.html) by Yihui Xie;
- the [whisker package](https://github.com/edwindj/whisker) maintained by Edwin de Jonge (used in for instance pkgddown);
- the [brew package](https://github.com/gregfrog/brew) maintained by Greg Hunt;
- [Pandoc](/blog/2023/06/01/troubleshooting-pandoc-problems-as-an-r-user/) by John MacFarlane.

The simplest example of the whisker package might furthermore remind you of the glue package.

A common workflow would be:

- You create a template in a file, where variable parts are indicated by strings such as `{{name}}`.
- You read this template in R using for instance the brio package.
- Mapping over your set of variables, you render the template using whisker and save each version to a file using the brio package. 

### Example

Here's an example markdown file that we can use as a template:

````markdown
---
title: "Homework assignment 1"
author: "{{name}}"
---

Create a normal distribution with a mean of {{mean}} and a standard deviation of {{sd}}:

```{r solution-1}
# hint: use the rnorm function
```
````

Using the workflow above, we can r


``` r
# generate student variables ----
students <- c("Maëlle", "Christophe", "Zhian")
n <- length(students)
key <- data.frame(
  name = students,
  mean = rpois(n, 5),
  sd   = sprintf("%.1f", runif(n)),
  file = sprintf("%s-hw.md", students)
)
# render and write assignment from template ---- 
make_assignment <- function(key, template) {
  lapply(seq(n), function(i) {
    new <- whisker::whisker.render(template, data = key[i, ])
    brio::write_lines(new, key$file[i])
  })
  return(invisible())
}
md <- brio::read_lines("hw-template.md")
make_assignment(key, template = md)
print(key)
```

```
##         name mean  sd             file
## 1     Maëlle    5 0.2     Maëlle-hw.md
## 2 Christophe    6 0.1 Christophe-hw.md
## 3      Zhian    5 0.7      Zhian-hw.md
```



## String Manipulation Tools

You can use string manipulation tools to parse Markdown if you are sure of the Markdown variants your code will get as input, or if you are willing to grow your codebase to accommodate many edge cases... which in the end means you are writing an actual Markdown parser. 
Not for the faint of heart... neither necessary if you read the section after this one. :relieved:

You'd detect heading using for instance `grep("^#", markdown_lines)`[^edge].

[^edge]: But this would also detect code comments! Don't do this!

Example of string manipulation tools include base R (`sub()`, `grep()` and friends), [stringr](https://stringr.tidyverse.org/) (and stringi), `xfun::gsub_file()`.

Although string manipulation tools are of a limited usefulness when parsing Markdown, they can _complement_ the actual parsing tools.
Even if using specific Markdown parsing tools will help you write less regular expressions yourself... they won't completely free you from them.

## Parsing Tools

Parsing tools are fantastic, and numerous.
These translate the Markdown document into a data structure called an [Abstract
Syntax Tree (AST)][AST] that gives you fine-grained control over specific elements of the document (e.g. individual headings or links regardless of how they are written).
With a formal data structure, you can programmatically manipulate the Markdown document by adding, removing, or manipulating pieces of Markdown in a standardized way. 
We will only mention the ones you can directly use from R.

[AST]: https://en.wikipedia.org/wiki/Abstract_syntax_tree

### Fine-grain Parsing

Let's say you have created a bunch of tutorials that link to a website containing a gallery of extensions for a popular plotting package. 
Let's also say that one day, someone discovers that the link to the website is suddenly [redirecting to a potentially malicious site that is most certainly not related to the grammar of graphics](https://github.com/ggplot2-exts/gallery/issues/112) and you
need to replace all instances of that link to `**redacted**`. Since links in Markdown could be written any number of ways, regex is not going to help you, but a fine-grained Markdown parser will!

A workflow for this situation would be:

- read in the Markdown AST with your favourite parser
- pull out all links that point to the rotten link
- replace them with emphasized text that says "redacted"
- convert the AST and write back to file

The [tinkr package](http://docs.ropensci.org/tinkr/) dreamed up by Maëlle Salmon and maintained by Zhian Kamvar parses Markdown to XML using Commonmark, allows you to extract and manipulate markdown using XPath via the xml2 package. 
Tinkr writes the XML back to Markdown using XSLT. 
The YAML metadata is available as a string.

The [md4r package](https://rundel.github.io/md4r/), is a recent experimental package maintained by Colin Rundel, and is an R wrapper around the MD4C (Markdown for C) library and represents the AST as a nested list with attributes in R. 
The development version of the package has utilities for constructing Markdown documents programmatically.

With Pandoc that we presented in a [tech note last year](blog/2023/06/01/troubleshooting-pandoc-problems-as-an-r-user/#raw-attributes), you can parse a Markdown files to a Pandoc Abstract Syntax Tree (in JSON format). 
Nic Crane has an experimental package called [parseqmd](https://github.com/thisisnic/parseqmd) that uses this strategy, parsing
the output with the jsonlite package.
You can also parse to, say HTML, and then back to Markdown. The benefit of parsing it to HTML is that you can use a package such as rvest to extract and manipulate the elements.

### High-level Parsing

If you are only interested in the heading structure of a document and code chunks, where you may or may not want to manipulate the rest of the Markdown, you might benefit from using a high-level parser. 
The [parsermd package](https://rundel.github.io/parsermd/) is another package maintained by Colin Rundel is "implementation of a formal grammar and parser for R Markdown documents using the Boost Spirit X3 library. 
It also includes a collection of high level functions for working with the resulting abstract syntax tree."
This package is different from other parsing options mentioned here because, in the words of its author, the aim of the package is "...to capture the fundamental structure of the document and as such we do not attempt to parse every detail of the Rmd."

This package has functionality for a tidy workflow allowing you to select different sections of the document.
One useful feature is that it has the function [`rmd_check_template()`](https://rundel.github.io/parsermd/articles/templates.html) allowing you to compare student markdown submissions against a standard template. 
You can watch his [RStudio::conf(2021) talk about it](https://posit.co/resources/videos/parsermd-parsing-r-markdown-for-fun-and-profit/).

### The Impossibility of a Perfect Roundtrip

When parsing and editing Markdown, then writing it back to Markdown, some undesired changes might appear.
For instance, with [tinkr](http://docs.ropensci.org/tinkr/#general-principles-and-solution) list items all start with a `-` even if in the original document they started with a `*`. With md4r, lists that are indented with extra space will be readjusted. 

Depending on your use case you might want to find ways to mitigate such losses, for instance only re-writing the lines you made intentional edits to.

### How to Choose a Parser?

You can choose a parser based on what it lets you manipulate the Markdown with: if you prefer XML[^maelle-approved] and HTML to nested lists for instance, you might prefer using tinkr or Pandoc.
If the high-level functions of md4r or parsermd are suitable for your use case, you might prefer one of them.

[^maelle-approved]: Both Maëlle and Zhian are _huge_ fans of XML and XPath (see: https://masalmon.eu/2022/04/08/xml-xpath/ and https://zkamvar.netlify.app/blog/gh-task-lists/). 

Another important criterion is to choose a parser that's a close to the use case of your Markdown files as possible.
If you are only going to work with Markdown files for GitHub, commonmark/tinkr is an excellent choice since GitHub itself uses commonmark. 
Now, your work might encompass different sorts of Markdown files that will be used by different tools.
For instance, the babeldown package processes any Markdown file[^caveat]: Markdown, R Markdown, Quarto, Hugo.
In that case, or if there is no R parser doing exactly what your Markdown's end user does, you need to pay attention to the quirks of that end user. 
Maybe you have to throw [Pandoc raw attributes](blog/2023/06/01/troubleshooting-pandoc-problems-as-an-r-user/#raw-attributes) around a Hugo shortcode, for instance.
Furthermore, if you need to parse certain elements, like again Hugo shortcodes, you might need to write the parsing code yourself, that is, regular expressions.

[^caveat]: Or at least it's supposed to :sweat_smile: Thankfully users report edge cases that are not covered yet.

## What about the Code Chunks

Programmatically parsing and editing R code is out of the scope of this post, but closely related enough to throw in a few tips.
As with Markdown, you might need to use regular expressions but try not to.
You can parse the code to XML using base R parsing and [xmlparsedata](https://r-lib.github.io/xmlparsedata/), then you manipulate the XML with [XPath](https://masalmon.eu/2022/04/08/xml-xpath/).
To write code back, you can make use of the attributes of each node that indicates the original lines and columns.

So a possible workflow is:

- parse the code to XML, use xmlparsedata to inform what to change and where. Out of these steps you'd get a list of elements' positions for instance.
- use brio to read the lines, change a few of them with base R tools, then use brio again to write the lines back.

## Examples of Markdown Parsing and Editing

The [pegboard package](https://carpentries.github.io/pegboard/) maintained by Zhian Kamvar, parses and validates Carpentries' lessons for structural markdown elements, including valid links, alt-text, and known fenced-divs thanks to tinkr.
This package was instrumental in converting all of The Carpentries lesson infrastructure from Jekyll's markdown syntax to Pandoc's markdown[^transition].

[^transition]: For examples, see [The Carpentries Workbench Transition Guide](https://carpentries.github.io/workbench/transition-guide.html).

The [babeldown package](https://docs.ropensci.org/babeldown/) maintained by Maëlle Salmon transforms Markdown to XML, sends it to DeepL API for translation, and writes the results back to Markdown, also using tinkr.

## Conclusion

In this post we explained how to best parse and edit Markdown files: using specific parsing tools, possibly complemented by ad-hoc string manipulation.
What do *you* use to handle Markdown files?