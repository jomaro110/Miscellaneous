# Iced Framework Arabic Text Support - Patch Documentation

## Overview

This document describes the patches made to the Iced GUI framework (v0.12) to enable proper Arabic text rendering in TextInput widgets. These patches are required because the default iced implementation does not correctly handle Right-to-Left (RTL) text rendering for Arabic characters.

## Problem Statement

Without these patches, iced's `TextInput` widget:
- Does not display Arabic characters correctly
- Fails to handle RTL text direction
- Does not properly shape Arabic glyphs (contextual forms)
- Has cursor positioning issues with bidirectional text

## Solution: RTL Hint Using Left-to-Right Mark (LRM)

The patch adds automatic RTL detection and inserts a **Left-to-Right Mark (LRM)** Unicode character (`U+200E`) at the beginning of text containing Arabic characters. This provides the text rendering engine with the necessary hint to properly shape and display RTL text.

## Patched Files

### 1. `vendor/iced_widget/src/text_input.rs`

**Location**: Lines 1498-1557

**Key Changes**:

#### Added Constants
```rust
const LRM: char = '\u{200E}';  // Left-to-Right Mark Unicode character
```

#### Added RTL Detection Functions

```rust
fn needs_rtl_hint(value: &str) -> bool {
    value
        .chars()
        .find(|ch| !ch.is_whitespace())
        .map(is_rtl_char)
        .unwrap_or(false)
}

fn is_rtl_char(ch: char) -> bool {
    const RTL_RANGES: &[(u32, u32)] = &[
        (0x0600, 0x06FF),  // Arabic
        (0x0750, 0x077F),  // Arabic Supplement
        (0x08A0, 0x08FF),  // Arabic Extended-A
        (0xFB50, 0xFDFF),  // Arabic Presentation Forms-A
        (0xFE70, 0xFEFF),  // Arabic Presentation Forms-B
    ];

    let code = ch as u32;
    RTL_RANGES
        .iter()
        .any(|(start, end)| code >= *start && code <= *end)
}
```

**Unicode Ranges Explained**:
- `0x0600-0x06FF`: Main Arabic block (letters, diacritics, punctuation)
- `0x0750-0x077F`: Arabic Supplement (additional letters for African languages)
- `0x08A0-0x08FF`: Arabic Extended-A (additional letters for Quranic text)
- `0xFB50-0xFDFF`: Arabic Presentation Forms-A (ligatures and contextual forms)
- `0xFE70-0xFEFF`: Arabic Presentation Forms-B (compatibility forms)

#### Modified VisualText Structure

```rust
#[derive(Debug, Clone, Copy, Default)]
struct DisplayOffset {
    bytes: usize,      // Byte offset for the LRM character
    graphemes: usize,  // Grapheme offset (always 1 for LRM)
}

struct VisualText {
    content: String,           // The displayed text (with LRM if needed)
    offset: DisplayOffset,     // Offset accounting for the LRM character
}

impl VisualText {
    fn new(value: &Value) -> Self {
        let raw = value.to_string();

        if needs_rtl_hint(&raw) {
            let mut rendered = String::with_capacity(raw.len() + LRM.len_utf8());
            rendered.push(LRM);       // Prepend LRM for RTL text
            rendered.push_str(&raw);
            Self {
                content: rendered,
                offset: DisplayOffset {
                    bytes: LRM.len_utf8(),  // 3 bytes for LRM
                    graphemes: 1,            // 1 grapheme cluster
                },
            }
        } else {
            Self {
                content: raw,
                offset: DisplayOffset::default(),  // No offset for LTR text
            }
        }
    }
}
```

#### How It Works

1. **Detection**: When text is typed, `needs_rtl_hint()` checks if the first non-whitespace character is Arabic
2. **Insertion**: If Arabic is detected, LRM (`U+200E`) is prepended to the visual text
3. **Offset Tracking**: The `DisplayOffset` structure tracks the invisible LRM character:
   - **Bytes**: 3 bytes (UTF-8 encoding of LRM)
   - **Graphemes**: 1 grapheme cluster
