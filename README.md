# ogygia all-islands build ships no client JS

Minimal reproduction for [ogygia](https://github.com/PuruVJ/ogygia) 0.4.3 with
`@sveltejs/kit` 2.70.2 and `vite` 8.2.1.

With `csr = false` on every route — ogygia's documented "all-islands" mode — `vite build`
reports success but emits **zero client JavaScript**. Every ogygia script 404s at runtime, so
no island hydrates and `<OgygiaRouter />` never boots.

## Run it

```sh
npm install
npm run build
find .svelte-kit/output/client -type f
```

Observed:

```
.svelte-kit/output/client/robots.txt
```

That is the entire client output. Meanwhile `.svelte-kit/ogygia-island-deps.json` exists and
names a chunk that is not on disk:

```json
{"/_app/immutable/ogygia-island.322bcc3ca43a.js":["/_app/immutable/disclose-version-BMsYaV5z.js"]}
```

The SSR bundle references that same path, so serving the build 404s it.

## Confirming the trigger

Add one route that keeps CSR on:

```sh
mkdir -p src/routes/other
echo 'export const csr = true;' > src/routes/other/+page.js
echo '<p>other</p>' > src/routes/other/+page.svelte
rm -rf .svelte-kit && npm run build
find .svelte-kit/output/client -name 'ogygia*'
```

Now both chunks ship:

```
.svelte-kit/output/client/_app/immutable/ogygia-island.322bcc3ca43a.js
.svelte-kit/output/client/_app/immutable/ogygia-runtime.860ebb6830eb.js
```

`/` is still `csr = false` and its island hydrates correctly. The failure needs *every* route to
be `csr = false`.

## Variant: hard crash instead of silent failure

Delete `static/` and rebuild. With no static asset to copy back, nothing recreates
`.svelte-kit/output/client`, and Kit's prerender throws:

```
Error: ENOENT: no such file or directory, scandir '.../.svelte-kit/output/client'
    at prerender (@sveltejs/kit/src/core/postbuild/prerender.js:205:24)
```

## Cause

`ogygia` is `enforce: 'pre'`, so its `buildStart` runs `runStandaloneClientBuild()`
(`dist/vite/index.js:571`, call at `:590`) and writes the runtime + island chunks into
`.svelte-kit/output/client`. Kit's own `buildStart` then runs `rimraf(out)`
(`@sveltejs/kit/src/exports/vite/index.js:1218`) and deletes the tree. Measured by
instrumenting both, one millisecond apart:

```
[ogygia] standalone build wrote chunks at 1786323102952
[kit]    rimraf(.svelte-kit/output)      at 1786323102953
```

Kit skips its own client build when every node has `page_options.csr === false`
(same file, `:1334`), so nothing recreates them. One `csr = true` route flips that
predicate and ogygia's islands are emitted through Kit's client build instead.

## Versions

| package | version |
| --- | --- |
| ogygia | 0.4.3 |
| @sveltejs/kit | 2.70.2 |
| vite | 8.2.1 |
| @sveltejs/vite-plugin-svelte | 7.3.0 |
| svelte | 5.56.8 |
