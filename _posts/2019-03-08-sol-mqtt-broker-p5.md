---
layout: post
title: "Sol - An MQTT broker from scratch. Part 5 - Topic abstraction"
description: "Writing an MQTT broker from scratch, to really understand something you have to build it."
categories: c unix tutorial structures
---

In the [Part 4](../sol-mqtt-broker-p4) we
explored some useful concepts and implemented two data structures on top of
those concepts.

<!--more-->

The MQTT protocol defines an abstraction named **topic**, in essence, a string,
a label that is used to filter messages for each client. It follow an
hierarchical model, described by some simple rules:

- a topic is an UTF-8 encoded string of max length of 65535 bytes
- a forward `/` is used to separate different levels, much like directories in
  a filesystem
- `#` can be used as a wildcard for multilevel subscription, to subscribe to
  multiple topics by following the hierarchy, e.g.: **foo/bar/#** will subscribe to:
  - foo/bar
  - foo/bar/baz
  - foo/bar/bat/yop
- `+` can be used for single level wildcard subscription, e.g: **foo/+/baz** will
  subscribe to:
  - foo/bar/baz
  - foo/zod/baz
  - foo/nop/baz

They share some traits with message queues, but way simpler, lightweight and
less powerful.

### Handling topic abstraction: the trie

We move now to the **trie**, the structure of choice to store topics. Trie is a
type of tree in which each node is a prefix for a key, the node position
define the keys and the associated values are set on the last node of each key.
They provide a big-O runtime complexity of O(m) on worst case, for insertion
and lookup, where m is the length of the key. The main advantage is the
possibility to query the tree by prefix, executing range scans in an easy way.

<hr>
**src/trie.h**
<hr>

{% highlight c %}

#ifndef TRIE_H
#define TRIE_H

#include <stdio.h>
#include <stdbool.h>
#include "list.h"

typedef struct trie Trie;

/*
 * Trie node, it contains a fixed size array (every node can have at max the
 * alphabet length size of children), a flag defining if the node represent
 * the end of a word and then if it contains a value defined by data.
 */
struct trie_node {
    char chr;
    List *children;
    void *data;
};

/*
 * Trie ADT, it is formed by a root struct trie_node, and the total size of the
 * Trie
 */
struct trie {
    struct trie_node *root;
    size_t size;
};

// Returns new trie node (initialized to NULLs)
struct trie_node *trie_create_node(char);

// Returns a new Trie, which is formed by a root node and a size
struct trie *trie_create(void);

void trie_init(Trie *);

// Return the size of the trie
size_t trie_size(const Trie *);

/*
 * The leaf represents the node with the associated data
 *           .
 *          / \
 *         h   s: s -> value
 *        / \
 *       e   k: hk -> value
 *      /
 *     l: hel -> value
 *
 * Here we got 3 <key:value> pairs:
 * - s   -> value
 * - hk  -> value
 * - hel -> value
 */
void *trie_insert(Trie *, const char *, const void *);

bool trie_delete(Trie *, const char *);

/* Returns true if key presents in trie, else false, the last pointer to
   pointer is used to store the value associated with the searched key, if
   present */
bool trie_find(const Trie *, const char *, void **);

void trie_node_free(struct trie_node *, size_t *);

void trie_release(Trie *);

/* Remove all keys matching a given prefix in a linear time complexity (O(n))*/
void trie_prefix_delete(Trie *, const char *);

/*
 * Apply a given function to all nodes which keys match a given prefix. The
 * function accepts two arguments, a struct trie_node pointer which correspond
 * to each node on the trie after the prefix node and a void pointer, used for
 * additional data which can be useful to the execution of `mapfunc`.
 */
void trie_prefix_map_tuple(Trie *, const char *,
                           void (*mapfunc)(struct trie_node *, void *), void *);

#endif

{% endhighlight %}
<hr>

Implementation of this data structure is a bit tricky and there are lot of
different approaches, the most simple one would involve the use of a fixed
length array on each node of the trie, with the complete alphabet size as
length.

{% highlight c %}

#define ALPHABET_SIZE 94

/* Trie node, it contains a fixed size array (every node can have at max the
   alphabet length size of children), a flag defining if the node represent
   the end of a word and then if it contains a value defined by data. */

struct trie_node {
    struct trie_node *children[ALPHABET_SIZE];
    void *data;
};

{% endhighlight %}

