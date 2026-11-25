---
name: pumpkin-plugin
description: Écrire des plugins Pumpkin (serveur Minecraft en Rust) compilés en WebAssembly avec la crate `pumpkin-plugin-api`. À utiliser pour initialiser un plugin ou ajouter une commande à un plugin existant.
---

# Plugins Pumpkin (Rust + WASM)

Point d'entrée du skill. Lis d'abord ce fichier, puis ouvre uniquement le fichier qui correspond à la tâche.

## Principe général

- Un plugin est un **composant WebAssembly** (`.wasm`) chargé par le serveur depuis le dossier `./plugins`.
- On l'écrit en Rust avec la crate `pumpkin-plugin-api` (licence MIT OR Apache-2.0, indépendante du code serveur en GPL-3.0).
- Un plugin = un type qui implémente le trait `Plugin`, enregistré avec la macro `register_plugin!`.
- Les définitions de l'API vivent dans `crates/pumpkin-plugin-wit` (fichiers `.wit`, versions `v0.1` et `v0.2`). La crate `pumpkin-plugin-api` est le SDK Rust par-dessus.

## Fichiers de ce skill

| Fichier | Quand le lire |
|---|---|
| [initialiser-un-plugin.md](initialiser-un-plugin.md) | Créer un nouveau plugin : projet Cargo, métadonnées, `on_load`, compilation, installation. |
| [creer-une-commande.md](creer-une-commande.md) | Ajouter une commande (avec sous-commandes et arguments) à un plugin. |

## Règles à respecter

- Ne jamais copier de code du serveur (GPL-3.0) dans un plugin : on passe uniquement par `pumpkin-plugin-api`.
- Si le code du dépôt et ces fichiers divergent, le code du dépôt a raison : corrige le fichier du skill.
- Chaque fichier signale les points non vérifiés dans une section « À vérifier ».
