# KGB client — writeup (English / Français)

---

## TL;DR — Summary / Résumé

**English:** We reversed the ELF client, found where it reads user input and where it compares that input to the locally generated “expected” word. Instead of reproducing the remote PRNG, we patched the client so it *uses its own expected buffer* for comparison and for building the passphrase. The patched client therefore sends the correct passphrase automatically.

**Français :** Nous avons rétro-ingénieré le client ELF, localisé la lecture de l’entrée et la comparaison avec le tampon *attendu* généré localement. Plutôt que de reproduire le PRNG distant, nous avons patché le client pour qu’il utilise son propre tampon attendu pour la comparaison et la construction de la phrase. Le client patché envoie donc la bonne phrase automatiquement.

---

# Full writeup (English)

## Goal

Obtain the dynamic passphrase the server expects and authenticate. The client asks the user for words and compares them to an *expected* word generated locally each round. The passphrase is the concatenation of those expected words.

Two approaches:

1. Reconstruct/replicate the PRNG and send the correct words externally.
2. Patch the client so it uses its own generated expected words (no external PRNG needed).

We used approach **(2)** — small, robust binary edits that force the client to send the expected words it already generates.

## High-level analysis

* In Ghidra we identified the main loop:

  * Print prompt `Enter next word:`
  * `std::getline` reads user input into a large buffer.
  * The client also generates an “expected” word into a different local buffer.
  * The client compares user buffer vs expected buffer (size + `memcmp`).
  * If equal, it appends the (user) word to an accumulator. After N words it sends the accumulator to the server.

Key observation: **the client already generates the correct word** each round. If we can make the client compare & append the expected buffer instead of the user buffer, it will send the correct passphrase.

## Minimal patches applied

All patches are tiny in-place modifications to the ELF. For our binary the relation `file_offset = VA - 0x400000` applied; you must verify offsets for your specific binary.

1. **NOP the `std::getline` call (stop blocking for user input)**

   * VA: `0x00401a57` → file offset `0x1a57`
   * Replace `e8 e4 f5 ff ff` (call rel32) with `90 90 90 90 90` (5x NOP)

2. **Force comparison to use expected buffer**

   * VA: `0x00401a80` → file offset `0x1a80`
   * Replace `48 89 c7` (`mov rdi, rax`) with `48 89 d7` (`mov rdi, rdx`)
   * This makes the compare function receive the expected buffer pointer instead of the user buffer pointer.

3. **Replace the user-input pointer so the client uses the expected word as its "input"**

   * Instruction: there is a `lea rdx, [rbp - X]` that originally loaded the address of the user-input buffer into `rdx` before appending; we change that displacement so `rdx` points to the small expected-word buffer instead.
   * In our binary that `lea`'s 4-byte displacement starts at VA `0x00401ad0` (file offset `0x1ad0`).
   * Replace original displacement `40 e7 fe ff` (== `-0x118c0`) with `b0 ff ff ff` (== `-0x50`) so `rdx` becomes `[rbp - 0x50]` (the expected-word buffer) instead of the large user buffer.

**Why step 3 is important:** even if we NOP the `getline` and force the comparison to use the expected buffer (step 2), the code later appends whatever pointer it thinks is the user input into the final passphrase. If that append still pointed at the (now-empty or uninitialized) user buffer, the final passphrase could be wrong. By redirecting the "user input" pointer to the expected buffer, the client will append the correct generated word to the accumulator and send the right passphrase.

**Combined effect:** no blocking read, comparisons succeed (expected vs expected), and the accumulator is built from expected words — client sends correct passphrase.

## Safe workflow (what we executed)

1. Backup the binary:

```bash
cp client.patch client.patch.bak
```

2. Inspect bytes (before patch):

```bash
xxd -g 1 -s 0x1a57 -l 8 client.patch   # confirm call bytes
xxd -g 1 -s 0x1a80 -l 8 client.patch   # confirm mov rdi,rax
xxd -g 1 -s 0x1ad0 -l 8 client.patch   # confirm displacement bytes
```

3. Apply patches (examples using dd/printf on WSL):

```bash
# Patch A: NOP call at 0x1a57
printf '\x90\x90\x90\x90\x90' > /tmp/newA.bin
dd if=/tmp/newA.bin of=client.patch bs=1 seek=$((0x1a57)) conv=notrunc

# Patch B: mov rdi,rax -> mov rdi,rdx at 0x1a80
printf '\x48\x89\xd7' > /tmp/newB.bin
dd if=/tmp/newB.bin of=client.patch bs=1 seek=$((0x1a80)) conv=notrunc

# Patch C: change lea displacement at 0x1ad0
printf '\xb0\xff\xff\xff' > /tmp/newC.bin
dd if=/tmp/newC.bin of=client.patch bs=1 seek=$((0x1ad0)) conv=notrunc
```

