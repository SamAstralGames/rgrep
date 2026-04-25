# Objectif du projet

Créer une version plus poussée du `minigrep` de la documentation Rust, avec une architecture propre et des options proches d'un vrai `grep`.

## Noms possibles

- `rgrep`
- `pedigrep`
- `minigrep-plus`

## Commandes cibles

```sh
rgrep "TODO" src --ignore-case --line-number --recursive
rgrep "fn main" . -n -C 2 --glob "*.rs"
rgrep "error" logs --json --count
```

## Architecture proposée

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

## Principe idiomatique

`main.rs` ne fait presque rien. Toute la logique testable reste dans `lib.rs` et les modules.

```rust
fn main() {
    if let Err(err) = rgrep::run() {
        eprintln!("error: {err}");
        std::process::exit(1);
    }
}
```

## Parsing CLI

Tu peux commencer avec `std::env::args`, comme dans le Rust Book, pour comprendre les bases. La fonction retourne un itérateur sur les arguments, et le premier élément est traditionnellement le chemin du programme.

Mais pour une version plus poussée, je te conseille ensuite `clap`, car il permet une CLI propre avec `derive(Parser)`. La documentation officielle montre l'usage de `#[derive(Parser)]` avec `#[arg(short, long)]`, valeurs par défaut, `--help`, etc.

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

## Options pédagogiques à ajouter

### Niveau 1 - Base solide

```sh
rgrep pattern file
rgrep pattern file --ignore-case
rgrep pattern file --line-number
rgrep pattern file --count
rgrep pattern file --invert-match
```

**Compétences travaillées**

| Option | Compétence Rust |
| --- | --- |
| `--ignore-case` | transformation de chaînes, ownership |
| `--line-number` | `enumerate()` |
| `--count` | accumulateurs, itérateurs |
| `--invert-match` | booléens, logique métier |
| chemins multiples | `Vec<PathBuf>` |

### Niveau 2 - Recherche idiomatique

**Ajoute :**

- `--regex`
- `--fixed-string`
- `--word`
- `--whole-line`
- `--smart-case`

Pour le mode regex, utilise le crate `regex`. `RegexBuilder` permet de configurer dynamiquement des options comme le mode insensible à la casse.

**Design possible :**

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

**C'est pédagogique parce que tu travailles :**

- `enum`
- `impl`
- abstraction sans sur-ingénierie
- séparation entre configuration et comportement

### Niveau 3 - Fichiers et dossiers

**Ajoute :**

- `-r, --recursive`
- `--hidden`
- `--no-ignore`
- `--glob "*.rs"`
- `--exclude target`
- `--max-depth 3`

Pour le parcours récursif, le crate `ignore` est très intéressant : il fournit un itérateur récursif rapide qui respecte notamment `.gitignore`, les globs et les fichiers cachés selon configuration.

**Module dédié :**

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

**Compétences :**

- `PathBuf`
- `Result`
- itérateurs
- gestion des erreurs par fichier
- distinction fichier / dossier

### Niveau 4 - Sortie propre

**Ajoute :**

- `--color auto|always|never`
- `--json`
- `--files-with-matches`
- `--files-without-match`
- `--quiet`
- `--heading`
- `-A, --after-context 2`
- `-B, --before-context 2`
- `-C, --context 2`

**Modèle de données :**

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

Puis tu peux avoir plusieurs renderers :

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

Ne commence pas forcément avec un trait si tu n'en as pas encore besoin. Rust idiomatique ne veut pas dire « plein d'abstractions », mais abstractions au moment où elles deviennent utiles.

## Design recommandé

### 1. `Cli` != `Config`

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

C'est très bon pédagogiquement : tu apprends à séparer input brut et état valide.

### 2. Lire en streaming, pas avec `read_to_string`

Le `minigrep` officiel commence simplement avec la lecture complète du fichier, ce qui est très bien pour apprendre. Mais une version plus avancée devrait utiliser `BufRead`.

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

### 3. Préparer `stdin`

Ajoute plus tard :

```sh
cat file.txt | rgrep error
```

**Architecture :**

```rust
pub enum Input {
    Stdin,
    Files(Vec<PathBuf>),
}
```

**Compétences :**

- `std::io::stdin`
- lifetimes éventuellement
- `Box<dyn BufRead>` si tu veux apprendre le dispatch dynamique
- ou génériques si tu veux rester statique

## Roadmap pédagogique

### Étape 1 - Refaire `minigrep` proprement

**Objectif :**

```sh
rgrep pattern file
```

**Modules :**

- `main.rs`
- `lib.rs`
- `config.rs`
- `search.rs`

