Hey there! I am Rejoan Sardar, a computer science undergraduate (as of writing this) at the Lovely Professional University, Punjab, India. Apart from my interest in computers and software, I have a dormant passion for audio DSP and synthesis.

# Big-Project-with-stdlib

-   REPL environment with integrated help and examples.

    - #### New REPL, New Art

    ![ascii](https://github.com/user-attachments/assets/429c3e74-2f78-447e-a855-27aaf9dcb998)

- #### Auto-closing brackets/quotations

  My first work on the REPL was implementing auto-closing brackets/quotations, a common feature in IDEs and code editors.

  ![auto-closing](https://github.com/user-attachments/assets/318963ec-8043-4429-accc-873e78849d0b)
  
  - feat: add support for auto-closing brackets/quotations in the REPL

  **Implementation details**:
  - The approach is to walk the abstract syntax tree (generated by *acorn*) for the current line and detect an auto-closing candidate. If found we just write the corresponding auto-closing symbol to the output. 
  - Now there are also cases where the user instinctively types the closing symbol themselves as if this feature never existed, and we should respect that and allow the user to type through the auto-appended symbol.
  - What about auto-deleting the appended symbol? When deleting an opening symbol, check if the corresponding closing symbol follows it. If it does, time to delete it. We need to avoid this behavior, if the user is editing a string literal and we again use acorn to find if the nodes around the cursor are strings.

- #### Pager

  Earlier, when an output was longer than the terminal height, it would scroll all the way to the end of the output. This meant the user had to scroll all the way back up to start reading the output. The pager aims to capture long outputs and display them in a scrollable way.

  ![scroll](https://github.com/user-attachments/assets/d0fcf1e0-fc9c-4f54-8414-bf5c4f809335)

  - feat: add pager to allow scrolling of long outputs in the REPL

  In general, pagers are simply implemented by halting the printing till the terminal height and waiting for user input to print further. But I wanted to do it differently. With our UI, we page in-place, meaning the pager appears like a screen, and we can still see the parent command on the top. The only downside to this might be the possible jittering of output as we rely on re-rendering the page upon every scroll.

  **Implementation details**:

  1. The first step is detecting a *pageable* output. We do this by checking if the number of rows in the output is greater than the height of the terminal stream (including space for the input command).
  2. Then we write the page UI and maintain the page indexing. During paging mode, the entire REPL is frozen and is only receptive to pager controls and `SIGINT` interrupts.
  3. As we receive the page up/down controls, we update the indices and re-render the page UI.
  4. We also listen to `SIGWINCH` events to make it receptive to terminal resizes.


- #### Syntax highlighting

  One of the most crucial additions to the REPL was syntax highlighting.
  
  ![typing](https://github.com/user-attachments/assets/71ce8887-486a-4ca1-b454-032f19145672)

  -  feat: add syntax highlighting in the REPL

    This adds the core modules for syntax highlighting, namely the tokenizer, and highlighter.

    ![syntax](https://github.com/user-attachments/assets/5af44178-e02d-43c2-bf82-2f53ccdd4a92)

    **Implementation details**:
    1. With every keypress, we capture the updated line.
    2. We then check if the updated line is changed. This is a short caching mechanism to avoid perf drag during events like moving left/right in the REPL.
    3. Tokenization. To support various token types, a good tokenizer is crucial. We use *acorn* to parse the line into an abstract syntax tree. During parsing, we keep a record of basic tokens like comments, operators, punctuation, strings, numbers, and regexps.
       To resolve declarations, we resolve all declarations (functions, classes, variables etc) in the local scope (not yet added to global context) by traversing the AST.
       To resolve all identifiers, we resolve the scopes in the order local > command > global. 
       To resolve member expressions, we recursively traverse (and compute where needed) the global context to tokenize each member.
    4. Highlight. Each of the token types is then colored accordingly using ANSI escape sequences, and the line is re-rendered with the highlighted line.

   - feat: add APIs, commands and tests for syntax-highlighting in the REPL

    A follow-up adding REPL prototype methods, in-REPL commands, and test suites for the syntax highlighter. This adds various APIs for theming in the REPL, allowing the user to configure it with their own set of themes. Another small thing I took take care of is to disable highlighting in non-TTY environments.

  ![showcase](https://github.com/user-attachments/assets/cd30fc4e-0dce-49fb-af98-9a47055386d5)


- #### Multi-line editing

  Prior to this, the REPL did support multi-line inputs using incomplete statements, but no way to edit them. Adding multi-line editing meant adding support for adding lines manually, and the ability to go up and edit like a normal editor.

    - Implementing this is not as easy as it seems. Initially, I thought, just updating the `_rli` instance and using escape sequences with the updated lines by tracking each up/down keypress event would do the trick. But internally, `readline` refreshes the stream after operations like left/right/delete etc. This meant, if we were at line 2 and the stream was refreshed, everything below that line was gone. So, to actually implement this, we had to implement manual rendering with each keypress event.

    **Implementation details**:
    1. We track each keypress event like up/down/right/left, backspace (for continuous deletion), and CTRL+O (for manually adding a new line).
    2. We also maintain line & cursor indices, and highlighted line buffers to store rendering data.
    3. After every keypress event, we visually render the remaining lines below the current line.
    4. Finally, we maintain the final `_cmd` buffer for final execution.

    Once multi-line editing was implemented, the ability to cycle through previous commands was broken, as the internal `readline` implementation handles it line by line. We listen to up/down arrow keys to override the default behavior and use our maintained `_history` buffer to support cycling through multi-line commands.

- #### Unicode table plotter

  A plot API for visualizing tabular data can be leveraged for downstream tasks like TTY rendering in the REPL or even in *jupyter* environments allowing users to easily work with tabular data when doing data analysis in the REPL (or elsewhere).

  The plot API supports data types like `Array<Array>`, `MatrixLike` (2D `<ndarray>`), `Array<Object>`, and `Object`. The API is highly configurable giving users full power over how the render looks like instead of giving them a pre-defined set of presets.
  This is how the default render looks like:

  ```
  ┌───────┬──────┬───────┐
  │  col1 │ col2 │  col3 │
  ├───────┼──────┼───────┤
  │    45 │   33 │ hello │
  │ 32.54 │ true │  null │
  └───────┴──────┴───────┘
  ```
  The plotter also supports the one-of-a-kind wrapping tables, which allows breaking the table into segmented sub-tables when given appropriate `maxOutputWidth` prop values.

  Implementing this API has been tedious (as evident from the footprint) mainly because of the number of properties and signatures it needs, to parse various datatypes and support this level of configurability.

- #### Fuzzy completions

  The initial scope was to just implement the fuzzy completions extension, but upon further discussion (and me having a lot of free time), we ended up re-writing an entirely new & improved completer engine from scratch.

    The new engine allows highlighted completions, persisted drawer, and the ability to cycle through the completions using arrow keys.

    ![completer](https://github.com/user-attachments/assets/3f7f742e-bcb3-4de6-aed5-d55f21b8de12)

    **Implementation details**:
    1. A renderer for the drawer view creates the grid-like view from available completions, with a maintained current completion index highlighted using ANSI escape sequences.
    2. Keypress events like up/down/left/right are tracked for navigation, and with each movement, the drawer is re-rendered and the selected completion is inserted to the current line.
    3. The engine is also receptive to `SIGWINCH` events meaning the terminal can be resized without any distortion.

    Fuzzy matching involves finding approximate completions that the user might be trying to type. This is different than fuzzy search as the length of the final completion shouldn't affect its relevancy.

    ![fuzzy](https://github.com/user-attachments/assets/456cf22f-d015-4107-a33b-09eaad239648)

    I wrote a fuzzy matching algorithm taking the following variables into account:

    1. Casing mismatch - *Low penalty*
    2. The first letter doesn't match - *Medium penalty*
    3. Gap between matching characters in the completion - *Medium penalty*
    4. Gap before the input was first encountered in the completion - *Low penalty* (as it already incurred the penalty from the 2nd clause)
    5. Missing character from the input in the completion - *High penalty*. This is generally not taken into account in most fuzzy completion algorithms but it can help detect spelling mistakes. The only downside being it increases the time complexity of the algorithm as we would still have to traverse the completion string even after a character was found to be missing.

    The fuzzy matching algorithm is based on a penalty-based scoring mechanism that negatively scores the completions for unfavorable characteristics in the completion string (mentioned above) that would make it a less ideal match for the input.

- #### Bracketed-paste mode

  Prior to this, pasting multi-line code into the REPL would execute it line-by-line instead of pasting it in a single prompt. Bracketed-paste mode aims to resolve this by allowing to paste without execution.

    Bracketed-paste mode is supported by most modern emulators and can be enabled/disabled via specific escape sequences. Once enabled, the emulator wraps pasted content with certain escape sequences that can then be parsed by downstream applications to detect and handle pasted content separately. We follow this very approach and disable execution when receiving paste sequences.

- #### Configuration and persistence

  Prior to this, if the user changed the settings or theme to their preference, those changes were lost once they exited the REPL. This can be annoying as there is no direct way to have a personalized REPL environment except by writing a custom initialization script.

    This adds support for having a default configuration, and the ability to load custom configuration or profiles.

    **Implementation details**:

    1. If any changes to REPL preferences are detected when exiting the REPL, the user is prompted whether they would like to save their preferences.
    2. This configuration is saved to the file from which the REPL was initialized from. Meaning the user can have their own configuration files, of the format `.stdlib_repl.json` in any of the parent directories to the `cwd`. 
    3. There is also a default configuration located at `~/.stdlib/repl.json`.

    While implementing support for configuration files, the initial plan was to also allow loading from `.stdlib_repl.js` files along with `stdlib_repl.json`. For this, I worked on a file system utility to resolve a path from a set of paths by traversing parent directories. But we later discarded the plan to support `.js` configurations as it made the idea of saving configurations more complex. So yeah, we didn't end up using this utility after all.

- #### Custom key-bindings

  Although `readline` comes with various actions that can be triggered by certain key combinations, there is no way to configure key-bindings.

  This adds support for configuring in-built `readline` editor actions and also implements some new ones inspired by *Julia*.
 
  **Implementation details**:

  1. To make the `readline` actions configurable, we just override the selected keybindings to trigger a different sequence that triggers the corresponding action once the keypress is processed by `readline`. For example, we can set a keybinding like CTRL+M to trigger the backspace action by emitting the required sequence before the CTRL+M keypress is processed.

  2. I also implemented some newer actions which can again be triggered by configurable keybindings.

  3. A challenge I faced was with key combinations involving symbols. (Ex: CTRL+/). `readline` doesn't parse such combinations into human-enriched formats and hence I had to write a custom parser (`parse_key.js`) to parse such key combinations.

-   150+ [special math functions][@stdlib/math/base/special].

    <div class="image" align="center">
        <img src="https://cdn.jsdelivr.net/gh/stdlib-js/stdlib@203839353bc74297fe641207270f7917d2bda560/docs/assets/readme/base_special_math.png" alt="Demo showcasing special math functions">
    </div>

-   35+ [probability distributions][@stdlib/stats/base/dists], with support for evaluating probability density functions (PDFs), cumulative distribution functions (CDFs), quantiles, moments, and more.

    <div class="image" align="center">
        <img src="https://cdn.jsdelivr.net/gh/stdlib-js/stdlib@e13885087939c064c69aa43ee80ea52710de5591/docs/assets/readme/base_dists.png" alt="Demo showcasing probability distributions">
    </div>

-   40+ [seedable pseudorandom number generators][@stdlib/random/base] (PRNGs).

    <div class="image" align="center">
        <img src="https://cdn.jsdelivr.net/gh/stdlib-js/stdlib@83dcd0fad98883320a8b1efc801b2fc1ed2a003d/docs/assets/readme/base_prngs.png" alt="Demo showcasing PRNGs">
    </div>
