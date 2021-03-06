# Adding Hashes to Filenames

Even though the generated build works the file names it uses is problematic. It doesn't allow to leverage client level cache efficiently as there's no way tell whether or not a file has changed. Cache invalidation can be achieved by including a hash to the filenames.

## Placeholders

Webpack provides **placeholders** for this purpose. These strings are used to attach specific information to webpack output. The most valuable ones are:

- `[id]` - Returns the chunk id.
- `[path]` - Returns the file path.
- `[name]` - Returns the file name.
- `[ext]` - Returns the extension. `[ext]` works for most available fields. `MiniCssExtractPlugin` is a notable exception to this rule.
- `[hash]` - Returns the build hash. If any portion of the build changes, this changes as well.
- `[chunkhash]` - Returns an entry chunk-specific hash. Each `entry` defined in the configuration receives a hash of its own. If any portion of the entry changes, the hash will change as well. `[chunkhash]` is more granular than `[hash]` by definition.
- `[contenthash]` - Returns a hash generated based on content.

It's preferable to use particularly `hash` and `contenthash` only for production purposes as hashing doesn't do much good during development.

T> It's possible to slice `hash` and `contenthash` using specific syntax: `[contenthash:4]`. Instead of a hash like `8c4cbfdb91ff93f3f3c5` this would yield `8c4c`.

T> There are more options available, and you can even modify the hashing and digest type as discussed at [loader-utils](https://www.npmjs.com/package/loader-utils#interpolatename) documentation.

### Example placeholders

Assume you have the following configuration:

```javascript
{
  output: {
    path: PATHS.build,
    filename: "[name].[contenthash].js",
  },
},
```

Webpack would generate filenames like these based on it:

```bash
main.d587bbd6e38337f5accd.js
vendor.dc746a5db4ed650296e1.js
```

If the file contents related to a chunk are different, the hash changes as well, thus the cache gets invalidated. More accurately, the browser sends a new request for the new file. If only `main` bundle gets updated, only that file needs to be requested again.

The same result can be achieved by generating static filenames and invalidating the cache through a querystring (i.e., `main.js?d587bbd6e38337f5accd`). The part behind the question mark invalidates the cache. According to [Steve Souders](http://www.stevesouders.com/blog/2008/08/23/revving-filenames-dont-use-querystring/), attaching the hash to the filename is the most performant option.

{pagebreak}

## Setting up hashing

The build needs tweaking to generate proper hashes. Adjust as follows:

**webpack.config.js**

```javascript
const productionConfig = merge([
leanpub-start-insert
  {
    output: {
      chunkFilename: "[name].[contenthash:4].js",
      filename: "[name].[contenthash:4].js",
    },
  },
leanpub-end-insert
  ...
  parts.loadImages({
    options: {
      limit: 15000,
leanpub-start-delete
      name: "[name].[ext]",
leanpub-end-delete
leanpub-start-insert
      name: "[name].[contenthash:4].[ext]",
leanpub-end-insert
    },
  }),
  ...
]);
```

{pagebreak}

To make sure extracted CSS receives hashes as well, adjust:

**webpack.parts.js**

```javascript
exports.extractCSS = ({ options = {}, loaders = [] } = {}) => {
  return {
    ...
    plugins: [
      new MiniCssExtractPlugin({
leanpub-start-delete
        filename: "[name].css",
leanpub-end-delete
leanpub-start-insert
        filename: "[name].[contenthash:4].css",
leanpub-end-insert
      }),
    ],
  };
};
```

W> The hashes have been sliced to make the output fit better in the book. In practice, you can skip slicing them.

If you generate a build now (`npm run build`), you should see something:

```bash
Hash: d165393a1d50d17e439d
Version: webpack 4.43.0
Time: 3676ms
Built at: 07/10/2020 4:07:37 PM
                     Asset       Size  Chunks                         Chunk Names
                 2.426a.js  191 bytes       2  [emitted] [immutable]
     2.426a.js.LICENSE.txt   15 bytes          [emitted]
                index.html  285 bytes          [emitted]
             main.0166.css   1.61 KiB       0  [emitted] [immutable]  main
              main.492e.js   2.73 KiB       0  [emitted] [immutable]  main
  main.492e.js.LICENSE.txt   15 bytes          [emitted]
            vendor.2ef5.js    126 KiB       1  [emitted] [immutable]  vendor
vendor.2ef5.js.LICENSE.txt  806 bytes          [emitted]
Entrypoint main = vendor.2ef5.js main.0166.css main.492e.js
```

The files have neat hashes now. To prove that it works for styling, you could try altering _src/main.css_ and see what happens to the hashes when you rebuild.

There's one problem, though. If you change the application code, it invalidates the vendor file as well! Solving this requires extracting a webpack **runtime**, but before that, you can improve the way the production build handles module IDs.

## Conclusion

Including hashes related to the file contents to their names allows to invalidate them on the client side. If a hash has changed, the client is forced to download the asset again.

To recap:

- Webpack's **placeholders** allow you to shape filenames and enable you to include hashes to them.
- The most valuable placeholders are `[name]`, `[contenthash]`, and `[ext]`. A content hash is derived based on the chunk content.
- If you are using `MiniCssExtractPlugin`, you should use `[contenthash]` as well. This way the generated assets get invalidated only if their content changes.

Even though the project generates hashes now, the output isn't flawless. The problem is that if the application changes, it invalidates the vendor bundle as well. The next chapter digs deeper into the topic and shows you how to extract webpack **runtime** to resolve the issue.
