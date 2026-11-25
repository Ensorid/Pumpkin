# Create a command in a Pumpkin plugin

Prerequisite: an initialized plugin (see [initialize-plugin.md](initialize-plugin.md)). Commands are registered in `on_load`, using the `Context`.

## 1. The building blocks of the API

Everything comes from `pumpkin_plugin_api` (modules `command`, `commands`, `text`):

| Item | Role |
|---|---|
| `Command::new(&names, description)` | Root command. The first name is the primary name, the rest are aliases. |
| `CommandNode::literal(name)` | Fixed subcommand (e.g. `/hello reload`). |
| `CommandNode::argument(name, type)` | Typed argument (`ArgumentType`). |
| `.then(node)` | Attaches a child node. |
| `.execute(handler)` | Attaches the code to run (on the root or on a node). |
| `trait CommandHandler` | `fn handle(&self, sender, server, args) -> Result<i32, CommandError>`. |
| `context.register_command(command, permission)` | Registers the command with the required permission node. |

The `then`, `execute` and `suggest` methods return `Self`, so they can be chained.

## 2. Simple command

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
        sender.send_message(TextComponent::text("Hello from my plugin!"));
        Ok(0) // exit code: 0 = success
    }
}

struct MyPlugin;

impl Plugin for MyPlugin {
    fn new() -> Self { MyPlugin }

    fn metadata(&self) -> PluginMetadata { /* see initialize-plugin.md */
        PluginMetadata {
            name: "my-plugin".into(),
            version: "0.1.0".into(),
            authors: vec![],
            description: String::new(),
            dependencies: vec![],
            permissions: vec![],
        }
    }

    fn on_load(&self, context: Context) -> Result<()> {
        let hello = Command::new(
            &["hello".to_string(), "hi".to_string()], // "hi" is an alias
            "Says hello",
        )
        .execute(HelloHandler);

        context.register_command(hello, "my-plugin:command.hello");
        Ok(())
    }
}

register_plugin!(MyPlugin);
```

## 3. Subcommand and argument

Resulting tree: `/hello say <message>`.

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
        // Read the argument by its name.
        let Arg::Simple(text) = args.get_value("message") else {
            return Err(CommandError::InvalidConsumption(Some("message".into())));
        };
        sender.send_message(TextComponent::text(&text));
        Ok(0)
    }
}

let say = CommandNode::literal("say").then(
    CommandNode::argument("message", &ArgumentType::String(StringType::Greedy))
        .execute(SayHandler),
);

let hello = Command::new(&["hello".to_string()], "Says hello")
    .execute(HelloHandler) // `/hello` alone
    .then(say);            // `/hello say <message>`
```

Rule: the handler runs on the **last node** that was matched. Put `.execute(...)` on every node that can end the command.

## 4. Errors and result

`CommandError` (defined in `command.wit`):

- `InvalidConsumption(Option<String>)`: missing or badly parsed argument.
- `InvalidRequirement`: an execution requirement is not met.
- `PermissionDenied`: permission refused.
- `CommandFailed(TextComponent)`: failure with a message shown to the sender.

Example: `Err(CommandError::CommandFailed(TextComponent::text("Not possible.")))`.

## 5. Who ran the command?

`CommandSender` exposes among others: `is_player()`, `as_player()`, `is_console()`, `get_name()`, `send_message(...)`, `send_error(...)`, `has_permission(&server, "node")`, `permission_level()`, `position()`, `world()`.

## 6. Permissions

The second parameter of `register_command` is the required permission node. To declare it to the server, use `context.register_permission(Permission { node, description, default, children })` before registering the command. `default` is `Deny`, `Allow` or `Op(level)` (levels 0 to 4).

## 7. Other useful argument types

`ArgumentType` covers, among others: `Bool`, `Float`, `Double`, `Integer`, `Long`, `String(StringType)` (`SingleWord`, `Quotable`, `Greedy`), `Players`, `Entities`, `BlockPos`, `Position3d`, `Item`, `Gamemode`, `Time`, `Message`. Parsed values arrive in the `Arg` enum (`Simple`, `Msg`, `Bool`, `Num`, `Players`, `BlockPos`, `Pos3d`, `Gamemode`, ...).

To offer custom suggestions on an argument, implement `CommandSuggestionHandler` and attach it with `CommandNode::suggest(...)`.

## To verify

- **Generated signatures**: `Command::new`, `CommandNode::literal/argument` and `send_message` are generated by `wit-bindgen` from `command.wit`. The forms above (for example `&[String]` for names, `&ArgumentType` for the type) are deduced from the WIT and were not compiled. Compile the plugin and fix according to the errors, or read the generated bindings.
- **Argument type to `Arg` variant mapping**: for example `String(Greedy)` probably gives `Arg::Simple` and `Message` gives `Arg::Msg`. Confirm in the host (`crates/pumpkin-wasm-host-v0_1/src`).
- **Whether the permission must be declared** before `register_command`: the host code simply reads the string without checking that it is declared. Declaring the permission is still recommended.
- No example plugin exists in this repository: this whole file comes from reading `pumpkin-plugin-api`, the `.wit` files and the host.