4. Verify writes:

```bash
xxd -g 1 -s 0x1a57 -l 8 client.patch
xxd -g 1 -s 0x1a80 -l 8 client.patch
xxd -g 1 -s 0x1ad0 -l 8 client.patch
```

5. Run the patched client:

```bash
chmod +x client.patch
./client.patch challenges.unitedctf.ca 32776
```

If everything is correct the client builds and sends the expected passphrase and the server replies with acceptance (or the flag).

## Thought process

* Spot the “source of truth”: the expected-word generator inside the client.
* Two broad routes: replicate PRNG (fragile: endianness, seed derivation, indexing variant) or force client to use its own values. The latter is less work and more robust.
* Minimal patches reduce risk — change only the needed instructions:

  * remove blocking read,
  * change a register-move to use expected buffer,
  * change a displacement so the append uses expected buffer.
* Test iteratively and keep backups.

## Caveats & ethics

* Use this only on CTF/owned/test binaries. Patching binaries on systems you do not own is unethical and illegal.
* Always keep backups (`client.patch.bak`).
* If the server still rejects the passphrase, the client’s generator may be different from the server’s expectation (unlikely if client uses the same generator; otherwise revert and attempt PRNG approach).

---

# Rapport complet (Français)

## Objectif

Récupérer la passphrase dynamique demandée par le serveur pour s’authentifier. Le client demande à l’utilisateur chaque mot et compare avec un mot *attendu* généré localement. Nous voulons que le client envoie automatiquement la bonne phrase.

## Analyse

* Repérage dans Ghidra :

  * `std::getline` lit l’entrée utilisateur.
  * Un générateur local remplit le tampon *attendu*.
  * Une fonction compare user-buffer vs expected-buffer (taille + `memcmp`).
  * Si égal → on concatène le mot dans l’accumulateur.
* On constate que le client produit déjà le mot correct. La solution simple : faire en sorte que le client compare et concatène le tampon attendu (et non le tampon utilisateur).

## Patches appliqués (résumé)

1. NOP le `call std::getline` (`0x00401a57`) → `e8 e4 f5 ff ff` → `90 90 90 90 90`
2. `mov rdi, rax` → `mov rdi, rdx` (`0x00401a80`) → `48 89 c7` → `48 89 d7`
3. changement de displacement dans `lea` (displacement à `0x00401ad0`) → `40 e7 fe ff` → `b0 ff ff ff`

## Procédure sûre

(Identique aux commandes en anglais — voir la section **Safe workflow**.)

## Raisonnement

* Réutiliser le code déjà présent (générateur) est plus rapide et plus fiable que de tenter de reconstituer un PRNG et de synchroniser seeds/endianness.
* Faire des modifications minimes et localisées facilite la vérification et la réversibilité.

## Éthique & précautions

* N’effectuez ces manips que dans des environnements autorisés (CTF / labo).
* Sauvegardez toujours l’original.

---

# Appendix — quick checklist (copy-paste)

```bash
# Backup
cp client.patch client.patch.bak

# Inspect locations
xxd -g 1 -s 0x1a57 -l 8 client.patch   # check call
xxd -g 1 -s 0x1a80 -l 8 client.patch   # check mov rdi,rax
xxd -g 1 -s 0x1ad0 -l 8 client.patch   # check displacement

# Apply patches
printf '\x90\x90\x90\x90\x90' > /tmp/newA.bin
dd if=/tmp/newA.bin of=client.patch bs=1 seek=$((0x1a57)) conv=notrunc

printf '\x48\x89\xd7' > /tmp/newB.bin
dd if=/tmp/newB.bin of=client.patch bs=1 seek=$((0x1a80)) conv=notrunc

printf '\xb0\xff\xff\xff' > /tmp/newC.bin
dd if=/tmp/newC.bin of=client.patch bs=1 seek=$((0x1ad0)) conv=notrunc

# Verify
xxd -g 1 -s 0x1a57 -l 8 client.patch
xxd -g 1 -s 0x1a80 -l 8 client.patch
xxd -g 1 -s 0x1ad0 -l 8 client.patch

# Run patched client
chmod +x client.patch
./client.patch challenges.unitedctf.ca 32776
```

---

If you want I can also produce a single binary patch file (diff) or a restore script. Let me know which format you prefer.