**À apprendre :**

- `Result`
- `?`
- tests unitaires
- séparation binaire / bibliothèque

### Étape 2 - Ajouter des options simples

**Options :**

- `-i`
- `-n`
- `-c`
- `-v`

**À apprendre :**

- `clap`
- `bool`
- `enumerate`
- logique conditionnelle propre

### Étape 3 - Multi-fichiers

```sh
rgrep TODO src/main.rs src/lib.rs
```

**À apprendre :**

- `Vec<PathBuf>`
- erreurs partielles
- continuer même si un fichier échoue

Décision importante : si un fichier échoue, le programme doit-il s'arrêter ou continuer ?

Je recommande : continuer, afficher l'erreur sur `stderr`, et retourner un code de sortie non zéro à la fin.

### Étape 4 - Dossiers récursifs

```sh
rgrep TODO src -r
```

**À apprendre :**

- parcours de fichiers
- `.gitignore`
- fichiers cachés
- dépendance `ignore`

### Étape 5 - Regex

```sh
rgrep "fn\s+\w+" src --regex
```

**À apprendre :**

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

**À apprendre :**

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

Pour une approche très pédagogique, tu peux introduire les dépendances dans cet ordre :

1. aucune dépendance
2. `clap`
3. `regex`
4. `ignore`
5. `serde_json`
6. `assert_cmd` / `tempfile`

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

## Ce que tu dois éviter

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

## Compétences Rust acquises à la fin

Avec ce projet, tu vas couvrir beaucoup plus que le `minigrep` de base :

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

Une belle version pédagogique pourrait supporter :

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

Mon conseil : construis-le en 6 petites versions, pas en une seule grosse passe. La meilleure progression serait :

- `v0.1` minigrep propre
- `v0.2` options simples
- `v0.3` multi-fichiers
- `v0.4` récursif + ignore
- `v0.5` regex + fixed string
- `v0.6` output avancé + tests d'intégration

### Post-MVP

**MVP pédagogique :**

- `v0.1` -> `v0.6`

**Post-MVP performance :**

- `v0.7` -> `v1.0+`

L'idée importante : ne pas optimiser trop tôt, mais préparer ton architecture pour pouvoir remplacer le moteur de recherche plus tard.

## Roadmap complète proposée

- `v0.1` minigrep propre
- `v0.2` options simples
- `v0.3` multi-fichiers
- `v0.4` récursif + ignore
- `v0.5` regex + fixed string
- `v0.6` output avancé + tests d'intégration
- `v0.7` moteur événementiel streaming
- `v0.8` lecture sans allocation ligne par ligne
- `v0.9` recherche sur bytes
- `v0.10` mmap pour gros fichiers
- `v0.11` parallélisation multi-fichiers
- `v0.12` gestion avancée du contexte
- `v0.13` benchmarks + profiling
- `v1.0` architecture stable

### v0.7 - Moteur événementiel streaming

À partir de là, tu changes un peu le cœur de l'architecture.

Avant :

```text
search_file() -> FileMatch
output.print(FileMatch)
```

Après :

```text
search_file() -> envoie des événements au fur et à mesure
```

Exemple :

```rust
pub trait SearchEventHandler {
    fn on_begin_file(&mut self, path: &Path);
    fn on_match(&mut self, event: MatchEvent<'_>);
    fn on_end_file(&mut self, path: &Path, stats: FileStats);
    fn on_error(&mut self, path: &Path, error: &SearchError);
}
```

Événement :

```rust
pub struct MatchEvent<'a> {
    pub path: &'a Path,
    pub line_number: usize,
    pub line: &'a str,
    pub ranges: Vec<Range<usize>>,
}
```

**Compétences acquises :**

- traits
- lifetimes
- callbacks
- séparation moteur / sortie
- architecture extensible

À ce stade, `TextOutput`, `JsonOutput`, `CountOutput`, `QuietOutput` peuvent tous implémenter le même handler.

### v0.8 - Lecture sans allocation ligne par ligne

La version simple utilise souvent :

```rust
for line in reader.lines() {
    let line = line?;
}
```

Mais `lines()` alloue une nouvelle `String` à chaque ligne.

Pour une version plus performante, tu passes à :

```rust
let mut line = String::new();

loop {
    line.clear();

    let bytes_read = reader.read_line(&mut line)?;
    if bytes_read == 0 {
        break;
    }

    // scanner line ici
}
```

Là, tu réutilises le même buffer.

C'est une excellente étape pédagogique parce que tu comprends mieux :

- allocation
- réutilisation de buffer
- durée de vie temporaire des références
- coût de `String`