The biggest advantage of the trie, beside the possibility of applying range
queries on the keyspace (this one will come handy for wildcard subscriptions
and management of topics), is in terms of average performances over hashtables
or B-Trees, it gives in fact on worst case O(L) insert, delete and search time
complexity where L is the length of the key. This comes at a cost, the main
drawback is that the structure itself, following this implementation is really
memory hungry. In the example case with an alphabet of size 96, starting from
the `<space>` character and ending with the `~` each node has 96 NULL pointer
to their children, this means that on a 64 bit arch machine with 8 bytes per
pointer we effectively have 768 bytes of allocated space per node. Let's
briefly analyze a case:

- insert key foo
- insert key foot

So we have the root f which have 1 non-null pointer o, the children have
another 1 non-null pointer o, here lies our first value for key foo, and the
last children o have 1 non-null pointer for t, which will also store our second
value for foot key. So we have a total of 4 nodes, that means 4 * 96 = 384
pointers, of which only 4 are used. Now that's a lot of wasted space! There's
some techniques to mitigate this humongous amount of wasting bytes while
maintaining good time-complexity performances, called compressed trie and
adaptive trie.

Without going too deep into these concepts, best solutions so far seems three:

- Use a single dynamic array (vector) in the Trie structure, each node must have
  a pointer to that vector and an array char children_idx[ALPHABET_SIZE] which
  store the index in the main vector for each children;

- Use sized node based on the number of children and adapting lookup
  algorithm accordingly e.g. with # children <= 4 use a fixed length array
  of 4 pointers and linear search for children, growing up set fixed steps of
  size and use a different mapping for characters on the array of children
  pointers.

- Replace the fixed length array on each node with a singly-linked **linked
  list**, maintained sorted on each insertion, this way there's an average
  performance of O(n/2) on each search, equal to O(n), which is the best case
  possible with the linked list data structure.

Luckily we've just written a **linked list** before (Perhaps I knew the answer?
:P) but also a **vector** could do well.

Let's implement our trie with the third solution:

<hr>
**src/trie.c**
<hr>

{% highlight c %}

#include <assert.h>
#include <stdlib.h>
#include <string.h>
#include "list.h"
#include "trie.h"

static struct list_node *merge_tnode_list(struct list_node *list1,
                                          struct list_node *list2) {
    struct list_node dummy_head = { NULL, NULL }, *tail = &dummy_head;
    while (list1 && list2) {

        /* cast to cluster_node */
        char chr1 = ((struct trie_node *) list1->data)->chr;
        char chr2 = ((struct trie_node *) list2->data)->chr;

        struct list_node **min = chr1 <= chr2 ? &list1 : &list2;
        struct list_node *next = (*min)->next;
        tail = tail->next = *min;
        *min = next;
    }
    tail->next = list1 ? list1 : list2;
    return dummy_head.next;
}

struct list_node *merge_sort_tnode(struct list_node *head) {
    struct list_node *list1 = head;
    if (!list1 || !list1->next)
        return list1;
    /* find the middle */
    struct list_node *list2 = bisect_list(list1);
    return merge_tnode_list(merge_sort_tnode(list1), merge_sort_tnode(list2));
}

/* Search for a given node based on a comparison of char stored in structure
 * and a value, O(n) at worst
 */
static struct list_node *linear_search(const List *list, int value) {
    if (!list || list->len == 0)
        return NULL;
    for (struct list_node *cur = list->head; cur != NULL; cur = cur->next) {
        if (((struct trie_node *) cur->data)->chr == value)
            return cur;
        else if (((struct trie_node *) cur->data)->chr > value)
            break;
    }
    return NULL;
}

/* Auxiliary comparison function, uses on list searches, this one compare the
 * char field stored in each struct trie_node structure contained in each node of the
 * list.
 */
static int with_char(void *arg1, void *arg2) {
    struct trie_node *tn1 = ((struct list_node *) arg1)->data;
    struct trie_node *tn2 = ((struct list_node *) arg2)->data;
    if (tn1->chr == tn2->chr)
        return 0;
    return -1;
}

// Check for children in a struct trie_node, if a node has no children is considered
// free
static bool trie_is_free_node(const struct trie_node *node) {
    return node->children->len == 0 ? true : false;
}

static struct trie_node *trie_node_find(const struct trie_node *node,
                                        const char *prefix) {
    struct trie_node *retnode = (struct trie_node *) node;

    // Move to the end of the prefix first
    for (; *prefix; prefix++) {

        // O(n), the best we can have
        struct list_node *child = linear_search(retnode->children, *prefix);

        // No key with the full prefix in the trie
        if (!child)
            return NULL;
        retnode = child->data;
    }
    return retnode;
}

