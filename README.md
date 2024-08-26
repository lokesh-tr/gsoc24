# [GSoC-2024] Expansion of Swift Macros in Visual Studio Code

Hello Everyone,

I'm Lokesh.T.R from India. I'm a sophomore currently pursuing my bachelor's degree
in Vel Tech University, Chennai. I'm thrilled to share with you on what I have accomplished
over the summer with my mentors @ahoppen and @adam-fowler for Google Summer of Code 2024.

## Overview

Over the summer, we worked on adding support for expansion of Swift Macros in Visual Studio Code.
Our project's main goal is to implement a code action in VS Code that allows users
to view the generated contents of a Swift Macro.

There were also some stretch goals which include:

1. Bringing Semantic Functionality (such as jump-to-defintion, quick help on hover,
   Syntax Highlighting, etc.) to the macro expansion being previewed.
2. Allowing to perform the "Expand Macro" Code Action on a macro that is present
   in the generated macro expansion to support the expansion of nested macros.

And as a bonus, we also worked on supporting macro expansions in other LSP-based
editors which by default cannot make use of the LSP extensions that we introduced.

## Implementation Details

### How Swift Language Features works in LSP-based editors

Let's have a look at three key components that you use in your everyday life in an LSP-based editor (e.g. VS Code):

1. VS Code-Swift Extension (Client):
   - Primarily acts as a bridge between VS Code and SourceKit-LSP
2. SourceKit-LSP (Server):
   - provides the necessary editor features to VS Code
   - communicates using Language Server Protocol (LSP)
3. SourceKitD (Background Service):
   - provides the raw data and operations to SourceKit-LSP
   - baked into the swift compiler

### Main Goal

In order to achieve the main goal, we introduced two new LSP extensions and new custom URL scheme as follows:

```typescript
// NEW LSP EXTENSIONS (SPECIFICATIONS)
// -----------------------------------

// workspace/peekDocuments (sourcekit-lsp -> vscode-swift)
export interface PeekDocumentsParams {
  uri: DocumentUri;
  position: Position;
  locations: DocumentUri[];
}

export interface PeekDocumentsResult {
  success: boolean;
}

// workspace/getReferenceDocument (vscode-swift -> sourcekit-lsp)
export interface GetReferenceDocumentParams {
  uri: DocumentUri;
}

export interface GetReferenceDocumentResult {
  content: string;
}

// NEW CUSTOM URL SCHEME (SPECIFICATIONS)
// --------------------------------------

// Reference Document URL
("sourcekit-lsp://<document-type>/<display-name>?<parameters>");

// Reference Document URL with Macro Expansion Document Type
("sourcekit-lsp://swift-macro-expansion/LaCb-LcCd.swift?fromLine=&fromColumn=&toLine=&toColumn=&bufferName=&parent=");
```

- `"workspace/peekDocuments"` allows the SourceKit-LSP Server to show the the contents stored in the `locations` inside
  the source file `uri` as a peek window
- Reference Document URL Scheme `sourcekit-lsp://` so that we can use it to encode the necessary data required to generate
  any form of content to which the URL corresponds to.
- We introduce the very first `document-type` of the Reference Document URL which is `swift-macro-expansion`. This will encode
  all the necessary data required to generate the macro expansion contents.
- `"workspace/getReferenceDocument"` is introduced so that the Editor Client can make a request to the SourceKit-LSP Server
  with the Reference Document URL to fetch its contents.

The way this works is that, we generate the Reference Document URLs from the macro expansions generated using sourcekitd and make a
`"workspace/peekDocuments"` request to the Editor Client. In VS Code, this executes the `"editor.action.peekLocations"` command
to present a peeked editor. Since VS Code can't resolve the contents of Reference Document URL, it makes
a `"workspace/getReferenceDocument"` request to the SourceKit-LSP Server, thereby retrieveing the contents and successfully
displaying it in the peeked editor.

### Stretch Goals

1. Achieving Semantic Functionality (jump-to-definition, quick help on hover, syntax highlighting, etc.):
   - SourceKit-LSP and SourceKitD by default doesn't know how to handle Reference Document URLs.
   - We need the build arguments of a file to provide semantic functionality.
   - We can use the source file's build arguments as arguments of the reference documents to trick sourcekitd to provide
     Semantic Functionality for the reference documents.
