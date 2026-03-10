# Sovereign Knight Solver

[![Python](https://img.shields.io/badge/python-3.10+-blue.svg)](https://python.org)
[![License: MIT](https://img.shields.io/badge/license-MIT-yellow.svg)](LICENSE)
![Math](https://img.shields.io/badge/math-exact%20integer-green)
![Research](https://img.shields.io/badge/status-research-orange)
![Architecture](https://img.shields.io/badge/geometry-A2%20lattice-purple)

Exact-integer research framework for exploring the Diophantine equation

\[
x^3 + y^3 + z^3 = n
\]

using algebraic identities derived from Eisenstein integers and the A₂ hexagonal lattice.

The solver is built around exact mathematical filters and only records a hit after full integer reconstruction and cube verification.

## Core compression

Let

\[
\omega = \frac{-1+\sqrt{-3}}{2}
\]

and define

\[
\alpha = x - y\omega,\qquad N(\alpha)=x^2-xy+y^2=q,\qquad d=x+y
\]

Then

\[
x^3+y^3=d\,q=d\,N(\alpha)
\]

so the problem becomes

\[
n=z^3+dN(\alpha)
\]

## Main identities

\[
k=n-z^3
\]

\[
d=x+y,\qquad s=x-y
\]

\[
q=x^2-xy+y^2
\]

\[
k=dq
\]

\[
4q=d^2+3s^2
\]

\[
4(n-z^3)=d^3+3ds^2
\]

\[
x=\frac{d+s}{2},\qquad y=\frac{d-s}{2}
\]

## Features

- exact integer arithmetic only
- q-first, d-first, bruteforce, and auto modes
- Eisenstein norm filters
- A₂ shell geometry
- modular and parity filters
- partial-factorization honesty
- checkpoint and resume
- JSONL logs
- SQLite factor cache
- safe parallel fallback to one worker
- independent packet verification mode
- built-in target sets

## Installation

Clone the repository:

```bash
git clone https://github.com/YOURUSERNAME/sovereign-knight-solver
cd sovereign-knight-solver