4. **Cursor Adjustment**: All cursor position calculations account for this offset
5. **Rendering**: The text rendering engine (glyphon/cosmic-text) receives the LRM hint and properly shapes Arabic glyphs

### 2. `vendor/glyphon/`

**Package**: glyphon v0.5.0

**Purpose**: Low-level text rendering library using `cosmic-text` for glyph shaping

**Dependencies**:
```toml
[dependencies.cosmic-text]
version = "0.10"
```

**Why vendored**: The `cosmic-text` library (which glyphon uses internally) handles the actual glyph shaping and BiDi (bidirectional text) algorithm. The LRM character provides the necessary hint for `cosmic-text` to correctly apply Arabic contextual shaping.

### 3. `vendor/iced_core/`

**Package**: iced_core v0.12.3

**Changes**: No direct code changes, but vendored to ensure compatibility with the patched `iced_widget` and `glyphon` versions.

## How to Apply Patches to a New Project

### Step 1: Setup Directory Structure

```bash
mkdir -p vendor
```

### Step 2: Download and Vendor Dependencies

```bash
# Clone or download the specific versions
cargo install cargo-vendor

# Or manually download from crates.io:
# - glyphon v0.5.0
# - iced_widget v0.12.3
# - iced_core v0.12.3

# Extract to vendor directory
# vendor/
#   ├── glyphon/
#   ├── iced_widget/
#   └── iced_core/
```

### Step 3: Apply the Patch to iced_widget

Edit `vendor/iced_widget/src/text_input.rs`:

1. **Add the LRM constant** (after line 1496):
```rust
const LRM: char = '\u{200E}';
```

2. **Add the DisplayOffset struct** (after line 1498):
```rust
#[derive(Debug, Clone, Copy, Default)]
struct DisplayOffset {
    bytes: usize,
    graphemes: usize,
}
```

3. **Replace the VisualText implementation** (lines 1506-1534):
```rust
struct VisualText {
    content: String,
    offset: DisplayOffset,
}

impl VisualText {
    fn new(value: &Value) -> Self {
        let raw = value.to_string();

        if needs_rtl_hint(&raw) {
            let mut rendered =
                String::with_capacity(raw.len() + LRM.len_utf8());
            rendered.push(LRM);
            rendered.push_str(&raw);
            Self {
                content: rendered,
                offset: DisplayOffset {
                    bytes: LRM.len_utf8(),
                    graphemes: 1,
                },
            }
        } else {
            Self {
                content: raw,
                offset: DisplayOffset::default(),
            }
        }
    }
}
```

4. **Add the RTL detection functions** (after line 1535):
```rust
fn needs_rtl_hint(value: &str) -> bool {
    value
        .chars()
        .find(|ch| !ch.is_whitespace())
        .map(is_rtl_char)
        .unwrap_or(false)
}

fn is_rtl_char(ch: char) -> bool {
    const RTL_RANGES: &[(u32, u32)] = &[
        (0x0600, 0x06FF),
        (0x0750, 0x077F),
        (0x08A0, 0x08FF),
        (0xFB50, 0xFDFF),
        (0xFE70, 0xFEFF),
    ];

    let code = ch as u32;
    RTL_RANGES
        .iter()
        .any(|(start, end)| code >= *start && code <= *end)
}
```

### Step 4: Update Cargo.toml

Add the patch directives to use your vendored versions:

```toml
[dependencies]
iced = { version = "0.12", features = ["tokio", "multi-window", "image", "advanced", "svg"] }
iced_style = "0.12"

[patch.crates-io]
glyphon = { path = "vendor/glyphon" }
iced_widget = { path = "vendor/iced_widget" }
iced_core = { path = "vendor/iced_core" }
```

### Step 5: Add Arabic Fonts

Ensure you have Arabic-compatible fonts in your project:

