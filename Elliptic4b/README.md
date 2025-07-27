# Elliptic4b

## Challenge
- elliptic4b.py
```python
import os
import secrets
from fastecdsa.curve import secp256k1
from fastecdsa.point import Point

flag = os.environ.get("FLAG", "CTF{dummy_flag}")
y = secrets.randbelow(secp256k1.p)
print(f"{y = }")
x = int(input("x = "))
if not secp256k1.is_point_on_curve((x, y)):
    print("// Not on curve!")
    exit(1)
a = int(input("a = "))
P = Point(x, y, secp256k1)
Q = a * P
if a < 0:
    print("// a must be non-negative!")
    exit(1)
if P.x != Q.x:
    print("// x-coordinates do not match!")
    exit(1)
if P.y == Q.y:
    print("// P and Q are the same point!")
    exit(1)
print("flag =", flag)
```

## Solution

Flagを取得する条件は4つです。

1. 提示された y 座標と入力した x 座標の点が楕円曲線上の点であること
2. 次に入力した `a` が負ではないこと
3. `P` と `Q` の x 座標が異なること
4. `P` と `Q` の y 座標が同一であること


### <ins>提示された y 座標と入力した x 座標の点が楕円曲線上の点であること</ins>

ここが一番苦戦しました。

使用されている楕円曲線の secp256k1 の数式は下記なります。

$y^2 \equiv x^3 + 7 \pmod{p}$

この中で既知ではないものは `x` だけです。<br>
そのため、下記の式で求まると考えました。

$x^3 \equiv y^2 - 7 \pmod{p}$

しかし、有限体上の三乗根をSageMath（`nth_root`）を利用しても解けませんでした。<br>
そのため、方針が違うという可能性を考慮し始めていましたが、

[secp256k1](https://www.secg.org/sec2-v2.pdf) を再確認していたところ、<br>
x -> y は導出できるというソースを確認しました。

```
2.3 Recommended 224-bit Elliptic Curve Domain Parameters over Fp SEC 2 (Draft) Ver. 2.0

G = 03 A1455B33 4DF099DF 30FC28A1 69A467E9 E47075A9 0F7E650E
B6B7A45C
and in uncompressed form is:
G = 04 A1455B33 4DF099DF 30FC28A1 69A467E9 E47075A9 0F7E650E
B6B7A45C 7E089FED 7FBA3442 82CAFBD6 F7E319F7 C0B0BD59 E2CA4BDB
556D61A5
```

圧縮時に y を省略するということは x -> y の導出が可能ということです。

であるならば、まだ y -> x と導出できる可能性があるのではないかと<br>
記事を漁ったところ下記のサイトに至りました。

[Bitcoin elliptic curve calculations](https://rawcdn.githack.com/nlitsme/bitcoinexplainer/aa50e86e8c72c04a7986f5f7c43bc2f98df94107/ecdsacrack.html)

> You can decompress from `y` as well.

[GitHub](https://github.com/nlitsme/bitcoinexplainer/blob/master/ec.js)まで行ってスクリプトを確認しましたが、<br>
この方がおそらく書かれた関数を利用されていました。<br>
https://github.com/nlitsme/bitcoinexplainer/blob/master/gfp.js#L137<br>
（y -> x が導出できるのは、secp256k1 などのように a が 0 である場合のみ）
```javascript
var x = y.square().sub(this.b).cubert(flag);
```

提示されている `y` を16進数に変換し、<br>
このサイトを利用することで y -> x を導出することができました。

おそらく、私のSageMathの使い方が悪かったのだろうと思います。

### <ins>P の逆元を求める</ins>

ここまで来たらあとは簡単です。

`P` と `Q` の x 座標が異なり、<br>
`P` と `Q` の y 座標が同一であること

それはつまり `Q` を `P` の逆元とすれば成立します。

楕円曲線暗号の前提知識として下記を覚えてください。

- P と Pの逆元(-P) を足すと 0 となる。（楕円曲線上での 0 は 無限遠点）<br>
$P - P = \mathcal{O}$
- P の位数が q であるとき、積で無限遠点となる。<br>
$q \cdot P = \mathcal{O}$

$q \cdot P$ を下記のように変形します。

$q \cdot P = (q - 1) \cdot P + P$

$q \cdot P$ と $(q - 1) \cdot P + P$ は同じなので<br>
前提知識の $q \cdot P = \mathcal{O}$ を利用して下記のように表せます。

$(q - 1) \cdot P + P = \mathcal{O}$

前提知識の $P - P = \mathcal{O}$ を利用することで下記のように表せます。<br>

$(q - 1) \cdot P = -P$

この $(q - 1)$ の部分を本問題の `a` に入力すれば、<br>
Q を P の逆元とすることができます。

```python
a = secp256k1.q - 1
```

`a` を求めることができたのであとは入力するだけです。

```
> nc elliptic4b.challenges.beginners.seccon.jp 9999
y = 31097083260074404764129653528764159377402723451468312247948553663106487315784
x = 87663981200543859399969406691460306431842045257738747336011788750499487265478
a = 115792089237316195423570985008687907852837564279074904382605163141518161494336
flag = ctf4b{1et'5_b3c0m3_3xp3r7s_1n_3ll1p71c_curv35!}
```

Flagを得ることができました。