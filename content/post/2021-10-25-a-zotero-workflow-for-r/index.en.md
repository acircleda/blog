---
title: A Zotero Workflow for R
author: Anthony Schmidt
date: '2021-10-25'
slug: []
categories: []
tags: []
subtitle: ''
summary: ''
authors: []
lastmod: '2021-10-25T10:45:10-04:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

In terms of citation management systems, there are not many options: EndNote, Excel, Mendeley, and Zotero. I was a big fan of Mendeley, especially with its integrated PDF reader. However, as I began to write more text in R, I realized using Mendeley was insufficient. When I heard that Zotero integrates nicely with R, I made the switch immediately.

There are several packages that make using R and Zotero pretty seamless. The purpose of this short blogpost is to outline the current packages and workflow I am using as I write my dissertation.

## Zotero Setup

Key to successfully using Zotero is installing add-ons. As part of my Zotero setup, I primarily use the [Better BibTex for Zotero](https://retorque.re/zotero-better-bibtex/) add-on. This facilitates a number of important processes for the workflow:

-   The generation of citation keys, which are necessary for referring to sources in-text (e.g. `@author2021`). You can set custom key formats and other key tricks. Mine is simple: \`\[auth:lower\]\[year\]\`, with each key unique to each library.

-   Better BibTex also converts Zotero's default unicode to LaTeX, ensuring citations are formatted and appear correctly (99% of the time).

-   Updated exports of your library. You can export your entire library and that export will be automatically updated every time you add, remove, or make changes in Zotero. There are numerous options for working with these auto exports as well.

## R Packages

[`citr`](https://github.com/crsh/citr) is the package I use to integrate R and Zotero. There are two key functions I use from citr:

1.  `citr::insert_citation()` (this is also an R Studio addin). This pops up a gui which allows you to select your Zotero library and search for citations. This makes it super convenient to insert references as you are typing. It is so convenient, in fact, that I have it mapped to a [custom keyboard command](https://support.rstudio.com/hc/en-us/articles/206382178-Customizing-Keyboard-Shortcuts): **Ctrl + ;** . As you insert citations into your document, they are also automatically added the the bibliography file you have specified in your YAML, keeping your end of document references page updated and formatted correctly. However, I have had some issues lately with the `insert_citation()` function crashing lately, mostly due to libraries with non-Roman letters in author names. So, I have not been using this as much lately, as will be shown below.

2.  `citr::tidy_bib_file()` . This handy function keep essentially keeps your bibliography file clean by removing duplicates and unwanted entries. I call this function every time I knit to keep my bibliography file sparkling.

## Workflow

I have a specific library dedicated to my project, in my current case, a "Dissertation" library with all my related readings. I typically add files to my library using Chrome's Save to Zotero extension.

I have previously exported this library to my R project folder, and selected the option to keep it updated as I add, remove, and make changes to the library within Zotero. This exported library has been cleverly named "Dissertation.bib".

As I write in my R Markdown document and need a reference, I search Zotero until I find it, and copy the citation key to include in text (e.g., `@author2021` ).

When I go to knit my document, the following code is called right after R libraries are loaded:

    citr::tidy_bib_file(
      rmd_file = c("index.Rmd", "03-introduction.Rmd", 
      "04-lit-review.Rmd"),
      messy_bibliography = "Dissertation.bib", #updated by Zotero
      file = "references.bib" #read in by R
    )

What do the arguments refer to?

-   `rmd_file` refers to the specific R Markdown files I have written and want `citr` to look through to update my bibliography

-   `messy_bibliography` refers to the Zotero-managed export. This is considered "messy" because it likely includes numerous citations I have not actually referred to in text.

-   `file` refers to the clean document `citr` creates that only includes citations referenced in my R files. This file is included in my YAML: `bibliography: references.bib`

The above process keeps a properly updated bibliography that contains no superfluous references. In addition, as I am knitting, the console shoots out warnings of any references missing, most often because of typos or issues with hyphens (tip: don't use hyphens). This allows me to fix citations and re-knit quickly, ensuring a properly cited final document.

I find the above workflow convenient and low maintenance. The only effort is the initial setup, adding additional Markdown files to the `rmd_file` argument as needed, and fixing missing citations. Combining R and Zotero is pretty easy and makes the writing experience pleasurable. So, I highly recommend writing in R. However, even if you choose to write in another platform (Word, Google Docs), I still highly recommend Zotero because it is free, easy-to-use, and open source - which means many robust addons to improve its functionality.
