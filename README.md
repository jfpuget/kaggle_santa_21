# Solution to Kaggle Santa 2021 Challenge

The challenge is described at https://www.kaggle.com/c/santa-2021/overview

I published three posts describing my solution using a MILP solver: cplex.  I may add a gurobi version of thes emodels in the near future.

My posts series:

* [A MILP Journey, part 1: proof of 2440 lower bound](https://www.kaggle.com/c/santa-2021/discussion/300507)
* [A MILP Journey, part 2: 2440 solution](https://www.kaggle.com/c/santa-2021/discussion/300516)
* [A MILP Journey, part 3: wildcards](https://www.kaggle.com/c/santa-2021/discussion/300522)

The text below is an edited version of these posts.

# Introduction

I decided to see if one of the optimization technique I know the best, namely mixed integer linear model (MILP) could be used here. To my surprise it worked fine. Let me try to explain how I worked with successive MILP models towards a 2428 solution.

One way to solve problems with MILP is to work with relaxations of the problem, i.e. models that include only a part of the problem specification, in the hope that the relaxation is easier to solve. Once a relaxation is solved we can add a bit more of the problem to it and iterate.

Before describing my models let's introduce some concepts

# Permutations, graph and cycles

We define a directed graph whose nodes are permutations of the string 1234567, and whose arcs are weighted with a cost. In the full problem, some permutations, the ones starting with 12, must appear in each of the 3 strings while the other permutations can appear only in one of them. We call the strings starting with 12 the mandatory strings.

In some variant I duplicate the nodes for mandatory strings 2 times or 3 times.  In that case the cost of arcs from a mandatory string to a copy of itself is set to 7.

Graph arcs cost is defined by the overlap between the two permutations.
For instance, the cost of the arc between 1234567 and 4567312 is 3, because if we merge the second string with the first with overlap overlap, then we get 1234567312 which is 3 chars longer than the first string. 

We call c-arc an arc of cost c.

The next concept is that of 1-cycle.  It is made of all the rotations of a given permutation.  For instance the 1-cycle starting with 1234567 visits these permutations in order, via 7 1-arcs:

1234567

2345671

3456712

4567123

5671234

6712345

7123456

1234567

If we cut the cycle some arc, say the first one above, and merge the strings with overlap, then we get a string of length 13:

2345671234567

That's  a general property: the length of a string obtained by breaking a cycle and merging its permutations is 7 plus the sum of the cost of the arcs in the cycle minus the arc we broke.

Our last concept is a 2-cycle.  It is made of 6 broken 1-cycles. For instance the 2 cycle starting with 1234567 is made of these 1-cycles:

2345671234567

3456721345672

4567231456723

5672341567234

6723451672345

7234561723456

If we merge the strings with overlap then we get

23456712345672134567231456723415672345167234561723456

# Lower bound proof

The first relaxation I tried was very crude. The code is in the lower_bound_proof.ipynb notebook.  I start with a graph without repeating mandatory permutations.  The MILP model has two sets of variables : 
- visit(p) is an integer  variable whose value is the number of time p appears in the problem solution.  The lower bound of visit(p)  is set to 3 for p mandatory, 1 otherwise.
- next(p1, p2) is an integer variable whose value is the number of times p1 is immediately followed by p2 in the strings of the solution to the problem.  

The graph does not contain all arcs.  If there is an arc(p1, p2) such that there exist two other arcs (p1, p3) (p3, p2) and cost(p1, p3) + cost(p3, p2) = cost(p1, p2) then I remove the arc (p1, p2).  Indeed, if there is a problem solution that goes from p1 to p2, then that solution goes from p1 to p3 to p2.

The constraints are simple:

- flow conservation.  The sum over p1 of next(p1, p0)) = sum over p2 of next(p0, p2) for all p0
- subtour elimination.  Given that the nodes may be visited more than once I cannot use the standard subtour elimination constraint which states that the sum of the variables for the edges in a cycle is strictly less than the number of arcs in of the cycle.  I used a variant using the incoming arcs to the cycle.  The sum of next(p1, p2) where p1 is not in the cycle and p2 is in the cycle is at least 1.  A similar constraint is stated for outgoing arcs. Subtour elimination is stated for every 1-cycle and for every 2-cycle.

The objective is also simple.  It is the sum of the costs time the next(p1, p2) variables.  I added a little twist: I wanted to maximize the number of 7-arcs in the solution, because it leads to a lot of freedom to cut the solution into strings with same length.  I therefore decreased the cost of 7 edges to 7 minus epsilon (epsilon = 0.01)

The resulting model has 24217200 integer variables,  13200 constraints, and 433802880 non zeros. It is solved in less than one hour with cplex on a machine with 20 cpu cores.  The solution objective value is 7317.6. It has 240 7-arc which explains why the objective is not integral.

Assume we are given a 3 string solution of value L of the original problem. If we append these strings and turn them into a cycle, then we get a cycle of length at most 3 * L. Therefore 3 * L >=  7317.6 because the MILP is a relaxation. Then L >= 7317.6 / 3 = 2439.2  . Given L is integral, L >= 2440.

This proves the [lowerbound of 2440 I shared during the competition](https://www.kaggle.com/c/santa-2021/discussion/292841). It was later [confirmed by William Cook using his Concorde solver](https://www.kaggle.com/c/santa-2021/discussion/294139).

I will now describe how a MILP model was used to find a 2440 solution.

# A first model

The first evolution of the model was to explicitly represent the duplicated mandatory permutations. I also removed all the 7-edges and replaced them by 2 binary variables per node.

The variables are:

 -  next(p1, p2) is an integer variable whose value is the number of times p1 is immediately followed by p2 in the strings of the solution to the problem.
- start(p) if p is preceded by a 7-edge, whatever the 7 edge
- end(p) if p is followed by a 7 edge, whatever the 7 edge

In order to make the problem easier to solve I enforced that nodes in my new graph were visited exactly once.  A s result I no longer can remove arcs as I did for the lower bound model, because a path that goes from p1 to p2 will not necessarily visit an extra node p3 if cost(p1, p3) + cost(p3, p1) = cost(p1, 2) when p3 is already visited by another path.

Another restriction was to remove 4-arcs, 5-arcs, and 6-arcs as none of them appeared in the lower bound model solution.  This restricts drastically the size of the model,

The constraints are simple:

-  flow conservation. The sum over p1 of next(p1, p)) + start(p) = sum over p2 of next(p0, p2) + end(p) for all p0
-   subtour elimination. I reused the same as in the lower bound model.  I later tried with the standard subtour elimination constraint, but to my surprise it took longer to solve that version of the MILP.

The objective is the same as before.  It is the sum of the costs time the next(p1, p2) variables where the cost of 7 edges to 7 minus epsilon (epsilon = 0.01).

Solving this model yields again a solution close to 7320. I then wrote  a little cycle extraction, that starts from a non zero next(p1, p2) variable, then looks for which variable next(p2, p3) is no zero, etc, until it comes back.  The hope was ot get 3 cycles of equal length, but this did not happen: Many more cycles were found.  My subtour elimination constraints were not strong enough.

# A Better model

I then added a variant of Miller–Tucker–Zemlin (MTZ) constraints to eliminate all subtours.  The difference is that I have implicit arcs via the start and end variables.  Another difference is that we look for 3 paths, not one.  For this I add one variable u(p) for every node p.  It represents the rank of p in the path that contains p.  The constraints are:

 u[i] - u[j] + N3 * nex(i,j) <= N3 - 1 for all i,j

u[i] <= (N3 - 1) * (1 - start_perm[i]) for all i

u[i] >= (1 - start_perm[i]) for all i

where N3 is the number of permutations divided by 3.

The first one is MTZ, the other 2 are saying that u[p] is zero if and only if start(p) is one.

A last twist is to use the know upper bound of 7320 as a cutoff.  It helps the solver during its branch and bound: any node of cost higher than the cutoff is pruned.

This model was better, less short cycles.  But it  produced a lot of cycles of similar nature.  Here is one:

123456712345672134567231456723415672345167234561723456

Issue is that the mandatory 1234567 is duplicated, which is not what we want.  We want duplicates of the same mandatory permutations to appear in 3 different strings.  for this I had to add a graph coloring piece to the model.

Other cycles were of the form

12345672134567231456723415672345167234561723456

These are 2-cycles for a mandatory minus 6 permutations.  The missing 6 permutations are the rotations of the mandatory.

# Graph coloring

I added 3 binary variables per node, class(p, k) with k=1,2,3 to represent which of the final 3 strings p belongs to.

The constraints state that a node can only have one class, and that the number of nodes having a given class is N3.  They also state that if next(p1, p2) is one, then class(p1,k) == class(p2,k) for all k.

Adding these constraints made the problem very hard to solve.  No solution was found after a day of computation.

I studied the previous model solution, and found few things:
- All nodes with a start were mandatory permutations.  This was easy to enforce with an additional constraint.
- All nodes with an end were mandatory permutations or were of the form 1x2xxxxx. Same, easy to enforce via constraints
- At least one copy of each mandatory had both a start and an end.  I decided to drop one duplicate for each mandatory.  This reduced the size of the problem further.
- The truncated 2-cycles for all mandatory cover all permutations that are not a rotation of a mandatory.  I decided to enforce this by setting to 1 all the arcs appearing in these truncated 2-cycles.

The resulting model solves in about 5 minutes, with a number of cycles.  When the cycles are grouped by class and concatenated we get 3 strings of length 2160.  Each of these string contain 80 mandatory.  By appending the 40 missing mandatory we get 3 strings of length 2440 who meet all the original problem constraints.

the code for this model is in the solution_2440.ipynb notebook.

# Adding wildcards

This solution of the previous model contains a number of truncated 2 cycles of the form 

`12345672134567231456723415672345167234561723456`

I quickly saw that by adding a wildcard near the end we could overlap another mandatory permutation.

```
12345672134567231456723415672345167234561723456
12345672134567231456723415672345167234561*234567
                                         1234567

```
Issue is that the mandatory we can add is already use in the string!  

A quick fix is to remove the mandatory at the start of the string, which yields a string one char shorter:

```
345672134567231456723415672345167234561*234567

```
Doing this twice per solution string yields a 2438 solution.

We can do better.  Indeed, we can now prepend another mandatory.  In this case we can use 1273456.  If we prepend it with overlap then we get this string:

`127345672134567231456723415672345167234561*234567`

This string is 5 chars shorter than these two strings combined:

```
12345672134567231456723415672345167234561723456
1273456

```
If we do this twice per solution string then we reduce the length of the solution by 10, yielding a 2430 solution.

In my case, I started from the 2160 MILP solution, and for each of my 3 strings, I looked at the missing 40  mandatory to see if 2 of them could be used as prefix as above.  Fortunately it was the case, which led to a 2430 solution quickly.

Then I got stuck for a while.  I tried lots of models with wildcards, but nothing worked, or lower bound were above 2428.

I then went onto something else and came back on this after few days break.  The issue was how to use wildcards as above, without having to remove the mandatory at the start of the truncated 2-cycle? Then the light came: we can use the widlcard to switch between two sibling strings.  

Let me explain it using an example.  We start from these two truncated 2-cycles:

```
12345672134567231456723415672345167234561723456
12347652134765231476523417652347165234761523476

```
We move from one to the other by swapping 5 and 7.  These are the values that appear where we insert a wildcard.

When we insert a widlcard then we also swap these values after it, and append the extra mandatory, to get these two modified strings:

```
12345672134567231456723415672345167234561*234765
12347652134765231476523417652347165234761*234567

```
We can the  check that we cover the same set of permutations plus two mandatory, at the cost of adding only one char per string
If we can do this for each of the 3 strings of a 2440 solution then we get a 2428 solution.

Starting from my 2160 solution, I looked for 3 pairs of truncated 2 cycles appearing in different strings, and which could be completed as above by unused mandatory.  I was fortunate to find them, and bingo, a 2428 submission!

But I was not fully satisfied as there was  a bit of luck here. I decided to check that I could make sure these pairs of truncated 2-cycles always existed.  The answer is yes, with a more elaborate MILP model.

# Final MILP model

We want to model the arcs corresponding to a change in the value of the wildcard.  These will be additional wildcard arcs in our graph.

For the string  12345672134567231456723415672345167234561*234765 we need to add two arcs:

5617234 to 6152347 ( `561*234 to 61*2347 ` )

1523476 to 1234765 (`1*23476 to*234765`)

There is a catch though.  The extra mandatory must be an unused one, which means we need to explicitly represent the 3 mandatory duplicates in our model.  Rather that doing it I only added the first wildcard arcs, and added constraint to make sure an unused mandatory will be available.

For each possible string pair as above I added a new binary variable.  If that binary was true then all the wild edges, and class constraints to make the wildcard string valid are enforced.

The resulting MILP is only slightly larger than the previous one and is solved in about 10 minutes.  Extracting cycles and appending them yields an out of the box 2428 solution.

The code for this model is in the solution_2428.ipynb notebook

