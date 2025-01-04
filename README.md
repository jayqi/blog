This is the source for my personal website and blog. It is a static site using [Hugo](https://gohugo.io/).

## Run development server

```bash
hugo server
```

## Update PaperMod theme

Update the submodule:

```bash
git submodule foreach git pull origin
```

To update to a specific version:

```bash
git submodule foreach git pull origin v8.0
```
