# Créer une commande dans un plugin Pumpkin

Prérequis : un plugin initialisé (voir [initialiser-un-plugin.md](initialiser-un-plugin.md)). On enregistre les commandes dans `on_load`, avec le `Context`.

## 1. Les briques de l'API

Tout vient de `pumpkin_plugin_api` (modules `command`, `commands`, `text`) :

| Élément | Rôle |
|---|---|
| `Command::new(&names, description)` | Commande racine. Le premier nom est le nom principal, les suivants sont des alias. |
| `CommandNode::literal(nom)` | Sous-commande fixe (ex. `/hello reload`). |
| `CommandNode::argument(nom, type)` | Argument typé (`ArgumentType`). |
| `.then(node)` | Attache un nœud enfant. |
| `.execute(handler)` | Attache le code à exécuter (sur la racine ou sur un nœud). |
| `trait CommandHandler` | `fn handle(&self, sender, server, args) -> Result<i32, CommandError>`. |
| `context.register_command(commande, permission)` | Enregistre la commande avec le nœud de permission requis. |

Les méthodes `then`, `execute` et `suggest` renvoient `Self` : on les enchaîne.

## 2. Commande simple

```rust
use pumpkin_plugin_api::{
    Context, Plugin, PluginMetadata, Result, Server, register_plugin,
    command::{Command, CommandError, CommandSender, ConsumedArgs},
    commands::CommandHandler,
    text::TextComponent,
};

struct HelloHandler;

impl CommandHandler for HelloHandler {
    fn handle(
        &self,
        sender: CommandSender,
        _server: Server,
        _args: ConsumedArgs,
    ) -> Result<i32, CommandError> {
        sender.send_message(TextComponent::text("Salut depuis mon plugin !"));
        Ok(0) // code de sortie : 0 = succès
    }
}

struct MonPlugin;

impl Plugin for MonPlugin {
    fn new() -> Self { MonPlugin }

    fn metadata(&self) -> PluginMetadata { /* voir initialiser-un-plugin.md */
        PluginMetadata {
            name: "mon-plugin".into(),
            version: "0.1.0".into(),
            authors: vec![],
            description: String::new(),
            dependencies: vec![],
            permissions: vec![],
        }
    }

    fn on_load(&self, context: Context) -> Result<()> {
        let hello = Command::new(
            &["hello".to_string(), "salut".to_string()], // "salut" est un alias
            "Dit bonjour",
        )
        .execute(HelloHandler);

        context.register_command(hello, "mon-plugin:command.hello");
        Ok(())
    }
}

register_plugin!(MonPlugin);
```

## 3. Sous-commande et argument

Arbre construit : `/hello say <message>`.

```rust
use pumpkin_plugin_api::command::{
    Arg, ArgumentType, CommandNode, StringType,
};

struct SayHandler;

impl CommandHandler for SayHandler {
    fn handle(
        &self,
        sender: CommandSender,
        _server: Server,
        args: ConsumedArgs,
    ) -> Result<i32, CommandError> {
        // Lire l'argument par son nom.
        let Arg::Simple(texte) = args.get_value("message") else {
            return Err(CommandError::InvalidConsumption(Some("message".into())));
        };
        sender.send_message(TextComponent::text(&texte));
        Ok(0)
    }
}

let say = CommandNode::literal("say").then(
    CommandNode::argument("message", &ArgumentType::String(StringType::Greedy))
        .execute(SayHandler),
);

let hello = Command::new(&["hello".to_string()], "Dit bonjour")
    .execute(HelloHandler) // `/hello` seul
    .then(say);            // `/hello say <message>`
```

Règle : le handler est exécuté sur le **dernier nœud** reconnu. Mets donc `.execute(...)` sur chaque nœud qui doit pouvoir terminer la commande.

## 4. Erreurs et résultat

`CommandError` (défini dans `command.wit`) :

- `InvalidConsumption(Option<String>)` : argument manquant ou mal lu.
- `InvalidRequirement` : condition d'exécution non remplie.
- `PermissionDenied` : refus de permission.
- `CommandFailed(TextComponent)` : échec avec un message affiché à l'expéditeur.

Exemple : `Err(CommandError::CommandFailed(TextComponent::text("Impossible.")))`.

## 5. Qui a exécuté la commande ?

`CommandSender` expose notamment : `is_player()`, `as_player()`, `is_console()`, `get_name()`, `send_message(...)`, `send_error(...)`, `has_permission(&server, "noeud")`, `permission_level()`, `position()`, `world()`.

## 6. Permissions

Le second paramètre de `register_command` est le nœud de permission exigé. Pour le déclarer au serveur, utilise `context.register_permission(Permission { node, description, default, children })` avant l'enregistrement de la commande. `default` vaut `Deny`, `Allow` ou `Op(niveau)` (niveaux 0 à 4).

## 7. Autres types d'arguments utiles

`ArgumentType` couvre entre autres : `Bool`, `Float`, `Double`, `Integer`, `Long`, `String(StringType)` (`SingleWord`, `Quotable`, `Greedy`), `Players`, `Entities`, `BlockPos`, `Position3d`, `Item`, `Gamemode`, `Time`, `Message`. Les valeurs lues arrivent dans l'enum `Arg` (`Simple`, `Msg`, `Bool`, `Num`, `Players`, `BlockPos`, `Pos3d`, `Gamemode`…).

Pour proposer des suggestions personnalisées sur un argument, implémente `CommandSuggestionHandler` et attache-le avec `CommandNode::suggest(...)`.

## À vérifier

- **Signatures générées** : `Command::new`, `CommandNode::literal/argument` et `send_message` sont générées par `wit-bindgen` à partir de `command.wit`. Les formes ci-dessus (par exemple `&[String]` pour les noms, `&ArgumentType` pour le type) sont déduites du WIT et non compilées. Compile le plugin et corrige selon les erreurs, ou lis les bindings générés.
- **Correspondance type d'argument → variante `Arg`** : par exemple `String(Greedy)` donne probablement `Arg::Simple` et `Message` donne `Arg::Msg`. À confirmer dans l'hôte (`crates/pumpkin-wasm-host-v0_1/src`).
- **Permission obligatoire ou non** avant `register_command` : le code hôte lit simplement la chaîne, sans vérifier qu'elle est déclarée. Déclarer la permission reste recommandé.
- Aucun exemple de plugin n'existe dans ce dépôt : tout ce fichier vient de la lecture de `pumpkin-plugin-api`, des fichiers `.wit` et de l'hôte.
