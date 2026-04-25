# sgrep

Version pédagogique plus ambitieuse du `minigrep` de la documentation Rust.

Le but est de construire progressivement un vrai petit `grep` en Rust, avec une architecture propre, une CLI correcte, des tests, puis des sujets plus avancés comme le streaming, les regex, le parcours récursif et les performances.

Les exemples ci-dessous utilisent parfois `rgrep` comme nom cible. Si tu gardes le nom du dépôt, remplace simplement par `sgrep`.

## Exemples visés

```sh
rgrep "TODO" src --ignore-case --line-number --recursive
rgrep "fn main" . -n -C 2 --glob "*.rs"
rgrep "error" logs --json --count
```

## Principe clé

`main.rs` doit rester très mince. Toute la logique testable vit dans `lib.rs` et dans les modules du crate.

```rust
fn main() {
    if let Err(err) = rgrep::run() {
        eprintln!("error: {err}");
        std::process::exit(1);
    }
}
```

## Architecture cible

```text
src/
├── main.rs      # Entrée binaire, très mince
├── lib.rs       # API publique du crate
├── cli.rs       # Parsing des arguments CLI
├── config.rs    # Config validée, indépendante de clap
├── error.rs     # Types d'erreurs métier
├── matcher.rs   # Logique de matching
├── search.rs    # Recherche dans un fichier / flux
├── walker.rs    # Parcours fichiers/dossiers
├── output.rs    # Formatage terminal, JSON, couleurs
└── model.rs     # Match, SearchResult, FileResult, etc.

tests/
├── cli.rs
├── search.rs
└── fixtures/
    ├── poem.txt
    └── rust_sample.rs
```

## Documentation

- [`docs/design.md`](docs/design.md) : architecture, parsing CLI, `Config`, streaming, API interne
- [`docs/roadmap.md`](docs/roadmap.md) : progression pédagogique, options, étapes `v0.1` à `v0.6`
- [`docs/advanced.md`](docs/advanced.md) : évolutions avancées `v0.7` à `v1.0`

## Progression recommandée

1. `v0.1` minigrep propre
2. `v0.2` options simples
3. `v0.3` multi-fichiers
4. `v0.4` récursif + ignore
5. `v0.5` regex + fixed string
6. `v0.6` output avancé + tests d'intégration

## Philosophie

**MVP :** apprendre à écrire un outil propre.

**Post-MVP :** apprendre à écrire un outil performant.

**v1.0 :** comprendre les compromis entre lisibilité, architecture, mémoire et vitesse.
