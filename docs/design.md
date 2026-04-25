# Design

## Noms possibles

- `rgrep`
- `pedigrep`
- `minigrep-plus`

## Parsing CLI

Tu peux commencer avec `std::env::args`, comme dans le Rust Book, pour comprendre les bases. La fonction retourne un itérateur sur les arguments, et le premier élément est traditionnellement le chemin du programme.

Mais pour une version plus poussée, `clap` est plus adapté, avec `derive(Parser)`, `#[arg(short, long)]`, valeurs par défaut et `--help` généré proprement.

### Exemple

```rust
use clap::{Parser, ValueEnum};
use std::path::PathBuf;

#[derive(Debug, Parser)]
#[command(name = "rgrep", version, about = "A pedagogical grep clone in Rust")]
pub struct Cli {
    pub pattern: String,

    #[arg(default_value = ".")]
    pub paths: Vec<PathBuf>,

    #[arg(short = 'i', long)]
    pub ignore_case: bool,

    #[arg(short = 'n', long)]
    pub line_number: bool,

    #[arg(short = 'r', long)]
    pub recursive: bool,

    #[arg(short = 'v', long)]
    pub invert_match: bool,

    #[arg(short = 'c', long)]
    pub count: bool,

    #[arg(long, value_enum, default_value_t = MatchMode::Plain)]
    pub mode: MatchMode,

    #[arg(long)]
    pub json: bool,
}

#[derive(Debug, Clone, ValueEnum)]
pub enum MatchMode {
    Plain,
    Regex,
}
```

## Design recommandé

### `Cli` != `Config`

`Cli` représente ce que l'utilisateur a tapé.

`Config` représente une configuration validée que ton programme comprend.

```rust
pub struct Config {
    pub pattern: String,
    pub paths: Vec<PathBuf>,
    pub ignore_case: bool,
    pub line_number: bool,
    pub recursive: bool,
    pub mode: MatchMode,
}

impl TryFrom<Cli> for Config {
    type Error = ConfigError;

    fn try_from(cli: Cli) -> Result<Self, Self::Error> {
        if cli.pattern.is_empty() {
            return Err(ConfigError::EmptyPattern);
        }

        Ok(Self {
            pattern: cli.pattern,
            paths: cli.paths,
            ignore_case: cli.ignore_case,
            line_number: cli.line_number,
            recursive: cli.recursive,
            mode: cli.mode,
        })
    }
}
```

C'est un très bon exercice pour séparer input brut et état valide.

### Lire en streaming, pas avec `read_to_string`

Le `minigrep` officiel commence avec la lecture complète du fichier, ce qui est très bien pour apprendre. Mais une version plus avancée devrait utiliser `BufRead`.

```rust
use std::io::{self, BufRead};

pub fn search_reader<R: BufRead>(
    reader: R,
    matcher: &Matcher,
) -> io::Result<Vec<LineMatch>> {
    let mut matches = Vec::new();

    for (index, line) in reader.lines().enumerate() {
        let line = line?;

        if matcher.is_match(&line) {
            matches.push(LineMatch {
                line_number: index + 1,
                line,
            });
        }
    }

    Ok(matches)
}
```

Avantage : tu peux tester avec une chaîne en mémoire, un fichier, ou plus tard `stdin`.

### Préparer `stdin`

```sh
cat file.txt | rgrep error
```

Architecture possible :

```rust
pub enum Input {
    Stdin,
    Files(Vec<PathBuf>),
}
```

Compétences travaillées :

- `std::io::stdin`
- lifetimes éventuellement
- `Box<dyn BufRead>` pour apprendre le dispatch dynamique
- ou génériques pour rester statique

## Exemple d'API interne propre

```rust
pub fn run(config: Config) -> Result<RunSummary, AppError> {
    let matcher = Matcher::new(&config)?;
    let files = collect_files(&config)?;

    let mut summary = RunSummary::default();

    for path in files {
        match search_file(&path, &matcher, &config) {
            Ok(result) => {
                summary.add(result);
            }
            Err(err) => {
                summary.add_error(path, err);
            }
        }
    }

    Ok(summary)
}
```

Ce style est intéressant parce que :

- `run` orchestre
- `Matcher` cherche
- `walker` fournit les fichiers
- `output` affiche
- les erreurs sont typées
- chaque partie est testable

## Ce qu'il faut éviter

Évite une version où tout est dans `main.rs`.

Évite aussi :

- `unwrap()`
- `expect("ça marche")`
- `std::process::exit()` partout
- `String` pour tous les chemins
- `read_to_string` pour les gros fichiers
- `println!` directement dans la logique de recherche

Préférer :

- `Result<T, E>`
- `PathBuf`
- `BufRead`
- `eprintln!` pour les erreurs
- tests unitaires sur la logique pure
- tests d'intégration sur la CLI
