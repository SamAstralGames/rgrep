# Advanced

## Roadmap complète proposée

- `v0.7` moteur événementiel streaming
- `v0.8` lecture sans allocation ligne par ligne
- `v0.9` recherche sur bytes
- `v0.10` `mmap` pour gros fichiers
- `v0.11` parallélisation multi-fichiers
- `v0.12` gestion avancée du contexte
- `v0.13` benchmarks + profiling
- `v1.0` architecture stable

## v0.7 - Moteur événementiel streaming

À partir de là, le cœur de l'architecture change un peu.

Avant :

```text
search_file() -> FileMatch
output.print(FileMatch)
```

Après :

```text
search_file() -> envoie des événements au fur et à mesure
```

```rust
pub trait SearchEventHandler {
    fn on_begin_file(&mut self, path: &Path);
    fn on_match(&mut self, event: MatchEvent<'_>);
    fn on_end_file(&mut self, path: &Path, stats: FileStats);
    fn on_error(&mut self, path: &Path, error: &SearchError);
}
```

```rust
pub struct MatchEvent<'a> {
    pub path: &'a Path,
    pub line_number: usize,
    pub line: &'a str,
    pub ranges: Vec<Range<usize>>,
}
```

Compétences acquises :

- traits
- lifetimes
- callbacks
- séparation moteur / sortie
- architecture extensible

À ce stade, `TextOutput`, `JsonOutput`, `CountOutput` et `QuietOutput` peuvent tous implémenter le même handler.

## v0.8 - Lecture sans allocation ligne par ligne

La version simple utilise souvent :

```rust
for line in reader.lines() {
    let line = line?;
}
```

Mais `lines()` alloue une nouvelle `String` à chaque ligne.

Version plus performante :

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

On réutilise le même buffer, ce qui permet de mieux comprendre :

- allocation
- réutilisation de buffer
- durée de vie temporaire des références
- coût de `String`

Le modèle événementiel devient alors naturel :

```rust
handler.on_match(MatchEvent {
    path,
    line_number,
    line: &line,
    ranges,
});
```

## v0.9 - Recherche sur bytes

Pour aller vers les gros fichiers, tu peux commencer à raisonner en `&[u8]` plutôt qu'en `String`.

Pourquoi :

- tous les fichiers ne sont pas forcément UTF-8
- travailler sur des bytes peut être plus rapide

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

```rust
pub struct ByteMatchEvent<'a> {
    pub path: &'a Path,
    pub line_number: usize,
    pub line: &'a [u8],
    pub ranges: Vec<Range<usize>>,
}
```

Cela complique l'output, car il faut gérer :

- UTF-8 valide
- UTF-8 invalide
- lossy display
- fichiers binaires

Donc c'est plutôt une étape post-MVP.

## v0.10 - `mmap` pour gros fichiers

Pour les très gros fichiers, tu peux ajouter un moteur basé sur memory mapping.

```text
SearchEngine
├── BufferedLineEngine
└── MmapEngine
```

Usage :

```sh
sgrep pattern huge.log --mmap
sgrep pattern huge.log --engine auto
```

Options possibles :

- `--engine buffered`
- `--engine mmap`
- `--engine auto`

Avec `mmap`, le fichier devient un grand tableau de bytes :

```rust
&[u8]
```

Les offsets absolus deviennent alors très utiles.

```rust
pub struct MmapMatchEvent<'a> {
    pub path: &'a Path,
    pub content: &'a [u8],
    pub line_number: usize,
    pub line_range: Range<usize>,
    pub match_ranges: Vec<Range<usize>>,
}
```

La ligne peut être reconstruite à la demande :

```rust
let line = &content[event.line_range.clone()];
```

## v0.11 - Parallélisation multi-fichiers

Une fois le récursif et le multi-fichiers en place, tu peux paralléliser par fichier.

```text
fichier A -> thread 1
fichier B -> thread 2
fichier C -> thread 3
```

Le vrai sujet n'est pas seulement la recherche, mais l'output.

Si plusieurs threads écrivent en même temps :

```text
src/a.rs:10:...
src/b.rs:2:...
src/a.rs:11:...
```

l'affichage peut devenir mélangé.

Deux modes possibles :

```sh
sgrep TODO src --threads 8
sgrep TODO src --threads 8 --sort path
```

**Stratégie A - Output immédiat**

- plus rapide, mais ordre moins stable
- thread cherche
- thread envoie événement
- output affiche tout de suite avec `Mutex`

**Stratégie B - Output ordonné**

- plus propre, mais plus de mémoire
- thread cherche
- résultat stocké par fichier
- main thread affiche dans l'ordre

Compétences :

- threads
- channels
- `Mutex`
- `Arc`
- ordonnancement
- trade-off performance / déterminisme

## v0.12 - Contexte sans stocker tout le fichier

Pour gérer :

```sh
sgrep error file.log -A 3
sgrep error file.log -B 3
sgrep error file.log -C 3
```

Il faut afficher des lignes autour du match.

Pour `--after-context`, c'est simple : après un match, afficher les `N` lignes suivantes.

Pour `--before-context`, il faut garder les `N` lignes précédentes.

```rust
VecDeque<LineBuffer>
```

```rust
let mut before = VecDeque::with_capacity(config.before_context);
```

Flux conceptuel :

- lire ligne
- si pas match : stocker dans `before`
- si match : afficher `before`
- afficher match
- activer `after_context`

Compétences :

- `VecDeque`
- état interne
- streaming
- éviter de charger tout le fichier

## v0.13 - Benchmarks et profiling

Ici, tu arrêtes de deviner.

Tu peux comparer :

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

Fixtures possibles :

- `10 KB`
- `1 MB`
- `100 MB`
- `1 GB`

Exemple de tableau de résultats :

```text
engine file size time allocations
buffered lines 100 MB ...
reused buffer 100 MB ...
mmap 100 MB ...
parallel src/ ...
```

## v1.0 - Architecture stable

Architecture cible possible :

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

Concepts clairs :

- `Cli` -> ce que l'utilisateur tape
- `Config` -> configuration validée
- `Walker` -> trouve les fichiers
- `Matcher` -> trouve les occurrences dans une ligne ou un buffer
- `Engine` -> lit les fichiers et produit des événements
- `Output` -> consomme les événements

Séparation visée :

```rust
pub trait Matcher {
    fn find_ranges(&self, haystack: &str) -> Vec<Range<usize>>;
}
```

```rust
pub trait ByteMatcher {
    fn find_ranges(&self, haystack: &[u8]) -> Vec<Range<usize>>;
}
```

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
