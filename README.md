# JAP

A private, password-protected static site hosted on GitHub Pages.

**Everything in this repository is encrypted.** Each HTML file is encrypted at rest with
[StatiCrypt](https://github.com/robinmoisson/staticrypt) (AES-256-CBC, PBKDF2 key derivation).
Opening any page shows a password prompt; decryption happens locally in the browser.

The password is **not** stored in this repository and is shared privately. Without it there is
nothing readable here.

The repository is public only because GitHub Pages requires it on the free plan.

---

Built with `staticrypt`. See `DEPLOY_INSTRUCTIONS.md` for the build and deploy steps.
