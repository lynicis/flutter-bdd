# Flutter BDD

Editor integration for the Dart package
[`bdd_widget_test`](https://pub.dev/packages/bdd_widget_test): Ctrl+Click any
Gherkin step in a `.feature` file to open the Dart step function it resolves to,
or scaffold the step file that does not exist yet.

Works in VS Code, VSCodium and Zed. Offline — no Dart tooling, generator run or
analyzer.

[![CI](https://github.com/lynicis/flutter-bdd/actions/workflows/ci.yml/badge.svg)](https://github.com/lynicis/flutter-bdd/actions/workflows/ci.yml)
[![Visual Studio Marketplace Version](https://vsmarketplacebadges.dev/version/lynicis.bdd-widget-test-helper.svg)](https://marketplace.visualstudio.com/items?itemName=lynicis.bdd-widget-test-helper)
[![Open VSX Version](https://img.shields.io/open-vsx/v/lynicis/bdd-widget-test-helper)](https://open-vsx.org/extension/lynicis/bdd-widget-test-helper)

Status: **0.2.x, pre-1.0, actively maintained.** Latest release:
[v0.2.2](https://github.com/lynicis/flutter-bdd/releases/tag/v0.2.2). The
extension installs as `lynicis.bdd-widget-test-helper` — the product name is
Flutter BDD.

> This resolves **`bdd_widget_test` naming only**. Projects that use a different
> step generator or runner (`flutter_gherkin`, custom step folders with their own
> naming) get no matches here — use their tooling instead.

## Install

VS Code 1.85 or newer.

```bash
code --install-extension lynicis.bdd-widget-test-helper
```

Or from the Extensions view: search **Flutter BDD** in the VS Code Marketplace,
or in Open VSX (which VSCodium uses by default). No configuration is needed; the
extension activates on `.feature` files.

For Zed, install the dev extension from a checkout of this repository:
`bun install && bun run build:zed`, then
**Extensions → Install Dev Extension → `zed/`**. Requirements and settings live
in the [Zed extension README](https://github.com/lynicis/flutter-bdd/blob/master/zed/README.md).

## Use it

1. Open any `.feature` file in a Flutter project that uses `bdd_widget_test`.
2. Put the cursor on a step line and press **F12**, or Ctrl/Cmd+Click it. The
   matching `step/*.dart` file opens at its `Future<void>` line.
3. On a step with no file yet, a warning appears. Open the lightbulb and choose
   **Create step file** — the content is what `bdd_widget_test` would write, with
   the same imports, parameter types and `/// Usage:` comment.

A Scenario Outline step and its generated file, both from
[`src/test/fixtures`](https://github.com/lynicis/flutter-bdd/tree/master/src/test/fixtures):

```gherkin
# test/outline.feature (excerpt)
Scenario Outline: eating
  Given there are <start> cucumbers
```

```dart
// test/step/there_are_cucumbers.dart
import 'package:flutter_test/flutter_test.dart';

/// Usage: there are <start> cucumbers
Future<void> thereAreCucumbers(WidgetTester tester, dynamic start) async {
  throw UnimplementedError();
}
```

The 20 built-in steps (`the app is running`, `I see {..} text`, …) resolve
without a file and are never reported missing. Commands: **Flutter BDD: Create
step file** (also the quick-fix) and **Flutter BDD: Rebuild step index**.

## Where it fits

- **Plain `bdd_widget_test`** generates step files at build time and gives the
  editor no way to navigate them. This adds the editor side and scaffolds the
  missing files; your build and generated code stay untouched.
- **Generic Gherkin or Cucumber extensions** highlight `.feature` files and
  resolve Cucumber-family step definitions. They do not implement
  `bdd_widget_test`'s naming — `I tap text` → `step/i_tap_text.dart`, function
  `iTapText` — so they will not find Dart steps.
- **Searching by hand** works, but knows nothing about `bdd_options.yaml`,
  `externalSteps` or built-in steps.

What it deliberately does not do: run tests, resolve other step generators, or
reach the network. Only the open-source `vscode` API is used, which is what keeps
VSCodium working.

## How a step resolves

Resolution order mirrors `bdd_widget_test`:

1. The 20 built-in steps — always built-ins, so a local `i_see_text.dart` is
   _not_ used for `I see {..} text`.
2. The `step/` folder for the feature (recursive; the nearest match wins).
3. `externalSteps` entries such as `package:my_lib/step/i_tap_text.dart`,
   resolved through `.dart_tool/package_config.json`.

The step folder and remaining options come from `bdd_options.yaml` (the nearest
file walking up to the workspace root), then `build.yaml`
(`targets → … → builders → bdd_widget_test → options`), then these settings:

| Setting                                          | Default  | Meaning                                        |
| ------------------------------------------------ | -------- | ---------------------------------------------- |
| `bddWidgetTestHelper.stepFolderName`             | `./step` | Step folder location                           |
| `bddWidgetTestHelper.relativeToTestFolder`       | `true`   | Interpret non-`./` folders relative to `test/` |
| `bddWidgetTestHelper.externalSteps`              | `[]`     | Extra `package:` step locations                |
| `bddWidgetTestHelper.showUnresolvedStepWarnings` | `true`   | Warn on unresolved steps                       |

Names are ported line by line from the generator, not guessed: `{...}` parameters
are removed from the name and typed `String` / `num` / `bool` / `dynamic`;
Scenario Outline `<placeholders>` become `dynamic`; accented Latin folds to ASCII
(`çalışıyor` → `calisiyor`); a data table adds `bdd.DataTable dataTable`; and a
step that cannot form a Dart identifier (non-Latin script, leading digit or
underscore) gets a rename warning instead of a broken file.

## Gherkin details

- `And` / `But` / `*` inherit the previous step keyword, as Gherkin specifies.
- `Background:` steps resolve like any other step.
- `Examples:` rows are never mistaken for steps; `<param>` placeholders map to
  `dynamic` parameters in scaffolds.
- Steps inside docstrings (triple-quoted or triple-backtick fences) and `#`
  comments are ignored.
- All 80 official Gherkin dialects are supported, including every
  `# language:` header.
- Diagnostics recompute as you type and when step files change; the step index is
  built once and kept current through a file watcher.

Zed has no document-link API, so navigation there is go-to-definition only, and
the index rebuilds on open/save rather than on a file watcher. Details:
[Zed extension README](https://github.com/lynicis/flutter-bdd/blob/master/zed/README.md).

## Development

Bun 1.4 and Node 26 (CI pins both). `bun install && bun run compile` builds the
bundle; `bun run test:unit` runs the Vitest suite without VS Code. The full
command list, integration-test setup and file map are in
[CONTRIBUTING.md](https://github.com/lynicis/flutter-bdd/blob/master/CONTRIBUTING.md).

`src/naming.ts` and the option-lookup code are a deliberate port of
`bdd_widget_test`; read the port-fidelity rule in `CONTRIBUTING.md` before
changing them. `src/naming.ts`, `bddOptions.ts`, `stepIndex.ts`,
`stepResolver.ts`, `featureParser.ts` and `simpleYaml.ts` must stay free of
`vscode` imports — they also back the Zed server.

Security reporting and community standards:
[SECURITY.md](https://github.com/lynicis/flutter-bdd/blob/master/SECURITY.md),
[CODE_OF_CONDUCT.md](https://github.com/lynicis/flutter-bdd/blob/master/CODE_OF_CONDUCT.md).
Release history: [CHANGELOG.md](https://github.com/lynicis/flutter-bdd/blob/master/CHANGELOG.md).

## License

MIT — see [LICENSE](https://github.com/lynicis/flutter-bdd/blob/master/LICENSE).
