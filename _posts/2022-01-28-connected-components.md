---
layout: post
title: "An Efficient Algorithm for 'Connected Components'"
excerpt_separator: <!--excerpt-->
tags: [python, docker]
---

A while back I was debugging some legacy code.
Profiling revealed that a few lines of code that computes
"connected components" was a major bottleneck.<!--excerpt-->

The concept of "connected components" here is simple: suppose we have
a number of sets, some of which overlap; we'll union any two sets that overlap,
and do this recursively; in the end, we'll get a number of non-ovrlapping sets,
and they are called "connected components".

For ease of description, we'll call the sets "components", the final non-overalapping
sets (or connected components) "groups", and elements in the sets "items".

The code uses a python package called [`networkx`](https://github.com/networkx/networkx);
it's as simple as the following:

```python
from typing import Iterable 
import networkx as nx


def connected_components(components: Iterable[Iterable[int]]):
    graph = nx.parse_adjlist(
            (' '.join(str(i) for i in c) for c in components),
            nodetype=int,
            )
    return list(nx.connected_components(graph))
```

Let's see it in action:

```
>>> COMPONENTS = [
...     (0, 1, 2, 3, 4, 5),
...     (0, 1, 8),
...     (2, 9),
...     (6, 7),
...     (5,),
...     (8, 9),
...     (10, 11, 12, 13),
...     (10, 14, 15),
...     ]
>>> from cc_nx import connected_components
>>> connected_components(COMPONENTS)
[{0, 1, 2, 3, 4, 5, 8, 9}, {6, 7}, {10, 11, 12, 13, 14, 15}]
>>>
```

This example is illustrated below:

![ex-1](/images/connected-components-1.png){:width="70%" height="70%"}


The connected components are depicted at the bottom:

![ex-2](/images/connected-components-2.png){:width="70%" height="70%"}

Please understand the problem in the diagram and compare with the result in the code above.

To make it a little more interesting and realistic, let's
randomize the order of the elements:

```
>>> COMPONENTS = [
...     (7, 6),
...     (5, 4, 3, 2, 1, 0),
...     (8, 1, 0),
...     (2, 9),
...     (8, 9),
...     (11, 13, 10, 12),
...     (10, 14, 15),
...     (5,),
...     ]
>>> connected_components(COMPONENTS)
[{6, 7}, {0, 1, 2, 3, 4, 5, 8, 9}, {10, 11, 12, 13, 14, 15}]
>>>
```

We see the algorithm has sorted the items within each group,
but there is no particular order between the groups.

This is all clean, concise, and nice. What's surprising is that
this apparently simple and uninsteresting step of processing turned out
to be a dominating bottleneck of the entire program, eclipsing all the
complex data pipelines and machine-learning modeling!
My hunch was that the `networkx` algorithm was poor crafted,
and the problem should be relatively easy. In fact, it's an interesting
little programming problem: it's clearly defined, rather generic, and may have a decent number of applications.
Without ever looking at the `networkx` source code, I started devising a new algorithm.


## The algorithm

It was a year ago, and my own algo took good parts of a day to finalize.
Let's assume all items across components are represented by sequential numbers
from `0` up to `n_items - 1`, hence each "component" is represented by a set of
such numbers. We need to work on these numbers to "union" overlapping components.

An ituition from the start of the effor is that it should be some kind of "sweep and mark"
algorithm. There is no avoiding walking through each component with its member items,
but let's try our best to do this walk only once (at least on the surface),
figure out and write down relations as we go,
and only do some trivial postprocessing on the findings after the single pass.
It will likely be quite procedural, with careful book-keeping along the way.
It should also watch out for careless use of dynamic `lists`: create new ones, `append` to them, and such.
The human conveniences all take computer time!
Beware of clever functional style that could triggerer recursion.
I did not start thinking down that direction at all
(there could be a solution in that direction but I don't know).

So, one pass through the items. That's the goal.

How do we represent the resultant groups? What are important intermediate relations to record?

Eventually, each group will be represented by its member items, i.e. the sequential numbers of the items.
However, as a middle step, we could also represent a group by its member *components*.
Then it will be a cheap postprocess to get the member *items* of the group.
A potential advantage is that the component-representation of groups will be small,
and can lend to using dynamic lists at will, if the algo so requires.

Let's stare at diagram 1. Suppose we walk over "component 0" first,
of course it does not overlap with any previous component.
Then we walk over "component 1", again, it does not overlap with any previous component.
Then we walk over "component 2" and realize it overlaps with "component 1", hence
components "2" and "2" belong to a common group.

Wait, this is easy to our eyes. But how does the code "see" the overlapping?
We can have a list to hold a mark for each of the items; see the bar at the top of the diagram.
When we walk "component 0", we'll record in the list that items `7` and `6` belong to "component 0".
Then we walk "component 1", and items `5, 4, 3, 2, 1, 0` belong to "component 1".
Note that none of these spots have been marked by a previous component, i.e. "component 0".
When we walk "component 2", we come across item `8` first---its spot is not marked yet.
We would be thinking "component 0" is standing along, "component 1" is standing alone,
and "component 2" is also standing alone---so far.
Then we come across item `1`. Boom! Its spot is already marked by "component 1".
Then immediately we know "component 2" overlaps with "component 1".
Not only item `1`, but also item `8`---which looked freestanding by itself---and all items
of "component 2" that come after component `1` belong to a group shared with "component 1".

At this point we've got to the tricky part: how do we do the relationship book-keeping and all kinds of updates to it?

Take a look at another situation. Suppose we walk components `0`, then `1`, then `4`, and then `2`.
Each of components `0`, `1`, and `4` would appear freestanding.
But then component `2` would connect the previously freestanding components `1` and `4`,
triggering an update to previously-found relations.

We have used a list for the items to hold component info of the items.
Similarly, we'll use a list for the *components* to hold *group* info of the *components*.
At the beginning, in particular, we'll assume each component is standing alone, and its group
has the same index number as the component itself, that is,
this list starts its life as `0, 1, ..., n_components - 1`.
This means, "component 0" belongs to "group 0", "component 1" belongs to "group 1",
and so on---until revised later.

Now let's restart the walk. At "component 0", none of its items are previously marked,
hence the component's group stays at "0".
At "component 1", similarly, its group stays at "1".
For "component 2", at its first item "8", its group stays at "2".
But at its second item "1", we see the item has been marked by "component 1",
hence the group of "component 2" should be the same as that of "component 1".
In the item list, we see "item 1" is marked by "component 1";
then in the component list, we see "component 1" has group "1".
Now there is a decision to be made: should we name the common group of components "1" and "2"
"group 1" or "group 2"?

To be sure, the final group numbers do not need to be consecutive.
They can be considered categorical labels.
The componen list looks like this now:

```
0, 1, 2, ...
```

and we need to update it to either

```
0, 1, 1, ...
```

or

```
0, 2, 2, ...
```

We will chose the latter and will revisit this decision in a moment.
Before moving on, we have to emphasize:

The component list `0, 2, 2, ...` does not mean that the group of "component 1"
is "group 2". Rather, it means that the group of "component 1" is the group
of "component 2", while the group of "component 2" is not necessarily
"group 2" (although it is for now), as it may very well change later.

So we have made this update at "item 1" of "component 2". Then we visit "item 0"
of "component 2". We see the item has been marked by "component 1".
Then checking the component mark list, we see "component 1" is marked "2".
There is no update to be made here, but that's after checking some things that
are hard to understand now. Let's proceed to the next step.

Next, we walk "component 3". First comes "item 2". We see the item is already marked
"1", meaning "component 1". Then in the "group list", we see "component 1" is marked
"2", meaning its group is the same as that of "component 2". This is a critical moment,
let's take a breather---

while walking "component 3" , we've found it shares group with "component 1", which shares
group with "component 2". How should we do the bookkeeping?

We can update the group mark of "component 1" to "3". That is in fact optional;
it's harmless but unnecessary.
What is necessary (and sufficient) is that we need to update the group mark of "component 2" to "3".
Why is it so?

We happened to have hit an item marked "component 2".
It is possible that "component 3" does not overlap with "component 2".
If we don't take the current opportunity to record that "component 2" shares group
with "component 3", it's possible that this info will be lost.
Suppose we mark "component 1" by "3" and move on, then "component 2" stays at group '2",
and there is no record that compoents "1" and "2" share a group.

On the other hand, it is unnecessary to update the group mark of "component 1",
as in that case the record continues to indicate that "component 1" shares a group with "component 2".

With this updating scheme, the group mark of "component k" is greater than or equal to "k".
When it's equal, the group number is also "k". Otherwise, the group number is found by following this chain "up"
until we find a component whose group mark is equal to its own component number.

To make the updating rule clear,

component mark: `C`
group mark: `G`

- Suppose we are visiting "item i" of "component j".
- If the component mark of "i" is missing, then we mark it and proceed to the next item of "component j".
- If the component mark of "i" is present, with value "k", then we check the group mark of "component k",
  and find it to be "m".
  - If `m == k`, then `G[k] <= i`
  - Else, check `G[m]`;
    - If `G[m] == m`, then `G[m] <= i`
    - Else, follow this chain until we find the component whose group number is the same as its component number, and we update it to "i".



```python
from collections import defaultdict
from typing import Iterable, Sequence


def _internal(components: Iterable[Iterable[int]], n_items: int, n_components: int):

    item_markers = [-1 for _ in range(n_items)]     # Component ID of each item
    component_markers = list(range(n_components))   # Group ID of each component

    for i_component, component_items in enumerate(components):
        for item in component_items:

            j_component = item_markers[item]

            if j_component < 0:
                item_markers[item] = i_component
            else:
                if component_markers[j_component] == i_component:
                    continue

                while True:
                    k_group = component_markers[j_component]
                    component_markers[j_component] = i_component

                    if k_group == j_component:
                        break

                    j_component = k_group

    for i in reversed(range(n_components)):
        if (k := component_markers[i]) != i:
            component_markers[i] = component_markers[k]

    return item_markers, component_markers
```

OK, this is not the most understandable code. Let's try it a different way.

a new . It took a few hours to finalize. The direction was guided by some intuition from the start. It was not too hard. But as I started to work on this post a few days ago, it took some effort to re-understand the algorithm. Furthermore, I've found it hard to find a pedagogical way to explain it.


```python
from collections import defaultdict
from typing import Iterable, Sequence


def _internal(components: Iterable[Iterable[int]], n_items:int, n_components: int):
    # Each component is a sequence of items.
    # The items are represented by their indices in the entire set of items
    # across all components, hence all the elements in `components`
    # are integers from 0 up to but not including `n_items`, which
    # is the total numbe of items across all components.

    item_markers = [-1 for _ in range(n_items)]     # Component ID of each item
    component_markers = list(range(n_components))   # Group ID of each component

    for i_component, component_items in enumerate(components):
        for item in component_items:

            # Get the component ID of this item, if it has been marked
            # by a previously-visited component; otherwise it is -1.
            j_component = item_markers[item]

            if j_component < 0:
                # The item has not been marked by a previous component,
                # hence mark it by the current component.
                item_markers[item] = i_component

                # This may or may not start a new group.
                # If the `else` block below has not executed for the current
                # component, then the current component is in a new group
                # (until a later item changes this situation).
                # This new group should have the largest ID possible so far,
                # which is `i_component` (see the `else` block below).
                # The value `component_markers[i_component]` is `i_component`
                # at this moment, hence no update is needed there.
                # Since the current component is the only component in the new
                # group (for now), no update is needed to other component's
                # group IDs.
                #
                # If the `else` block below has ever executed for the current
                # component, then this component belongs to a group with ID
                # `i_component`, which has been taken care of in that block.
                # Nothing else needs to be done about the current item in addition
                # to marking it.
                #
                # Apparently, in any case, the current component belongs to group
                # with ID `i_component`.
            else:
                # The item has been marked by a previous component,
                # with ID `j_component` hence the components `i_component` and
                # `j_component` belong to the same group,
                # because they are "connected" by the current common item.
                #
                # This previous component may share its group with other previous
                # components. All these components as well as the current one
                # share a group. This block ensures that this group gets ID value
                # equal to `i_component`, and all it member components can find
                # this group ID directly or indirectly.

                # Check the group ID of the previous component.
                if component_markers[j_component] == i_component:
                    # A previous item of the current component was also marked by
                    # the component `j_component`. When the previous item was
                    # processed, it updated the group ID to `i_component` for
                    # the component `j_component`, and possibly for other connected
                    # components. All is good now.
                    continue

                # Now the current component has group ID `i_component`, but
                # the connected component `j_component` has group ID that is
                # smaller than `i_component`. Update the group ID of component
                # `j_component` and possibly others that are connected to
                # `j_component`.

                while True:
                    k_group = component_markers[j_component]
                    component_markers[j_component] = i_component
                    # This is the only line that changes `component_markers[idx]`,
                    # and the change is always upwards. Before change, the value
                    # is `idx`. After change, say `component_markers[idx] = jdx`,
                    # the component `idx` shares a group with component `jdx`.
                    # This change happens when and only when a later component
                    # (i.e. `idx < jdx`) sees that the two components are connected.

                    if k_group == j_component:
                        # This suggests the group ID for component `j_component`
                        # was never changed. Until now, the group ID for component
                        # `j_component` has been `j_component`. There may be components
                        # before `j_component` that are connected to `j_component`,
                        # but there are no components after `j_component` that
                        # are connected to it. We have updated
                        # `component_markers[j_component]` to `i_component`.
                        # We do not need to update this for components that are
                        # before `j_component` and connected to it; they will find
                        # their group ID by checking `component_markers[j_component]`.
                        break

                    # The component `j_component` is connected to another component
                    # (whose index is `k_group`) that is after it (`j_component` < `k_group`).
                    # Now that we have updated `component_markers[j_component]`, we
                    # follow the chain to update that of `k_group`.
                    j_component = k_group

    # Now `component_markers` contains the group ID of each component
    # directly or indirectly. The next block makes them all direct.
    for i in reversed(range(n_components)):
        if (k := component_markers[i]) != i:
            # Now it must be that `mark > i`, indicating
            # that the component `i` shares a group
            # with component `mark`. We follow this chain
            # to get the group ID.
            component_markers[i] = component_markers[k]

    return item_markers, component_markers


def connected_components(components: Sequence[Sequence[int]], n_items: int):
    n_components = len(components)
    _, component_markers = _internal(components, n_items, n_components)

    groups = defaultdict(set)
    # Item IDs in each group, indexed by group ID.

    for i_comp, i_grp in enumerate(component_markers):
        groups[i_grp].update(components[i_comp])

    return list(groups.values())


```

```python
from typing import Sequence

import numba
import numpy as np


# Numba cache of the compiled function is in __pycache__
# in the directory of the source code.
# First run of this function includes compilation time.
# Compiling this function in particular prints a warning;
# don't worry about it.
@numba.jit(nopython=True, cache=True)
def _internal(
        item_markers: np.ndarray,
        component_markers: np.ndarray,
        i_component: int,
        component_items: np.ndarray,
        ):
    for item in component_items:
        j_component = item_markers[item]

        if j_component < 0:
            item_markers[item] = i_component
        else:
            if component_markers[j_component] == i_component:
                continue

            while True:
                k_group = component_markers[j_component]
                component_markers[j_component] = i_component
                if k_group == j_component:
                    break
                j_component = k_group


def connected_components(components: Sequence[Sequence[int]], n_items: int):
    components = [
            c if isinstance(c, np.ndarray) else np.array(c)
            for c in components
            ]
    n_components = len(components)

    item_markers = np.full(n_items, -1)
    component_markers = np.arange(n_components)

    for i, component_items in enumerate(components):
        _internal(item_markers, component_markers, i, component_items)

    for i in reversed(range(n_components - 1)):
        if (k := component_markers[i]) != i:
            component_markers[i] = component_markers[k]

    item_markers = item_markers[item_markers >= 0]
    item_markers = component_markers[item_markers]

    return [
            np.where(item_markers == grp)[0]
            for grp in np.unique(component_markers)
            ]

```


```python
import math
import time
import numpy as np
import cc_nx
import cc_py
import cc_numba


def bench(mod, n_items, repeats=10):
    for _ in range(repeats):
        n_components = int(math.sqrt(n_items))
        component_size = int(n_items / n_components * 0.8)
        components = [
                np.random.choice(n_items, component_size, replace=False)
                for _ in range(n_components)]

        t0 = time.perf_counter()
        _ = mod.connected_components(components, n_items)
        t1 = time.perf_counter()
        tpy = t1 - t0

        t0 = time.perf_counter()
        _ = cc_nx.connected_components(components)
        t1 = time.perf_counter()
        tnx = t1 - t0

        print(n_items, f'{tnx:.6f}', f'{tpy:.6f}', f'{tnx/tpy:.2f}')


if __name__ == '__main__':
    from argparse import ArgumentParser
    parser = ArgumentParser()
    parser.add_argument('mod', choices=['py', 'nu'])
    args = parser.parse_args()

    mod = {'py': cc_py, 'nu': cc_numba}[args.mod] 
    for n_items in (1000, 10000, 100000, 1000000):
        bench(mod, n_items)

```
