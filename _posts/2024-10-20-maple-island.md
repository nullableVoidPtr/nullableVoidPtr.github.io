---
layout: single
title: Maple Island (misc; 100 points) writeup - Maple CTF 2023
---
> Initially published 2024-10-08, immediately after Maple CTF 2023.


> "And now for something completely different"


<i>special thanks to fanfiction.net and wattpad for the """intuition"""</i>

In MapleCTF 2023, we are presented with the following[^1]:

> **Title:** `Maple Island <3`
> 
> **Tags:** `matching`
>
> **Description:**
> > The name of the game is simple. It's love. They say opposites attract. You know like North and South, Hot and Cold, etc. The same is said to be true for parity too, the odd (the ones) and even DWORDS (the zeroes) have always had quite steamy and passionate relationships.
> > 
> > Historically speaking, tradition was paramount for this species. The zeroes scour the world in hopes of find their special One. (Where do you think the saying comes from? duh.) However, we are in the 21st century and must adapt to the new.
> > 
> > So, we made an entire reality TV show about it. The premise is simple: Screw tradition, in this show, only the Ones are allowed to court the zeroes.
> > 
> > Stay tuned for the most drama-filled season of Maple Island as of yet with even more tears, arguments, and passionate moments than ever before. Will every match made in Maple heaven be stable?
> > 
> > Maple Island streaming next month on MapleTV!
> > 
> > But wait, lucky viewers have a chance to catch exclusive early-access content if they can solve the following puzzle below and text the answer to 1-800-MAPLE-1337.
> > 
> > Author: hiswui

[^1]: i never really understood love island tbh

While this challenge in and of itself is pretty simple seeing as the points reward reduced to 100 after 47 solves, this writeup consists of my approach when solving this challenge.

`nc`-ing to the provided remote gave us the same prelude in the description with some Python literal values in the format `f"{key}: {value}"`:
```py
ones: list[bytes]
zeroes: list[bytes]
oprefs: list[list[bytes]]
zprefs: list[list[bytes]]
ctext: bytes
```

Looking at the provided `server.py` yields some insights:
* The contestant data are created in `generate_contestants` with the following:
	* `ones` are random DWORDs with the last bit set to 1, `zeroes` likewise, until a minimum of 20 for each are available
	* `oprefs` are 20 randomly selected `zeroes`, and vice versa
* The `ctext` is encrypted with a XOR key generated as such:
	* Contestant info `(ones, zeroes, oprefs, zprefs)` are passed to a hidden `create_a_perfect_world` function which yields to a `couple` variable
	* Each `couple` are then appended as-is to `otp: bytes`
	* `otp` is the resultant XOR key

With my experience from years in fandom participation[^2], I had deduced that `otp` is just a string of [*"One True Pairings"*](https://tvtropes.org/pmwiki/pmwiki.php/Main/OneTruePairing) - a couple that is a "perfect fit", as hinted by it being a result of a "perfect world". Hence, each `couple` or keypart would be the "ship name" from the two contestants, i.e. the two names - byte strings in this case - joined together.

Moreover, as it was said that in this reality TV show, the "Ones" are to court the "Zeroes", meaning that following fandom convention, the Ones are on the "left" of the ship names[^3], hence their coupled Zero "on the right" are appended to their names to create their ship name.

Additionally, because each couple is an OTP, it should follow that there would be one OTP for each "One" contestant, and hence would not be reused in another couple later in the total key.

With the sum of these untested total guesses, I fashioned the following code to test:

```py
# FIXME: too lazy to parse using pwntools.remote
# legit just monkey copy-paste from stdout
# then s/: /=/ thx

ones = {one: set(opref) for one, opref in zip(ones, oprefs)}
zeroes = {zero: set(zpref) for zero, zpref in zip(zeroes, zprefs)}

potential_otps = {
    one: [
        zero
        for zero in opref
        if one in zeroes[zero]
    ]
    for (one, opref) in ones.items()
}

import string
from Crypto.Util.strxor import strxor

def find(offset: int, excluded: set) -> tuple[bytes, bytes, bytes] | None:
    for one, mprefs in potential_otps.items():
        if one in excluded:
            continue

        for zero in mprefs:
            otp = one + zero
            attempt = strxor(ctext[offset:offset + len(otp)], otp)
            # had to tweak allowed characters a couple of times
            if all(chr(c) in string.ascii_letters + string.digits + ' .{_}' for c in attempt):
                return (attempt, one, zero)

    return None

flag = ""
matched = set()
while len(flag) < len(ctext):
    result = find(len(flag), matched)
    if result is None:
        break

    part, one, _ = result
    print(part)
    matched.add(one)
    flag += part.decode()

```

And, the result is:
```sh
$ python test.py
b'Love wil'
b'l never '
b'give up '
b'on you. '
b'Love wil'
b'l never '
b'let you '
b'down. Lo'
b've will '
b'never ru'
b'n around'
b' and des'
b'ert you '
b'maple{G0'
b'5h_1_w4n'
b't_4_st4b'
b'l3_m4tch'
b'_t00_pls'
b'_1_4m_50'
b'_l0n3ly}'
Love will never give up on you. Love will never let you down. Love will never run around and desert you maple{G05h_1_w4nt_4_st4bl3_m4tch_t00_pls_1_4m_50_l0n3ly}
```

It works! First try!

**Flag:** `maple{G05h_1_w4nt_4_st4bl3_m4tch_t00_pls_1_4m_50_l0n3ly}`[^4]

# Post Script

According to the [published challenge](https://github.com/ubcctf/maple-ctf-2023-public/blob/main/misc/maple-island/solve/solve.py), the actual intended solve was some nerd stuff called the [Gale-Shapley algorithm](https://en.wikipedia.org/wiki/Gale%E2%80%93Shapley_algorithm), and that this challenge was actually a formulation of the [stable matching problem](https://en.wikipedia.org/wiki/Stable_marriage_problem)

I'm so cooked. I have crippling brainrot.

[^2]: my ao3 bookmarks are between me and god stop asking
[^3]: and i'm not explaining "being on top" in this writeup either
[^4]: ~~me too buddy~~