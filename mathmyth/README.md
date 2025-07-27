# mathmyth

## Challenge

- problem.py
```python
from Crypto.Util.number import getPrime, isPrime, bytes_to_long
import os, hashlib, secrets


def next_prime(n: int) -> int:
    n += 1
    while not isPrime(n):
        n += 1
    return n


def g(q: int, salt: int) -> int:
    q_bytes = q.to_bytes((q.bit_length() + 7) // 8, "big")
    salt_bytes = salt.to_bytes(16, "big")
    h = hashlib.sha512(q_bytes + salt_bytes).digest()
    return int.from_bytes(h, "big")


BITS_q = 280
salt = secrets.randbits(128)

r = 1
for _ in range(4):
    r *= getPrime(56)

for attempt in range(1000):
    q = getPrime(BITS_q)
    cand = q * q * next_prime(r) + g(q, salt) * r
    if isPrime(cand):
        p = cand
        break
else:
    raise RuntimeError("Failed to find suitable prime p")

n = p * q
e = 0x10001
d = pow(e, -1, (p - 1) * (q - 1))

flag = os.getenv("FLAG", "ctf4b{dummy_flag}").encode()
c = pow(bytes_to_long(flag), e, n)

print(f"n = {n}")
print(f"e = {e}")
print(f"c = {c}")
print(f"r = {r}")
```

- output.txt
```
n = 23734771090248698495965066978731410043037460354821847769332817729448975545908794119067452869598412566984925781008642238995593407175153358227331408865885159489921512208891346616583672681306322601209763619655504176913841857299598426155538234534402952826976850019794857846921708954447430297363648280253578504979311210518547
e = 65537
c = 22417329318878619730651705410225614332680840585615239906507789561650353082833855142192942351615391602350331869200198929410120997195750699143505598991770858416937216272158142281144782652750654697847840376002907226725362778292640956434687927315158519324142726613719655726444468707122866655123649786935639872601647255712257
r = 4788463264666184142381766080749720573563355321283908576415551013379
```

## Solution

時間内に解けませんでした。<br>
自分用の備忘録です。

p = $(q^2 \cdot R) + (g \cdot r)$

n = $(q^3 \cdot R) + (g \cdot q \cdot r)$

n ≡ $R \cdot q^3 \pmod{r}$

$n / R \equiv q^3 \pmod{r}$

$q \bmod r$ が求められる。

p = $(q \bmod r) + k \cdot r$

これで求められると思ったが $2^{56}$ 通りを総当たりできず...<br>
（ここまでは辿り着けてた）

その先の想定解

n = $(q^3 \cdot R) + (g \cdot q \cdot r)$

$n / R = q^3 + \dfrac{g \cdot q \cdot r}{R}$

$\left( \dfrac{n}{R} \right)^{-3} = \left( q^3 + \dfrac{g \cdot q \cdot r}{R} \right)^{-3}$

$\dfrac{g \cdot q \cdot r}{R}$ は bit 数が $q^3$ に比べて小さいため、<br>
$\left( \dfrac{n}{R} \right)^{-3}$ を軸に前後を探索する。

$k = \dfrac{ \left( \dfrac{n}{R} \right)^{-3} - (q \bmod r) }{r}$

p = $(q \bmod r) + k \cdot r$

前と式は同じだが、$k$ の軸が定められているため、計算量が削減されている。

- q_mod_r.py
```python
n = 23734771090248698495965066978731410043037460354821847769332817729448975545908794119067452869598412566984925781008642238995593407175153358227331408865885159489921512208891346616583672681306322601209763619655504176913841857299598426155538234534402952826976850019794857846921708954447430297363648280253578504979311210518547
r = 4788463264666184142381766080749720573563355321283908576415551013379

R = next_prime(Integer(r))

z = Mod(n, r) * inverse_mod(R, r)

factors = factor(r)
moduli = [int(p) for (p, _) in factors]

roots_per_prime = []
for p in moduli:
    Fp = GF(p)
    R_poly.<x> = PolynomialRing(Fp)
    f = x^3 - int(z % p)
    roots = f.roots(multiplicities=False)
    roots_per_prime.append(roots)

from itertools import product
q_mod_r_candidates = []
for comb in product(*roots_per_prime):
    comb_ints = [int(x) for x in comb]
    q = crt(comb_ints, moduli)
    q_mod_r_candidates.append(q)

for i, q in enumerate(q_mod_r_candidates):
    print(f"[{i}] q mod r candidate: {q}")
```

- solver.py
```python
n = 23734771090248698495965066978731410043037460354821847769332817729448975545908794119067452869598412566984925781008642238995593407175153358227331408865885159489921512208891346616583672681306322601209763619655504176913841857299598426155538234534402952826976850019794857846921708954447430297363648280253578504979311210518547
r = 4788463264666184142381766080749720573563355321283908576415551013379
R = next_prime(Integer(r))


q_mod_r_candidates = [
    4200798870418079743183373640690101637639619324954989327173283704835,
    3989732896157417202719300984522116216404235045924973291866646132116,
    1462887293675083884699153856274442031003069830086435930647905652415,
    3306964422699180513371995395917959355933026978430178798303195867103,
    3095898448438517972907922739749973934697642699400162762996558294384,
    569052845956184654887775611502299749296477483561625401777817814683,
    1887646452409914871723975231699699802328557333654157996981670868770,
    1676580478149252331259902575531714381093173054624141961675033296051,
    3938198140333103155621521528033760769255363160069513176871843829729
]

q_approx = (n // R).nth_root(3, truncate_mode=True)[0]
print(f"[*] Approximate q ≈ {q_approx}")

for q0 in q_mod_r_candidates:
    print(f"\n[>] Trying q0 ≡ {q0} mod r")

    k_center = (q_approx - q0) // r

    for delta in range(-1000, 1000):
        k = k_center + delta
        if k < 0:
            continue
        q = q0 + k * r
        if not is_prime(q):
            continue
        if n % q == 0:
            p = n // q
            print("p = ", p)
            print("q = ", q)
            break
    else:
        print("Not Found.")
        continue
    break
else:
    print("Not Found.")
```