```rust
// In your app initialization
use iced::Font;

const NOTO_NASKH_ARABIC: Font = Font::External {
    name: "Noto Naskh Arabic",
    bytes: include_bytes!("../fonts/NotoNaskhArabic.ttf"),
};

// Use the font in your TextInput
TextInput::new("placeholder", &value)
    .font(NOTO_NASKH_ARABIC)
    .on_input(Message::TextChanged)
```

**Recommended Fonts**:
- Noto Naskh Arabic (Traditional style)
- Noto Sans Arabic (Modern sans-serif)
- Amiri (Classical Arabic)
- Cairo (Modern geometric)

### Step 6: Build and Test

```bash
cargo build
```

The patches will now be active. Test with Arabic text in your TextInput widgets.

## Testing the Patches

### Test Case 1: Pure Arabic Text
```rust
TextInput::new("أدخل النص هنا", "مرحبا")
```
Expected: Text displays correctly from right to left with proper glyph shaping.

### Test Case 2: Mixed Arabic and English
```rust
TextInput::new("Enter text", "Hello مرحبا 123")
```
Expected: Bidirectional text displays correctly with proper segmentation.

### Test Case 3: Cursor Movement
- Type Arabic text
- Use arrow keys to move cursor
- Expected: Cursor moves correctly through Arabic grapheme clusters

### Test Case 4: Text Selection
- Type Arabic text
- Double-click to select word
- Expected: Correct word boundaries for Arabic words

## Technical Details

### Why LRM Works

The Left-to-Right Mark is a **zero-width formatting character** that:
1. **Does not render visually** (invisible to users)
2. **Provides BiDi context** to the Unicode BiDi algorithm
3. **Forces proper shaping** of adjacent Arabic characters
4. **Is widely supported** by all modern text rendering engines

### Offset Management

The `DisplayOffset` structure is critical because:
- **Visual text** includes the LRM (what's rendered)
- **Logical text** excludes the LRM (what the user types)
- All cursor calculations must account for this 1-character offset

Example:
```
User types: "مرحبا"
Visual text: "\u{200E}مرحبا"
             ^------- LRM (invisible)

Cursor position 0 (user) = position 1 (visual)
Cursor position 5 (user) = position 6 (visual)
```

### Alternative Approaches Considered

1. **Full BiDi Implementation**: Too complex, would require major iced changes
2. **RTL TextInput Widget**: Limited, wouldn't support mixed text
3. **External Text Shaping**: Performance overhead
4. **LRM Insertion (chosen)**: Simple, effective, minimal changes

## Limitations

1. **First-character detection only**: Only checks the first non-whitespace character to determine RTL mode
2. **No paragraph-level BiDi**: Each TextInput is treated as a single line
3. **Limited to TextInput**: Other text widgets may need separate patches
4. **Performance**: Adds small overhead for RTL detection on every keystroke

## Maintenance Notes

### When Updating Iced

When updating to a newer iced version:

1. **Check release notes** for text rendering changes
2. **Re-vendor** the new versions of iced_widget, iced_core, glyphon
3. **Reapply patches** to `text_input.rs` (diff against old version)
4. **Test thoroughly** with Arabic text

### Upstream Contribution

Consider contributing these patches upstream to the iced project:
- Create a GitHub issue describing the problem
- Submit a PR with the LRM-based solution
- Reference Unicode Technical Report #9 (BiDi Algorithm)

## References

- [Unicode Standard - Arabic Block](https://unicode.org/charts/PDF/U0600.pdf)
- [Unicode BiDi Algorithm (UAX #9)](https://unicode.org/reports/tr9/)
- [Iced Framework](https://github.com/iced-rs/iced)
- [Glyphon Text Rendering](https://github.com/grovesNL/glyphon)
- [Cosmic Text (Shaping Engine)](https://github.com/pop-os/cosmic-text)

## Version History

- **2024-11**: Initial implementation for alaraby_tv_file_manager
- **Iced v0.12**: Patches tested and working
- **Glyphon v0.5.0**: Compatible version

## License

These patches maintain the original MIT license of the iced framework.

---

**Author**: White Cabbage
**Last Updated**: 2024-11-14
**Iced Version**: 0.12.3
