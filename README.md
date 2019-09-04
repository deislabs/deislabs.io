# deislabs.io

A simple static site, built with Hugo and Netlify.


## development

The site uses a custom theme, based off of the Fresh theme for Hugo Luc Perkins. The underlying CSS framework is bulma, [docs](https://bulma.io/documentation/) are here. Any changes to the CSS/SCSS need to be rebuilt and commited to update the site appearance.

```
// run hugo to have the pipes rebuild and recompile
# hugo

// commit the generated results to git
# git add resources/_gen/*
```
