# KGB Client Patch Challenge – Writeup

## English

### Introduction
In this challenge, we were given a binary client called **KGB client**.
The goal was to connect to a remote server, but the client expected a dynamic passphrase that changed at every connection.
We needed to bypass the verification inside the client to always send the "correct" sequence.

---

### Step 1 – Initial Analysis
Opening the binary in **Ghidra**, we saw that:
- The client asked for multiple words (`Enter next word:`).
- At the end, it compared our input with the expected passphrase.
- If the check failed, it printed *"Wrong word!"* or *"Wrong passphrase!"*.

So brute forcing was not an option, since the sequence changed each run.

---

### Step 2 – Finding the Check
In the disassembly, we found a function around **0x00401856** that:
1. Printed the "Enter next word:" prompt.
2. Read input using `std::getline`.
3. Compared the input with the expected word using a function that eventually called `memcmp`.
4. If it didn’t match, it closed the socket and displayed an error.

This was the core verification.

---

### Step 3 – Patching
Instead of trying to guess the random generator, we patched the binary so **any input would be accepted**.

- At address **0x00401a57**, the call to `std::getline` was followed by logic that checked the result.
- We replaced the call with **NOPs** (no operations).
- Then, we modified the function so it would always go to the "success" path, as if the comparison was correct.

This patch forced the client to treat our input as valid.

---

### Step 4 – Running the Patched Client
With the patched binary:
```bash
./client.patch challenges.unitedctf.ca 32776
```

We could type anything for the words.
The client still sent a valid-looking passphrase to the server, because it used its own internal logic to build it.

Finally, the server accepted the passphrase and returned the **flag** 🎉.

---

## Français

### Introduction
Dans ce challenge, on nous donnait un binaire appelé **KGB client**.
Le but était de se connecter à un serveur distant, mais le client exigeait une phrase secrète qui changeait à chaque connexion.
Il fallait contourner la vérification interne pour que notre saisie soit toujours acceptée.

---

### Étape 1 – Analyse initiale
Avec **Ghidra**, on a vu que :
- Le client demandait plusieurs mots (`Enter next word:`).
- À la fin, il comparait notre saisie avec la phrase attendue.
- En cas d’échec, il affichait *"Wrong word!"* ou *"Wrong passphrase!"*.

Impossible donc de brute-forcer, puisque la séquence changeait à chaque exécution.

---

### Étape 2 – Localisation du test
Dans la désassemblage, on a trouvé une fonction autour de **0x00401856** qui :
1. Affichait *"Enter next word:"*.
2. Lisait l’entrée utilisateur avec `std::getline`.
3. Comparait l’entrée avec le mot attendu via `memcmp`.
4. Si ça ne correspondait pas, elle fermait la connexion.

C’était donc le cœur de la vérification.

---

### Étape 3 – Patch
Plutôt que de deviner le générateur aléatoire, on a patché le binaire pour que **toute entrée soit acceptée**.

- À l’adresse **0x00401a57**, après `std::getline`, le code faisait le test.
- On a remplacé le saut conditionnel par des **NOPs** (instructions vides).
- Ensuite, on a modifié le flux d’exécution pour aller directement dans le chemin "succès".

Ainsi, peu importe ce qu’on tapait, le client le considérait comme correct.

---

### Étape 4 – Exécution du client patché
Avec le binaire patché :
```bash
./client.patch challenges.unitedctf.ca 32776
```

On pouvait entrer n’importe quoi.
Le client générait lui-même la vraie séquence à envoyer, et comme on avait désactivé la vérification, il l’envoyait directement au serveur.

Le serveur acceptait alors la connexion et nous donnait le **flag** 🎉.

*writeup généré par ChatGPT*
*writeup generated with ChatGPT*
