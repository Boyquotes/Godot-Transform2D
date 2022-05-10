André's proposal
===

* auto-gen TOC:
{:toc}

I imagine two kinds of `Transform2D` users:
1. **Regular users.**
Users that just want to place and manipulate an object in some (parent) coordinate system.
2. **Megazord matrix lovers power users.**
People that want to rotate things inside things that rotate and move around.

This proposal aims to be easy and intuitive for regular users,
and yet, powerful and fully functional for megazord users.

Although *regular users* usually manipulate `Node2D` and not `Transform2D` directly,
this proposal aims to keep methods of `Node2D` consistent with those of `Transform2D`.


What `Transform2D` is
---

Things in Godot are positioned according to coordinates.
We can imagine that the whole *world* has a **global** system of coordinates.
However, in general, we do not use global coordinates.
It is a good practice to develop things *locally*,
*encapsulated* in their own *context*.
In particular, each thing to be positioned somewhere should have its own set of *local coordinates*.

![Coordinate system](figs/coordinate_system.jpg)


In Godot, things are sort of *chained*. For example, some `Node A` can be put in the world, and some other `Node B` is put inside this first node. Each node has an instance of `Transform2D` that describes how its local coordinates can be converted to its parent's coordinate system. To **place a node** in its parent's coordinate system corresponds to properly setting its `Transform2D` instance.

Although we might think of *node* positioning, the `Transform2D` is not aware of the concept of *nodes*. What `Transform2D` does is to convert some pair of numbers (coordinates) into some other pair of numbers. We interpret this as if there is a (local) *coordinate system* inside some other (parent) *coordinate system*.

![Coordinate system inside another coordinate system](figs/coordinate_system_inside_coordinate_system.jpg)

When we instantiate a `Transform2D` object, we assume there is some *parent coordinate system*.

Therefore, an instance of `Transform2D` is, from the public interface point of view, as a set of three variables. We will give them name in this document, but this does not mean that `Transform2D` needs to be implemented internally using those names.
1. The `origin` (Vector2 origin --- $\vec{o}$).
This is where the coordinate $(0,0)$ is mapped to in the *parent coordinate system*.
From the *regular user*'s point of view, it is `position`.
From the *megazord user*'s point of view, it is `origin`.
2. The `x-axis` (Vector2 b1 --- $\vec{b_1}$).
Departing from `origin`, this represents one unit in the $x$ direction in the *parent coordinate system*.
3. The `y-axis` (Vector2 b2 --- $\vec{b_2}$).
Departing from `origin`, this represents one unit in the $y$ direction in the *parent coordinate system*.

All vectors above are given in the *parent coordinate system*.

A *local coordinate* $(a,b)$, shall yeld
\begin{equation\*}
  (a_{\text{new}}, b_{\text{new}})
  =
  \vec{o} + a \vec{b_1} + b \vec{b_2}
\end{equation\*}
when converted to the *parent coordinate system*.

For those who like matrices, let us agree that vectors will generally be given as *column* matrices. In this case, `Transform2D` is represented by an <a href="https://en.wikipedia.org/wiki/Affine_transformation#Augmented_matrix">augmented matrix</a>. Local to parent coordinate conversion corresponds to
\begin{equation\*}
  \begin{bmatrix}
    a_{\text{new}}
    {}\\\\{}
    b_{\text{new}}
    {}\\\\{}
    1
  \end{bmatrix}
  =
  \begin{bmatrix}
    b1.x & b2.x & origin.x
    {}\\\\{}
    b1.y & b2.y & origin.y
    {}\\\\{}
    0 & 0 & 1
  \end{bmatrix}
  \begin{bmatrix}
    a
    {}\\\\{}
    b
    {}\\\\{}
    1
  \end{bmatrix}.
\end{equation\*}

In this document,
let us agree to use the symbols $T$ and $B$ to represent the following matrices
\begin{equation\*}
  T
  =
  \begin{bmatrix}
    b1.x & b2.x & origin.x
    {}\\\\{}
    b1.y & b2.y & origin.y
    {}\\\\{}
    0 & 0 & 1
  \end{bmatrix}
  \quad\text{e}\quad
  B
  =
  \begin{bmatrix}
    b1.x & b2.x
    {}\\\\{}
    b1.y & b2.y
  \end{bmatrix}
\end{equation\*}

### Chaining transforms

When we have two `Transform2D`s, $T_1$ and $T_2$,
they can be chained and form a new `Transform2D` $T$.
To be precise, applying $T$ to some `Vector2D` $\vec{v}$
is the same as applying $T_1$ to $\vec{v}$ and then applying $T_2$ to the result.
In terms of matrix notation,
\begin{equation\*}
  T = T_2 * T_1.
