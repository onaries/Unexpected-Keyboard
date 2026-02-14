# AGENTS.md — Unexpected Keyboard

Android virtual keyboard app. Java, Gradle (Kotlin DSL), no Kotlin source code.

## Build Commands

```sh
./gradlew assembleDebug          # Build debug APK → build/outputs/apk/debug/app-debug.apk
./gradlew assembleRelease        # Build release APK (requires signing env vars)
./gradlew installDebug           # Build + install on connected device via adb
```

Requires: OpenJDK 17, Android SDK (build tools ≥28.0.1, platform 35).
Nix users: `nix-shell ./shell.nix` provides the full environment.

## Test Commands

```sh
./gradlew test                                              # Run all unit tests
./gradlew test --tests "juloo.keyboard2.KeyValueTest"       # Single test class
./gradlew test --tests "juloo.keyboard2.ComposeKeyTest.fnEquals"  # Single test method
```

Test framework: **JUnit 4**. Tests live in `test/juloo.keyboard2/`.
Tests auto-depend on `genLayoutsList`, `checkKeyboardLayouts`, `compileComposeSequences`, `genMethodXml`.

## Code Generation (requires Python 3)

```sh
./gradlew genLayoutsList           # Regenerate res/values/layouts.xml from srcs/layouts/
./gradlew checkKeyboardLayouts     # Validate layouts → updates check_layout.output
./gradlew compileComposeSequences  # Regenerate ComposeKeyData.java from srcs/compose/
./gradlew genMethodXml             # Regenerate res/xml/method.xml
./gradlew genEmojis                # Regenerate res/raw/emojis.txt
```

After running generators, commit any changed output files (`check_layout.output`, etc.).

## CI (GitHub Actions)

- **make-apk.yml**: Builds debug APK on every push/PR. Uses JDK 17.
- **check-layouts.yml**: Runs `gen_layouts.py` and `check_layout.py`, then `git diff --exit-code`.
  If generated files are stale, CI fails. Run the generators locally and commit.

## Project Structure

```
srcs/juloo.keyboard2/          # Main Java sources (package: juloo.keyboard2)
srcs/juloo.keyboard2/dict/     # Dictionary support
srcs/juloo.keyboard2/prefs/    # Preference UI components
srcs/juloo.keyboard2/suggestions/  # Autocomplete/suggestions
srcs/layouts/                  # Keyboard layout XML definitions
srcs/compose/                  # Compose sequence data (JSON + Python compiler)
srcs/special_font/             # SVG sources for the special font
test/juloo.keyboard2/          # JUnit 4 tests
vendor/cdict/                  # Git submodule — C dictionary library with JNI
res/                           # Android resources (layouts, strings, XML)
build.gradle.kts               # Single-module Gradle build (Kotlin DSL)
```

Source dirs are non-standard — configured in `build.gradle.kts` `sourceSets`.
There is no `src/main/java` directory.

## Code Style

### Formatting

- **Indentation**: 2 spaces. No tabs.
- **Braces**: Allman style — opening brace on its own line for classes, methods, enums, and multi-line blocks.
- **Single-statement bodies**: No braces required for single-line `if`/`while`/`for`.
- **Line width**: No strict limit, but keep lines reasonable (~100 chars).
- No formatter config file exists. Match surrounding code exactly.

```java
public final class Example
{
  public void method_name()
  {
    if (condition)
      return single_statement;
    for (Item i : items)
      process(i);
    if (complex_condition)
    {
      do_thing_a();
      do_thing_b();
    }
  }
}
```

### Naming Conventions

| Element | Convention | Examples |
|---------|-----------|----------|
| Classes | PascalCase | `KeyValue`, `KeyboardData`, `ComposeKey` |
| Methods | snake_case | `get_gesture()`, `capitalize_string()`, `read_all_utf8()` |
| Private fields | `_prefixed` snake_case | `_prefs`, `_debug_logs`, `_key_pos` |
| Public fields | snake_case | `editor_config`, `swipe_dist_px`, `bottom_row` |
| Constants | UPPER_SNAKE_CASE | `FLAG_LATCH`, `ROTATION_THRESHOLD`, `TAG` |
| Enum values | Mixed — follow existing file | `SHIFT`, `CONFIG` or `Swiped`, `Rotating_clockwise` |
| Parameters | snake_case | `starting_direction`, `key_descr` |
| Local variables | snake_case | `prev_length`, `buff_length` |

**IMPORTANT**: Methods use `snake_case`, NOT Java-standard `camelCase`. Follow this project convention.

### Imports

- Use specific imports: `import java.util.HashMap;`
- Avoid wildcard imports. Exception: `android.view.*` is occasionally used.
- Group order: android, java, org, then project (`juloo.*`). No blank lines between groups.

### Classes and Access

- Mark utility/data classes `final`.
- Use package-private (no modifier) for internal methods — not everything needs `private`.
- Prefer concrete classes over interfaces/abstractions.
- Inner classes and enums are common within their parent class.

### Null Handling

- Document nullable fields/returns with `/** Might be null. */` or `/** [null] means ... */`.
- Check null inline: `if (x == null) return;` — no Optional, no annotations.

### Error Handling

- Use `Logs.exn(msg, exception)` for logging exceptions (wraps `Log.e`).
- Use `Logs.debug(msg)` for debug logging (only prints when debug mode is on).
- Custom exceptions (e.g., `KeyValueParser.ParseError`) extend `Exception`.
- Fail fast — throw exceptions, don't silently swallow errors.

### Comments

- `/** Javadoc */` on public API and fields. Can use `[]` to reference other symbols: `/** See [KeyValue.Kind]. */`
- `/* Block comments */` for implementation notes and section headers.
- `//` for inline clarifications.
- Comments in this codebase use `[]` brackets for code references (OCaml-doc style), not `{@link}`.

### Tests

- One test class per source class: `KeyValueTest`, `ComposeKeyTest`, etc.
- Tests use a nested `static class Utils` for shared assertion helpers.
- Test classes have an explicit no-arg constructor: `public FooTest() {}`
- Method names: `snake_case` or short descriptive names.

```java
public class FooTest
{
  public FooTest() {}

  @Test
  public void some_behavior() throws Exception
  {
    assertEquals(expected, actual);
  }

  static class Utils
  {
    static void check(String input, KeyValue expected) throws Exception
    {
      assertEquals(expected, SomeClass.parse(input));
    }
  }
}
```

### Layout XML Files

- Layouts live in `srcs/layouts/`, named as `{script}_{layout}_{country}.xml`.
- Do NOT edit `res/values/layouts.xml` directly — it is generated.
- After adding/modifying layouts, run `./gradlew genLayoutsList checkKeyboardLayouts`.
- See `srcs/layouts/latn_qwerty_us.xml` as the canonical reference.

### Generated Files — Do Not Edit

These files are generated. Edit their sources instead:
- `res/values/layouts.xml` — generated by `gen_layouts.py`
- `srcs/juloo.keyboard2/ComposeKeyData.java` — generated by `srcs/compose/compile.py`
- `res/xml/method.xml` — generated by `gen_method_xml.py`
- `res/raw/emojis.txt` — generated by `gen_emoji.py`
- `check_layout.output` — generated by `check_layout.py`
