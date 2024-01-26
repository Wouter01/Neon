<div align="center">

[![Build Status][build status badge]][build status]
[![Platforms][platforms badge]][platforms]
[![Documentation][documentation badge]][documentation]
[![Discord][discord badge]][discord]

</div>

# Neon
A Swift library for efficient, flexible content-based text styling.

- Lazy content processing
- Minimal invalidation calculation
- Support for multiple sources of token data
- Support for versionable text data storage
- A hybrid sync/async system for targeting flicker-free styling on keystrokes
- [tree-sitter](https://tree-sitter.github.io/tree-sitter/) integration
- Text-system agnostic

Neon has a strong focus on efficiency and flexibility. It sits in-between your text system and wherever you get your semantic token information. Neon was developed for syntax highlighting and it can serve that need very well. However, it is more general-purpose than that and could be used for any system that needs to manage the state of range-based content.

Many people are looking for a drop-in editor View subclass that does it all. This is a lower-level library. You could, however, use Neon to drive highlighting for a view like this.

> Warning: The code on the main branch is still in beta. It differs significantly from the 0.6.x releases. Both your patience and bug reports are very appreciated.

## Installation

```swift
dependencies: [
    .package(url: "https://github.com/ChimeHQ/Neon", branch: "main")
],
targets: [
    .target(
        name: "MyTarget",
        dependencies: [
            "Neon",
            .product(name: "TreeSitterClient", package: "Neon"),
            .product(name: "RangeState", package: "Neon"),
        ]
    ),
]
```

## Concepts

Neon is made up of three parts: the core library, `RangeState` and `TreeSitterClient`.

### RangeState

Neon's lowest-level component is called RangeState. This module contains the core building blocks used for the rest of the system. RangeState is built around the idea of hybrid synchronous/asynchronous execution. Making everything async is a lot easier, but that makes it impossible to provide a low-latency path for small documents. It is content-independent.

- `Hybrid(Throwing)ValueProvider`: a fundamental type that defines work in terms of both synchronous and asynchronous functions
- `RangeProcessor`: performs on-demand processing of range-based content (think parsing)
- `RangeValidator`: manages the validation of range-based content (think highlighting)
- `RangeInvalidationBuffer`: buffer and consolidate invalidations so they can be applied at the optimal time

Many of these support versionable content. If you are working with a backing store structure that supports efficient versioning, like a [piece table](https://en.wikipedia.org/wiki/Piece_table), expressing this to RangeState can improve its efficiency.

It might be surprising to see that many of the types in RangeState are marked `@MainActor`. Right now, I have found no way to both support the hybrid sync/async functionality while also not being tied to a global actor. I think this is the most resonable trade-off, but I would very much like to lift this restriction. However, I believe it will require [language changes](https://forums.swift.org/t/isolation-assumptions/69514/47).

### TreeSitterClient

This library is a hybrid sync/async interface to [SwiftTreeSitter][SwiftTreeSitter]. It features:

- UTF-16 code-point (`NSString`-compatible) API for edits, invalidations, and queries
- Processing edits of `String` objects, or raw bytes
- Invalidation translation to the current content state regardless of background processing
- On-demand nested language resolution via tree-sitter's injection system
- Background processing when needed to scale to large documents

Tree-sitter uses separate compiled parsers for each language. There are a variety of ways to use tree-sitter parsers with SwiftTreeSitter. Check out that project for details.

### Neon

The top-level module includes systems for managing text styling. It is also text-system independent. It makes very few assumptions about how text is stored, displayed, or styled. It also includes some components for use with stock AppKit and UIKit systems. These are provided for easy integration, not maximum performance. 

- `TextViewHighlighter`: simple integration between `NSTextView`/`UITextView` and `TreeSitterClient`
- `TextViewSystemInterface`: implementation of the `TextSystemInterface` protocol for `NSTextView`/`UITextView`
- `LayoutManagerSystemInterface`, `TextLayoutManagerSystemInterface`, and `TextStorageSystemInterface`: Specialized TextKit 1/2 implementations `TextSystemInterface`

There is also an example project that demonstrates how to use `TextViewHighlighter` for macOS and iOS.

### Token Data Sources

Neon was designed to accept and overlay token data from multiple sources simultaneously. Here's a real-world example of how this is used:

- First pass: pattern-matching system with ok quality and guaranteed low-latency
- Second pass: [tree-sitter](https://tree-sitter.github.io/tree-sitter/), which has good quality and **could** be low-latency
- Third pass: [Language Server Protocol](https://microsoft.github.io/language-server-protocol/)'s [semantic tokens](https://microsoft.github.io/language-server-protocol/specifications/specification-current/#textDocument_semanticTokens), which can augment existing highlighting, but is high-latency

### Theming

A highlighting theme is really just a mapping from semantic labels to styles. Token data sources apply the semantic labels and the `TextSystemInterface` uses those labels to look up styling.

This separation makes it very easy for you to do this look-up in a way that makes the most sense for whatever theming formats you'd like to support. This is also a convenient spot to adapt/modify the semantic labels coming from your data sources into a normalized form.

### TextKit Integration

In a traditional `NSTextStorage`-backed system (TextKit 1 and 2), it can be challenging to achieve flicker-free on-keypress highlighting. You need to know when a text change has been processed by enough of the system that styling is possible. This point in the text change lifecycle is not natively supported by `NSTextStorage` or `NSLayoutManager`. It requires an `NSTextStorage` subclass. Such a subclass, `TSYTextStorage` is available in [TextStory](https://github.com/ChimeHQ/TextStory).

But, even that isn't quite enough unfortunately. You still need to precisely control the timing of invalidation and styling. This is where `RangeInvalidationBuffer` comes in.

## Usage

### TreeSitterClient

Here's a minimal sample using TreeSitterClient. It is involved, but should give you an idea of what needs to be done.

```swift
import Neon
import SwiftTreeSitter
import TreeSitterClient

import TreeSitterSwift // this parser is available via SPM (see SwiftTreeSitter's README.md)

// assume we have a text view available that has been loaded with some Swift source

let languageConfig = try LanguageConfiguration(
    tree_sitter_swift(),
    name: "Swift"
)

let clientConfig = TreeSitterClient.Configuration(
    languageProvider: { identifier in
        // look up nested languages by identifier here. If done
        // asynchronously, inform the client they are ready with
        // `languageConfigurationChanged(for:)`
        return nil
    },
    contentProvider: { [textView] length in
        // given a maximum needed length, produce a `Content` structure
        // that will be used to access the text data

        // this can work for any system that efficiently produce a `String`
        return .init(string: textView.string)
    },
    lengthProvider: { [textView] in
        textView.string.utf16.count

    },
    invalidationHandler: { set in
        // take action on invalidated regions of the text
    },
    locationTransformer: { location in
        // optionally, use the UTF-16 location to produce a line-relative Point structure.
        return nil
    }
)

let client = try TreeSitterClient(
    rootLanguageConfig: languageConfig,
    configuration: clientConfig
)

let source = textView.string

let provider = source.predicateTextProvider

// this uses the synchronous query API, but with the `.required` mode, which will force the client
// to do all processing necessary to satisfy the request.
let highlights = try client.highlights(in: NSRange(0..<24), provider: provider, mode: .required)!

print("highlights:", highlights)
```

## Contributing and Collaboration

I would love to hear from you! Issues or pull requests work great. A [Discord server][discord] is also available for live help, but I have a strong bias towards answering in the form of documentation.

I prefer collaboration, and would love to find ways to work together if you have a similar project.

I prefer indentation with tabs for improved accessibility. But, I'd rather you use the system you want and make a PR than hesitate because of whitespace.

By participating in this project you agree to abide by the [Contributor Code of Conduct](CODE_OF_CONDUCT.md).

[build status]: https://github.com/ChimeHQ/Neon/actions
[build status badge]: https://github.com/ChimeHQ/Neon/workflows/CI/badge.svg
[platforms]: https://swiftpackageindex.com/ChimeHQ/Neon
[platforms badge]: https://img.shields.io/endpoint?url=https%3A%2F%2Fswiftpackageindex.com%2Fapi%2Fpackages%2FChimeHQ%2FNeon%2Fbadge%3Ftype%3Dplatforms
[documentation]: https://swiftpackageindex.com/ChimeHQ/Neon/main/documentation
[documentation badge]: https://img.shields.io/badge/Documentation-DocC-blue
[discord]: https://discord.gg/esFpX6sErJ
[discord badge]: https://img.shields.io/badge/Discord-purple?logo=Discord&label=Chat&color=%235A64EC
[SwiftTreeSitter]: https://github.com/ChimeHQ/SwiftTreeSitter