\end{equation\*}
However,
**it does not mean** that *matrix multiplication* is always a good / adequate way
to manipulate of a `Transform2D` instance.


Public methods
===

**Attention:**
This does not tell how each method shall be implemented internally.
The idea is to simply describe the method behaviour...
but some times it is easier to use code instead of words.


Position
---

Those methods place or move the local coordinate system inside another (parent) coordinate system.

### get_origin()

```
  const Vector2 Transform2D::get_origin()
  {
    return origin;
  }
```

### set_origin()

```
  void Transform2D::set_origin(const Vector2 &position)
  {
    origin = position;
  }
```

### translate()

It is like if the *parent* is moving the *child* coordinate system around.
If you throw a stone,
you **don't do it** in the stone's internal coordinate system!
You do it in the coordinate system where the stone is placed.
That is,
you do it in the **parent coordinate system**.
For that reason,
`translation` is given in the *parent coordinate system*.
Not in *local coordinates*.

```
  void Transform2D::translate(const Vector2 &translation)
  {
    origin += translation;
  }
```

### translate_local()

This is like if the object is moving **itself**, relative to its own units / size.
Like a spaceship moving forward.
If you double its size, the speed will be doubled as well.

```
  void Transform2D::translate_local(const Vector2 &translation)
  {
    origin += translation.x * b1 + translation.y * b2;
  }
```


### Compatibility issue

In current implementation, our `translate` does not exist.
There is no method for **moving things** around.
Current's implementation `translate` is what we call `translate_local`.


Scaling
---

The current scaling implementation is, IMO, a little messy.
I really think there is **not much use** for uneven scaling (different scale for $x$ and $y$).
Uneven scaling can be achieved by directly manipulating $\vec{b_1}$ and $\vec{b_2}$.

Here is an ideal world.
But it is important to notice that the real world is not ideal,
and `Node2D` makes use of the uneven version of *scaling*
(although not through the methods described here).
Therefore,
I shall write a second proposal that specifies uneven scaling if there is a demand.


### get_scale()

This function makes the assumption that there is a uniform scale.
That is, it assumes that `b1.length() == b2.length()`.
It makes no sense to call `get_scale()` if the scaling is uneven.

```
  real_t get_scale() const
  {
    return b1.length();
  }
```


### set_scale()

Sets the size of the vectors $\vec{b_1}$ and $\vec{b_2}$.
The *position* (*origin*) does not change:
the object will be scaled *in place*.

```
  void set_scale(real_t s)
  {
    b1.normalize();
    b2.normalize();
    scale(s);
  }
```


### scale()

Multiplies the size of the vectors $\vec{b_1}$ and $\vec{b_2}$ by $s$.
The *position* (*origin*) does not change:
the object will be scaled *in place*.

```
  void scale(real_t s)
  {
    b1*= scale;
    b2*= scale;
  }
```



### Compatibility issue

#### Uneven scaling

The current implementation allows uneven scaling.
That is,
it allows a different scaling on the $x$ and $y$ directions.
The implementation is not consistent, however,
because sometimes $x$ and $y$ directions refers to the local $x$ and $y$ directions,
and some times it refers to the parents $x$ and $y$ directions.

For example,
the methods `Transform2D::set_rotation_and_scale`
and `Transform2D::set_rotation_scale_and_skew`
assume that the $x$ and $y$ directions are the local versions.
For that reason they execute code like
```
  columns[0][0] = Math::cos(p_rot) * p_scale.x;
  columns[0][1] = Math::sin(p_rot) * p_scale.x;
```
The same is true of `Transform2D::set_scale`, as it does
```
  columns[0] *= p_scale.x;
```
However, `Transform2D::scale` calls `Transform2D::scale_basis`, that does
```
  columns[0][0] *= p_scale.x;
  columns[0][1] *= p_scale.y;
```

In this document we suggest that the methods `set_scale` and `scale`
simply do not take two parameters.
And `get_scale` returns only one real number.

In the long term,
we suggest that `Node2D` use
`Translate2D::set_rotation`,
`Translate2D::rotate`,
`Translate2D::set_scale`,
`Translate2D::scale`,
`Translate2D::translate`,
etc
instead of calling
`Transform2D::set_rotation_and_scale`
and `Transform2D::set_rotation_scale_and_skew`
through `Transform2D::_update_transform`.
Notice that `Node2D::rotate` does
```
  set_rotation(get_rotation() + p_radians);
```
while it would be much more efficient to call
```
  transform.rotate(p_radians);
```


