# Mendel Core - Production resolution of file system based experiments

Mendel Core loads all pre-built Mendel manifests in memory on a per process basis and evaluates the variations in a request time (or on a per user) basis.

```
+-----------------------------------------------+
|                                               |
|  MendelTrees long running                     |
|  instance                                     |
|                                               |
|          (contains all variations of          |
|            all pre-compiled files)            |
|                                               |
+----+--------------+------------+----------+---+
     |              |            |          |
     v              v            v          |
                                            |
+-----------+ +-----------+ +-----------+   v
| Tree for  | | Tree for  | | Tree for  |
| variations| | variations| | variations|  etc
| A         | | B, C      | | A, C      |
+-----------+ +-----------+ +-----------+
```

Each tree representation can be used to server side render, to hash generation or to resolve a bundle for a particular variation or for a particular set of variations.

## Mendel hashing - why is it needed and how we do it

In order to understand the problems we need to solve, be sure to recap the differences between multivariate and multilayer concepts. Assuming you understand that, let's analyze a simple scenario:


**4 experiments, divided in 2 layers, 2 experiments in each layer**


Since Mendel is based on file system we can understand the potential bundles for this scenario with the following diagram:

[![4 experiments in 2 layers graph](https://cdn.rawgit.com/yahoo/mendel/master/docs/Mendel-4-buckets-2-layers.svg)](../../docs/Mendel-4-buckets-2-layers.svg)

The image above can be used to understand a number of Mendel responsibilities. Here are the ones relevant to `mendel-core`:

  1. With 4 experiments in those layers Mendel can potentially generate 9 bundles.
  2. If file `F6` is changed in your application between one version and the other, it only affects 3 bundles

That's why `mendel-core` generates a hash for each bundle. The hash is based in the file list for a bundle (or what we can a tree representation), and the content of each file. This is very similar to how `git` works internally, but it is meant to be evaluated dynamically in production. The reason this is not feasible at build time is simple: We are analyzing 4 buckets in 2 layers, but large applications can yield a huge number of variations, for instance, 40 experiments evenly distributed in 8 layers will yield 6.700 permutations.

#### Mendel Hashing algorithm

The Mendel hash is a binary format encoded in URLSafeBase64 (RFC4648 section 5). The binary has the format below, where each number in parenthesis is the number of bytes used for a particular information:

```
      1           2           3           4          5          6
+------------+---------+--------------+---------+----------+---------+
| ascii(6*8) | uint(8) | loop uint(8) | uint(8) | uint(16) | bin(20) |
| === mendel |         | !== 255      | == 255  |          |         |
+------------+---------+--------------+---------+----------+---------+
```

The 6 pieces stand for:

1. ID: The string “mendel” (lowercase) in ascii encoding
2. VERSION: The version of this binary, right now it is version 1, we reserved this for future compatibility
3. BRANCHES_ARRAY: 0+ segments of 8 bit unsigned integers, that are different than 255
4. BRANCHES_LIMITER: Integer with value 255 that marks end of the branches
5. FILE_COUNT: The number of total files that were hashed during tree walking
6. CONTENT_HASH: 20 byte sha1 binary

Mendel starts with ID and VERSION and will walk the manifest, starting in the entry points of the package. Each "dep" is a browserify-like payload and has a sha1 of the source code, and also the dependencies of this file. Mendel will collect sha1 of all files and count how many files were walked.

Every time Mendel finds a file with variations, it uses a one of two desired method for choosing a variation (discussed below), and it will then push **the index of the chosen variation** to the **branch index array**. The numbers are also appended to BRANCHES_ARRAY of the binary.

Once walking is done, Mendel adds BRANCHES_LIMITER, to the binary, adds FILE_COUNT with the number of files walked, and computes CONTENT_HASH, the sha1 of all walked sha1 of the bundle. The final binary is than encoded with URLSafeBase64.

The variations can be resolved in two ways: By an array of desired **variation names**. This is useful for generating the hash for the first time. The second way, is when we decode a hash binary, and walk the manifest to collect and bundle source code, using the **branch index array**.

Because we need to make sure the contents are the same requested by user, the hash is calculated both when generating it for the first time (when generating HTML request) and when collecting the source code payload (when dynamically serving the bundle). If it is a match, the source code can be concatenated using `browser_pack`.


## Reference Usage

Usually, you can use the `mendel-middleware` instead of using `mendel-core` directly. In case you need advanced use of Mendel, the minimal server bellow should be enough for you to start your custom implementation.

```js
const MendelTrees = require('mendel-core');

const trees = MendelTrees(); // if no options passed, will read .mendelrc

function requestHandler(request, response) {
    // bundle should match .mendelrc bundle and can be multiple bundles
    const bundle = 'main';

    if (/index.html/.test(request.url)) {
        // variations can come from request cookie
        const variations = ['base'];
        // if you have multiple bundles you need to find one tree per bundle
        const tree = trees.findTreeForVariations(bundle, variations);
        // bundle url can be wrapped in a cookie-less CDN url
        const src = '/' + bundle + '.' + tree.hash + '.js';
        const script = '<script src="'+ src + '"></script>';
        response.end('<html><head>'+ script + '</head></html>');
    } else {
        // if you have multiple bundles you will need routes based on bundle
        const jsPath = new RegExp('/' + bundle + '\.(.*)\.js');
        if (jsPath.test(request.url)) {
            // your CDN can safely cache this url without cookies
            // and without "Vary" header
            const hash = request.url.match(jsPath)[1];
            const tree = trees.findTreeForHash(bundle, hash);
            response.end(JSON.stringify(tree)); // use your bundler
        } else {
            response.end('Not found');
        }
    }
}

const http = require('http');
const server = http.createServer(requestHandler);

server.listen(3000);
```

Usually, you won't be serving a JSON representation for your bundle, you will instead add error control, conflict detection and etc and eventually pack your bundle using UMD, browser-pack, rollup or even just concatenating all sources on global scope. Here is a minimal example of using browser pack:

```js
// on the example above, add this line at the top of the file

const bpack = require('browser-pack');

// and replace the line `response.end(JSON.stringify(tree));` by:

if (!tree || tree.error || tree.conflicts) {
    return response.end('Error ' + tree && tree.error || tree.conflictList);
}

const pack = bpack({raw: true, hasExports: true});
pack.pipe(response);

const modules = tree.deps.filter(Boolean);
for (var i = 0; i < modules.length; i++) {
    pack.write(modules[i]);
}
pack.end();
```

Please, see `mendel-middlware` for a production ready implementation of mendel-core with an express compatible API.

Please, see `mendel-development-middleware` for a viable implementation of development bundles without using `mendel-core` and with cached re-bundling based on source file changes.