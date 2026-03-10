---

1. README.md

# Sovereign Knight Solver
[![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://python.org)
[![License: MIT](https://img.shields.io/badge/license-MIT-yellow.svg)](LICENSE)
![Math](https://img.shields.io/badge/Math-Exact%20Integer-green)
![Architecture](https://img.shields.io/badge/Architecture-A2%20Lattice-purple)
![Status](https://img.shields.io/badge/status-research-orange)

Exact-integer research framework for the Diophantine equation

\[
x^3 + y^3 + z^3 = n
\]

using a structured pipeline built from

- Eisenstein integer norms
- A₂ hexagonal lattice geometry
- packet / shell decomposition
- strict mathematical filter identities

The solver searches for integer solutions using only **verified algebraic constraints**.

---

## Mathematical Structure

Define

\[
\omega=\frac{-1+\sqrt{-3}}{2}
\]

\[
\alpha = x - y\omega
\]

Eisenstein norm

\[
N(\alpha)=x^2-xy+y^2=q
\]

Let

\[
d=x+y
\]

Then

\[
x^3+y^3=d\,N(\alpha)
\]

so the cubic equation becomes

\[
n = z^3 + d\,N(\alpha)
\]

---

## Core Identities

\[
k=n-z^3
\]

\[
d=x+y
\]

\[
s=x-y
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
x=\frac{d+s}{2},\quad y=\frac{d-s}{2}
\]

---

## Mathematical Filters

Each candidate must satisfy

1. admissible residue
2. cube transport
3. packet factorization
4. shell identity
5. parity constraints
6. Eisenstein norm rules
7. primitive packet laws
8. exact reconstruction

The solver only records a hit if the cube identity verifies exactly.

---

## Features

- Exact integer arithmetic
- Eisenstein lattice shell geometry
- mirror symmetry reduction
- modular residue filters
- Pollard-Rho factorization
- SQLite factor cache
- checkpoint / resume
- multi-process workers with safe fallback
- fixture validation tests
- Everest target runner

---

## Installation

Requires Python 3.10+

```bash
git clone https://github.com/YOURUSERNAME/sovereign-knight-solver
cd sovereign-knight-solver
python3 sovereign_knight_solver.py --help


---

Example

python3 sovereign_knight_solver.py \
--n 153 \
--z-start -200 \
--z-stop 200 \
--mode qfirst_corridor


---

Everest Targets

Current difficult values

114
390
627
633
732
921
975

Run:

python3 sovereign_knight_solver.py \
--target-set everest \
--z-start -1000 \
--z-stop 1000


---

Research Scope

This repository implements a computational framework for exploring solutions to

x^3+y^3+z^3=n

It is not a proof of unsolved targets unless a verified integer triple is found.


---

Author

Riley Cameron Byrd


---

License

MIT

---

# 2. LICENSE (MIT)

```text
MIT License

Copyright (c) 2026 Riley Cameron Byrd

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files to deal in the Software
without restriction, including without limitation the rights to use, copy,
modify, merge, publish, distribute, sublicense, and/or sell copies of the
Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND.


---

3. CONTRIBUTING.md

# Contributing

Thank you for your interest in contributing.

This project focuses on **exact integer mathematics** and research related to the equation

x³ + y³ + z³ = n.

## Guidelines

1. All algorithms must maintain **exact integer correctness**
2. No floating-point heuristics may replace verification
3. All proposed improvements must include mathematical justification
4. Code must remain readable and documented
5. Pull requests should include tests when applicable

## Development Setup

Clone repository

git clone https://github.com/YOURUSERNAME/sovereign-knight-solver

Run tests

python3 sovereign_knight_solver.py --self-check

## Types of Contributions

- algorithm improvements
- factorization optimizations
- mathematical filters
- visualization tools
- documentation

Thank you for helping advance computational number theory.


---

4. CODE_OF_CONDUCT.md

# Code of Conduct

This project follows a standard open-source conduct policy.

Participants are expected to:

- be respectful
- focus on constructive discussion
- avoid harassment or discrimination

The goal of this repository is collaborative mathematical research.

Issues or concerns can be reported to the maintainer.


---

5. pyproject.toml

This allows people to install it like a Python package.

[project]
name = "sovereign-knight-solver"
version = "0.1.0"
description = "Exact integer solver for the sum of three cubes using Eisenstein lattice methods"
authors = [{name="Riley Cameron Byrd"}]
readme = "README.md"
requires-python = ">=3.10"

[project.urls]
Homepage = "https://github.com/YOURUSERNAME/sovereign-knight-solver"

[tool.setuptools]
py-modules = ["sovereign_knight_solver"]


---

6. Suggested Repository Layout

sovereign-knight-solver/
│
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── pyproject.toml
│
├── sovereign_knight_solver.py
│
├── fixtures/
├── docs/
│
└── examples/


---

Final Advice

To make the repo look very legitimate, also:

Add a GitHub Topics list

number-theory
diophantine-equations
sum-of-three-cubes
eisenstein-integers
computational-mathematics