2. Achieving Nested Macro Expansion:
   - Due to the flexible nature of the Reference Document URLs, nested macro expansions becomes trivial.
   - The beauty of the Reference Document URLs is that we can nest the Reference Document URLs.
   - We set the `parent` parameter of the macro expansion reference document URL to the source file if it's a first level
     macro expansion or to the reference document from which the macro expansion originates if it was second or third or
     n-th level macro expansion. This allows us to expand nested macros efficiently.

### Bonus

While the new LSP Extensions which we introduced, doesn't work out-of-the-box in other LSP-based editors and requires
the extension / plugin developers of respective editors to make use of it, We worked on providing a basic form of first level
macro expansion support using the standard LSP Requests to support other LSP-based editors.

This works as follows:

1. SourceKitLSP asks SourceKitD for macro expansions.
2. It then stitches all the macro expansions together in a single file.
3. It stores the file in some temporary location in the disk.
4. It then makes a `ShowDocumentRequest` to open and show the file in the editor.

That wraps up the GSoC project successfully!

### What's left to do?

- I will be working on implementing a test case that encompasses all the semantic features in all nested macro levels in
  sourcekit-lsp.
- I will also implement some end-to-end test cases in the vscode-swift side which ensures that they really work as intended in
  a real world situation.

Test cases for freestanding macros, attached macros and nested macros are already in place.
Code Documentation is also in place for everything that was implemented so far.

### Future Directions

1. Feedback, Feedback, Feedback!

   - We had put so much attention to detail and ensured that every decision made in the design process was thoughtful.
   - But, you may have a better idea, or you may find some issues, and we would love to improve this feature.
   - Please file an issue to suggest an idea or report bugs in the sourcekit-lsp or vscode-swift repository wherever you
     face the issue.

2. Migrating the non-standard `"workspace/getReferenceDocument"` to the new standard `"workspace/textDocumentContent"` Request

   - We should be able to perform this migration when the specifications of LSP 3.18 gets finalised.
   - Thanks to @fwcd for noticing the new change and making a PR to be ready for migration when LSP 3.18 releases.

3. Migrate from generating temporary files in the Disk to Reference Document URLs for other LSP-based editors

   - We are currently unable to generate macro expansions on-the-fly in other LSP-based editors since we don't have
     our LSP Extension for getting the contents of the reference document.
   - Building on top of the previous idea, when LSP 3.18 gets finalised, we should be able to completely eliminate
     temporary file storage in favour of the standard `"workspace/textDocumentContent"` Request.

4. Adding Semantic Functionality & Nested Macro Expansion support for other LSP-based editors

   - This is tricky to implement since the temporary file (or reference document after LSP 3.18) will have all the macro
     expansions of a given macro in a single file.
   - Although the same approach can be used such as passing the source file's build arguments and `parent` to be the macro's
     originating file, there will be line and character position shifts that should be taken into consideration.
   - For example,
     the third macro expansion of a given attached macro will be expected to start at `0:0` but its actual location will be shifted
     by the first and second macro expansion's content length and three lines of comments that describes where the macro will be
     present in the original file

5. Other Use Cases for the Reference Document URL

   - Reference Document URLs where built from the ground up to allow for encoding the data required to show any form of content,
     one such example which we discussed today is `swift-macro-expansion` document type.
   - This should allow anyone to show any content of their choice, in a peeked editor or a fully open document, as long
     as its generated during compile time.
   - The following use cases are not related to macro expansions but uses the Reference Document URL that we created to
     show other document types:
     - Migrating `OpenInterfaceRequest` to Reference Document URLs to show Swift Generated Interfaces
       *(Thanks to @ahoppen & @adam-fowler for the idea)*
     - Showing Implicitly generated constructors and Synthesized code upon `Equatable`, `Hashable` and `Codable` Conformances.
       *(Thanks to @douglas-gregor and @rintaro for pointing this out and some use cases for reference documents in their projects)*
     - Showing a preview of generated HTML or rendering the generated HTML from the Mustache template engine by hooking up its CLI.
       *(Thanks to @adam-fowler for his feedback on my idea)*
   - These are just few examples, and the fact that you can show various document formats based on various code generation behaviours
     brings a wide range of possibilities, and hence, you can bring your own idea.
   - And with LSP 3.18, this will be standardised across all editors, not just VS Code.

