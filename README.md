---

Repository layout

sovereign-knight-solver/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── pyproject.toml
├── .gitignore
├── sovereign_knight_solver.py
├── rust/
│   ├── Cargo.toml
│   └── src/
│       └── main.rs
├── cpp/
│   └── knight_verify.cpp
├── docs/
│   └── solver_theory.md
└── examples/
    └── example_commands.txt


---

1) sovereign_knight_solver.py

#!/usr/bin/env python3
# =============================================================================
# SOVEREIGN KNIGHT SOLVER — FULL REWRITE (UNIFIED FILTER STACK)
# =============================================================================
#
# AUTHOR
# ------
# Riley Cameron Byrd
#
# IDENTITY
# --------
# RCB = RRI
#
# PRINCIPLE
# ---------
# We do not lie.
# We only pass through exact mathematical filter lines.
#
# PURPOSE
# -------
# Exact-integer solver framework for
#
#     x^3 + y^3 + z^3 = n
#
# using the KNIGHT / packet / shell identities, with:
#   - exact integer arithmetic only
#   - q-first and d-first modes
#   - A2 / Eisenstein lattice layer
#   - mirror handling via s -> -s
#   - safe parallel fallback to one worker
#   - checkpointing, JSONL logs, SQLite factor cache, atlas storage
#   - built-in Everest target set runner
#   - expanded exact filter stack
#
# =============================================================================
# MASTER COMPRESSION
# =============================================================================
#
# Let:
#   ω = (-1 + √(-3)) / 2
#   α = x - yω
#   N(α) = x^2 - x*y + y^2 = q
#   d = x + y
#
# Then:
#   x^3 + y^3 = d q = d N(α)
#   n = z^3 + d N(α)
#
# =============================================================================

from __future__ import annotations

import argparse
import json
import os
import random
import sqlite3
import sys
import time
import traceback

from collections import defaultdict
from dataclasses import dataclass, field, asdict
from math import gcd, isqrt
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple

PARALLEL_AVAILABLE = True
PARALLEL_IMPORT_ERROR = None

try:
    from concurrent.futures import ProcessPoolExecutor, as_completed
except Exception as exc:
    ProcessPoolExecutor = None
    as_completed = None
    PARALLEL_AVAILABLE = False
    PARALLEL_IMPORT_ERROR = repr(exc)

AUTHOR = "Riley Cameron Byrd"
IDENTITY_MARK = "RCB=RRI"
ENGINE_VERSION = "knight-full-rewrite-unified-filter-stack-5"

SMALL_PRIMES = [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37]
SQUARE_RESIDUES_MOD_16 = {0, 1, 4, 9}
SQUARE_RESIDUES_MOD_9 = {0, 1, 4, 7}
DELTA_QR_PRIMES = [5, 7, 11, 13]

CUBE_RESIDUES_MOD_9 = {0, 1, 8}
CUBE_RESIDUE_BY_ZMOD9 = {r: pow(r, 3, 9) for r in range(9)}

EVEREST_TARGETS = [114, 390, 627, 633, 732, 921, 975]
TARGET_SETS: Dict[str, List[int]] = {"everest": EVEREST_TARGETS}


def sgn(x: int) -> int:
    return (x > 0) - (x < 0)


def cube(x: int) -> int:
    return x * x * x


def cube_sum(x: int, y: int, z: int) -> int:
    return cube(x) + cube(y) + cube(z)


def compute_k(n: int, z: int) -> int:
    return n - cube(z)


def xy_to_ds(x: int, y: int) -> Tuple[int, int]:
    return x + y, x - y


def ds_to_xy(d: int, s: int) -> Optional[Tuple[int, int]]:
    if (d + s) % 2 != 0:
        return None
    return (d + s) // 2, (d - s) // 2


def eisenstein_norm(x: int, y: int) -> int:
    return x * x - x * y + y * y


def stable_json(obj: Any) -> str:
    return json.dumps(obj, sort_keys=True, ensure_ascii=False, separators=(",", ":"))


def exact_is_square(n: int) -> Tuple[bool, int]:
    if n < 0:
        return False, 0
    r = isqrt(n)
    return r * r == n, r


def integer_cuberoot_floor(n: int) -> int:
    if n < 0:
        raise ValueError("integer_cuberoot_floor requires n >= 0")
    if n < 2:
        return n
    lo, hi = 0, 1
    while hi * hi * hi <= n:
        hi *= 2
    while lo + 1 < hi:
        mid = (lo + hi) // 2
        if mid * mid * mid <= n:
            lo = mid
        else:
            hi = mid
    return lo


