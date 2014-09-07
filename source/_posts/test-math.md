title: test math
date: 2014-09-06 21:42:29
tags:
---
Simple inline $a = b + c$. ab

$$\frac{\partial u}{\partial t}
= h^2 \left( \frac{\partial^2 u}{\partial x^2} +
\frac{\partial^2 u}{\partial y^2} +
\frac{\partial^2 u}{\partial z^2}\right)$$

This equation {% math \cos 2\theta = \cos^2 \theta - \sin^2 \theta =  2 \cos^2 \theta - 1 %} is inline.

{% math-block %}
\begin{aligned}
\dot{x} & = \sigma(y-x) \\
\dot{y} & = \rho x - y - xz \\
\dot{z} & = -\beta z + xy
\end{aligned}
{% endmath-block %}