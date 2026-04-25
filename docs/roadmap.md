# Roadmap

## Options pédagogiques à ajouter

### Niveau 1 - Base solide

```sh
rgrep pattern file
rgrep pattern file --ignore-case
rgrep pattern file --line-number
rgrep pattern file --count
rgrep pattern file --invert-match
```

| Option | Compétence Rust |
| --- | --- |
| `--ignore-case` | transformation de chaînes, ownership |
| `--line-number` | `enumerate()` |
| `--count` | accumulateurs, itérateurs |
| `--invert-match` | booléens, logique métier |
| chemins multiples | `Vec<PathBuf>` |

### Niveau 2 - Recherche idiomatique

Ajoute :

- `--regex`
- `--fixed-string`
- `--word`
- `--whole-line`
- `--smart-case`

Pour le mode regex, le crate `regex` est le bon point d'entrée. `RegexBuilder` permet de configurer dynamiquement des options comme l'insensibilité à la casse.

Design possible :

```rust
pub enum Matcher {
    Plain(PlainMatcher),
    Regex(RegexMatcher),
}

impl Matcher {
    pub fn is_match(&self, line: &str) -> bool {
        match self {
            Self::Plain(matcher) => matcher.is_match(line),
            Self::Regex(matcher) => matcher.is_match(line),
        }
    }
}
```

Tu travailles alors :

- `enum`
- `impl`
- abstraction sans sur-ingénierie
- séparation entre configuration et comportement

### Niveau 3 - Fichiers et dossiers

Ajoute :

- `-r, --recursive`
- `--hidden`
- `--no-ignore`
- `--glob "*.rs"`
- `--exclude target`
- `--max-depth 3`

Pour le parcours récursif, le crate `ignore` est très intéressant : il fournit un itérateur récursif rapide qui respecte `.gitignore`, les globs et les fichiers cachés selon configuration.

```rust
pub struct FileWalker {
    config: WalkConfig,
}

impl FileWalker {
    pub fn walk(&self, paths: &[PathBuf]) -> impl Iterator<Item = Result<PathBuf, WalkError>> {
        // ignore::WalkBuilder ici
    }
}
```

Compétences :

- `PathBuf`
- `Result`
- itérateurs
- gestion des erreurs par fichier
- distinction fichier / dossier

### Niveau 4 - Sortie propre

Ajoute :

- `--color auto|always|never`
- `--json`
- `--files-with-matches`
- `--files-without-match`
- `--quiet`
- `--heading`
- `-A, --after-context 2`
- `-B, --before-context 2`
- `-C, --context 2`

Modèle de données :

```rust
pub struct Match {
    pub path: PathBuf,
    pub line_number: usize,
    pub line: String,
    pub ranges: Vec<MatchRange>,
}

pub struct MatchRange {
    pub start: usize,
    pub end: usize,
}
```

Plusieurs renderers sont possibles :

```rust
pub trait Output {
    fn print_match(&mut self, item: &Match) -> Result<(), OutputError>;
}
```

Ou plus simplement :

```rust
pub enum OutputFormat {
    Text,
    Json,
}
```

Ne commence pas forcément avec un trait si tu n'en as pas encore besoin.

## Roadmap pédagogique

### Étape 1 - Refaire `minigrep` proprement

Objectif :

```sh
rgrep pattern file
```

Modules :

- `main.rs`
- `lib.rs`
- `config.rs`
- `search.rs`

À apprendre :

- `Result`
- `?`
- tests unitaires
- séparation binaire / bibliothèque

### Étape 2 - Ajouter des options simples

Options :

- `-i`
- `-n`
- `-c`
- `-v`

À apprendre :

- `clap`
- `bool`
- `enumerate`
- logique conditionnelle propre

### Étape 3 - Multi-fichiers

```sh
rgrep TODO src/main.rs src/lib.rs
```

À apprendre :

- `Vec<PathBuf>`
- erreurs partielles
- continuer même si un fichier échoue

Décision importante : si un fichier échoue, le programme doit-il s'arrêter ou continuer ?

Je recommande : continuer, afficher l'erreur sur `stderr`, et retourner un code de sortie non zéro à la fin.

### Étape 4 - Dossiers récursifs

```sh
rgrep TODO src -r
```

À apprendre :

- parcours de fichiers
- `.gitignore`
- fichiers cachés
- dépendance `ignore`

### Étape 5 - Regex

```sh
rgrep "fn\s+\w+" src --regex
```

À apprendre :

- dépendance externe
- erreur de compilation regex
- builder pattern
- enums de stratégie

### Étape 6 - Sorties avancées

```sh
rgrep error logs --json
rgrep TODO src --files-with-matches
rgrep panic src -C 2
```

À apprendre :

- sérialisation avec `serde`
- modèles de données
- séparation recherche / présentation
- tests d'intégration

## Dépendances utiles

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
regex = "1"
ignore = "0.4"
thiserror = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[dev-dependencies]
assert_cmd = "2"
predicates = "3"
tempfile = "3"
```

Ordre d'introduction recommandé :

1. aucune dépendance
2. `clap`
3. `regex`
4. `ignore`
5. `serde_json`
6. `assert_cmd` / `tempfile`

## Compétences Rust acquises à la fin

| Sujet | Où tu l'apprends |
| --- | --- |
| Ownership / borrowing | passage de `Config`, `Matcher`, lignes |
| `Result` / `?` | lecture fichier, regex invalide |
| `Path` / `PathBuf` | gestion de fichiers |
| `BufRead` | lecture streaming |
| Itérateurs | lignes, fichiers, filtres |
| `enum` | mode regex/plain, output text/json |
| Traits | output ou matcher avancé |
| Tests unitaires | `search_reader` |
| Tests d'intégration | CLI complète |
| Crates externes | `clap`, `regex`, `ignore` |
| Architecture | séparation CLI / domaine / I/O |
| UX terminal | couleurs, codes de sortie, `stderr` |

## Version finale visée

```text
rgrep <pattern> [paths...]

Options:
-i, --ignore-case
-s, --smart-case
-n, --line-number
-c, --count
-v, --invert-match
-r, --recursive
--regex
-F, --fixed-string
-w, --word
-x, --whole-line
--glob <GLOB>
--exclude <PATTERN>
--hidden
--no-ignore
--json
--color <auto|always|never>
-A, --after-context <N>
-B, --before-context <N>
-C, --context <N>
--files-with-matches
--files-without-match
-q, --quiet
```

Progression recommandée :

1. `v0.1` minigrep propre
2. `v0.2` options simples
3. `v0.3` multi-fichiers
4. `v0.4` récursif + ignore
5. `v0.5` regex + fixed string
6. `v0.6` output avancé + tests d'intégration

Pour les évolutions avancées au-delà du MVP, voir [`docs/advanced.md`](advanced.md).
