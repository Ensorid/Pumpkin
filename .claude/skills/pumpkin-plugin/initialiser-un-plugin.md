# Initialiser un plugin Pumpkin (Rust + WASM)

## 1. Ce que le serveur attend

- Un fichier `.wasm` au format **component model** placé dans `./plugins/` (à côté de l'exécutable du serveur). Le dossier est créé s'il n'existe pas, et le serveur surveille les changements de fichiers `.wasm` pour le rechargement à chaud.
- Le serveur détecte la version de l'API (`v0.1` ou `v0.2`) depuis le composant. La crate `pumpkin-plugin-api` actuelle génère ses bindings sur **v0.1**. L'hôte v0.2 est désactivé par défaut (feature `plugin-v0_2` du binaire `pumpkin`).
- Le plugin exporte `init-plugin` (généré par `register_plugin!`), puis le serveur appelle `metadata`, puis `on-load`.

## 2. Créer le projet

```bash
cargo new --lib mon-plugin
cd mon-plugin
rustup target add wasm32-wasip2
```

`Cargo.toml` :

```toml
[package]
name = "mon-plugin"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
pumpkin-plugin-api = { git = "https://github.com/Pumpkin-MC/Pumpkin" }
tracing = "0.1"
```

Notes :
- `crate-type = ["cdylib"]` est nécessaire pour produire un module WASM.
- Pour tester contre ton fork, remplace l'URL par `https://github.com/Ensorid/pumpkin` (ajoute `branch = "..."` si besoin), ou utilise `path = "../pumpkin/crates/pumpkin-plugin-api"` en local.
- `tracing` sert à journaliser : `register_plugin` installe un abonné qui renvoie les logs au serveur.

## 3. Code minimal (`src/lib.rs`)

```rust
use pumpkin_plugin_api::{
    Context, Plugin, PluginMetadata, Result, register_plugin,
};

struct MonPlugin;

impl Plugin for MonPlugin {
    // Appelé une seule fois avant on_load.
    fn new() -> Self {
        MonPlugin
    }

    fn metadata(&self) -> PluginMetadata {
        PluginMetadata {
            name: "mon-plugin".into(),
            version: "0.1.0".into(),
            authors: vec!["ton-nom".into()],
            description: "Mon premier plugin Pumpkin.".into(),
            dependencies: vec![],
            // Permissions demandées à l'hôte (voir `pumpkin_plugin_api::permissions`).
            permissions: vec![],
        }
    }

    // C'est ici qu'on enregistre commandes, événements, permissions, etc.
    fn on_load(&self, _context: Context) -> Result<()> {
        tracing::info!("mon-plugin chargé");
        Ok(())
    }

    fn on_unload(&self, _context: Context) -> Result<()> {
        tracing::info!("mon-plugin déchargé");
        Ok(())
    }
}

register_plugin!(MonPlugin);
```

Points importants du trait `Plugin` :
- Le trait exige `Send + Sync` et les méthodes prennent `&self` : tout état modifiable passe par de la mutabilité intérieure thread-safe (`Mutex`, atomiques…).
- Ne garde pas un verrou non réentrant pendant un appel à l'hôte : l'appel peut redéclencher un événement ou un message IPC dans le même plugin avant de revenir.
- `register_plugin!` ne doit être appelée qu'une fois.

## 4. Permissions

`metadata().permissions` liste les accès demandés, sous forme de chaînes. Les constantes sont dans `pumpkin_plugin_api::permissions` :

- réseau : `NETWORK_DNS`, `NETWORK_TCP`, `NETWORK_OUTBOUND`, `HTTP_OUTBOUND`…
- fichiers du dossier de données du plugin : `FS_READ_DATA`, `FS_WRITE_DATA` (dossier `plugins/data/<nom>`)
- système : `SYS_ENV`, `SYS_INFO`…

Sans permission, la fonctionnalité correspondante n'est pas accessible. Demande le strict nécessaire.

Pour la persistance, il n'y a pas d'API de stockage dédiée : le plugin lit et écrit ses fichiers avec `std::fs`, dans le chemin renvoyé par `context.get_data_folder()` (à appeler dans `on_load` ou `on_unload`).

## 5. Compiler et installer

```bash
cargo build --release --target wasm32-wasip2
cp target/wasm32-wasip2/release/mon_plugin.wasm <dossier-du-serveur>/plugins/
```

Le nom du fichier produit utilise des underscores (`mon_plugin.wasm`). Au démarrage (ou à la copie, si le rechargement à chaud est actif), le serveur charge le plugin et journalise son nom et sa version.

## À vérifier

- **Commande de build** : le dépôt ne documente pas la commande exacte. La cible `wasm32-wasip2` produit directement un composant, mais l'API utilise des métadonnées `cargo-component` (`[package.metadata.component]`) et le chargeur reconnaît `cargo-component` parmi les outils. Si le serveur refuse le fichier (« ComponentNewFailed » ou « ApiMismatch »), essaie `cargo component build --release`.
- **Forme de la dépendance** : `pumpkin-plugin-api` est un membre du workspace du dépôt, pas confirmée comme publiée sur crates.io. Adapte `git`/`path` selon la source.
- **Valeur par défaut du rechargement à chaud** : le code surveille `./plugins` mais le drapeau `hot_reload_enabled` n'a pas été vérifié.
- **Exemple de la doc** : le commentaire de `lib.rs` montre `on_load(&self, context: &Context)`, alors que le trait déclare `on_load(&self, context: Context)`. Suis la signature du trait.
