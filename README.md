RBush
=====

RBush is a high-performance JavaScript library for 2D **spatial indexing** of points and rectangles.
It's based on an optimized **R-tree** data structure with **bulk insertion** support.

*Spatial index* is a special data structure for points and rectangles
that allows you to perform queries like "all items within this bounding box" very efficiently
(e.g. hundreds of times faster than looping over all items).
It's most commonly used in maps and data visualizations.

[![Build Status](https://travis-ci.org/mourner/rbush.svg?branch=master)](https://travis-ci.org/mourner/rbush)
[![](https://img.shields.io/badge/simply-awesome-brightgreen.svg)](https://github.com/mourner/projects)

## Demos

The demos contain visualization of trees generated from 50k bulk-loaded random points.
Open web console to see benchmarks;
click on buttons to insert or remove items;
click to perform search under the cursor.

* [uniformly distributed random data](http://mourner.github.io/rbush/viz/viz-uniform.html)
* [randomly clustered data](http://mourner.github.io/rbush/viz/viz-cluster.html)

## Usage

### Creating a Tree

```js
var tree = rbush(9);
```

An optional argument to `rbush` defines the maximum number of entries in a tree node.
It drastically affects the performance, so you should adjust it
considering the type of data and search queries you perform.

### Adding and Removing Data

Insert an item:

```js
var item = [20, 40, 30, 50]; // [x1, y1, x2, y2]
tree.insert(item);
```

Remove a previously inserted item:

```js
tree.remove(item);
```

Clear all items:

```js
tree.clear();
```

Items inserted in the tree can have other arbitrary properties/elements that you can access later:

```js
var item1 = [20, 40, 30, 50, {foo: 'bar'}];
tree.insert(item1);

var item2 = [15, 15, 30, 30];
item2.foo = 'bar';
tree.insert(item2);
```

### Data Format

By default, RBush assumes the format of data points to be `[minX, minY, maxX, maxY]`
(bounding box coordinates, or just `[x, y, x, y]` for points).
You can customize this by providing an array with `minX`, `minY`, `maxX`, `maxY` accessor strings
as a second argument to `rbush` like this:

```js
var tree = rbush(9, ['.minLng', '.minLat', '.maxLng', '.maxLat']);
tree.insert({id: 'foo', minLng: 30, minLat: 50, maxLng: 40, maxLat: 60});
```

### Bulk-Inserting Data

Bulk-insert the given data into the tree:

```js
tree.load([
	[10, 10, 15, 20],
	[12, 15, 40, 64.5],
	...
]);
```

Bulk insertion is usually ~2-3 times faster than inserting items one by one.
After bulk loading (bulk insertion into an empty tree), subsequent query performance is also ~20-30% better.

When you do bulk insertion into an existing tree, it bulk-loads the given data into a separate tree
and inserts the smaller tree into the larger tree.
This means that bulk insertion works very well for clustered data (where items are close to each other),
but makes query performance worse if the data is scattered.

### Search

```js
var result = tree.search([40, 20, 80, 70]);
```

Returns an array of data items (points or rectangles) that the given bounding box (`[minX, minY, maxX, maxY]`) intersects.

```js
var allItems = tree.all();
```

Returns all items of the tree.

### Collisions

```js
var result = tree.collides([40, 20, 80, 70]);
```

Returns `true` if there are any items intersecting the given bounding box, otherwise `false`.


### Export and Import

```js
// export data as JSON object
var treeData = tree.toJSON();

// import previously exported data
var tree = rbush(9).fromJSON(treeData);
```

Importing and exporting as JSON allows you to use RBush on both the server (using Node.js) and the browser combined,
e.g. first indexing the data on the server and and then importing the resulting tree data on the client for searching.

### K-Nearest Neighbors

For "_k_ nearest neighbors around a point" type of queries for RBush,
check out [rbush-knn](https://github.com/mourner/rbush-knn).

## Performance

The following sample performance test was done by generating
random uniformly distributed rectangles of ~0.01% area and setting `maxEntries` to `16`
(see `debug/perf.js` script).
Performed with Node.js v5.2.0 on a Retina Macbook Pro 15 (mid-2012).

Test                         | RBush  | [old RTree](https://github.com/imbcmdth/RTree) | Improvement
---------------------------- | ------ | ------ | ----
insert 1M items one by one   | 4.7s   | 9.26s  | 2x
1000 searches of 0.01% area  | 0.06s  | 1.12s  | 20x
1000 searches of 1% area     | 0.43s  | 2.73s  | 6.3x
1000 searches of 10% area    | 2.19s  | 11.56s | 5.3x
remove 1000 items one by one | 0.02s  | 1.44s  | 65x
bulk-insert 1M items         | 1.38s  | n/a    | 6.7x

## Algorithms Used

* single insertion: non-recursive R-tree insertion with overlap minimizing split routine from R\*-tree (split is very effective in JS, while other R\*-tree modifications like reinsertion on overflow and overlap minimizing subtree search are too slow and not worth it)
* single deletion: non-recursive R-tree deletion using depth-first tree traversal with free-at-empty strategy (entries in underflowed nodes are not reinserted, instead underflowed nodes are kept in the tree and deleted only when empty, which is a good compromise of query vs removal performance)
* bulk loading: OMT algorithm (Overlap Minimizing Top-down Bulk Loading) combined with Floyd–Rivest selection algorithm
* bulk insertion: STLT algorithm (Small-Tree-Large-Tree)
* search: standard non-recursive R-tree search

## Papers

* [R-trees: a Dynamic Index Structure For Spatial Searching](http://www-db.deis.unibo.it/courses/SI-LS/papers/Gut84.pdf)
* [The R*-tree: An Efficient and Robust Access Method for Points and Rectangles+](http://dbs.mathematik.uni-marburg.de/publications/myPapers/1990/BKSS90.pdf)
* [OMT: Overlap Minimizing Top-down Bulk Loading Algorithm for R-tree](http://ftp.informatik.rwth-aachen.de/Publications/CEUR-WS/Vol-74/files/FORUM_18.pdf)
* [Bulk Insertions into R-Trees Using the Small-Tree-Large-Tree Approach](http://www.cs.arizona.edu/~bkmoon/papers/dke06-bulk.pdf)
* [R-Trees: Theory and Applications (book)](http://www.apress.com/9781852339777)

## Development

```bash
npm install  # install dependencies

npm test     # check the code with JSHint and run tests
npm run perf # run performance benchmarks
npm run cov  # report test coverage (with more detailed report in coverage/lcov-report/index.html)
```

## Compatibility

RBush should run on Node and all major browsers. The only caveat: IE 8 needs an [Array#indexOf polyfill](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Array/indexOf#Polyfill) for `remove` method to work.

## Changelog

#### 1.4.3 &mdash; May 17, 2016

- Fixed an error when inserting many empty bounding boxes.

#### 1.4.2 &mdash; Dec 16, 2015

- 50% faster insertion.

#### 1.4.1 &mdash; Sep 16, 2015

- Fixed insertion in IE8.

#### 1.4.0 &mdash; Apr 22, 2015

- Added `collides` method for fast collision detection.

#### 1.3.4 &mdash; Aug 31, 2014

- Improved bulk insertion performance for a large number of items (e.g. up to 100% for inserting a million items).
- Fixed performance regression for high node sizes.

#### 1.3.3 &mdash; Aug 30, 2014

- Improved bulk insertion performance by ~60-70%.
- Improved insertion performance by ~40%.
- Improved search performance by ~30%.

#### 1.3.2 &mdash; Nov 25, 2013

- Improved removal performance by ~50%. [#18](https://github.com/mourner/rbush/pull/18)

#### 1.3.1 &mdash; Nov 24, 2013

- Fixed minor error in the choose split axis algorithm. [#17](https://github.com/mourner/rbush/pull/17)
- Much better test coverage (near 100%). [#6](https://github.com/mourner/rbush/issues/6)

#### 1.3.0 &mdash; Nov 21, 2013

- Significantly improved search performance (especially on large-bbox queries — up to 3x faster). [#11](https://github.com/mourner/rbush/pull/11)
- Added `all` method for getting all of the tree items. [#11](https://github.com/mourner/rbush/pull/11)
- Made `toBBox`, `compareMinX`, `compareMinY` methods public, made it possible to avoid Content Security Policy issues by overriding them for custom format. [#14](https://github.com/mourner/rbush/pull/14) [#12](https://github.com/mourner/rbush/pull/12)

#### 1.2.5 &mdash; Nov 5, 2013

- Fixed a bug where insertion failed on a tree that had all items removed previously. [#10](https://github.com/mourner/rbush/issues/10)

#### 1.2.4 &mdash; Oct 25, 2013

- Added Web Workers support. [#9](https://github.com/mourner/rbush/pull/9)

#### 1.2.3 &mdash; Aug 30, 2013

- Added AMD support. [#8](https://github.com/mourner/rbush/pull/8)

#### 1.2.2 &mdash; Aug 27, 2013

- Eliminated recursion when recalculating node bboxes (on insert, remove, load).

#### 1.2.0 &mdash; Jul 19, 2013

First fully functional RBush release.