#### Scaling does not change `origin`

The current implementation is not very consistent.
Sometimes it assumes scaling is done about the `origin`,
and some times it assumes it is done about the parents `origin`.
That is,
sometimes it rescales `origin`, and some times it does not.

In this proposal,
we completely abolish scaling that is not about the local `origin`.
That is,
no proposed scaling method changes `origin`.


### What scaling is NOT



Rotating
---

### rotate

This method rotates the `axis` $\vec{b_1}$ and $\vec{b_2}$ about the `origin`.
It does not touch the `origin`.

```
  void Transform2D::rotate(const real_t radians)
  {
    real_t cs = Math:cos(radians);
    real_t sn = Math:sin(radians);
    b1 = Vector2(b1.x * cs - b1.y * sn, b1.x * sn + b1.y * cs);
    b2 = Vector2(b2.x * cs - b2.y * sn, b2.x * sn + b2.y * cs);
  }
```

In matrix notation,
\begin{equation\*}
  \begin{bmatrix}
    b_j.x_{\text{new}}
    {}\\\\{}
    b_j.y_{\text{new}}
  \end{bmatrix}
  =
  \begin{bmatrix}
    \cos(\theta) & -\sin(\theta)
    {}\\\\{}
    \sin(\theta) & \cos(\theta)
  \end{bmatrix}
  \begin{bmatrix}
    b_j.x
    {}\\\\{}
    b_j.y
  \end{bmatrix}
\end{equation\*}


### get_rotation

Returns the angle in radians from the parent's $x$-axis to $\vec{b_1}$.


### set_rotation

Rotates $\vec{b_1}$ and $\vec{b_2}$ the exact amount so that the angle
from the parent $x$-axis to $\vec{b_1}$ is the given `radians`.

The implementation can simply rotate backwards to get an angle of $0$,
and then call rotate.

```
  void Transform2D::set_rotation(const real_t radians)
  {
    real_t len = b1.length();
    real_t cs = b1.x / len;
    real_t sn = -b1.y / len;
    b1 = Vector2(b1.x * cs - b1.y * sn, b1.x * sn + b1.y * cs);
    b2 = Vector2(b2.x * cs - b2.y * sn, b2.x * sn + b2.y * cs);
    rotate(radians);
  }
```
In matrix notation,
\begin{equation\*}
  \begin{bmatrix}
    b_j.x_{\text{new}}
    {}\\\\{}
    b_j.y_{\text{new}}
  \end{bmatrix}
  =
  \frac{1}{\sqrt{\vec{b_1}.x^2 + \vec{b_1}.y^2}}
  \begin{bmatrix}
    \cos(\theta) & -\sin(\theta)
    {}\\\\{}
    \sin(\theta) & \cos(\theta)
  \end{bmatrix}
  \begin{bmatrix}
    \vec{b_1}.x & \vec{b_1}.y
    {}\\\\{}
    -\vec{b_1}.y & \vec{b_1}.x
  \end{bmatrix}
  \begin{bmatrix}
    b_j.x
    {}\\\\{}
    b_j.y
  \end{bmatrix}
\end{equation\*}
Therefore,
a more efficient implementation would use a precalculated matrix product formula,
instead of calling `rotate()`.

Notice that `set_rotation()` is much more complicated then `rotation()`.
The current implementation is not only more complicate.
It is also **buggy**!!!
In the current implementation,
`set_rotation()` **always** returns orthogonal axis!!!


Also, notice that `Node2D::rotate` does
```
  set_rotation(get_rotation() + p_radians);
```
while it would be much more efficient to call
```
  transform.rotate(p_radians);
```
The `Node2D` implementation is an indicative that
`Translate2D` is poorly designed / specified.


Flipping
---

These methods are not really needed.
Its specifications were motivated by the fact that with *uneven scaling* on can actually call
`scale(Vector2(1,-1))`.
Since we are not allowing uneven scaling in this specification,
the *flipping* operations can be useful.


### flip_axis()

Flips (mirror) the coordinate system about the given `axis` through the `origin`.
The `origin` is kept fixed.

```
  void flip_axis(Vector2 axis)
  {
    real_t len2 = axis.x * axis.x + axis.y * axis.y;
    real_t p = (axis.y * axis.y - axis.x * axis.x) / len2;
    real_t q = -2 * axis.x * axis.y / len2;
    b1 = Vector2(p * b1.x + q * b1.y, q * b1.x - p * b1.y);
    b2 = Vector2(p * b2.x + q * b2.y, q * b2.x - p * b2.y);
  }
```