### Thanks & Gratitude

I offer my deepest gratitude to my mentors @ahoppen and @adam-fowler without whom this journey is impossible. GSoC is not
only the work of me as a contributor but also the work of my mentors in guiding me and helping me out whenever possible.

Hey @ahoppen, Thank you for accepting my proposal, spending as many hours as possible with me to set up my environment and trying to
debug all the bugs in my environment. Thanks for getting me started with the PRs. Thanks for your wonderful ideas and feedback
on my ideas. Thanks for waking up to attend the 8:00 AM meeting. Thanks for your detailed PR reviews. Thanks for helping me out when I got constrained by time. Thanks for allowing me to change meeting dates due to my schedule. Thanks for all the immediate and quick responses. Pardon me for any mistakes that I did. I hope I gave you a good GSoC experience too. And, Thank you for all the work which you did in sourcekit-lsp.

Hey @adam-fowler, Thank you for accepting my proposal. Thank you so much for suggesting me to get started with the project
in a docker container. Thanks for your wonderful ideas and feedback on my ideas. Thanks for reviewing my PRs. Thank you
for always being ready to jump into LSP specfications and VS Code's codebase to identify things. Thanks for allowing me to
change meeting dates due to my schedule. Thanks for all the immediate and quick responses. Pardon me for any mistakes that
I did. I believe this is your first time also, I hope I gave you a good GSoC experience too. And, Thank you for all the work
which you did in vscode-swift.

If not for you two people, this project wouldn't be a success.

#### Special Thanks

- I was in the middle of lots of works & exams in my university when GSoC's contributor proposal submission period, I didn't have
much time to go in detail into any project from any organisation. I always wanted to do some contribution to Swift. Among
all the ideas, The initial foundations and discussions laid by @fwcd helped me to understand the project and its requirements
and also allowed me to understand both the sourcekit-lsp and vscode-swift code base faster, and make a detailed proposal
that got me selected in the first place.

- Thanks to @plemarquand for testing out the very first implementation of my project. I certainly didn't expect that. And,
it did give me a boost in confidence that the community is very welcoming and also motivated me that I'm in my right direction.
Also, Thanks to you for offering me some initial guidance on writing end-to-end test cases in vscode-swift.

- Thanks to @douglas-gregor and @rintaro for making me realise that you guys have already started finding use cases of my work in
your own projects. I certainly didn't expect that the design decision which me and my mentors made would have immediate benefits
to the community in its own way.

- Thanks to @Matejkob and @parispittman for encouraging me to present my project in conferences whenever and wherever possible.

- Thanks to the entire Swift Community for being so welcoming, I started my journey with Swift in 2019 and I always wanted
to do some contribution, I didn't take a step forward since I wasn't very confident with my communication / social skills.
Now, years have passed, I believe I grew up seeing Swift evolve. GSoC gave me a perfect excuse to push me out of my comfort zone and made me to contribute to Swift and to have weekly meetings with unimaginably intelligent mentors and thus, Here we are with a
successful project.
(My mentors might have noticed the difference in how I spoke in the first introductory meeting and in the final
presentation).

### Appendix: Pull Request Stats

#### Pre-GSoC (Community Bonding Period)

**`sourcekit-lsp [GOOD FIRST ISSUE]`**

