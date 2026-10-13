# My Process

<img width="298" height="127" alt="image" src="https://github.com/user-attachments/assets/8d78430c-f82b-4de5-bb9f-8dc70d2df0c0" />

Why this layout:

- Caldera on Kali: Kali packages Caldera in its repos, so you avoid a third VM. It only needs to be reachable by its agents, which comes in Phase 6.
- Nessus on its own VM: plugin compilation is RAM- and CPU-heavy and would bog Kali down. A dedicated scanner is also how it’s deployed in real environments.
- Running set for Phase 5: 100, 101, 130, 131, 140, 141 = about 24 GB. AD VMs stay off.
















