// Returns new trie node (initialized to NULL)
struct trie_node *trie_create_node(char c) {
    struct trie_node *new_node = malloc(sizeof(*new_node));
    if (new_node) {
        new_node->chr = c;
        new_node->data = NULL;
        new_node->children = list_create(NULL);
    }
    return new_node;
}

// Returns new Trie, with a NULL root and 0 size
Trie *trie_create(void) {
    Trie *trie = malloc(sizeof(*trie));
    trie_init(trie);
    return trie;
}

void trie_init(Trie *trie) {
    trie->root = trie_create_node(' ');
    trie->size = 0;
}

size_t trie_size(const Trie *trie) {
    return trie->size;
}

/*
 * If not present, inserts key into trie, if the key is prefix of trie node,
 * just marks leaf node by assigning the new data pointer. Returns a pointer
 * to the new inserted data.
 *
 * Being a Trie, it should guarantees O(m) performance for insertion on the
 * worst case, where `m` is the length of the key.
 */
static void *trie_node_insert(struct trie_node *root, const char *key,
                              const void *data, size_t *size) {
    struct trie_node *cursor = root;
    struct trie_node *cur_node = NULL;
    struct list_node *tmp = NULL;

    // Iterate through the key char by char
    for (; *key; key++) {

        /*
         * We can use a linear search as on a linked list O(n) is the best find
         * algorithm we can use, as binary search would have the same if not
         * worse performance by not having direct access to node like in an
         * array.
         *
         * Anyway we expect to have an average O(n/2) -> O(n) cause at every
         * insertion the list is sorted so we expect to find our char in the
         * middle on average.
         *
         * As a future improvement it's advisable to substitute list with a
         * B-tree or RBTree to improve searching complexity to O(logn) at best,
         * avg and worst while maintaining O(n) space complexity, but it really
         * depends also on the size of the alphabet.
         */
        tmp = linear_search(cursor->children, *key);

        // No match, we add a new node and sort the list with the new added link
        if (!tmp) {
            cur_node = trie_create_node(*key);
            cursor->children = list_push(cursor->children, cur_node);
            cursor->children->head = merge_sort_tnode(cursor->children->head);
        } else {
            // Match found, no need to sort the list, the child already exists
            cur_node = tmp->data;
        }
        cursor = cur_node;
    }

    /*
     * Clear out if already taken (e.g. we are in a leaf node), rc = 0 to not
     * change the trie size, otherwise 1 means that we added a new node,
     * effectively changing the size
     */
    if (!cursor->data)
        (*size)++;
    cursor->data = (void *) data;
    return cursor->data;
}

/*
 * Private function, iterate recursively through the trie structure starting
 * from a given node, deleting the target value
 */
static bool trie_node_recursive_delete(struct trie_node *node, const char *key,
                                       size_t *size, bool *found) {
    if (!node)
        return false;

    // Base case
    if (*key == '\0') {
        if (node->data) {

            // Update found flag
            *found = true;

            // Free resources, covering the case of a sub-prefix
            if (node->data) {
                free(node->data);
                node->data = NULL;
            }
            free(node->data);
            node->data = NULL;
            if (*size > 0)
                (*size)--;

            // If empty, node to be deleted
            return trie_is_free_node(node);
        }
    } else {

        // O(n), the best we can have
        struct list_node *cur = linear_search(node->children, *key);
        if (!cur)
            return false;
        struct trie_node *child = cur->data;
        if (trie_node_recursive_delete(child, key + 1, size, found)) {

            // Messy solution, requiring probably avoidable allocations
            struct trie_node t = {*key, NULL, NULL};
            struct list_node tmp = {&t, NULL};
            list_remove(node->children, &tmp, with_char);

            // last node marked, delete it
            trie_node_free(child, size);

            // recursively climb up, and delete eligible nodes
            return (!node->data && trie_is_free_node(node));
        }
    }
    return false;
}

/*
 * Returns true if key is present in trie, else false. Also for lookup the
 * big-O runtime is guaranteed O(m) with `m` as length of the key.
 */
static bool trie_node_search(const struct trie_node *root,
                             const char *key, void **ret) {

    // Walk the trie till the end of the key
    struct trie_node *cursor = trie_node_find(root, key);
    *ret = (cursor && cursor->data) ? cursor->data : NULL;

    // Return false if no complete key found, true otherwise
    return !*ret ? false : true;
}