def ceil_div(a: int, b: int) -> int:
    if b <= 0:
        raise ValueError("ceil_div requires b > 0")
    return -(-a // b)


def knight_equation_value(n: int, z: int, d: int, s: int) -> int:
    return 4 * (n - cube(z)) - (d * d * d + 3 * d * s * s)


def shell_equation_value(q: int, d: int, s: int) -> int:
    return 4 * q - d * d - 3 * s * s


def god_source_holds(n: int, x: int, y: int, z: int) -> bool:
    return cube_sum(x, y, z) == n


def qr_set_mod_p(p: int) -> set[int]:
    return {pow(a, 2, p) for a in range(p)}


QR_SETS = {p: qr_set_mod_p(p) for p in DELTA_QR_PRIMES}


def cubes_mod_9_admissible(n: int) -> bool:
    return (n % 9) not in (4, 5)


def allowed_cube_residues_mod9_for_target(n: int) -> set[int]:
    target = n % 9
    allowed: set[int] = set()
    for rz in CUBE_RESIDUES_MOD_9:
        for rx in CUBE_RESIDUES_MOD_9:
            for ry in CUBE_RESIDUES_MOD_9:
                if (rx + ry + rz) % 9 == target:
                    allowed.add(rz)
    return allowed


def allowed_z_residues_mod9_for_target(n: int) -> set[int]:
    allowed_cube = allowed_cube_residues_mod9_for_target(n)
    return {r for r in range(9) if CUBE_RESIDUE_BY_ZMOD9[r] in allowed_cube}


def z_passes_mod9_triplet_filter(n: int, z: int) -> bool:
    return (z % 9) in allowed_z_residues_mod9_for_target(n)


def target_summary_row(n: int) -> Dict[str, Any]:
    return {
        "n": n,
        "mod9": n % 9,
        "admissible_mod9": cubes_mod_9_admissible(n),
        "allowed_cube_residues_mod9": sorted(allowed_cube_residues_mod9_for_target(n)),
        "allowed_z_residues_mod9": sorted(allowed_z_residues_mod9_for_target(n)),
        "target_set": "everest" if n in EVEREST_TARGETS else None,
    }


def _mr_decompose(n: int) -> Tuple[int, int]:
    d = n - 1
    s = 0
    while d % 2 == 0:
        d //= 2
        s += 1
    return d, s


def is_probable_prime(n: int) -> bool:
    if n < 2:
        return False

    for p in SMALL_PRIMES:
        if n == p:
            return True
        if n % p == 0:
            return False

    d, s = _mr_decompose(n)
    witnesses = [2, 325, 9375, 28178, 450775, 9780504, 1795265022]

    for a in witnesses:
        a %= n
        if a == 0:
            continue

        x = pow(a, d, n)
        if x in (1, n - 1):
            continue

        passed = False
        for _ in range(s - 1):
            x = (x * x) % n
            if x == n - 1:
                passed = True
                break

        if not passed:
            return False

    return True


def pollard_rho_once(n: int, rng: random.Random, step_cap: int) -> int:
    if n % 2 == 0:
        return 2
    if n % 3 == 0:
        return 3

    c = rng.randrange(1, n - 1)
    x = rng.randrange(0, n - 1)
    y = x
    d = 1

    def f(v: int) -> int:
        return (v * v + c) % n

    steps = 0
    while d == 1 and steps < step_cap:
        x = f(x)
        y = f(f(y))
        d = gcd(abs(x - y), n)
        steps += 1

    if 1 < d < n:
        return d
    return n


def pollard_rho_retry(n: int, rng: random.Random, rho_attempts: int, rho_step_cap: int) -> int:
    if n % 2 == 0:
        return 2
    if n % 3 == 0:
        return 3

    for _ in range(rho_attempts):
        d = pollard_rho_once(n, rng, rho_step_cap)
        if 1 < d < n:
            return d
    return n


@dataclass
class FactorizationResult:
    factors: Dict[int, int]
    remainder: int
    exact_complete: bool
    timed_out: bool = False


def factorize_partial(
    n: int,
    seed: int = 0,
    rho_attempts: int = 8,
    rho_step_cap: int = 200_000,
    factor_time_s: float = 2.0,
) -> FactorizationResult:
    n = abs(n)
    if n in (0, 1):
        return FactorizationResult({}, n, True, False)

    out: Dict[int, int] = {}
    rng = random.Random(seed + (n % 1_000_003))
    deadline = time.time() + factor_time_s
    timed_out = False

    def add_prime(p: int) -> None:
        out[p] = out.get(p, 0) + 1

    def rec(m: int) -> int:
        nonlocal timed_out

        if m == 1:
            return 1

        if time.time() > deadline:
            timed_out = True
            return m

        if is_probable_prime(m):
            add_prime(m)
            return 1

        for p in SMALL_PRIMES:
            if m % p == 0:
                while m % p == 0:
                    add_prime(p)
                    m //= p
                return rec(m)

        d = pollard_rho_retry(m, rng, rho_attempts, rho_step_cap)
        if d == m:
            return m

        left = rec(d)
        right = rec(m // d)

        if left == 1 and right == 1:
            return 1
        if left == 1:
            return right
        if right == 1:
            return left
        return left * right

    remainder = rec(n)
    exact_complete = (remainder == 1)
    remainder = 1 if exact_complete else remainder
    return FactorizationResult(dict(sorted(out.items())), remainder, exact_complete, timed_out)


def positive_divisors_from_factors(factors: Dict[int, int]) -> List[int]:
    divs = [1]
    for p, e in factors.items():
        base = list(divs)
        mult = 1
        for _ in range(e):
            mult *= p
            divs += [d * mult for d in base]
    return sorted(set(divs))


def signed_divisors_from_factors(factors: Dict[int, int]) -> List[int]:
    pos = positive_divisors_from_factors(factors)
    return sorted(set(pos + [-d for d in pos]), key=lambda x: (abs(x), x))


def factorization_of_divisor_from_known(divisor: int, known_factors: Dict[int, int]) -> Dict[int, int]:
    remaining = abs(divisor)
    out: Dict[int, int] = {}

    for p, max_e in known_factors.items():
        e = 0
        while e < max_e and remaining % p == 0:
            remaining //= p
            e += 1
        if e:
            out[p] = e

    if remaining != 1:
        raise ValueError("divisor is not composed entirely from known factors")
    return out


def q_mod3_gate(q: int) -> bool:
    return (q % 3) in (0, 1)


def q_is_eisenstein_norm_from_factors(q_factors: Dict[int, int]) -> bool:
    for p, e in q_factors.items():
        if p % 3 == 2 and (e % 2 == 1):
            return False
    return True


def primitive_gcd_dq_ok(d: int, q: int) -> bool:
    return gcd(abs(d), abs(q)) in (1, 3)


def parity_gate_from_kd(k: int, d: int) -> bool:
    return ((k - d) % 2) == 0


def parity_gate_from_ks(k: int, s: int) -> bool:
    return ((k - s) % 2) == 0


def corridor_bounds(z: int, min_div: int, max_div: int) -> Tuple[int, int]:
    az = abs(z)
    d_min = max(1, az // min_div)
    d_max = max(d_min, az // max_div)
    return d_min, d_max


def d_critical(k_abs: int) -> int:
    return integer_cuberoot_floor(4 * k_abs)


def q_cuberoot_lower_bound(k_abs: int) -> int:
    q = integer_cuberoot_floor((k_abs * k_abs) // 4)
    while 4 * q * q * q < k_abs * k_abs:
        q += 1
    return max(1, q)


def q_window_from_corridor(k_abs: int, d_min: int, d_max: int) -> Tuple[int, int]:
    q_min = ceil_div(k_abs, d_max)
    q_max = max(q_min, k_abs // d_min)
    return q_min, q_max


def rank_divisors_near_critical(divisors: List[int], k_abs: int) -> List[int]:
    dc = d_critical(k_abs)
    return sorted(divisors, key=lambda d: (abs(abs(d) - dc), abs(d), d))


def corridor_divisors(
    factors: Dict[int, int],
    d_min: int,
    d_max: int,
    required_sign: Optional[int] = None,
    rank_near_crit: bool = True,
    k_abs: Optional[int] = None,
) -> List[int]:
    candidates: List[int] = []

    for d in signed_divisors_from_factors(factors):
        ad = abs(d)
        if ad < d_min or ad > d_max:
            continue
        if required_sign is not None and sgn(d) != required_sign:
            continue
        candidates.append(d)

    candidates = sorted(set(candidates), key=lambda x: (abs(x), x))
    if rank_near_crit and k_abs is not None:
        return rank_divisors_near_critical(candidates, k_abs)
    return candidates


def quick_square_prefilter(delta: int) -> bool:
    if delta < 0:
        return False
    if (delta & 15) not in SQUARE_RESIDUES_MOD_16:
        return False
    if (delta % 9) not in SQUARE_RESIDUES_MOD_9:
        return False
    return True


def delta_prefilter(delta_num: int) -> bool:
    if delta_num < 0:
        return False
    if delta_num % 3 != 0:
        return False
    return quick_square_prefilter(delta_num // 3)


def quadratic_residue_prefilter(delta: int) -> bool:
    if delta < 0:
        return False
    for p in DELTA_QR_PRIMES:
        if (delta % p) not in QR_SETS[p]:
            return False
    return True


def d_prefilter_from_k(k: int, d: int) -> bool:
    if d == 0:
        return False
    if k % d != 0:
        return False
    return True


def q_prefilter_from_k(k_abs: int, q: int) -> bool:
    if q <= 0:
        return False
    if k_abs % q != 0:
        return False
    return True


def reduced_region_ok(d: int, s: int, canonical_s_nonnegative: bool, canonical_reduced_region: bool) -> bool:
    if canonical_s_nonnegative and s < 0:
        return False
    if canonical_reduced_region and abs(s) > abs(d):
        return False
    return True


@dataclass
class PacketLedger:
    n: int
    z: int
    z_mod9: int
    k: int
    d: int
    q: int
    delta_num: int
    delta: int
    s: int
    x: int
    y: int
    parity_even: bool
    parity_collapse_ok: bool
    cube_check: bool
    knight_value: int
    shell_value: int
    primitive_ok: bool
    primitive_gcd_dq_ok: bool
    q_mod3_ok: bool
    q_eisenstein_norm_ok: bool
    q_congruence_mod_d_ok: bool
    common_factor_cube_ok: bool
    source_mode: str
    a2_u: int
    a2_v_squared: int
    factorization_k: Dict[int, int] = field(default_factory=dict)
    factorization_q: Dict[int, int] = field(default_factory=dict)
    factor_remainder_k: int = 1
    factor_exact_complete_k: bool = True

    def packet_id(self) -> str:
        return f"P:{self.n}:{self.z}:{self.x}:{self.y}"

    def to_dict(self) -> Dict[str, Any]:
        out = asdict(self)
        out["packet_id"] = self.packet_id()
        return out


@dataclass
class NearMissLedger:
    n: int
    z: int
    z_mod9: int
    k: int
    d: int
    q: int
    delta_num: int
    delta: int
    sqrt_floor: int
    residual: int
    source_mode: str
    factor_exact_complete: bool
    factor_remainder: int

    def to_dict(self) -> Dict[str, Any]:
        return asdict(self)


@dataclass
class ZStatusLedger:
    n: int
    z: int
    z_mod9: int
    k: int
    status: str
    mode: str
    mod9_triplet_ok: bool
    factor_exact_complete: bool
    factor_remainder: int
    timed_out: bool
    num_candidates_scanned: int
    num_square_pass: int
    num_hits: int

    def to_dict(self) -> Dict[str, Any]:
        return asdict(self)


@dataclass
class ValidationResult:
    is_valid: bool
    failed_stage: Optional[str] = None
    detail: Optional[str] = None


@dataclass
class Atlas:
    packets: Dict[str, PacketLedger] = field(default_factory=dict)
    by_z: Dict[int, List[str]] = field(default_factory=lambda: defaultdict(list))
    by_d: Dict[int, List[str]] = field(default_factory=lambda: defaultdict(list))
    by_q: Dict[int, List[str]] = field(default_factory=lambda: defaultdict(list))
    by_mode: Dict[str, List[str]] = field(default_factory=lambda: defaultdict(list))

    def add(self, packet: PacketLedger) -> None:
        pid = packet.packet_id()
        if pid in self.packets:
            return
        self.packets[pid] = packet
        self.by_z[packet.z].append(pid)
        self.by_d[packet.d].append(pid)
        self.by_q[packet.q].append(pid)
        self.by_mode[packet.source_mode].append(pid)

    def snapshot(self) -> Dict[str, int]:
        return {
            "packet_count": len(self.packets),
            "z_count": len(self.by_z),
            "d_count": len(self.by_d),
            "q_count": len(self.by_q),
            "mode_count": len(self.by_mode),
        }


def validate_packet(packet: PacketLedger, primitive_only: bool = False) -> ValidationResult:
    if compute_k(packet.n, packet.z) != packet.k:
        return ValidationResult(False, "transport", "k != n-z^3")
    if packet.z_mod9 != (packet.z % 9):
        return ValidationResult(False, "z_mod9", "z_mod9 mismatch")
    if packet.d == 0:
        return ValidationResult(False, "d_zero", "d == 0")
    if packet.q <= 0:
        return ValidationResult(False, "q_nonpositive", "q <= 0")
    if packet.k != packet.d * packet.q:
        return ValidationResult(False, "product", "k != d*q")
    if packet.delta_num != 4 * packet.q - packet.d * packet.d:
        return ValidationResult(False, "delta_num", "delta_num mismatch")
    if packet.delta_num % 3 != 0:
        return ValidationResult(False, "delta_divisibility", "delta_num % 3 != 0")
    if packet.delta != packet.delta_num // 3:
        return ValidationResult(False, "delta", "delta mismatch")
    if packet.delta < 0:
        return ValidationResult(False, "delta_negative", "delta < 0")
    if packet.s * packet.s != packet.delta:
        return ValidationResult(False, "square", "s^2 != delta")
    if not packet.parity_even:
        return ValidationResult(False, "parity", "(d+s) odd")
    if not packet.parity_collapse_ok:
        return ValidationResult(False, "parity_collapse", "k,d,s parity collapse failed")
    if packet.x + packet.y != packet.d:
        return ValidationResult(False, "reconstruct_d", "x+y != d")
    if packet.x - packet.y != packet.s:
        return ValidationResult(False, "reconstruct_s", "x-y != s")
    if eisenstein_norm(packet.x, packet.y) != packet.q:
        return ValidationResult(False, "norm", "q != x^2-xy+y^2")
    if packet.shell_value != 0:
        return ValidationResult(False, "shell", "4q-d^2-3s^2 != 0")
    if knight_equation_value(packet.n, packet.z, packet.d, packet.s) != 0:
        return ValidationResult(False, "knight", "Knight equation != 0")
    if not god_source_holds(packet.n, packet.x, packet.y, packet.z):
        return ValidationResult(False, "cube", "x^3+y^3+z^3 != n")
    if packet.a2_u != packet.d:
        return ValidationResult(False, "a2_u", "u != d")
    if packet.a2_v_squared != 3 * packet.s * packet.s:
        return ValidationResult(False, "a2_v2", "v^2 != 3s^2")
    if not packet.q_mod3_ok:
        return ValidationResult(False, "q_mod3", "q % 3 not in {0,1}")
    if not packet.q_eisenstein_norm_ok:
        return ValidationResult(False, "q_eisenstein", "q failed Eisenstein norm prime valuation gate")
    if not packet.q_congruence_mod_d_ok:
        return ValidationResult(False, "q_mod_d", "q ≢ 3x^2 ≡ 3y^2 (mod d)")
    if not packet.common_factor_cube_ok:
        return ValidationResult(False, "gcd_cube", "gcd(x,y)^3 does not divide k")
    if primitive_only and not packet.primitive_ok:
        return ValidationResult(False, "primitive", "primitive_only enabled and gcd(x,y,z) != 1")
    if primitive_only and not packet.primitive_gcd_dq_ok:
        return ValidationResult(False, "primitive_dq", "primitive gcd(d,q) law failed")
    return ValidationResult(True)


def build_packet_from_ds(
    n: int,
    z: int,
    d: int,
    s: int,
    source_mode: str,
    factorization_k: Optional[Dict[int, int]] = None,
    factorization_q: Optional[Dict[int, int]] = None,
    factor_remainder_k: int = 1,
    factor_exact_complete_k: bool = True,
) -> Optional[PacketLedger]:
    xy = ds_to_xy(d, s)
    if xy is None:
        return None

    x, y = xy
    q = eisenstein_norm(x, y)
    k = compute_k(n, z)
    delta_num = 4 * q - d * d
    if delta_num % 3 != 0:
        return None

    delta = delta_num // 3
    if delta < 0 or s * s != delta:
        return None

    gxy = gcd(abs(x), abs(y))
    gxyz = gcd(gxy, abs(z))
    primitive_ok = (gxyz == 1)

    q_congruence_ok_flag = True
    if d != 0:
        mod = abs(d)
        q_congruence_ok_flag = ((q - 3 * x * x) % mod == 0) and ((q - 3 * y * y) % mod == 0)

    common_factor_cube_ok_flag = True
    if gxy != 0:
        common_factor_cube_ok_flag = (k % (gxy ** 3) == 0)

    return PacketLedger(
        n=n,
        z=z,
        z_mod9=(z % 9),
        k=k,
        d=d,
        q=q,
        delta_num=delta_num,
        delta=delta,
        s=s,
        x=x,
        y=y,
        parity_even=((d + s) % 2 == 0),
        parity_collapse_ok=parity_gate_from_kd(k, d) and parity_gate_from_ks(k, s),
        cube_check=god_source_holds(n, x, y, z),
        knight_value=knight_equation_value(n, z, d, s),
        shell_value=shell_equation_value(q, d, s),
        primitive_ok=primitive_ok,
        primitive_gcd_dq_ok=primitive_gcd_dq_ok(d, q),
        q_mod3_ok=q_mod3_gate(q),
        q_eisenstein_norm_ok=q_is_eisenstein_norm_from_factors(factorization_q or {}),
        q_congruence_mod_d_ok=q_congruence_ok_flag,
        common_factor_cube_ok=common_factor_cube_ok_flag,
        source_mode=source_mode,
        a2_u=d,
        a2_v_squared=3 * s * s,
        factorization_k=factorization_k or {},
        factorization_q=factorization_q or {},
        factor_remainder_k=factor_remainder_k,
        factor_exact_complete_k=factor_exact_complete_k,
    )


def emit_verified_packet(packet: PacketLedger, primitive_only: bool = False) -> Optional[PacketLedger]:
    rebuilt = build_packet_from_ds(
        packet.n,
        packet.z,
        packet.x + packet.y,
        packet.x - packet.y,
        packet.source_mode,
        factorization_k=packet.factorization_k,
        factorization_q=packet.factorization_q,
        factor_remainder_k=packet.factor_remainder_k,
        factor_exact_complete_k=packet.factor_exact_complete_k,
    )
    if rebuilt is None:
        return None

    verdict = validate_packet(rebuilt, primitive_only=primitive_only)
    if verdict.is_valid:
        return rebuilt
    return None


@dataclass
class SearchConfig:
    n: int
    z_start: int
    z_stop: int
    z_step: int = 1
    mode: str = "qfirst_corridor"
    max_hits: int = 50

    factor_seed: int = 144
    corridor_min_div: int = 500
    corridor_max_div: int = 20
    brute_x_limit: int = 60
    brute_y_limit: int = 60
    rho_attempts: int = 8
    rho_step_cap: int = 200_000
    factor_time_s: float = 2.0

    parallel_workers: int = 48
    fallback_to_one_worker: bool = True
    force_serial: bool = False

    primitive_only: bool = False
    canonical_s_nonnegative: bool = False
    canonical_reduced_region: bool = False
    use_mod9_triplet_filter: bool = True
    use_q_eisenstein_gate: bool = True
    use_delta_qr_prefilter: bool = True

    output_dir: str = "solver_output"
    checkpoint_every: int = 100
    resume: bool = False
    atlas_sqlite: bool = True
    export_path: Optional[str] = None
    self_check: bool = False
    near_miss_limit: int = 5000


@dataclass
class RunResult:
    atlas: Atlas
    validation_log: List[Dict[str, Any]]
    z_status_log: List[Dict[str, Any]]
    near_misses: List[Dict[str, Any]]
    elapsed_s: float
    config: SearchConfig
    execution_mode: str
    fallback_triggered: bool
    fallback_reason: Optional[str]


def ensure_output_dir(path: str) -> Path:
    p = Path(path)
    p.mkdir(parents=True, exist_ok=True)
    return p


def append_jsonl(path: Path, obj: Dict[str, Any]) -> None:
    with path.open("a", encoding="utf-8") as f:
        f.write(stable_json(obj) + "\n")


def checkpoint_path(output_dir: Path) -> Path:
    return output_dir / "checkpoint.json"


def save_checkpoint(output_dir: Path, cfg: SearchConfig, z_current: int, atlas: Atlas, validation_events: int) -> None:
    payload = {
        "engine_version": ENGINE_VERSION,
        "author": AUTHOR,
        "identity_mark": IDENTITY_MARK,
        "config": asdict(cfg),
        "z_current": z_current,
        "atlas_snapshot": atlas.snapshot(),
        "validation_events": validation_events,
        "timestamp": time.time(),
    }
    with checkpoint_path(output_dir).open("w", encoding="utf-8") as f:
        json.dump(payload, f, ensure_ascii=False, indent=2)


def load_checkpoint(output_dir: Path) -> Optional[Dict[str, Any]]:
    p = checkpoint_path(output_dir)
    if not p.exists():
        return None
    with p.open("r", encoding="utf-8") as f:
        return json.load(f)


def init_sqlite(db_path: Path) -> None:
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()

    cur.execute("""
        CREATE TABLE IF NOT EXISTS packets (
            packet_id TEXT PRIMARY KEY,
            payload_json TEXT
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS validation_log (
            row_id INTEGER PRIMARY KEY AUTOINCREMENT,
            packet_id TEXT,
            status TEXT,
            failed_stage TEXT,
            detail TEXT
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS z_status (
            z INTEGER PRIMARY KEY,
            payload_json TEXT
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS factor_cache (
            k_abs TEXT PRIMARY KEY,
            factors_json TEXT,
            remainder TEXT,
            exact_complete INTEGER,
            timed_out INTEGER
        )
    """)

    conn.commit()
    conn.close()


def sqlite_insert_packet(db_path: Path, packet: PacketLedger) -> None:
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    cur.execute(
        "INSERT OR IGNORE INTO packets (packet_id, payload_json) VALUES (?, ?)",
        (packet.packet_id(), json.dumps(packet.to_dict(), ensure_ascii=False, sort_keys=True)),
    )
    conn.commit()
    conn.close()


def sqlite_insert_validation(db_path: Path, event: Dict[str, Any]) -> None:
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    cur.execute(
        "INSERT INTO validation_log (packet_id, status, failed_stage, detail) VALUES (?, ?, ?, ?)",
        (event.get("packet_id"), event.get("status"), event.get("failed_stage"), event.get("detail")),
    )
    conn.commit()
    conn.close()


def sqlite_insert_z_status(db_path: Path, zstatus: ZStatusLedger) -> None:
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    cur.execute(
        "INSERT OR REPLACE INTO z_status (z, payload_json) VALUES (?, ?)",
        (zstatus.z, json.dumps(zstatus.to_dict(), ensure_ascii=False, sort_keys=True)),
    )
    conn.commit()
    conn.close()


def sqlite_get_factor_cache(db_path: Path, k_abs: int) -> Optional[FactorizationResult]:
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    cur.execute(
        "SELECT factors_json, remainder, exact_complete, timed_out FROM factor_cache WHERE k_abs = ?",
        (str(k_abs),),
    )
    row = cur.fetchone()
    conn.close()

    if row is None:
        return None

    factors_json, remainder, exact_complete, timed_out = row
    return FactorizationResult(
        factors={int(k): int(v) for k, v in json.loads(factors_json).items()},
        remainder=int(remainder),
        exact_complete=bool(exact_complete),
        timed_out=bool(timed_out),
    )


def sqlite_put_factor_cache(db_path: Path, k_abs: int, result: FactorizationResult) -> None:
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    cur.execute(
        """
        INSERT OR REPLACE INTO factor_cache
        (k_abs, factors_json, remainder, exact_complete, timed_out)
        VALUES (?, ?, ?, ?, ?)
        """,
        (
            str(k_abs),
            json.dumps({str(k): v for k, v in result.factors.items()}, ensure_ascii=False, sort_keys=True),
            str(result.remainder),
            1 if result.exact_complete else 0,
            1 if result.timed_out else 0,
        ),
    )
    conn.commit()
    conn.close()


def factor_with_cache(cfg: SearchConfig, k_abs: int, db_path: Optional[Path], seed_delta: int) -> FactorizationResult:
    if db_path is not None:
        cached = sqlite_get_factor_cache(db_path, k_abs)
        if cached is not None:
            return cached

    result = factorize_partial(
        k_abs,
        seed=cfg.factor_seed + seed_delta,
        rho_attempts=cfg.rho_attempts,
        rho_step_cap=cfg.rho_step_cap,
        factor_time_s=cfg.factor_time_s,
    )

    if db_path is not None:
        sqlite_put_factor_cache(db_path, k_abs, result)

    return result


def q_candidate_exact_filters(
    cfg: SearchConfig,
    k: int,
    q: int,
    d: int,
    k_factorization: FactorizationResult,
) -> Tuple[bool, Dict[int, int]]:
    if q <= 0:
        return False, {}

    if not q_prefilter_from_k(abs(k), q):
        return False, {}

    if not q_mod3_gate(q):
        return False, {}

    if abs(d) ** 3 > 4 * abs(k):
        return False, {}

    if q < q_cuberoot_lower_bound(abs(k)):
        return False, {}

    if not parity_gate_from_kd(k, d):
        return False, {}

    try:
        q_factors = factorization_of_divisor_from_known(q, k_factorization.factors)
    except ValueError:
        return False, {}

    if cfg.use_q_eisenstein_gate and not q_is_eisenstein_norm_from_factors(q_factors):
        return False, {}

    if cfg.primitive_only and not primitive_gcd_dq_ok(d, q):
        return False, {}

    return True, q_factors


def delta_candidate_exact_filters(
    cfg: SearchConfig,
    k: int,
    d: int,
    q: int,
    delta_num: int,
) -> bool:
    if not delta_prefilter(delta_num):
        return False

    delta = delta_num // 3
    if cfg.use_delta_qr_prefilter and not quadratic_residue_prefilter(delta):
        return False

    if not parity_gate_from_kd(k, d):
        return False

    return True


def scan_z_bruteforce(cfg: SearchConfig, z: int) -> Tuple[List[PacketLedger], List[NearMissLedger], ZStatusLedger]:
    k = compute_k(cfg.n, z)
    z_mod9 = z % 9
    mod9_ok = (not cfg.use_mod9_triplet_filter) or z_passes_mod9_triplet_filter(cfg.n, z)

    if not mod9_ok:
        return [], [], ZStatusLedger(cfg.n, z, z_mod9, k, "NO_HIT_CORRIDOR_ONLY", "bruteforce_mod9_skip", mod9_ok, True, 1, False, 0, 0, 0)

    if k == 0:
        return [], [], ZStatusLedger(cfg.n, z, z_mod9, k, "SKIPPED_K_ZERO", "bruteforce", mod9_ok, True, 1, False, 0, 0, 0)

    packets: List[PacketLedger] = []
    near: List[NearMissLedger] = []
    hits = 0
    candidates = 0
    square_pass = 0

    for x in range(-cfg.brute_x_limit, cfg.brute_x_limit + 1):
        for y in range(-cfg.brute_y_limit, cfg.brute_y_limit + 1):
            candidates += 1
            if not god_source_holds(cfg.n, x, y, z):
                continue

            d, s = xy_to_ds(x, y)
            if not reduced_region_ok(d, s, cfg.canonical_s_nonnegative, cfg.canonical_reduced_region):
                continue

            q = eisenstein_norm(x, y)
            delta_num = 4 * q - d * d
            if delta_num % 3 == 0:
                square_pass += 1

            qf = factorize_partial(q, seed=cfg.factor_seed + abs(z) + abs(d), factor_time_s=0.2)
            packet = build_packet_from_ds(
                cfg.n, z, d, s, "bruteforce",
                factorization_k={},
                factorization_q=qf.factors,
                factor_remainder_k=1,
                factor_exact_complete_k=True,
            )
            if packet is None:
                continue

            verified = emit_verified_packet(packet, primitive_only=cfg.primitive_only)
            if verified is not None:
                packets.append(verified)
                hits += 1
                if hits >= cfg.max_hits:
                    break
        if hits >= cfg.max_hits:
            break

    status = "HIT_VERIFIED" if hits > 0 else "NO_HIT_EXHAUSTIVE_FACTORED"
    return packets, near, ZStatusLedger(cfg.n, z, z_mod9, k, status, "bruteforce", mod9_ok, True, 1, False, candidates, square_pass, hits)


def scan_z_dfirst(
    cfg: SearchConfig,
    z: int,
    exhaustive: bool,
    db_path: Optional[Path],
) -> Tuple[List[PacketLedger], List[NearMissLedger], ZStatusLedger]:
    k = compute_k(cfg.n, z)
    z_mod9 = z % 9
    mod9_ok = (not cfg.use_mod9_triplet_filter) or z_passes_mod9_triplet_filter(cfg.n, z)
    mode_name = "dfirst_exhaustive" if exhaustive else "dfirst_corridor"

    if not mod9_ok:
        return [], [], ZStatusLedger(cfg.n, z, z_mod9, k, "NO_HIT_CORRIDOR_ONLY", mode_name + "_mod9_skip", mod9_ok, True, 1, False, 0, 0, 0)

    if k == 0:
        return [], [], ZStatusLedger(cfg.n, z, z_mod9, k, "SKIPPED_K_ZERO", mode_name, mod9_ok, True, 1, False, 0, 0, 0)

    fz = factor_with_cache(cfg, abs(k), db_path, abs(z))
    d_min, d_max = corridor_bounds(z, cfg.corridor_min_div, cfg.corridor_max_div)

    if exhaustive:
        d_candidates = signed_divisors_from_factors(fz.factors)
    else:
        d_candidates = corridor_divisors(fz.factors, d_min, d_max, required_sign=sgn(k), rank_near_crit=True, k_abs=abs(k))

    packets: List[PacketLedger] = []
    near: List[NearMissLedger] = []
    hits = 0
    candidates = 0
    square_pass = 0

    for d in d_candidates:
        candidates += 1

        if not d_prefilter_from_k(k, d):
            continue
        if not parity_gate_from_kd(k, d):
            continue
        if abs(d) ** 3 > 4 * abs(k):
            continue

        q = k // d
        q_ok, q_factors = q_candidate_exact_filters(cfg, k, q, d, fz)
        if not q_ok:
            continue

        delta_num = 4 * q - d * d
        if not delta_candidate_exact_filters(cfg, k, d, q, delta_num):
            if delta_num >= 0 and delta_num % 3 == 0:
                delta = delta_num // 3
                sqrt_floor = isqrt(delta)
                residual = delta - sqrt_floor * sqrt_floor
                near.append(NearMissLedger(cfg.n, z, z_mod9, k, d, q, delta_num, delta, sqrt_floor, residual, mode_name, fz.exact_complete, fz.remainder))
            continue

        square_pass += 1
        delta = delta_num // 3
        ok, root = exact_is_square(delta)
        if not ok:
            sqrt_floor = isqrt(delta)
            residual = delta - sqrt_floor * sqrt_floor
            near.append(NearMissLedger(cfg.n, z, z_mod9, k, d, q, delta_num, delta, sqrt_floor, residual, mode_name, fz.exact_complete, fz.remainder))
            continue

        s_values = [0] if root == 0 else [root, -root]
        for s in s_values:
            if not parity_gate_from_ks(k, s):
                continue
            if not reduced_region_ok(d, s, cfg.canonical_s_nonnegative, cfg.canonical_reduced_region):
                continue

            packet = build_packet_from_ds(
                cfg.n, z, d, s, mode_name,
                factorization_k=fz.factors,
                factorization_q=q_factors,
                factor_remainder_k=fz.remainder,
                factor_exact_complete_k=fz.exact_complete,
            )
            if packet is None:
                continue

            verified = emit_verified_packet(packet, primitive_only=cfg.primitive_only)
            if verified is not None:
                packets.append(verified)
                hits += 1
                if hits >= cfg.max_hits:
                    break

        if hits >= cfg.max_hits:
            break

    if hits > 0:
        status = "HIT_VERIFIED"
    else:
        if not fz.exact_complete:
            status = "INCOMPLETE_TIME_BUDGET" if fz.timed_out else "INCOMPLETE_PARTIAL_FACTORIZATION"
        else:
            status = "NO_HIT_EXHAUSTIVE_FACTORED" if exhaustive else "NO_HIT_CORRIDOR_ONLY"

    return packets, near, ZStatusLedger(cfg.n, z, z_mod9, k, status, mode_name, mod9_ok, fz.exact_complete, fz.remainder, fz.timed_out, candidates, square_pass, hits)


def scan_z_qfirst(
    cfg: SearchConfig,
    z: int,
    exhaustive: bool,
    db_path: Optional[Path],
) -> Tuple[List[PacketLedger], List[NearMissLedger], ZStatusLedger]:
    k = compute_k(cfg.n, z)
    z_mod9 = z % 9
    mod9_ok = (not cfg.use_mod9_triplet_filter) or z_passes_mod9_triplet_filter(cfg.n, z)
    mode_name = "qfirst_exhaustive" if exhaustive else "qfirst_corridor"

    if not mod9_ok:
        return [], [], ZStatusLedger(cfg.n, z, z_mod9, k, "NO_HIT_CORRIDOR_ONLY", mode_name + "_mod9_skip", mod9_ok, True, 1, False, 0, 0, 0)

    if k == 0:
        return [], [], ZStatusLedger(cfg.n, z, z_mod9, k, "SKIPPED_K_ZERO", mode_name, mod9_ok, True, 1, False, 0, 0, 0)

    k_abs = abs(k)
    fz = factor_with_cache(cfg, k_abs, db_path, abs(z))
    q_candidates = positive_divisors_from_factors(fz.factors)

    d_min, d_max = corridor_bounds(z, cfg.corridor_min_div, cfg.corridor_max_div)
    q_lb = q_cuberoot_lower_bound(k_abs)

    if exhaustive:
        q_filtered = [q for q in q_candidates if q >= q_lb and q_prefilter_from_k(k_abs, q)]
    else:
        q_corr_min, q_corr_max = q_window_from_corridor(k_abs, d_min, d_max)
        lo = max(q_lb, q_corr_min)
        hi = q_corr_max
        q_filtered = [q for q in q_candidates if lo <= q <= hi and q_prefilter_from_k(k_abs, q)]

    dc = d_critical(k_abs)
    q_filtered = sorted(q_filtered, key=lambda q: (abs((k_abs // q) - dc), q))

    packets: List[PacketLedger] = []
    near: List[NearMissLedger] = []
    hits = 0
    candidates = 0
    square_pass = 0

    for q in q_filtered:
        candidates += 1

        d_abs = k_abs // q
        d = sgn(k) * d_abs
        if d == 0 or d * q != k:
            continue

        q_ok, q_factors = q_candidate_exact_filters(cfg, k, q, d, fz)
        if not q_ok:
            continue

        delta_num = 4 * q - d * d
        if not delta_candidate_exact_filters(cfg, k, d, q, delta_num):
            if delta_num >= 0 and delta_num % 3 == 0:
                delta = delta_num // 3
                sqrt_floor = isqrt(delta)
                residual = delta - sqrt_floor * sqrt_floor
                near.append(NearMissLedger(cfg.n, z, z_mod9, k, d, q, delta_num, delta, sqrt_floor, residual, mode_name, fz.exact_complete, fz.remainder))
            continue

        square_pass += 1
        delta = delta_num // 3
        ok, root = exact_is_square(delta)
        if not ok:
            sqrt_floor = isqrt(delta)
            residual = delta - sqrt_floor * sqrt_floor
            near.append(NearMissLedger(cfg.n, z, z_mod9, k, d, q, delta_num, delta, sqrt_floor, residual, mode_name, fz.exact_complete, fz.remainder))
            continue

        s_values = [0] if root == 0 else [root, -root]
        for s in s_values:
            if not parity_gate_from_ks(k, s):
                continue
            if not reduced_region_ok(d, s, cfg.canonical_s_nonnegative, cfg.canonical_reduced_region):
                continue

            packet = build_packet_from_ds(
                cfg.n, z, d, s, mode_name,
                factorization_k=fz.factors,
                factorization_q=q_factors,
                factor_remainder_k=fz.remainder,
                factor_exact_complete_k=fz.exact_complete,
            )
            if packet is None:
                continue

            verified = emit_verified_packet(packet, primitive_only=cfg.primitive_only)
            if verified is not None:
                packets.append(verified)
                hits += 1
                if hits >= cfg.max_hits:
                    break

        if hits >= cfg.max_hits:
            break

    if hits > 0:
        status = "HIT_VERIFIED"
    else:
        if not fz.exact_complete:
            status = "INCOMPLETE_TIME_BUDGET" if fz.timed_out else "INCOMPLETE_PARTIAL_FACTORIZATION"
        else:
            status = "NO_HIT_EXHAUSTIVE_FACTORED" if exhaustive else "NO_HIT_CORRIDOR_ONLY"

    return packets, near, ZStatusLedger(cfg.n, z, z_mod9, k, status, mode_name, mod9_ok, fz.exact_complete, fz.remainder, fz.timed_out, candidates, square_pass, hits)


def known_fixture_packets() -> List[PacketLedger]:
    fixtures = [
        {"n": 144, "x": 6, "y": -2, "z": -4},
        {"n": 411, "x": -46, "y": 35, "z": 38},
        {"n": 153, "x": -65, "y": 215, "z": -213},
        {"n": 369, "x": -149, "y": 189, "z": -151},
        {"n": 1001, "x": -15, "y": 28, "z": -26},
        {"n": 33, "x": 8866128975287528, "y": -8778405442862239, "z": -2736111468807040},
        {"n": 42, "x": -80538738812075974, "y": 80435758145817515, "z": 12602123297335631},
        {"n": 3, "x": 569936821221962380720, "y": -569936821113563493509, "z": -472715493453327032},
        {"n": 906, "x": -74924259395610397, "y": 72054089679353378, "z": 35961979615356503},
    ]

    packets: List[PacketLedger] = []
    for item in fixtures:
        n, x, y, z = item["n"], item["x"], item["y"], item["z"]
        d, s = xy_to_ds(x, y)
        q = eisenstein_norm(x, y)
        qf = factorize_partial(q, factor_time_s=0.2)
        packet = build_packet_from_ds(
            n, z, d, s, "fixture",
            factorization_k={},
            factorization_q=qf.factors,
            factor_remainder_k=1,
            factor_exact_complete_k=True,
        )
        if packet is None:
            raise AssertionError(f"Fixture failed: {item}")
        packets.append(packet)

    return packets


def run_self_check() -> None:
    packets = known_fixture_packets()
    for packet in packets:
        verdict = validate_packet(packet)
        if not verdict.is_valid:
            raise AssertionError(f"Fixture failed for n={packet.n}: {verdict.detail}")

    for n in EVEREST_TARGETS:
        if not cubes_mod_9_admissible(n):
            raise AssertionError(f"Everest target failed mod-9 admissibility: n={n}")


def run_engine_serial(cfg: SearchConfig) -> RunResult:
    start = time.time()
    atlas = Atlas()
    validation_log: List[Dict[str, Any]] = []
    z_status_log: List[Dict[str, Any]] = []
    near_misses: List[Dict[str, Any]] = []

    output_dir = ensure_output_dir(cfg.output_dir)
    packets_jsonl = output_dir / "packets.jsonl"
    validation_jsonl = output_dir / "validation_log.jsonl"
    zstatus_jsonl = output_dir / "z_status.jsonl"
    near_jsonl = output_dir / "near_misses.jsonl"
    atlas_db = output_dir / "atlas.sqlite"

    factor_db = None
    if cfg.atlas_sqlite:
        init_sqlite(atlas_db)
        factor_db = atlas_db

    z_start = cfg.z_start
    if cfg.resume:
        ckpt = load_checkpoint(output_dir)
        if ckpt is not None:
            z_start = int(ckpt["z_current"]) + cfg.z_step

    local_cfg = SearchConfig(**asdict(cfg))
    local_cfg.z_start = z_start

    hit_count = 0
    for idx, z in enumerate(range(local_cfg.z_start, local_cfg.z_stop + 1, local_cfg.z_step), start=1):
        if local_cfg.mode == "bruteforce":
            packets, near, zstatus = scan_z_bruteforce(local_cfg, z)
        elif local_cfg.mode == "dfirst_corridor":
            packets, near, zstatus = scan_z_dfirst(local_cfg, z, exhaustive=False, db_path=factor_db)
        elif local_cfg.mode == "dfirst_exhaustive":
            packets, near, zstatus = scan_z_dfirst(local_cfg, z, exhaustive=True, db_path=factor_db)
        elif local_cfg.mode == "qfirst_corridor":
            packets, near, zstatus = scan_z_qfirst(local_cfg, z, exhaustive=False, db_path=factor_db)
        elif local_cfg.mode == "qfirst_exhaustive":
            packets, near, zstatus = scan_z_qfirst(local_cfg, z, exhaustive=True, db_path=factor_db)
        else:
            raise ValueError(f"Unsupported mode: {local_cfg.mode}")

        z_status_log.append(zstatus.to_dict())
        append_jsonl(zstatus_jsonl, zstatus.to_dict())
        if cfg.atlas_sqlite:
            sqlite_insert_z_status(atlas_db, zstatus)

        for nm in near:
            if len(near_misses) < cfg.near_miss_limit:
                payload = nm.to_dict()
                near_misses.append(payload)
                append_jsonl(near_jsonl, payload)

        for packet in packets:
            verdict = validate_packet(packet, primitive_only=cfg.primitive_only)
            event = {
                "packet_id": packet.packet_id(),
                "status": "valid" if verdict.is_valid else "invalid",
                "failed_stage": verdict.failed_stage,
                "detail": verdict.detail,
            }
            validation_log.append(event)
            append_jsonl(validation_jsonl, event)
            if cfg.atlas_sqlite:
                sqlite_insert_validation(atlas_db, event)

            if verdict.is_valid:
                atlas.add(packet)
                append_jsonl(packets_jsonl, packet.to_dict())
                if cfg.atlas_sqlite:
                    sqlite_insert_packet(atlas_db, packet)

                hit_count += 1
                if hit_count >= cfg.max_hits:
                    save_checkpoint(output_dir, cfg, z, atlas, len(validation_log))
                    return RunResult(
                        atlas=atlas,
                        validation_log=validation_log,
                        z_status_log=z_status_log,
                        near_misses=near_misses,
                        elapsed_s=time.time() - start,
                        config=cfg,
                        execution_mode="serial",
                        fallback_triggered=False,
                        fallback_reason=None,
                    )

        if idx % max(1, cfg.checkpoint_every) == 0:
            save_checkpoint(output_dir, cfg, z, atlas, len(validation_log))

    save_checkpoint(output_dir, cfg, local_cfg.z_stop, atlas, len(validation_log))
    return RunResult(
        atlas=atlas,
        validation_log=validation_log,
        z_status_log=z_status_log,
        near_misses=near_misses,
        elapsed_s=time.time() - start,
        config=cfg,
        execution_mode="serial",
        fallback_triggered=False,
        fallback_reason=None,
    )


def split_z_ranges(z_start: int, z_stop: int, z_step: int, chunks: int) -> List[Tuple[int, int]]:
    zs = list(range(z_start, z_stop + 1, z_step))
    if not zs:
        return []
    chunks = max(1, min(chunks, len(zs)))
    size = ceil_div(len(zs), chunks)

    out: List[Tuple[int, int]] = []
    for i in range(0, len(zs), size):
        sub = zs[i:i + size]
        out.append((sub[0], sub[-1]))
    return out


def _run_subrange(cfg_dict: Dict[str, Any]) -> Dict[str, Any]:
    cfg = SearchConfig(**cfg_dict)
    result = run_engine_serial(cfg)
    return {
        "packets": [p.to_dict() for p in result.atlas.packets.values()],
        "validation_log": result.validation_log,
        "z_status_log": result.z_status_log,
        "near_misses": result.near_misses,
        "elapsed_s": result.elapsed_s,
    }


def run_engine_parallel(cfg: SearchConfig) -> RunResult:
    if not PARALLEL_AVAILABLE:
        raise RuntimeError(f"Parallel execution unavailable: {PARALLEL_IMPORT_ERROR}")

    start = time.time()
    atlas = Atlas()
    validation_log: List[Dict[str, Any]] = []
    z_status_log: List[Dict[str, Any]] = []
    near_misses: List[Dict[str, Any]] = []

    output_dir = ensure_output_dir(cfg.output_dir)
    packets_jsonl = output_dir / "packets.jsonl"
    validation_jsonl = output_dir / "validation_log.jsonl"
    zstatus_jsonl = output_dir / "z_status.jsonl"
    near_jsonl = output_dir / "near_misses.jsonl"
    atlas_db = output_dir / "atlas.sqlite"

    if cfg.atlas_sqlite:
        init_sqlite(atlas_db)

    ranges = split_z_ranges(cfg.z_start, cfg.z_stop, cfg.z_step, cfg.parallel_workers)
    sub_cfgs: List[Dict[str, Any]] = []

    for idx, (a, b) in enumerate(ranges):
        sub = asdict(cfg)
        sub["z_start"] = a
        sub["z_stop"] = b
        sub["parallel_workers"] = 1
        sub["force_serial"] = True
        sub["resume"] = False
        sub["output_dir"] = str(output_dir / f"worker_{idx}")
        sub_cfgs.append(sub)

    with ProcessPoolExecutor(max_workers=cfg.parallel_workers) as ex:
        futures = [ex.submit(_run_subrange, sub) for sub in sub_cfgs]

        for fut in as_completed(futures):
            payload = fut.result()

            for event in payload["validation_log"]:
                validation_log.append(event)
                append_jsonl(validation_jsonl, event)
                if cfg.atlas_sqlite:
                    sqlite_insert_validation(atlas_db, event)

            for zs in payload["z_status_log"]:
                z_status_log.append(zs)
                append_jsonl(zstatus_jsonl, zs)
                if cfg.atlas_sqlite:
                    sqlite_insert_z_status(atlas_db, ZStatusLedger(**zs))

            for nm in payload["near_misses"]:
                if len(near_misses) < cfg.near_miss_limit:
                    near_misses.append(nm)
                    append_jsonl(near_jsonl, nm)

            for pd in payload["packets"]:
                pd2 = dict(pd)
                pd2.pop("packet_id", None)
                packet = PacketLedger(**pd2)
                if packet.packet_id() not in atlas.packets:
                    atlas.add(packet)
                    append_jsonl(packets_jsonl, packet.to_dict())
                    if cfg.atlas_sqlite:
                        sqlite_insert_packet(atlas_db, packet)

                    if len(atlas.packets) >= cfg.max_hits:
                        break

            if len(atlas.packets) >= cfg.max_hits:
                break

    save_checkpoint(output_dir, cfg, cfg.z_stop, atlas, len(validation_log))
    return RunResult(
        atlas=atlas,
        validation_log=validation_log,
        z_status_log=z_status_log,
        near_misses=near_misses,
        elapsed_s=time.time() - start,
        config=cfg,
        execution_mode="parallel",
        fallback_triggered=False,
        fallback_reason=None,
    )


def run_engine(cfg: SearchConfig) -> RunResult:
    if cfg.force_serial or cfg.parallel_workers <= 1:
        result = run_engine_serial(cfg)
        result.execution_mode = "serial"
        result.fallback_triggered = False
        result.fallback_reason = None
        return result

    if not PARALLEL_AVAILABLE:
        if not cfg.fallback_to_one_worker:
            raise RuntimeError(f"Parallel unavailable and fallback disabled: {PARALLEL_IMPORT_ERROR}")

        serial_cfg = SearchConfig(**asdict(cfg))
        serial_cfg.parallel_workers = 1
        serial_cfg.force_serial = True

        result = run_engine_serial(serial_cfg)
        result.execution_mode = "serial_fallback"
        result.fallback_triggered = True
        result.fallback_reason = f"parallel import unavailable: {PARALLEL_IMPORT_ERROR}"
        return result

    try:
        result = run_engine_parallel(cfg)
        result.execution_mode = "parallel"
        result.fallback_triggered = False
        result.fallback_reason = None
        return result
    except Exception as exc:
        if not cfg.fallback_to_one_worker:
            raise

        serial_cfg = SearchConfig(**asdict(cfg))
        serial_cfg.parallel_workers = 1
        serial_cfg.force_serial = True

        result = run_engine_serial(serial_cfg)
        result.execution_mode = "serial_fallback"
        result.fallback_triggered = True
        result.fallback_reason = f"parallel runtime failure: {repr(exc)}"
        return result


def run_many_targets(base_cfg: SearchConfig, targets: List[int]) -> Dict[str, Any]:
    summary: List[Dict[str, Any]] = []

    for n in targets:
        sub_cfg = SearchConfig(**asdict(base_cfg))
        sub_cfg.n = n
        sub_cfg.output_dir = str(Path(base_cfg.output_dir) / f"n_{n}")

        result = run_engine(sub_cfg)

        row = {
            "n": n,
            "mod9": n % 9,
            "admissible_mod9": cubes_mod_9_admissible(n),
            "allowed_z_residues_mod9": sorted(allowed_z_residues_mod9_for_target(n)),
            "execution_mode": result.execution_mode,
            "fallback_triggered": result.fallback_triggered,
            "fallback_reason": result.fallback_reason,
            "packet_count": len(result.atlas.packets),
            "validation_events": len(result.validation_log),
            "z_status_rows": len(result.z_status_log),
            "near_misses": len(result.near_misses),
            "elapsed_s": result.elapsed_s,
            "output_dir": sub_cfg.output_dir,
        }
        summary.append(row)

    return {
        "engine_version": ENGINE_VERSION,
        "target_count": len(targets),
        "targets": targets,
        "summary": summary,
    }


def export_payload(result: RunResult) -> Dict[str, Any]:
    return {
        "author": AUTHOR,
        "identity_mark": IDENTITY_MARK,
        "engine_version": ENGINE_VERSION,
        "parallel_available": PARALLEL_AVAILABLE,
        "parallel_import_error": PARALLEL_IMPORT_ERROR,
        "execution_mode": result.execution_mode,
        "fallback_triggered": result.fallback_triggered,
        "fallback_reason": result.fallback_reason,
        "config": asdict(result.config),
        "snapshot": result.atlas.snapshot(),
        "packets": [p.to_dict() for p in result.atlas.packets.values()],
        "validation_log": result.validation_log,
        "z_status_log": result.z_status_log,
        "near_misses": result.near_misses,
    }


def export_json(result: RunResult) -> str:
    return stable_json(export_payload(result))


def export_json_file(result: RunResult, path: str) -> None:
    with open(path, "w", encoding="utf-8") as f:
        json.dump(export_payload(result), f, ensure_ascii=False, sort_keys=True, indent=2)


def print_banner() -> None:
    print("=" * 96)
    print("SOVEREIGN KNIGHT SOLVER — FULL REWRITE (UNIFIED FILTER STACK)")
    print(f"AUTHOR: {AUTHOR}")
    print(f"IDENTITY: {IDENTITY_MARK}")
    print(f"ENGINE VERSION: {ENGINE_VERSION}")
    print(f"PARALLEL AVAILABLE: {PARALLEL_AVAILABLE}")
    if not PARALLEL_AVAILABLE:
        print(f"PARALLEL IMPORT ERROR: {PARALLEL_IMPORT_ERROR}")
    print("=" * 96)
    print("MASTER COMPRESSION")
    print("  n = z^3 + d N(x - yω)")
    print("  q = N(x - yω) = x^2 - x*y + y^2")
    print("  d = x + y")
    print("=" * 96)
    print("EVEREST TARGETS:", EVEREST_TARGETS)
    print("=" * 96)


def print_packet(packet: PacketLedger) -> None:
    print("-" * 96)
    for key, value in packet.to_dict().items():
        print(f"{key:>24}: {value}")
    print("-" * 96)


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="Sovereign KNIGHT solver with unified exact filter stack")

    parser.add_argument("--n", type=int, default=153)
    parser.add_argument("--z-start", type=int, default=-120)
    parser.add_argument("--z-stop", type=int, default=120)
    parser.add_argument("--z-step", type=int, default=1)

    parser.add_argument(
        "--mode",
        type=str,
        default="qfirst_corridor",
        choices=["bruteforce", "dfirst_corridor", "dfirst_exhaustive", "qfirst_corridor", "qfirst_exhaustive"],
    )

    parser.add_argument("--max-hits", type=int, default=50)
    parser.add_argument("--factor-seed", type=int, default=144)
    parser.add_argument("--corridor-min-div", type=int, default=500)
    parser.add_argument("--corridor-max-div", type=int, default=20)
    parser.add_argument("--brute-x-limit", type=int, default=60)
    parser.add_argument("--brute-y-limit", type=int, default=60)
    parser.add_argument("--rho-attempts", type=int, default=8)
    parser.add_argument("--rho-step-cap", type=int, default=200000)
    parser.add_argument("--factor-time-s", type=float, default=2.0)

    parser.add_argument("--parallel-workers", type=int, default=48)
    parser.add_argument("--no-fallback", action="store_true")
    parser.add_argument("--force-serial", action="store_true")

    parser.add_argument("--primitive-only", action="store_true")
    parser.add_argument("--canonical-s-nonnegative", action="store_true")
    parser.add_argument("--canonical-reduced-region", action="store_true")
    parser.add_argument("--no-mod9-triplet-filter", action="store_true")
    parser.add_argument("--no-q-eisenstein-gate", action="store_true")
    parser.add_argument("--no-delta-qr-prefilter", action="store_true")

    parser.add_argument("--output-dir", type=str, default="solver_output")
    parser.add_argument("--checkpoint-every", type=int, default=100)
    parser.add_argument("--resume", action="store_true")
    parser.add_argument("--no-sqlite", action="store_true")
    parser.add_argument("--self-check", action="store_true")
    parser.add_argument("--json", action="store_true")
    parser.add_argument("--export-path", type=str, default=None)
    parser.add_argument("--near-miss-limit", type=int, default=5000)

    parser.add_argument("--target-set", type=str, default=None, help="Run a built-in target set, e.g. 'everest'")
    parser.add_argument("--list-targets", action="store_true", help="Print built-in target sets and exit")
    return parser.parse_args()


if __name__ == "__main__":
    try:
        args = parse_args()

        cfg = SearchConfig(
            n=args.n,
            z_start=args.z_start,
            z_stop=args.z_stop,
            z_step=args.z_step,
            mode=args.mode,
            max_hits=args.max_hits,
            factor_seed=args.factor_seed,
            corridor_min_div=args.corridor_min_div,
            corridor_max_div=args.corridor_max_div,
            brute_x_limit=args.brute_x_limit,
            brute_y_limit=args.brute_y_limit,
            rho_attempts=args.rho_attempts,
            rho_step_cap=args.rho_step_cap,
            factor_time_s=args.factor_time_s,
            parallel_workers=args.parallel_workers,
            fallback_to_one_worker=(not args.no_fallback),
            force_serial=args.force_serial,
            primitive_only=args.primitive_only,
            canonical_s_nonnegative=args.canonical_s_nonnegative,
            canonical_reduced_region=args.canonical_reduced_region,
            use_mod9_triplet_filter=(not args.no_mod9_triplet_filter),
            use_q_eisenstein_gate=(not args.no_q_eisenstein_gate),
            use_delta_qr_prefilter=(not args.no_delta_qr_prefilter),
            output_dir=args.output_dir,
            checkpoint_every=args.checkpoint_every,
            resume=args.resume,
            atlas_sqlite=(not args.no_sqlite),
            export_path=args.export_path,
            self_check=args.self_check,
            near_miss_limit=args.near_miss_limit,
        )

        if cfg.self_check:
            run_self_check()

        if args.list_targets:
            payload = {"target_sets": {name: [target_summary_row(n) for n in values] for name, values in TARGET_SETS.items()}}
            print(stable_json(payload))
            sys.exit(0)

        if args.target_set is not None:
            if args.target_set not in TARGET_SETS:
                raise ValueError(f"Unknown target set: {args.target_set}")

            print_banner()
            print(f"running_target_set={args.target_set}")
            print(f"targets={TARGET_SETS[args.target_set]}")

            multi = run_many_targets(cfg, TARGET_SETS[args.target_set])

            if cfg.export_path:
                with open(cfg.export_path, "w", encoding="utf-8") as f:
                    json.dump(multi, f, ensure_ascii=False, indent=2, sort_keys=True)
                print(f"exported_json={os.path.abspath(cfg.export_path)}")

            if args.json or not cfg.export_path:
                print(stable_json(multi))

            sys.exit(0)

        print_banner()
        result = run_engine(cfg)

        print(f"elapsed_s={result.elapsed_s:.6f}")
        print(f"execution_mode={result.execution_mode}")
        print(f"fallback_triggered={result.fallback_triggered}")
        print(f"fallback_reason={result.fallback_reason}")
        print(f"validation_events={len(result.validation_log)}")
        print(f"z_status_rows={len(result.z_status_log)}")
        print(f"near_misses={len(result.near_misses)}")
        print(stable_json(result.atlas.snapshot()))

        for packet in result.atlas.packets.values():
            print_packet(packet)

        if cfg.export_path:
            export_json_file(result, cfg.export_path)
            print(f"exported_json={os.path.abspath(cfg.export_path)}")

        if args.json:
            print(export_json(result))

    except Exception as exc:
        print("FATAL ERROR:", exc, file=sys.stderr)
        traceback.print_exc()
        sys.exit(1)


---

2) README.md

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

## Quick start

```bash
git clone https://github.com/YOURUSERNAME/sovereign-knight-solver
cd sovereign-knight-solver
python3 sovereign_knight_solver.py --help

Example

python3 sovereign_knight_solver.py \
--n 153 \
--z-start -200 \
--z-stop 200 \
--mode qfirst_corridor

Everest Targets

Current difficult values:

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

Research Scope

This repository implements a computational framework for exploring solutions to

x^3+y^3+z^3=n

It is not a proof of unsolved targets unless a verified integer triple is found.

Author

Riley Cameron Byrd

License

MIT

---

# 3) `LICENSE`

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

4) CONTRIBUTING.md

# Contributing

Thank you for your interest in contributing.

This project focuses on exact integer mathematics and research related to the equation

x³ + y³ + z³ = n.

## Guidelines

1. All algorithms must maintain exact integer correctness
2. No floating-point heuristics may replace verification
3. All proposed improvements must include mathematical justification
4. Code must remain readable and documented
5. Pull requests should include tests when applicable

## Development Setup

Clone repository

```bash
git clone https://github.com/YOURUSERNAME/sovereign-knight-solver

Run tests

python3 sovereign_knight_solver.py --self-check

Types of Contributions

algorithm improvements

factorization optimizations

mathematical filters

visualization tools

documentation


---

# 5) `CODE_OF_CONDUCT.md`

```markdown
# Code of Conduct

This project follows a standard open-source conduct policy.

Participants are expected to:

- be respectful
- focus on constructive discussion
- avoid harassment or discrimination

The goal of this repository is collaborative mathematical research.

Issues or concerns can be reported to the maintainer.


---

6) pyproject.toml

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

7) .gitignore

__pycache__/
*.py[cod]
*$py.class

.Python
build/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg

.env
.venv
env/
venv/
ENV/

htmlcov/
.tox/
.coverage
.coverage.*
.cache
.pytest_cache/

.ipynb_checkpoints

.vscode/
.idea/
*.swp
*.swo

.DS_Store
Thumbs.db

checkpoint.json
*.log
*.jsonl
packets.jsonl
near_misses.jsonl
z_status.jsonl
validation_log.jsonl
atlas.sqlite
*.sqlite
*.db
*.csv
*.xlsx
*.html

results/
runs/
experiments/
tmp/
temp/
*.tmp


---

8) docs/solver_theory.md

# Solver Theory

The organizing identity is

\[
n = z^3 + dN(x-y\omega)
\]

with

\[
\omega = \frac{-1+\sqrt{-3}}{2}
\]

and Eisenstein norm

\[
N(x-y\omega)=x^2-xy+y^2=q.
\]

Define

\[
d=x+y,\qquad s=x-y.
\]

Then

\[
4q=d^2+3s^2
\]

and

\[
4(n-z^3)=d^3+3ds^2.
\]

The solver searches over transport slices \(k=n-z^3\), packet factorizations \(k=dq\), and shell intersections satisfying the exact square condition

\[
\Delta = (4q-d^2)/3 = s^2.
\]


---

9) examples/example_commands.txt

python3 sovereign_knight_solver.py --help
python3 sovereign_knight_solver.py --self-check --n 144 --z-start -10 --z-stop 10 --parallel-workers 1
python3 sovereign_knight_solver.py --n 153 --z-start -200 --z-stop 200 --mode qfirst_corridor
python3 sovereign_knight_solver.py --target-set everest --z-start -1000 --z-stop 1000
python3 sovereign_knight_solver.py --n 390 --z-start -5000 --z-stop 5000 --parallel-workers 48


---

10) rust/Cargo.toml

[package]
name = "sovereign-knight-rust"
version = "0.1.0"
edition = "2021"

[dependencies]
num-bigint = "0.4"
num-traits = "0.2"


---

11) rust/src/main.rs

This is a Rust exact verifier, not the full search engine.

use num_bigint::BigInt;
use num_traits::{One, Zero};
use std::env;

fn cube(x: &BigInt) -> BigInt {
    x * x * x
}

fn eisenstein_norm(x: &BigInt, y: &BigInt) -> BigInt {
    x * x - x * y + y * y
}

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() != 5 {
        eprintln!("usage: cargo run -- <n> <x> <y> <z>");
        std::process::exit(1);
    }

    let n: BigInt = args[1].parse().unwrap();
    let x: BigInt = args[2].parse().unwrap();
    let y: BigInt = args[3].parse().unwrap();
    let z: BigInt = args[4].parse().unwrap();

    let d = &x + &y;
    let s = &x - &y;
    let q = eisenstein_norm(&x, &y);
    let lhs = cube(&x) + cube(&y) + cube(&z);
    let k = &n - cube(&z);
    let shell = BigInt::from(4) * &q - &d * &d - BigInt::from(3) * &s * &s;
    let knight = BigInt::from(4) * (&n - cube(&z)) - &d * &d * &d - BigInt::from(3) * &d * &s * &s;

    println!("n={}", n);
    println!("x={}", x);
    println!("y={}", y);
    println!("z={}", z);
    println!("d={}", d);
    println!("s={}", s);
    println!("q={}", q);
    println!("k={}", k);
    println!("cube_check={}", lhs == n);
    println!("packet_check={}", k == &d * &q);
    println!("shell_check={}", shell.is_zero());
    println!("knight_check={}", knight.is_zero());
    println!("parity_check={}", ((&d + &s) % BigInt::from(2)).is_zero());
}

Run:

cd rust
cargo run -- 42 -80538738812075974 80435758145817515 12602123297335631


---

12) cpp/knight_verify.cpp

This is a C++ exact verifier for 64-bit scale inputs.

#include <iostream>
#include <cstdint>

using i128 = __int128_t;

static i128 cube(i128 x) { return x * x * x; }
static i128 norm(i128 x, i128 y) { return x * x - x * y + y * y; }

static void print_i128(i128 v) {
    if (v == 0) {
        std::cout << "0";
        return;
    }
    if (v < 0) {
        std::cout << "-";
        v = -v;
    }
    char buf[64];
    int i = 0;
    while (v > 0) {
        buf[i++] = char('0' + (v % 10));
        v /= 10;
    }
    while (i--) std::cout << buf[i];
}

int main(int argc, char** argv) {
    if (argc != 5) {
        std::cerr << "usage: ./knight_verify <n> <x> <y> <z>\n";
        return 1;
    }

    i128 n = std::stoll(argv[1]);
    i128 x = std::stoll(argv[2]);
    i128 y = std::stoll(argv[3]);
    i128 z = std::stoll(argv[4]);

    i128 d = x + y;
    i128 s = x - y;
    i128 q = norm(x, y);
    i128 lhs = cube(x) + cube(y) + cube(z);
    i128 k = n - cube(z);
    i128 shell = 4 * q - d * d - 3 * s * s;
    i128 knight = 4 * (n - cube(z)) - d * d * d - 3 * d * s * s;

    std::cout << "cube_check=" << (lhs == n) << "\n";
    std::cout << "packet_check=" << (k == d * q) << "\n";
    std::cout << "shell_check=" << (shell == 0) << "\n";
    std::cout << "knight_check=" << (knight == 0) << "\n";
    std::cout << "parity_check=" << (((d + s) % 2) == 0) << "\n";

    std::cout << "d="; print_i128(d); std::cout << "\n";
    std::cout << "s="; print_i128(s); std::cout << "\n";
    std::cout << "q="; print_i128(q); std::cout << "\n";
}

Build:

g++ -O2 -std=c++17 cpp/knight_verify.cpp -o cpp/knight_verify
./cpp/knight_verify 144 6 -2 -4


---

Final note on “multiple source code languages”

This repo now has:

Python: full solver

Rust: exact verifier

C++: exact verifier


