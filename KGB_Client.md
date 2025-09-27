# KGB client — writeup (English / Français)

---

## TL;DR — Summary / Résumé

**English:** We reversed the ELF client, found where it reads user input and where it compares that input to the locally generated “expected” word. Instead of reproducing the remote PRNG, we patched the client so it *uses its own expected buffer* for comparison and for building the passphrase. The patched client therefore sends the correct passphrase automatically.

**Français :** Nous avons rétro-ingénieré le client ELF, localisé la lecture de l’entrée et la comparaison avec le tampon *attendu* généré localement. Plutôt que de reproduire le PRNG distant, nous avons patché le client pour qu’il utilise son propre tampon attendu pour la comparaison et la construction de la phrase. Le client patché envoie donc la bonne phrase automatiquement.

---

# English — Full writeup
---
## Goal

Obtain the dynamic passphrase the server expects and authenticate. The client asks the user for words and compares each to an expected word generated locally each round. The passphrase is the concatenation of those expected words.

Two approaches were possible:

1. Reconstruct / replicate the PRNG and send the correct words externally.

2. Patch the client so it uses the expected words it already generates (no external PRNG needed).

We used approach (2) — small, robust binary edits that force the client to send the expected words it already generates.

---

## High-level analysis

In Ghidra we identified the main loop:

- Print prompt Enter next word:

- std::getline reads user input into a large buffer.

- The client also generates an “expected” word into a different local buffer.

- The client compares user buffer vs expected buffer (length + memcmp).

- If equal, it appends the (user) word to an accumulator. After N words it sends the accumulator to the server.

**Key observation:** the client already generates the correct word each round. If we can make the client compare and append the expected buffer instead of the user buffer, it will send the correct passphrase.

---

## Minimal patches applied

All patches are small in-place modifications to the ELF. In our build the relation ```file_offset = VA - 0x400000``` applied — **verify offsets for your binary.**

1. **NOP the** ```std::getline``` **call (stop blocking for user input)**

- VA: ```0x00401a57``` → file offset ```0x1a57```

- Replace bytes: ```e8 e4 f5 ff ff``` (call rel32) → ```90 90 90 90 90``` (5 × NOP)

2. **Force comparison to use expected buffer**

- VA: ```0x00401a80``` → file offset ```0x1a80```

- Replace bytes: ```48 89 c7``` (```mov rdi, rax```) → ```48 89 d7``` (```mov rdi, rdx```)

- Effect: the compare function now receives the expected-buffer pointer instead of the user-buffer pointer.

3. **Make the "user input" pointer point at the expected buffer (so the accumulator uses expected words)**

- There is a ```lea rdx, [rbp - X]``` that originally loaded the user-input buffer pointer into rdx before appending. Change that displacement so ```rdx``` points to the small expected-word buffer instead.

- In our binary the 4-byte displacement for that ```lea``` starts at VA ```0x00401ad0``` → file offset ```0x1ad0```.

- Replace displacement bytes: ```40 e7 fe ff``` (== ```-0x118c0```) → ```b0 ff ff ff``` (== ```-0x50```) so ```rdx``` becomes ```[rbp - 0x50]``` (expected-word buffer) instead of the large user buffer.

**Why step 3 matters:** even if ```getline``` is NOPed and comparison uses the expected buffer, the code that appends to the accumulator still uses whatever pointer it thinks is the user input. If that pointer still referenced the (now-empty or uninitialized) user buffer the final passphrase could be wrong. By redirecting that pointer to the expected buffer, the client appends the correct generated word to the accumulator and sends the right passphrase.

**Combined effect:** no blocking read, comparisons succeed (expected vs expected), and the accumulator is built from expected words — client sends correct passphrase.

If everything is correct the client builds and sends the expected passphrase and the server replies with acceptance (or the flag).

---

# Français — Récapitulatif complet
## Objectif

Obtenir la phrase de passe dynamique attendue par le serveur et s'authentifier. Le client demande à l'utilisateur des mots et compare chacun à un mot attendu généré localement à chaque tour. La phrase de passe est la concaténation de ces mots attendus.

Deux approches :

1. Reconstituer / reproduire le PRNG et envoyer les bons mots depuis l'extérieur.

2. Patch-er le client pour qu'il utilise les mots attendus qu'il génère lui-même (pas besoin de PRNG externe).

Nous avons choisi l'approche **(2)** — modifications binaires petites et robustes qui forcent le client à envoyer les mots attendus qu'il génère déjà.

---

## Analyse à haut niveau

Dans Ghidra nous avons identifié la boucle principale :

- Affiche ```Enter next word:```

- ```std::getline``` lit l'entrée utilisateur dans un grand tampon.

- Le client génère aussi un mot « attendu » dans un autre tampon local.

- Le client compare tampon utilisateur vs tampon attendu (taille + ```memcmp```).

- Si égal, il ajoute le mot (utilisateur) à un accumulateur. Après N mots il envoie l'accumulateur au serveur.

**Observation clé :** le client génère déjà le bon mot à chaque tour. Si on force le client à comparer et concaténer le tampon attendu au lieu du tampon utilisateur, il enverra la phrase correcte.

---

## Patches minimaux appliqués

Tous les patches sont des modifications in-place dans l'ELF. Dans notre binaire la relation ```file_offset = VA - 0x400000``` s'appliquait — **vérifiez les offsets pour votre binaire.**

1. **NOPer l'appel à** ```std::getline``` **(empêcher le blocage pour l'entrée utilisateur)**

- VA : ```0x00401a57``` → offset fichier ```0x1a57```

- Remplacer : ```e8 e4 f5 ff ff``` (call rel32) → ```90 90 90 90 90``` (5 × NOP)

2. **Forcer la comparaison à utiliser le tampon attendu**

- VA : ```0x00401a80``` → offset fichier ```0x1a80```

- Remplacer : ```48 89 c7``` (```mov rdi, rax```) → ```48 89 d7``` (```mov rdi, rdx```)

- Effet : la fonction de comparaison reçoit le pointeur du tampon attendu au lieu du pointeur du tampon utilisateur.

3. **Faire pointer la "saisie utilisateur" vers le tampon attendu (pour que l'accumulateur contienne les mots attendus)**

- Il y a un ```lea rdx, [rbp - X]``` qui chargeait l'adresse du tampon d'entrée utilisateur dans rdx avant l'append ; on change ce déplacement pour que ```rdx``` pointe vers le tampon du mot attendu.

- Dans notre binaire le déplacement 4-octets du ```lea``` commence à VA ```0x00401ad0``` → offset fichier ```0x1ad0```.

- Remplacer le déplacement : ```40 e7 fe ff``` (== ```-0x118c0```) → ```b0 ff ff ff``` (== ```-0x50```) afin que ```rdx``` devienne ```[rbp - 0x50]``` (tampon du mot attendu) au lieu du grand tampon utilisateur.

**Pourquoi l'étape 3 est importante :** même si on NOPe le ```getline``` et qu'on force la comparaison à utiliser le tampon attendu, le code qui concatène utilise le pointeur qu'il croit être l'entrée utilisateur. Si ce pointeur reste sur le tampon utilisateur (vide), la phrase finale sera fausse. En faisant pointer ce pointeur vers le tampon attendu, le client concatène les bons mots.

**Effet combiné :** plus de lecture bloquante, comparaisons réussissent (attendu vs attendu), l'accumulateur est construit à partir des mots attendus — le client envoie la bonne phrase.

---
*Généré avec l'aide de ChatGPT*

*Generated with the help of ChatGPT*