In matrix notation,
assuming `axis` is normalized and equal to $(a_1, a_2)$,
$\vec{b_1}$ and $\vec{b_2}$ shall be processed by the matrix
\begin{equation\*}
  \begin{bmatrix}
    p & q
    {}\\\\{}
    q & -p
  \end{bmatrix}
  =
  \begin{bmatrix}
    a_2^2 - a_1^2 & -2 a_1 a_2
    {}\\\\{}
    -2 a_1 a_2 & a_1^2 - a_2^2
  \end{bmatrix}
  =
  \begin{bmatrix}
    a_1 & -a_2
    {}\\\\{}
    a_2 & a_1
  \end{bmatrix}
  \begin{bmatrix}
    1 & 0
    {}\\\\{}
    0 & -1
  \end{bmatrix}
  \begin{bmatrix}
    a_1 & a_2
    {}\\\\{}
    -a_2 & a_1
  \end{bmatrix}.
\end{equation\*}
This can be made more efficient if we do not normalize `axis`,
and instead divide the result by $a_1^2 + a_2^2$.


### flip_direction()

Flips (mirror) the image toward the given `direction` through the `origin`.
The `origin` is kept fixed.

One possible implementation is to simply rotate `direction` 90 degrees to get the rotation axis.
And then, call `flip_axis()`.

```
  void flip_direction(Vector2 direction)
  {
    flip_axis(Vector2(direction.y, -direction.x));
  }
```


### is_flipped()

Returns true is the image is *flipped* (mirrored).

```
  bool is_flipped()
  {
    return (b1.x * b2.y - b1.y * b2.x < 0);
  }
```


Setting skew
---

Set axis
---

Discussion
===

Scaling about what point?
---

In its simplest form,
*scaling* is just making the thing bigger or smaller.
But there is one catch:
> When you *scale* something,
> you do it about a certain point, that is kept fixed.

With all due respect,
the current implementation is quite messy about this.
The method used by `Node2D` (`set_rotation_scale_and_skew()`)
assumes that scaling is always done keeping the `origin` fixed.
Exactly as we propose `scale` and `set_scale` should behave.

However,
in current implementation,
`scale` and `set_scale` assume that the `origin` shall also move with *scaling*.
That is,
scaling is done about the parent's `origin`.
This is **not consistent** with `Node2D` behaviour.
Of course,
it would make more sense if `Translatb2D::set_scale()`
was consistent with `Node2D::set_scale()`.

Also,
by the way, there is no `Node2D::scale()`.
But there is a `Node2D::apply_scale()`.
Again,
it would make sense to make `Transform2D::scale()`
consistent with `Node2D::apply_scale()`.

I would say that the need to scaling about the *parent*'s origin
is an indication of *bad design*.
I see two use cases:

1. **Scenario 1:** There is a node `A` with many child nodes.
The programmer wants to rescale the whole contents of node `A` by rescaling every child.
The **correct** approach here would be to rescale `A`, and not its children.
2. **Scenario 2:** There is a node `A` and a group of its child nodes.
The programmer wants to rescale only those child nodes in this group, not the other child nodes.
The **correct** approach here would be to create a node `G`, put this special group of child nodes of `A` inside `G`, instead. And then, and place `G` in `A`. Now, instead of rescaling every child node and their `origin`, one should simply rescale `G`.


Uneven scaling :-(
---

In current implementation, scaling is a little more complicated then here,
because it allows one to apply a different scale to the $x$ and $y$ directions.
Let's say $s_x$ in the direction $x$ and $s_y$ in the direction $y$.
Again,
there is a catch:
> When we say $x$ direction...
> are we referring to the *local* $x$ direction,
> or to the *parent* $x$ direction?

To scale relative to the local coordinate system, all we have to do is
```
  b1 *= sx;
  b2 *= sy;
```
To scale relative to the parent coordinate system, we need to scale the $x$ coordinate of $\vec{b_1}$ and $\vec{b_2}$, and also the $y$ coordinate of both.
But `Vector2` also has a method that can be used:
```
  b1 *= Vector2(sx, sy);
  b2 *= Vector2(sx, sy);
```



With all due respect,
current implementation is certainly messy about it.
Some methods assume...
There are methods called `set_rotation_and_scale` and `set_rotation_scale_and_skew`.
Those two methods scale about the `origin`, keeping it fixed.

The implementation of...

