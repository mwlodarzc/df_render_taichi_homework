# Tai Chi Graphics Course S1-Tai Chi Ray Tracing (Flat kd-tree Implementation)

##   Successful Effect Display
First, let's post the effect images. I just learned Blender, so the modeling is a bit rough, please forgive me. Since I don't have a graphics card, it runs purely on CPU (6 cores x 3.00GHz).

![mushroom](./imgs/ray-trace-mushroom.png)

![crystal](./imgs/ray-trace-crystal-01.png)

## Introduction

Based on the Taichi programming language, it implements ray tracing based on a flat kd-tree.

Main issues addressed:
- [Intersection of triangle surfaces and rays](#三角形面片) 
- [Generation of kd-tree](#kd树生成)
- [Determination of the dividing line](#分割线的确定)
- [Flattening of the kd-tree](#扁平化kd树)
- [Flat traversal](#扁平的遍历)

3D models:

- Currently, only supports PLY ascii format. See plyreader2.py for details.
- Due to axis orientation, textures need to rotate by $-\pi/2$.

## File Structure

```
- raytrace-c.py (Crystal model rendering program)
- raytrace-m.py (Mushroom model rendering program)
- Scene.py (Scene, camera, ray tracing, ...)
- plyreader2.py (Reading PLY models)
- view_uv.py (Viewing textures)
- mesh (Folder for models and textures)
```

Running method:
- Crystal 'python3 raytrace-c.py'
- Mushroom 'python3 raytrace-m.py'
- Viewing 'textures python3 view_uv.py'

## Symbol Explanation
Ray: $p = o + t d$. $e$ is the endpoint of the ray, $t$ is time, $d$ is direction, in the code $|d|_F = 1$ fixed length to $1$;
Plane: $n(x - y) = 0$. $n$ is the normal vector of the plane, $x$ is any point on the plane, $y$ is another point.
## Triangle Surfaces
3D models most commonly use triangular or quadrilateral surfaces. So, a core issue is how to determine the intersection of a ray with a triangle surface.

### One method is direct calculation.
The "Tiger Book" (Steve Marschner and Peter Shirley. Fundamentals of Computer Graphics 4th ed.) section 4.4.2 provides a detailed explanation.
But this algorithm requires calculating a $3 \times 3$ determinant for each intersection calculation, which is relatively complex.

### Here's another approach.
A triangle surface is a two-dimensional object, and a ray intersecting it must first intersect the plane on which the triangle lies.
We can first calculate the intersection of the ray with the plane, and then check if the intersection point is inside the triangle.

### To calculate the time $t$ when the ray $p = o + td$ intersects the plane $n(x - (o + td)) = 0$, $t = n(x - o) / (nd)$.
If $t$ is greater than the shortest intersection time before, then this triangle is not needed, and can be skipped directly.
If $t$ is less than the shortest intersection time before, then check if the intersection point is inside the triangle.

For a triangle:
```
   a
  / \
 /   \
x --- b
```
With intersection coordinate $p$, parameters $\alpha, \beta$ satisfy
$$
p = x + \alpha (a-x) + \beta (b-x)
$$

Moving $x$ to the left side:
$$
p - x = \alpha (a-x) + \beta (b-x)
$$
For convenience, let $\hat{p} = p - x$, $\hat{a} = a - x$, $\hat{b} = b - x$, expanding this yields
$$
\begin{aligned}
    \hat{p}[0] &= \alpha \hat{x}[0] + \beta \hat{b}[0]\\
    \hat{p}[1] &= \alpha \hat{x}[1] + \beta \hat{b}[1]\\
    \hat{p}[2] &= \alpha \hat{x}[2] + \beta \hat{b}[2]\\
\end{aligned}
$$
There are three equations here, but only two are linearly independent (in terms of linear algebra| this can be roughly understood as 3 equations equal to 1 equation + 2 equations). Therefore, only two equations are needed to solve.

Which two equations to choose?
We can first combine them in pairs, solve if possible, and switch to another combination if not solvable (e.g., choosing equations 1

Specifically, this means: determine whether three determinants are $0$
$$
\begin{aligned}
\det(A_0) &= \left|\begin{matrix}
\hat{a}[1] & \hat{a}[2] \\
\hat{b}[1] & \hat{b}[2] \\
\end{matrix}\right| 
&= \hat{a}[1] ~ \hat{b}[2] - \hat{a}[2] ~ \hat{b}[1] \\
\det(A_1) &= \left|\begin{matrix}
\hat{a}[0] & \hat{a}[2] \\
\hat{b}[0] & \hat{b}[2] \\
\end{matrix}\right| 
&= \hat{a}[0] ~ \hat{b}[2] - \hat{a}[2] ~ \hat{b}[0] \\
\det(A_2) &= \left|\begin{matrix}
\hat{a}[0] & \hat{a}[1] \\
\hat{b}[0] & \hat{b}[1] \\
\end{matrix}\right| 
&= \hat{a}[0] ~ \hat{b}[1] - \hat{a}[1] ~ \hat{b}[0] \\
\end{aligned}
$$

It's found that all three determinants are unrelated to the ray, so they can be calculated in the initialization phase to save time.

After calculating the three determinants, one of the following three cases can be used to directly solve for $\alpha,\beta$. The amount of calculation is reduced from a $3 \times 3$ determinant to a $2 \times 2$ determinant.

If $\det(A_0) \ne 0$,
$$
\begin{aligned}
    \alpha &= (\hat{p}[0] \hat{a}[1] + \hat{p}[1] \hat{a}[2]) / \det(A_0) \\
    \beta  &= (\hat{p}[0] \hat{b}[1] + \hat{p}[1] \hat{b}[2]) / \det(A_0) \\
\end{aligned}
$$

If $\det(A_1) \ne 0$,
$$
\begin{aligned}
    \alpha &= (\hat{p}[0] \hat{a}[0] + \hat{p}[1] \hat{a}[2]) / \det(A_1) \\
    \beta  &= (\hat{p}[0] \hat{b}[0] + \hat{p}[1] \hat{b}[2]) / \det(A_1) \\
\end{aligned}
$$

If $\det(A_2) \ne 0$,
$$
\begin{aligned}
    \alpha &= (\hat{p}[0] \hat{a}[0] + \hat{p}[1] \hat{a}[1]) / \det(A_2) \\
    \beta  &= (\hat{p}[0] \hat{b}[0] + \hat{p}[1] \hat{b}[1]) / \det(A_2) \\
\end{aligned}
$$

Solving for $\alpha, \beta$ can determine whether point $p$ is inside the triangle
$$
\alpha > 0;~ \beta > 0;~ \alpha + \beta < 1.
$$

If it's a quadrilateral surface, the process is the same as before. When judging $\alpha, \beta$, use
$$
\alpha > 0;~ \beta > 0;~ \alpha < 1; ~ \beta < 1.
$$

(The principle is vector addition)

```
    a
   /
  /---p
 /   /
x---------b
```

Here, $\alpha, \beta$ are also the basis for texture positioning.

## Generation of kd-tree
Since Taichi neither supports recursion nor dynamic memory creation in ti.func (a significant challenge, as many trees cannot be used), the kd-tree is created in the Python environment and then converted for use in Taichi.

Let's use an example:
For instance, if you're looking to find someone across the country, you'd first locate the province they're in, then the city, and so on, narrowing down step by step. This method is much faster than a blanket search.

This approach is also applicable when determining intersections. We can compose objects into a series of boxes, first identifying a large box and then finding smaller boxes within it, continuing this process.

How are the boxes drawn? A common method is to draw dividing lines, with one box to the left of the line and another to the right. Objects in the middle go into both sides.

However, searching for boxes requires recursion or a queue...

Without dynamic memory or recursion... many common methods become unusable /(ㄒoㄒ)/. After several attempts and errors, a method was finally found.

Due to Taichi's characteristics, calculations must be completed within loops, so the relationship between the boxes must be determined in a single judgment. For well-separated boxes, not much consideration is needed. But for overlapping situations
```
|------
|     |
|   --|----
|   | |   |
|---|--   |
    |-----|
```

```
|---------|
|         |
|     ----|
|     |   |
|     ----|
|---------|
```

it becomes complicated.

Therefore, we draw three boxes: one strictly on the left, another strictly on the right, and a third in the middle. Then, the tree can be created recursively in Python.

## Determination of the Dividing Line
The dividing line is determined so that the number of objects on the left is close to the number on the right, with fewer objects in the middle. Based on this, for each dimension, a grid search (a somewhat lazy method) is used to find a dividing line that minimizes $\text{abs}(c_l - c_r) + 0.5 c_m$, where $c_l, c_m, c_r$ are the number of objects on the left, middle, and right, respectively.

## Flattening of kd-tree
Due to the characteristics of Taichi... the tree constructed previously is compressed into one dimension.

The data format (the numbers at the beginning indicate relative offsets):
```
# [kdflat]
# 0: corner index; positions of the lower-left and upper-right corners
# 1: mark; a symbol
# if mark >= 0: # Offset area
#   1: left index; offset of the left tree
#   2: right index; offset of the right tree
#   3: mid index; offset of objects in the middle
#   4: lr data index; offsets of objects in the left and right trees
# else: # Data area
#   2: data len (n); length of data
#   3+0: triangle index 0; index of the triangle...
#   3+1: triangle index 1
#   ...: .....
#   3+n-1: triangle index n-1
#
# next block

```


## Flat Traversal

Starting with the node offset at 0
- First, traverse the middle
- If intersecting only with the left side:
  - Use the left tree node offset as the root
- If intersecting only with the right side
  - Use the right tree node offset as the root
- If intersecting with both
  - Traverse lr data index;`
- Repeat

Intersecting with just the left or right side is easy to understand. Intersecting with both is illustrated in the situation below. Although the ray hits the left box, it does not intersect with any object inside the left box. Since the calculation must be completed in one go, all objects on both sides must be traversed.

```
    |------|   |------|
    |   A  |   |      |
-----------------> B  |
    |   A  |   |      |
    |------|   |------|
```

## Acknowledgments
A huge thank you to Taichi Graphics, Professor Tian Tian, all the teaching assistants, and everyone who contributed to the course!!!
