# CIDR Math — Deriving Subnet Ranges by Hand

Companion to `README.md` §2.4. That section gave you the shortcut; this doc
derives *why* the shortcut works, in binary, so you can compute any `/N`
range without memorizing a lookup table.

---

## 1. Start from the VPC: `172.31.0.0/16`

An IPv4 address is 32 bits, written as 4 octets of 8 bits each:

```text
172       . 31        . 0        . 0
10101100  . 00011111  . 00000000 . 00000000
```

`/16` means **the first 16 bits are the fixed network portion**; everything
after is host space:

```text
10101100.00011111 | 00000000.00000000
     network (16)  |     host (16)
```

Host bits = `32 - 16 = 16` → addresses = `2^16 = 65,536`.

So the VPC owns every address from `172.31.0.0` through `172.31.255.255` —
the two fixed octets (`172.31`) never change; the last two octets (`0.0`
through `255.255`) are entirely free for subnets to carve up.

---

## 2. What `/20` carves out of that

`/20` fixes the first 20 bits, leaving `32 - 20 = 12` host bits →
`2^12 = 4096` addresses per subnet.

Those 12 host bits split across the third and fourth octets:

```text
172.31.0.0/20

10101100.00011111.00000000.00000000
|------ 20 fixed bits -----|-- 12 host bits --|
```

The third octet contributes its **last 4 bits** as host bits (8 bits of
the fourth octet + 4 bits of the third octet = 12):

```text
third octet:  0000 | 0000
              ^^^^   ^^^^
            network  host (4 bits → 0–15)

fourth octet: 00000000
              ^^^^^^^^
              host (8 bits → 0–255)
```

So `172.31.0.0/20` ranges over:
- third-octet host bits: `0000`→`1111` = **0 to 15**
- fourth-octet: **0 to 255** (fully free)

Range: `172.31.0.0` → `172.31.15.255`.

---

## 3. Where the next `/20` starts

The previous block ended at `172.31.15.255`. The next block starts one
address later: `172.31.16.0`. Its network bits fix the third octet's upper
4 bits to `0001` (=16), and the same 4+8 host bits apply again:

```text
172.31.16.0/20   →  172.31.16.0  – 172.31.31.255
172.31.32.0/20   →  172.31.32.0  – 172.31.47.255
172.31.48.0/20   →  172.31.48.0  – 172.31.63.255
172.31.64.0/20   →  172.31.64.0  – 172.31.79.255
172.31.80.0/20   →  172.31.80.0  – 172.31.95.255
172.31.96.0/20   →  172.31.96.0  – 172.31.111.255
```

Each block's start is the previous block's end, plus 1.

---

## 4. The shortcut: subnet mask → block size

Instead of redoing the binary each time, convert `/N` to its dotted-decimal
subnet mask and read the block size off the first octet that isn't `255`
or `0`:

```text
/20  =  11111111.11111111.11110000.00000000  =  255.255.240.0
                                    ^^^^^^^^
                            "interesting octet" = 240

block size = 256 - 240 = 16
```

That `16` is the increment between consecutive `/20` networks in the third
octet — which is exactly the sequence from §3: `0, 16, 32, 48, 64, 80, 96, ...`.

**General rule:** find the octet in the mask that isn't `255` or `0`;
`256 - <that value>` is the step size in that octet.

---

## 5. Why re-creating `172.31.0.0/20` fails

An existing subnet already owns `172.31.0.0` → `172.31.15.255`. Asking AWS
to create another subnet on the exact same CIDR is asking for the *same*
address space twice:

```text
Existing:  172.31.0.0 ─────────────── 172.31.15.255
New (same request): 172.31.0.0 ─────── 172.31.15.255
                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                     100% overlap → AWS rejects it
```

Overlap doesn't require an *exact* match — any range intersection is
rejected. Two blocks only coexist safely if neither's range intersects the
other's.

---

## 6. The formula, generalized

For any CIDR `/N` on a 32-bit IPv4 address:

```text
host bits        = 32 - N
addresses/subnet  = 2^(32 - N)
mask octet value  = 256 - (block size in that octet)   [derived from binary]
block size        = 256 - mask_octet_value
```

| CIDR | Host bits | Addresses | Mask | Increment octet | Step |
|------|-----------|-----------|------|------------------|------|
| /16  | 16        | 65,536    | 255.255.0.0     | 3rd octet | 256 (whole octet) |
| /20  | 12        | 4,096     | 255.255.240.0   | 3rd octet | 16 |
| /24  | 8         | 256       | 255.255.255.0   | 4th octet | 1 (whole octet) |
| /26  | 6         | 64        | 255.255.255.192 | 4th octet | 64 |
| /28  | 4         | 16        | 255.255.255.240 | 4th octet | 16 |

---

## 7. Practice — do these by hand before checking

For a base network `10.0.0.0`, derive the first three consecutive blocks
for each mask (network address, last address, block count):

- [ ] `/24` (hint: mask `255.255.255.0`, whole 4th octet is host space)
- [ ] `/26` (hint: mask `255.255.255.192`, step = `256-192`)
- [ ] `/28` (hint: mask `255.255.255.240`, step = `256-240`)

Then answer without a calculator:

- [ ] Given a VPC `10.0.0.0/16` with subnets already at `10.0.0.0/24` and
      `10.0.1.0/24`, what's the next available `/24`?
- [ ] Why does a `/24` always land on a whole-number third octet
      (`10.0.0.0`, `10.0.1.0`, `10.0.2.0`, ...) instead of splitting a
      octet like `/20` or `/26` do?
- [ ] A subnet is `172.31.96.0/20`. Without listing every address, state
      its first and last address using §2–3's method.

---

## 8. Where this plugs back into the lab

`README.md` §2.4 uses steps 4–5 of this document as a given; this file is
the derivation behind that shortcut. Use §6's table as the quick-reference
you actually keep open while working; use §1–5 to rebuild it from scratch
if you ever forget *why* the table is true.
