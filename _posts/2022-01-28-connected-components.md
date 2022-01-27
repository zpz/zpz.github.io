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


```python
import numpy as np
import cc_nx
import cc_py
import cc_numba


COMPONENTS_1 = [
        (0, 1, 2, 3, 4, 5),
        (0, 1, 8),
        (2, 9),
        (6, 7),
        (5,),
        (8, 9),
        (10, 11, 12, 13),
        (10, 14, 15),
        ]
N_1 = 16


COMPONENTS_2 = [
        (7, 6),
        (5, 4, 3, 2, 1, 0),
        (8, 1, 0),
        (2, 9),
        (8, 9),
        (11, 13, 10, 12),
        (10, 14, 15),
        (5,),
        ]
N_2 = 16


def intro_nx():
    cc = cc_nx.connected_components(COMPONENTS_1)
    print(cc)


def check_nx():
    cc = cc_nx.connected_components(COMPONENTS_2)
    print(cc)


def check_py(mod):
    cc = mod.connected_components(COMPONENTS_2, N_2)
    print(mod.__name__)
    print(cc)


if __name__ == '__main__':
    intro_nx()
    check_nx()
    check_py(cc_py)
    check_py(cc_numba)


```


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
