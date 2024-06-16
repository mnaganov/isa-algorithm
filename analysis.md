# Analysis of the Pseudocode

The code from the paper "Improved Sliding Algorithm for Generating No-Fit
Polygon in the 2D Irregular Packing Problem" by Q. Luo and Y. Rao is
analyzed in order to determine requirements for Python libraries that will
be used to implement it and propose data structures used for computations.

## Algorithm 1: Determine Translation Vector

This algorithm works with **touching groups.** Each group consists of 2D
vectors that represent edges from two polygons `A` and `B`. One of the
polygons is **fixed,** the other is called **orbital.** By convention, the
polygon `A` is the fixed one.  The algorithm determines whether one of the
vectors can be used for **translation** (moving) of the orbital polygon
along it w/o intersecting with the fixed polygon. This is called
**feasibility test.** Note that in order to conduct this test, the
algorithm does not consider entire polygons, but rather only the edges from
touching groups. Thus, there must be an attribution of each edge to one of
the polygons.

The algorithm calculates the **angular direction** for each touching group.
The angle is calculated for the **touch point**: it is either on an edge,
or it's the point where the edges start or end (both polygons touch only
with one of their angles). This is the direction for which the edge of the
orbital polygon can move without intersecting with the edge of the fixed
polygon. The algorithm does not specify how to do that, but the paper
provides some examples in **3.2.2.ii**. This may be non-trivial. The
angular direction does not seem to be used outside of the **Algorithm 1.**
Note that the vector has its own direction, relative to the coordinate
axis, this is the direction that will actually be used for moving the
polygon `B`.

The algorithm specifies that the output is a touching group, however the
code also selects, that is, flags one of its vectors as "feasible." This is
an important side effect of the algorithm. This flag must be stored
somewhere in the touching group. The flagged vector is called the
**translation vector.**

Note that the algorithm selects only one touching group. However, later
steps of further algorithms may find that this touching group can not be
used. In that case, the algorithm is called again to resume past the
previous selected group. The input list of touching groups does not change.

```
Input: T list; // T list,
Input: t1; // last touching group
Output: t2; // the selected touching group

t = the number of touching groups in T;

for each touching group T[i] in T do
  Get the potential translation vector of T[i];
  Calculate angular direction of T[i];
end for

if t is equal to 1
  t2 = T[0];
return;

i = 0;
for i < t do
  for j < t do
    if j is not equal to i
      // do feasible test
      Get the potential vector v[k] in T[i];
      if v[k] is not suitable for T[j]
        Flag v[k] is infeasible;
        break;  // breaks inner loop
    j = j + 1;
  end for
  i = i + 1;
end for
// ??? Can we only consider the diagonal part: j < i ???

if only one feasible translation vector in T
  t2 = feasible translation vector in T;
  return;
t2 = Select a feasible vector from T based on t1;
return;
```

### Proposed data structures

```
Point := struct
  int x, y

Vector := struct
  Point start, end

TouchingGroup := struct
  List<Vector> edges
  nullable Vector& translation_vec
```

Alternative:

```
FlaggedVector := struct
  Vector vec
  Flag flag

TouchingGroup := List<FlaggedVector>
```

The `FlaggedVector` can be used first for flagging vectors as "infeasible",
and then for marking the translation vector. The `FlaggedVector` can also
be reused in the **Algorithm 3** for polygon edges traversal. One caveat is
that use of `FlaggedVector` may allow marking several vectors as
translation vectors by mistake.

## Algorithm 2: Compute Translation Distance

The translation vector determined by the **Algorithm 1** may not always be
used at its full length. The actual distance of the translation is
determined using another algorithm. This algorithm iteratively shortens the
translation vector from both sides based on intersection tests.  Provided
with the translation vector, the algorithm considers each vertex of the
polygon `A` and translates it along the vector, by placing the starting
point of the vector to the vertex. If it is determined that the translation
vector intersects with any of the edges of the polygon `B`, the translation
vector gets shortened up to the intersection point (see **Figure 9**).
Then the same procedure is repeated with polygons swapping their roles.
Indeed, these two parts of the algorithm are identical except for swapping
`PA` with `PB` and considering the translation vector in the opposite
direction—this fact is not specified in the algorithm clearly.

Since the run time complexity of this algorithm is *O(nm)*, where *n* and
*m* are the number of vertices in each of two polygons, the paper proposes
an optimization. The optimization removes out of consideration those
vertices that are proven not to cause shortening of the translation vector.
The algorithm uses two lines, called **upper** and **lower boundary** which
bound a strip on the plane. Vertices outside of this strip are not
considered. The exact steps for determining the boundaries are not shown in
the algorithm, but the principle is described in the section **3.2.3.ii**
of the paper. There seems to be a contradiction between the description of
the boundaries in the paper and in the algorithm. According to the paper,
if the strip is defined based on the polygon `B`, then vertices of the
other polygon (polygon `A`) outside of this strip are not considered, and
vice versa. However, the algorithm uses the same polygon both for
determining the boundaries and considering vertices of. That does not make
sense since by definition all vertices of the polygon will be inside the
bounding strip.