Le modèle événementiel prend alors tout son sens :

```rust
handler.on_match(MatchEvent {
    path,
    line_number,
    line: &line,
    ranges,
});
```

Tu ne copies pas la ligne. Tu l'empruntes juste pendant l'appel.

### v0.9 - Recherche sur bytes

Pour aller vers les gros fichiers, tu peux commencer à raisonner non plus en `String`, mais en `&[u8]`.

Pourquoi ? Parce que tous les fichiers ne sont pas forcément UTF-8, et parce que travailler sur des bytes peut être plus rapide.

Tu pourrais avoir deux moteurs :

```rust
pub enum SearchMode {
    TextUtf8,
    Bytes,
}
```

Ou mieux :

```rust
pub trait SearchEngine {
    fn search_file(
        &self,
        path: &Path,
        handler: &mut dyn SearchEventHandler,
    ) -> Result<FileStats, SearchError>;
}
```

Puis :

- `Utf8LineEngine`
- `ByteLineEngine`
- `MmapEngine`

Pour le modèle byte :

```rust
pub struct ByteMatchEvent<'a> {
    pub path: &'a Path,
    pub line_number: usize,
    pub line: &'a [u8],
    pub ranges: Vec<Range<usize>>,
}
```

Mais ça complique l'output, car afficher du `&[u8]` demande de gérer :

- UTF-8 valide
- UTF-8 invalide
- lossy display
- fichiers binaires

Donc je mettrais ça après le MVP.

### v0.10 - `mmap` pour gros fichiers

Pour les très gros fichiers, tu peux ajouter un moteur basé sur memory mapping.

Architecture possible :

```text
SearchEngine
├── BufferedLineEngine
└── MmapEngine
```

Usage :

```sh
sgrep pattern huge.log --mmap
```

Ou en automatique :

```sh
sgrep pattern huge.log --engine auto
```

Options possibles :

- `--engine buffered`
- `--engine mmap`
- `--engine auto`

Avec `mmap`, ton fichier est vu comme un grand tableau de bytes :

```rust
&[u8]
```

Là, les offsets absolus deviennent beaucoup plus intéressants.

Tu peux avoir :

```rust
pub struct MmapMatchEvent<'a> {
    pub path: &'a Path,
    pub content: &'a [u8],
    pub line_number: usize,
    pub line_range: Range<usize>,
    pub match_ranges: Vec<Range<usize>>,
}
```

Ici, tu n'as plus besoin de stocker la ligne. Tu peux reconstruire :

```rust
let line = &content[event.line_range.clone()];
```

Donc oui, dans cette architecture, ton idée initiale :

```text
line_number + offsets
```

devient vraiment pertinente.

### v0.11 - Parallélisation multi-fichiers

Une fois que tu as le récursif et le multi-fichiers, tu peux paralléliser par fichier.

```text
fichier A -> thread 1
fichier B -> thread 2
fichier C -> thread 3
```

Mais attention : la difficulté n'est pas seulement la recherche. C'est surtout l'output.

Si plusieurs threads écrivent en même temps :

```text
src/a.rs:10:...
src/b.rs:2:...
src/a.rs:11:...
```

l'affichage peut devenir mélangé.

Tu peux proposer deux modes :

```sh
sgrep TODO src --threads 8
sgrep TODO src --threads 8 --sort path
```

Deux stratégies :

**Stratégie A - Output immédiat**

- plus rapide, mais ordre moins stable
- thread cherche
- thread envoie événement
- output affiche tout de suite avec `Mutex`

**Stratégie B - Output ordonné**

- plus propre, mais utilise plus de mémoire
- thread cherche
- résultat stocké par fichier
- main thread affiche dans l'ordre

C'est une excellente étape pour apprendre :

- threads
- channels
- `Mutex`
- `Arc`
- ordonnancement
- trade-off performance / déterminisme

### v0.12 - Contexte sans stocker tout le fichier

Pour gérer :

```sh
sgrep error file.log -A 3
sgrep error file.log -B 3
sgrep error file.log -C 3
```

Tu dois afficher des lignes autour du match.

Pour `--after-context`, c'est simple : après un match, tu affiches les `N` lignes suivantes.

Pour `--before-context`, il faut garder les `N` lignes précédentes.

Tu peux utiliser un buffer circulaire :

```rust
VecDeque<LineBuffer>
```

Exemple conceptuel :

```rust
let mut before = VecDeque::with_capacity(config.before_context);
```

Flux :

- lire ligne
- si pas match : stocker dans `before`
- si match : afficher `before`
- afficher match
- activer `after_context`

**Compétences :**

