# zpz.github.io

## Render the blog posts for development

Do

```
$ docker run --rm -v ~/work/src/zpz.github.io:/srv/jekyll -p 127.0.0.1:4000:4000 jekyll/jekyll:3.5 jekyll serve
```

then on a browser visit `localhost:4000`.
