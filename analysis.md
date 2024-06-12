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
somewhere in the touching group.

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

## Algorithm 2: Compute Translation Distance

The translation vector determined by the **Algorithm 1** only specifies the
direction. The actual distance of the translation is determined using
another algorithm.