There is no explicit output from the algorithm, it just modifies the
translation vector as a side effect.

```
Input: PA, PB //two polygons,
Input: t2 // the selected touching group

Get the translation vector v in t2;
Initialize ub, lb; // the upper and lower bound

Get the boundary ub and lb about PA based on v;  // ??? PB ???
i = 0;
for each vertex point pa of PA do
  if pa is not in ub and lb bound
    continue; // point exclusion test
  for each edge e of PB do
    Calculate the cross point pc of the pa along v with edge e;
    if pc exists // intersection happened
      Get distance d between pc and pa;
      if d less than the current length of v;
        The starting point of v = pc;
        The ending point of v = pa;
  end for
end for

// !!! Note that here v is cosidered in the opposite direction !!!
Get the boundary ub and lb about PB based on v;  // ??? PA ???
i = 0;
for each vertex point pb of PB do
  if pb is not in ub and lb bound do
    continue; // point exclusion test
  for each edge e of PA do
    Calculate the cross point pc of the pb along v with edge e;
    if pc exists // intersection happened
      Get distance d between pc and pb;
      if d less than the current length of v;
        The starting point of v = pb;
        The ending point of v = pc;
  end for
end for
```

### Proposed data structures

```
Polygon := List<Vector>

Line := struct
  int k, b  // y = k*x + b

Stripe := struct
  Line lower, upper
```

Also see an alternative definition for the `Polygon` after the
**Algorithm 3**.

## Algorithm 3: Search for Other Starting Points

The aim of the overall algorithm is calculation of a **no-fit polygon**
(NFP) around the fixed polygon (`A`) which defines the area that can be
used to test quickly whether the second polygon (`B`) touches the first
one, by checking whether an arbitrary point of the polygon `B` belongs to
the NFP. This is much faster than a full polygon intersection test, thus
using an NFP optimizes trying different polygon layouts. This area is
calculated by choosing an arbitrary placement of the polygon `B` so that it
touches the polygon `A`, and then sliding it around.

The sliding method works well unless the polygon `A` has regions that are
unreachable via sliding, such as a hole inside it, or as shown in example
on the **Figure 11**. To calculate the NFP in such a case, multiple
slidings from different starting points must be performed. The
**Algorithm 3** tries to find alternative starting points. This algorithm
depends on the **Algorithm 4** for getting a list of translation vectors.

The **Algorithm 3** is designed to be called multiple times for searching
another starting point, until it has considered all the edges of the
polygon `A`. That's why the algorithm flags them. This information is used
in subsequent re-runs of the algorithm.

The algorithm tests each pair of edges from polygons `A` and `B`. If after
placing the polygon `B` in a way that the starting point of the edge from
`B` at the starting point of the edge from `A`, the polygons do overlap,
then this pair should not be considered. Note the typo in the paper's
algorithm. The algorithm includes an optimization for not considering edges
of the polygon `B` that fail the **right side test** (see **Figure 12.a** and
**12.b**).

If the edge passes the check, then the **Algorithm 3** uses the
**Algorithm 4** for obtaining all translation vectors. If these exist, they
are used to slide the polygon `B` along the current edge of the polygon `A`
and test while sliding, if they overlap or not. If they do not overlap,
then the current position is a new starting point. Otherwise, the algorithm
determines the translation distance, and slides the polygon `B` by it.  The
paper says that the translation distance is *calculated similar to the
steps described in the Section 3.2.3*. This section describes the
**Algorithm 2**, however the **Algorithm 3** does not explicitly into it.
This is likely due to the fact that the **Algorithm 2** only considers one
vector, while the **Algorithm 3** needs to shorten a list of vectors (`V`
in the algorithm pseudocode). After shortening, the algorithm somehow
chooses one translation vector from `V`—the exact criteria is not clear.

```
Input: PA, PB //two polygons,
Input: Ptnfp; // the starting point of new NFP
Returns: whether a new starting point has been found

Initialize V = ∅; // the list of storing all translation vectors

for each edge e[n] of PA do
  if e[n] is visited do
    continue;
  Set e[n] is visited;  // This is because the algorithm may be called iteratively
  for each edge e[m] of PA  // !!! PB !!!
    // ??? is it assumed that holes have edges oriented inversely to the outer shell ???
    Move PB by making starting point of e[n] and e[m] coincidence;  // ??? coincident
    if both e[m−1] and e[m] are not on the right side of e[n] do
      continue; // “right side” test
    V = ∅; // clear stored vectors
    If the result of getting translation vectors (PA, PB, e[n]) is false do
      continue; // Algorithm 4
    // Algorithm 4 outputs to the list V
    while the starting point of e[m] does not on the ending point of e[n] do
      if PA does not overlap with PB at this position
        Ptnfp = PBrf; // the reference point PBrf of PB
        return true;
      Shorten all vectors in V and get the translation vector v;
      Translate PB by vector v;
    end while
  end for
end for
return false;
```

### Proposed data structures

```
Polygon := List<FlaggedVector>
```
