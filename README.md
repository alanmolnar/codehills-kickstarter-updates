# Codehills Kickstarter updates

The public update channel for Codehills Kickstarter, the parent theme
Codehills builds client sites on. It holds two things:

- **Releases**, one per Kickstarter version (`vX.Y.Z`). Each carries the
  theme's production build, `codehills-kickstarter-X.Y.Z-production.zip`,
  its signature `codehills-kickstarter-X.Y.Z-production.zip.sig`, and a
  `SHA256SUMS` that lists the ZIP. The release notes are that version's
  CHANGELOG section without its "Files changed in this release"
  subsection, the same text as `latest.json`'s `notes`, which every site's
  Kickstarter Settings > Updates screen shows.
- **`latest.json`** on `main`: the newest version, where its ZIP is, the
  ZIP's SHA-256 and signature, the WordPress and PHP versions it requires,
  and its notes. Sites read it from
  `https://raw.githubusercontent.com/alanmolnar/codehills-kickstarter-updates/main/latest.json`.

Everything here is written by the release workflow of the theme's private
repository when a version tag is pushed, and never by hand: a release is
signed there, its signature is verified there against the public key in the
theme's source, and only then are the release and `latest.json` published.
A release that already exists is never replaced with different files. Do not
open pull requests or edit files here; a change made by hand would not be
signed, and sites refuse it.

## What `latest.json` holds

| Field | What it is |
|---|---|
| `version`, `tag`, `released_at` | the version, its tag, and when the release workflow published it (UTC, ISO 8601) |
| `package_url` | the production ZIP, an asset of the release |
| `sha256` | the ZIP's SHA-256, hexadecimal |
| `signature` | the Ed25519 signature of the ZIP's SHA-384, base64 (the same text as the `.sig` asset) |
| `requires_wp`, `requires_php` | the `Requires at least` and `Requires PHP` headers of the theme's `style.css` |
| `notes` | the version's CHANGELOG section without its "Files changed in this release" subsection, as text: the release notes, which every site's Kickstarter Settings > Updates screen shows |
| `release_url` | the release page |

## How a site verifies a package

A site running Kickstarter 2.6.2 or later reads `latest.json` from wp-admin
(never on a front-end request) and offers the version in Appearance > Themes
only when it is newer than the installed parent and the site meets
`requires_wp` and `requires_php`. The offer carries no package, so neither
WordPress's own updater nor a site manager can install it. An administrator
updates from Kickstarter Settings > Updates, which, in this order:

1. downloads `package_url`;
2. compares the file's SHA-256 with `sha256`;
3. verifies `signature` with `sodium_crypto_sign_verify_detached()` over the
   raw SHA-384 of the file (48 bytes), against the public key the installed
   theme carries in its own code. This is the scheme WordPress core uses for
   its own packages (`verify_file_signature()`), but the key is never added
   to `wp_trusted_keys`, which also decides what core packages are trusted;
4. installs the build exactly as `wp codehills build install` does: the
   installed parent must match its manifest, the build must match the
   manifest it carries and name the version `latest.json` names, and the
   theme folder is swapped with two renames that roll back on failure.

A failed check leaves the installed theme as it was and says which check
failed.

A site trusts only the key it ships with. A package signed with any other
key is refused, including one that this repository's `latest.json` points
to.

### Checking a release by hand

In a folder holding the three assets of a release:

```bash
sha256sum -c SHA256SUMS
php -r '
  $zip = $argv[1];
  $key = base64_decode( "yHm6+lGsZ3+znLL+IcKXQxLINIFnTBoFKq37o8KoBSE=" );
  $sig = base64_decode( trim( file_get_contents( $zip . ".sig" ) ) );
  exit( sodium_crypto_sign_verify_detached( $sig, hash_file( "sha384", $zip, true ), $key ) ? 0 : 1 );
' codehills-kickstarter-X.Y.Z-production.zip && echo "signature OK"
```

The key above is the one Kickstarter ships with from 2.6.2 (Ed25519, 32
bytes, base64). The authoritative copy is the one in the theme a site runs;
compare against that one when the two could differ.