/*
 * Insert a new key-value pair in the Trie structure, returning a pointer to
 * the new inserted data in order to simplify some operations as the addition
 * of expiring keys with a set TTL.
 */
void *trie_insert(Trie *trie, const char *key, const void *data) {
    assert(trie && key);
    return trie_node_insert(trie->root, key, data, &trie->size);
}

bool trie_delete(Trie *trie, const char *key) {
    assert(trie && key);
    bool found = false;
    if (strlen(key) > 0)
        trie_node_recursive_delete(trie->root, key, &(trie->size), &found);
    return found;
}

bool trie_find(const Trie *trie, const char *key, void **ret) {
    assert(trie && key);
    return trie_node_search(trie->root, key, ret);
}

/*
 * Remove and delete all keys matching a given prefix in the trie
 * e.g. hello*
 * - hello
 * hellot
 * helloworld
 * hello
 */
void trie_prefix_delete(Trie *trie, const char *prefix) {
    assert(trie && prefix);

    // Walk the trie till the end of the key
    struct trie_node *cursor = trie_node_find(trie->root, prefix);

    // No complete key found
    if (!cursor)
        return;

    // Simply remove the key if it has no children, no need to clear the list
    if (cursor->children->len == 0) {
        trie_delete(trie, prefix);
        return;
    }
    struct list_node *cur = cursor->children->head;
    // Clear out all possible sub-paths
    for (; cur; cur = cur->next) {
        trie_node_free(cur->data, &(trie->size));
        cur->data = NULL;
    }

    // Set the current node (the one storing the last character of the prefix)
    // as a leaf and delete the prefix key as well
    trie_delete(trie, prefix);
    list_clear(cursor->children, 1);
}

/* Iterate through children of each node starting from a given node, applying
   a defined function which take a struct trie_node as argument */
static void trie_prefix_map_func2(struct trie_node *node,
                                  void (*mapfunc)(struct trie_node *, void *), void *arg) {
    if (trie_is_free_node(node)) {
        mapfunc(node, arg);
        return;
    }
    struct list_node *child = node->children->head;
    for (; child; child = child->next)
        trie_prefix_map_func2(child->data, mapfunc, arg);
    mapfunc(node, arg);
}

/*
 * Apply a function to every key below a given prefix, if prefix is null the
 * function will be applied to all the trie. The function applied accepts an
 * additional arguments for optional extra data.
 */
void trie_prefix_map_tuple(Trie *trie, const char *prefix,
                           void (*mapfunc)(struct trie_node *, void *), void *arg) {
    assert(trie);
    if (!prefix) {
        trie_prefix_map_func2(trie->root, mapfunc, arg);
    } else {

        // Walk the trie till the end of the key
        struct trie_node *node = trie_node_find(trie->root, prefix);

        // No complete key found
        if (!node)
            return;

        // Check all possible sub-paths and add to count where there is a leaf
        trie_prefix_map_func2(node, mapfunc, arg);
    }
}

/* Release memory of a node while updating size of the trie */
void trie_node_free(struct trie_node *node, size_t *size) {

    // Base case
    if (!node)
        return;

    // Recursive call to all children of the node
    if (node->children) {
        struct list_node *cur = node->children->head;
        for (; cur; cur = cur->next)
            trie_node_free(cur->data, size);
        list_release(node->children, 0);
        node->children = NULL;
    }

    // Release memory on data stored on the node
    if (node->data) {
        free(node->data);
        if (*size > 0)
            (*size)--;
    } else if (node->data) {
        free(node->data);
        if (*size > 0)
            (*size)--;
    }

    // Release the node itself
    free(node);
}

void trie_release(Trie *trie) {
    if (!trie)
        return;
    trie_node_free(trie->root, &(trie->size));
    free(trie);
}

{% endhighlight %}
<hr>

Well, we have enough in our plate for now, our project should now have 3 more
modules:

{% highlight bash %}
sol/
 ├── src/
 │    ├── mqtt.h
 |    ├── mqtt.c
 │    ├── network.h
 │    ├── network.c
 │    ├── list.h
 │    ├── list.c
 │    ├── hashtable.h
 │    ├── hashtable.c
 │    ├── trie.h
 │    ├── trie.c
 │    ├── util.h
 │    ├── util.c
 │    ├── pack.h
 │    └── pack.c
 ├── CHANGELOG
 ├── CMakeLists.txt
 ├── COPYING
 └── README.md
{% endhighlight %}

The [Part 6](../sol-mqtt-broker-p6) awaits
with the server side handlers.