1. Change static method `DocumentURI.for(_:testName:)` to an initializer `DocumentURI.init(for:testName:)` [#1348](https://github.com/swiftlang/sourcekit-lsp/pull/1348)
2. Rename `note` to `notification` throughout the codebase wherever necessary [#1353](https://github.com/swiftlang/sourcekit-lsp/pull/1353)

#### GSoC (Coding Period)

##### What got merged?

**`sourcekit-lsp`**

3. Add LSP support for showing Macro Expansions [#1436](https://github.com/swiftlang/sourcekit-lsp/pull/1436)
4. Add LSP extension to show Macro Expansions (or any document) in a "peeked" editor (and some minor quality improvements) [#1479](https://github.com/swiftlang/sourcekit-lsp/pull/1479)
5. Skip `testFreestandingMacroExpansion` if host toolchain does not support background indexing [#1548](https://github.com/swiftlang/sourcekit-lsp/pull/1548) by @ahoppen
6. Skip `testAttachedMacroExpansion` if host toolchain does not support background indexing [#1553](https://github.com/swiftlang/sourcekit-lsp/pull/1553)
7. Allow macro expansions to be viewed through `GetReferenceDocumentRequest` instead of storing in temporary files [#1567](https://github.com/swiftlang/sourcekit-lsp/pull/1567)
8. Support expansion of nested macros [#1631](https://github.com/swiftlang/sourcekit-lsp/pull/1631) by @ahoppen (Co-authored by me from #1610)
9. Add support for semantic functionality in macro expansion reference documents [#1634](https://github.com/swiftlang/sourcekit-lsp/pull/1634)
10. Remove `ExperimentalFeature.showMacroExpansions` flag for macro expansions [#1635](https://github.com/swiftlang/sourcekit-lsp/pull/1635)
11. Add an extra percent encoding layer when encoding DocumentURIs to LSP requests [#1636](https://github.com/swiftlang/sourcekit-lsp/pull/1636) by @ahoppen
12. Address review comments to #1631 [#1637](https://github.com/swiftlang/sourcekit-lsp/pull/1637) by @ahoppen

**`vscode-swift`**

13. Handle `PeekDocumentsRequest` to show documents from sourcekit-lsp (like, Macro Expansions) in a "peeked" editor [#945](https://github.com/swiftlang/vscode-swift/pull/945)
14. Add missing licence header to peekDocuments.ts [#953](https://github.com/swiftlang/vscode-swift/pull/953) by @plemarquand
15. Retrieve macro expansions for `PeekDocumentsRequest` through `GetReferenceDocumentRequest` instead of showing temporary files [#971](https://github.com/swiftlang/vscode-swift/pull/971)
16. Allow VS Code to recognise files with "sourcekit-lsp" scheme to provide Semantic Functionality through SourceKitLSP [#990](https://github.com/swiftlang/vscode-swift/pull/990)
17. Fix peeked editor closing without reopening with new contents when triggered again at the same position in the same file [#1019](https://github.com/swiftlang/vscode-swift/pull/1019) (Workaround for a bug in VS Code)

##### What should be merged?

18. Work around `Uri` round-tripping issue in VS Code for `sourcekit-lsp` scheme [#1026](https://github.com/swiftlang/vscode-swift/pull/1026) by @ahoppen

##### What got closed?

**`sourcekit-lsp`**

19. Add LSP extension request for retrieving macro expansions [#892](https://github.com/swiftlang/sourcekit-lsp/pull/892) by @fwcd 🙏🏻

20. Add Semantic Functionality to Macro Expansion Reference Documents (including Nested Macro Expansion) 🚦 [#1610](https://github.com/swiftlang/sourcekit-lsp/pull/1610) (Explored a variety of ideas everything took shape into #1631,
last commit squashed previous ideas by the way) - Closed
in favour of #1631

**`vscode-swift`**

21. Add client-side support for macro expansions [#621](https://github.com/swiftlang/vscode-swift/pull/621) by @fwcd 🙏🏻

22. Fix semantic functionality not working for macro expansion reference documents due to URL encoding [#1017](https://github.com/swiftlang/vscode-swift/pull/1017) (Workaround for a bug in VS Code) - Closed in favour of #1026 by @ahoppen

##### What has to be done?

1. Revert [sourcekit-lsp#1636](https://github.com/swiftlang/sourcekit-lsp/pull/1636) when [vscode-swift#1026](https://github.com/swiftlang/vscode-swift/pull/1026) gets merged.
2. Add test cases for Semantic Functionality in sourcekit-lsp.
3. Add end-to-end test cases for the "Expand Macro" Code Action in vscode-swift.
4. Merge [sourcekit-lsp#1639](https://github.com/swiftlang/sourcekit-lsp/pull/1639) by @fwcd when LSP 3.18 gets finalised
5. Merge [vscode-swift#1027](https://github.com/swiftlang/vscode-swift/pull/1027) by @fwcd when LSP 3.18 gets finalised
