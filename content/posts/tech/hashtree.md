---
title: "hashtree"
date: '2023-09-18'
draft: false
summary: 'hashtree is a new data structure to efficiently store trees in
unordered hashtables.'
tags:
  - data structure
image: https://upload.wikimedia.org/wikipedia/commons/thumb/5/5f/Tree_%28computer_science%29.svg/800px-Tree_%28computer_science%29.svg.png
author: Manas
---

A data structure which is almost a map but instead of mapping keys to
concrete (`singular`) values, this maps its keys to a tree of values. Mapping a
tree of values is expensive during insertion and searching (as compared to
ordinary std::map) but it may provide extensive information according to the
users' needs.

A hashtree will require two types, `key_type` and `mapped_type` (while a
`value_type` will be a pair of these two). Moreover, the user must either
implement a policy for internal tree representation (or it defaults to using a
rbtree) which should provide `insert` and `remove` methods which operate on the
`mapped_value`. Consider the following examples,

```cpp

#include <hashtree>
using namespace std;
auto foo() {
  hashtree<string, int> ht;
  ht.insert({"foo", 1}); // inserts `foo` key and creates a new tree with node 1
  ht.insert({"foo", 2}); // inserts a new child node (in rbtree) after node 1
  ht.insert({"foo", 3}); // inserts a new child node (in rbtree) after node 1
                         // structure looks:
                         // foo ---->  1
                         //          2   3
}

```

If you want to override rbtree implementation, you can provide an alternative
tree policy:

```cpp
#include <hashtree>
using namespace std;

// FIXME This is not compile-checked.
template <class T> struct AVLTree {
  typedef typename hashtree::mapped_type mapped_type;
  typedef typename hashtree::key_type key_type;
  typedef typename hashtree::insert_return_type insert_return_type;
  T* __root;

public:
  insert_return_type insert(T &) {}
  mapped_type remove(T &) {}
};

auto foo() {
  hashtree<string, int, AVLTree> ht;
  // this insert method is hashtree's member, which under the hood calls above
  // provided insert method through the tree policy (AVLTree::insert).
  ht.insert({"foo", 1}); // inserts `foo` key and creates a new tree with node 1
  ht.insert({"foo", 2}); // inserts a new child node (in AVLTree) after node 1
  ht.insert({"foo", 3}); // inserts a new child node (in AVLTree) after node 1
                         // structure looks:
                         // foo ---->  1
                         //          2   3
}

```

A hashtree with a single key and a tree of values will be:

```text
hashtree = {key1 : tree1}

  where : tree1 =       root
                  l1-left   l1-right
                        ...
```

and there are following methods.

1. `insert` : a template member which will take a _key-type_ object and a
   _value-type_ object, as `insert(key, value)`. If _key_ isn't present then a
   new node containing hashed key and constructing a tree with value node using
   TreePolicy methods will be inserted.

   And if this key is already present then a `insert` will be called on its
   value object. This `insert` method will also be provided through the
   TreePolicy.

2. `delete` : ...
3. `operator[]` : ...


TreePolicy will provide insert method which enables us to insert a new node
(_value-type_) into the tree corresponding to a key.




#### You can choose how to grow your company

More companies die of drowning in opportunities than of starvation. Focusing your company on a very specific target market, finding a way to dominate that constrained market and then expanding from there is a well-documented approach to significantly reducing the risk of a startup’s failure.

#### You can weather storms

But, you say, what about market downturns? What if it becomes harder to raise money? Won’t a big cache of cash ensure survivability through hard times?

Not necessarily. The single largest determinant of whether you can survive a downturn in the funding market is your burn rate. You can weather storms better if you are capital efficient. If your cash burn is low — or even better, if you can get to cash-flow break-even — you can get through hard times.

Keep this mantra in mind: less burn, more options.

#### You can choose your exit strategy

Choosing an exit path is the greatest startup existential question of all. It’s also where raising too much capital limits your optionality the most. Venture capitalists don’t like to talk about this — shooting for the moon is expected.


