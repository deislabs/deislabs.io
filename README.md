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

[![Netlify Status](https://api.netlify.com/api/v1/badges/e90a0480-8860-458a-ae8d-e0707f54b5d9/deploy-status)](https://app.netlify.com/sites/deislabsio/deploys)
