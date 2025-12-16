# Obsidian-Wypst Architecture

This document provides a comprehensive overview of the obsidian-wypst plugin architecture for developers familiar with the JavaScript ecosystem.

## Overview

**obsidian-wypst** is an [Obsidian](https://obsidian.md/) plugin that enables Typst math typesetting in Obsidian notes. It replaces the default MathJax math rendering with Typst-based rendering, powered by the [wypst](https://github.com/0xpapercut/wypst) library.

## Development Environment

### Technology Stack

| Component | Technology |
|-----------|------------|
| Language | TypeScript |
| Build Tool | esbuild (via `esbuild.config.mjs`) |
| Package Manager | npm |
| Target Platform | Obsidian Plugin (CommonJS format) |
| Math Rendering | wypst → Typst WASM → KaTeX CSS |

### Project Structure

```
obsidian-wypst/
├── main.ts              # Plugin entry point
├── package.json         # npm dependencies and scripts
├── tsconfig.json        # TypeScript configuration
├── esbuild.config.mjs   # esbuild bundler configuration
├── custom.d.ts          # TypeScript type declarations for WASM imports
├── manifest.json        # Obsidian plugin manifest
├── versions.json        # Plugin version compatibility mapping
├── default.css          # Custom styles (KaTeX font-size override)
└── version-bump.mjs     # Version bump automation script
```

### Build Commands

```bash
npm install      # Install dependencies
npm run dev      # Development mode with watch
npm run build    # Production build (outputs main.js and styles.css)
```

### Build Process

1. **TypeScript Compilation**: `tsc -noEmit -skipLibCheck` (type checking only)
2. **Bundling**: esbuild bundles `main.ts` into `main.js`
3. **Asset Handling**:
   - `.wasm` files are loaded as binary data (inlined)
   - `.woff`, `.woff2`, `.ttf` fonts are converted to data URLs
   - CSS is extracted and renamed to `styles.css`

---

## How Typst Compiler is Imported via WASM

### The Integration Chain

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         obsidian-wypst (this plugin)                         │
│                                                                              │
│   main.ts                                                                    │
│     ├── import wypst from 'wypst'        ← JavaScript API                    │
│     └── import wasm from 'wypst/core/core_bg.wasm'  ← WASM binary           │
│                                                                              │
│   Initialization:                                                            │
│     await wypst.init(wasm)               ← Load WASM into JavaScript engine  │
└──────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────────┐
│                         wypst npm package (v0.0.4)                           │
│                                                                              │
│   wypst.js                                                                   │
│     ├── Exports: render(), renderToString(), parseTree(), init()            │
│     └── Uses KaTeX for DOM tree building                                     │
│                                                                              │
│   core/                                                                      │
│     ├── core.js          ← WASM bindings (init, parseTree, typstContentTree)│
│     ├── core_bg.js       ← WASM glue code (memory management, imports)      │
│     └── core_bg.wasm     ← Compiled Typst compiler (~13.8 MB)               │
└──────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────────┐
│                      wypst source (Rust → WASM)                              │
│                                                                              │
│   src/core/Cargo.toml                                                        │
│     └── typst = { git = "https://github.com/typst/typst.git", tag = "v0.10.0" }│
│     └── typst-syntax = { git = "...typst.git", tag = "v0.10.0" }            │
│                                                                              │
│   Build: wasm-pack build --target web                                        │
│     → Compiles Rust code + Typst library to WebAssembly                      │
└──────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────────┐
│                         typst (v0.10.0)                                      │
│                                                                              │
│   The official Typst typesetting system                                      │
│   https://github.com/typst/typst                                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Key Integration Points

#### 1. TypeScript WASM Declaration (`custom.d.ts`)

```typescript
declare module '*.wasm' {
    const content: any;
    export default content;
}
```

This allows TypeScript to import `.wasm` files as modules.

#### 2. esbuild WASM Loader (`esbuild.config.mjs`)

```javascript
loader: {
    '.wasm': 'binary',  // Load WASM as raw binary ArrayBuffer
}
```

The WASM binary is inlined into the JavaScript bundle as binary data.

#### 3. Plugin Initialization (`main.ts`)

```typescript
import wypst from 'wypst';
import wasm from 'wypst/core/core_bg.wasm';

// In onload():
await wypst.init(wasm);  // Initialize WASM with binary data
```

The plugin passes the WASM binary ArrayBuffer to `wypst.init()`, which instantiates the WebAssembly module.

#### 4. WASM Instantiation (`core.js` in wypst)

The `init()` function:
1. Creates imports object with JavaScript callbacks for WASM
2. Instantiates the WebAssembly module using `WebAssembly.instantiate()`
3. Exports functions like `parseTree()` that call into the WASM module

### Runtime Flow

```
User writes $sum_(n>=1) 1/n^2$
         ↓
Obsidian calls MathJax.tex2chtml()
         ↓
Plugin intercepts and calls wypst.renderToString()
         ↓
wypst calls parseTree() (JavaScript)
         ↓
parseTree() calls wasm.parseTree() (WASM boundary)
         ↓
Typst parses expression and returns AST (WASM → JavaScript)
         ↓
KaTeX buildTree() converts AST to DOM tree
         ↓
DOM tree rendered in Obsidian preview
```

---

## Bumping the Typst Version

The current setup uses **Typst v0.10.0** (released January 2024). As of December 2024, Typst has reached **v0.12.0+**. Here are the possible approaches to upgrade:

### Option 1: Wait for wypst Package Update (Recommended)

**Effort**: Low  
**Risk**: Low

The simplest approach is to wait for the upstream `wypst` package maintainer to update the Typst version.

```bash
# Check for new wypst versions
npm show wypst versions
# Currently: ['0.0.1', '0.0.2', '0.0.3', '0.0.4', '0.0.5', '0.0.6', '0.0.7', '0.0.8']

# Update to latest wypst
npm install wypst@latest
```

**Current status**: wypst v0.0.8 is available on npm and may have a newer Typst version.

**To upgrade obsidian-wypst to use a newer wypst:**
1. Update `package.json`: `"wypst": "^0.0.8"` (or latest)
2. Run `npm install`
3. Test thoroughly for any breaking changes in the API
4. Rebuild: `npm run build`

**Note**: The vypst v0.0.8 release notes indicate a build system change to esbuild, which may affect how the WASM is loaded. Testing is essential.

### Option 2: Fork and Rebuild wypst

**Effort**: Medium  
**Risk**: Medium

If you need a specific Typst version not yet released in wypst:

1. **Fork the wypst repository**:
   ```bash
   git clone https://github.com/0xpapercut/wypst.git
   cd wypst
   ```

2. **Update Typst version** in `src/core/Cargo.toml`:
   ```toml
   [dependencies]
   typst = { git = "https://github.com/typst/typst.git", tag = "v0.12.0" }
   typst-syntax = { git = "https://github.com/typst/typst.git", tag = "v0.12.0" }
   ```

3. **Install Rust and wasm-pack**:
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   cargo install wasm-pack
   ```

4. **Build the WASM**:
   ```bash
   cd src/core
   wasm-pack build --target web -d ../../core
   ```

5. **Use the local fork** in obsidian-wypst:
   ```json
   // package.json
   "dependencies": {
       "wypst": "file:../wypst"
   }
   ```
   
   Or publish to npm under your own scope:
   ```bash
   npm publish --access public  # as @your-scope/wypst
   ```

**Important considerations**:
- Newer Typst versions may have breaking changes in the Rust API
- The `converter.rs` file in wypst may need updates for API changes
- WASM binary size may change significantly (~13.8 MB currently)

### Option 3: Direct Typst WASM Integration (Advanced)

**Effort**: High  
**Risk**: High

For complete control, bypass wypst entirely and integrate Typst WASM directly:

1. **Use official typst-preview** or similar projects that provide Typst WASM builds

2. **Create custom Rust wrapper** exposing minimal API:
   ```rust
   use typst::syntax::parse;
   use wasm_bindgen::prelude::*;
   
   #[wasm_bindgen]
   pub fn parse_math(input: &str) -> JsValue {
       // Custom implementation
   }
   ```

3. **Build and integrate** into obsidian-wypst

**This approach is only recommended if**:
- You need very specific Typst features not exposed by wypst
- You want to minimize bundle size by removing unused code
- You're planning to contribute the changes back upstream

### Option 4: Use Different Typst Rendering Library

**Effort**: Medium-High  
**Risk**: Medium

Alternative libraries that provide Typst rendering:

- **typst.ts**: Official TypeScript bindings (if available)
- **typst-preview**: Browser-based Typst rendering

This would require rewriting the integration layer but might provide:
- Better maintained bindings
- Newer Typst versions
- Different rendering approaches

---

## Version Compatibility Matrix

| obsidian-wypst | wypst | Typst | Notes |
|----------------|-------|-------|-------|
| 0.0.2 (current) | 0.0.4 | 0.10.0 | Current stable |
| (proposed) | 0.0.8 | 0.10.0 | Test required |

---

## Testing After Version Bump

After updating the Typst version, test these scenarios:

1. **Basic math expressions**: `$x^2 + y^2 = z^2$`
2. **Complex expressions**: `$sum_(n>=1) 1/n^2 = pi^2/6$`
3. **Edge cases**: Empty expressions, very long expressions
4. **LaTeX fallback**: `$\pi$` should still render via MathJax
5. **Performance**: Large documents with many math blocks
6. **Error handling**: Invalid Typst syntax should show error messages

---

## Summary

The obsidian-wypst plugin integrates Typst math typesetting through a multi-layer architecture:

1. **This plugin** (`main.ts`) intercepts Obsidian's math rendering
2. **wypst** (`wypst.js`) provides the JavaScript API and KaTeX integration
3. **Core WASM** (`core_bg.wasm`) contains the compiled Typst parser
4. **Typst** (Rust) is the underlying typesetting engine

To bump the Typst version, the most practical approach is to:
1. First, try updating to the latest `wypst` package
2. If that doesn't have the needed Typst version, fork wypst and update `Cargo.toml`
3. Rebuild the WASM and test thoroughly

The WASM loading is handled through esbuild's binary loader, which inlines the ~13.8 MB WASM file into the JavaScript bundle as binary data.