- `VecDeque`
- état interne
- streaming
- éviter de charger tout le fichier

C'est très intéressant pédagogiquement, car tu ajoutes une fonctionnalité avancée sans sacrifier le streaming.

### v0.13 - Benchmarks et profiling

Là tu arrêtes de deviner.

Tu ajoutes des benchmarks pour comparer :

- `read_to_string`
- `BufRead::lines`
- `read_line` avec buffer réutilisé
- `mmap`
- `regex`
- fixed string
- multi-thread

Scénarios de test :

- petit fichier, beaucoup de matchs
- gros fichier, peu de matchs
- gros fichier, aucun match
- beaucoup de petits fichiers
- fichier avec longues lignes
- fichier non UTF-8

Tu peux aussi générer des fixtures :

- `10 KB`
- `1 MB`
- `100 MB`
- `1 GB`

Objectif : apprendre à mesurer avant d'optimiser.

À cette étape, tu peux produire un tableau du genre :

```text
engine file size time allocations
buffered lines 100 MB ...
reused buffer 100 MB ...
mmap 100 MB ...
parallel src/ ...
```

### v1.0 - Architecture stable

À la fin, tu pourrais avoir une architecture comme ça :

```text
src/
├── main.rs
├── lib.rs
├── cli.rs
├── config.rs
├── error.rs
├── model.rs
├── matcher/
│   ├── mod.rs
│   ├── fixed.rs
│   ├── regex.rs
│   └── bytes.rs
├── engine/
│   ├── mod.rs
│   ├── buffered.rs
│   ├── mmap.rs
│   └── parallel.rs
├── walker/
│   ├── mod.rs
│   └── ignore.rs
├── output/
│   ├── mod.rs
│   ├── text.rs
│   ├── json.rs
│   ├── count.rs
│   └── quiet.rs
└── context.rs
```

Avec des concepts clairs :

- `Cli` -> ce que l'utilisateur tape
- `Config` -> configuration validée
- `Walker` -> trouve les fichiers
- `Matcher` -> trouve les occurrences dans une ligne ou un buffer
- `Engine` -> lit les fichiers et produit des événements
- `Output` -> consomme les événements

Très bonne séparation finale.

Tu peux viser cette séparation :

```rust
pub trait Matcher {
    fn find_ranges(&self, haystack: &str) -> Vec<Range<usize>>;
}
```

Puis plus tard :

```rust
pub trait ByteMatcher {
    fn find_ranges(&self, haystack: &[u8]) -> Vec<Range<usize>>;
}
```

Et côté moteur :

```rust
pub trait SearchEngine {
    fn search(
        &self,
        path: &Path,
        matcher: &dyn Matcher,
        handler: &mut dyn SearchEventHandler,
    ) -> Result<FileStats, SearchError>;
}
```

Le moteur ne sait pas s'il affiche en texte, JSON ou count.

Le matcher ne sait pas d'où vient le texte.

L'output ne sait pas comment le fichier a été lu.

C'est exactement ce que tu veux pour un projet pédagogique propre.

## Roadmap finale que je choisirais

- `v0.1` minigrep propre - un pattern - un fichier - tests unitaires
- `v0.2` options simples - ignore-case - line-number - count - invert-match
- `v0.3` multi-fichiers - plusieurs paths - erreurs par fichier - code de sortie propre
- `v0.4` récursif + ignore - dossiers - hidden files - ignore patterns - max-depth
- `v0.5` regex + fixed string - matcher abstrait - regex invalide - fixed-string explicite
- `v0.6` output avancé + tests d'intégration - color - json - files-with-matches - quiet - assert_cmd / fixtures
- `v0.7` événementiel streaming - `SearchEventHandler` - `MatchEvent<'a>` - output découplé du search
- `v0.8` buffers réutilisés - `read_line` - moins d'allocations - lifetimes plus strictes
- `v0.9` byte mode - `&[u8]` - fichiers non UTF-8 - binary detection
- `v0.10` mmap engine - offsets absolus - `line_range` - gros fichiers
- `v0.11` parallel engine - threads - channels - output ordonné ou rapide
- `v0.12` context streaming - `-A / -B / -C` - `VecDeque` - pas de chargement complet
- `v0.13` benchmark suite - mesures - comparaison moteurs - profiling
- `v1.0` stabilisation - API propre - docs - README - exemples

## Philosophie

Et pour résumer :

**MVP :**

apprendre à écrire un outil propre.

**Post-MVP :**

apprendre à écrire un outil performant.

**v1.0 :**

comprendre les compromis entre lisibilité, architecture, mémoire et vitesse.
