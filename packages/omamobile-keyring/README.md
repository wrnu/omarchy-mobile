# omamobile-keyring

The public key that signs Omarchy Mobile's packages and the `[omamobile]` repository
database, in pacman's keyring format. Installing it runs
`pacman-key --populate omamobile`, which imports the key and marks it trusted
(`omamobile-trusted`), the same way `archlinux-keyring` works.

The private keys never leave their build machines (`keys/`, ignored by git).
Fingerprints:

- `6C84FE3D240709E70E27CC94B301F5EA62B249FE`: the project's packages
- `ED1F8C61F70743686E2E2D37CCCF3E18CD8C2FDF`: a local build of this tree
