Background
===

New user facing changes to Luau require deep consideration due to myriad downstream impacts. An RFC process ensures that we are able to consider
the many implications of those changes and can effectively evolve the language in the best way possible.

In addition to these questions, we also need to consider that every addition carries a cost, and too many features will result in a language that is harder to learn, harder to implement and ensure consistent implementation quality throughout, slower, etc. In addition, any language is greater than the sum of its parts and features often have non-intuitive interactions with each other.

Reversing these decisions is incredibly costly and can be impossible due to backwards compatibility implications, so all user facing changes to Luau language and core libraries must go through an RFC process.

RFC Design Principles
===

This section (non-exhaustively) articulates some design principles RFCs should follow.

### Examples, examples, examples!

Talking about code in the abstract is hard. The more examples you provide of your feature, the easier it is for everyone to participate in the discussion and visualize what life will be like with it. Feel free to draw upon prior art, especially if you cite the design of those features.

### Is your change backwards compatible?

Before opening an RFC, consider how your change interacts with other existing code. Luau has a fairly generous backwards compatibility guarantee: there are no tools for versioning such as epochs, language editions, or standards. If you are introducing new syntax, how might this interact with paren-less function calls in Luau? How might it interact with code using your new keyword as an identifier? You should have good answers to these questions.

### Is your feature compositional?

We often propose new features to solve a problem users currently have. However, language changes do not exist in a vacuum, and not all features will compose nicely with each other. We consider removing features impossible, and judge new language features with that bar in mind.

### Do I need to care about performance?

When we design language features, we should consider whether the feature is good for the language as a whole. Generic statements about a feature being non-performant are not useful without evidence, and oftentimes it is impossible to know without implementing the feature. Historically, the first implementation of a new language feature might be non-performant, but with additional time and care, can be made much faster. The introduction of your change should not regress performance for users who decide not to use this feature. Features developed under the RFC process explicitly spend time in an experimental/debug state internally so any performance issues may be surfaced at this time.

### Your change is runtime focused, what about the type system?

Changes to the runtime should also articulate what changes to the type system will occur. The type system may reject programs, but is ultimately an optional extension to the language. Additionally, changes that are type system forward need to consider the runtime behavior: types are optional and are not present at runtime, so features which depend on type inference for runtime behavior will be rejected outright.

### Other questions you should think about:
Whenever new syntax is introduced, you should also think about:

- Is it easy for machines and humans to parse?
- Is it stylistically coherent with the rest of the language?
- Does it present challenges with editor integration like autocomplete?

For changes in semantics, we should be asking:

- Is behavior easy to understand and non-surprising?
- Can it be sandboxed assuming malicious usage?
- Is it compatible with type checking and other forms of static analysis?

For new standard library functions, we should be asking:

- Is the new functionality used/useful often enough in existing code?
- Does the standard library implementation carry important performance benefits that can't be achieved in user code?
- Is the behavior general and unambiguous, as opposed to solving a problem / providing an interface that's too specific?
- Is the function interface amenable to type checking / linting?



## Non-constraints

### Backtracking

Historically, the Luau team has rejected RFCs that would require parser backtracking. This was originally done to keep our LL(2) parser simple to write, easy to reason about, and performant. 

After much deliberation, the Luau team is lifting the blanket restriction on language features that involve parser backtracking or unbounded lookahead (as of Oct 15, 2026). Our new policy is to consider new features that would benefit from the lifting of this restriction on a case by case basis, paying careful attention to performance, the editor experience (syntax highlighting, autocomplete etc), and ergonomics. Our aim is still to ensure that our parser still remains reasonably fast, and that the error messages are of high quality.

Parser changes will still be rejected if they are completely ambiguous.


Process
===

To open an RFC, a Pull Request must be opened which creates a new Markdown file in `docs/` folder. The RFCs should follow the template `TEMPLATE.md`, and should have a file name that is a short human readable description of the feature (using lowercase alphanumeric characters and dashes only). Try using the general area of the RFC as a prefix, e.g. `syntax-generic-functions.md` or `function-debug-info.md`.

Every open RFC will be open for at least two calendar weeks. This is to make sure that there is sufficient time to review the proposal and raise concerns or suggest improvements. The discussion points should be reflected on the PR comments; when discussion happens outside of the comment stream, the points salient to the RFC should be summarized as a followup.

When the initial comment period expires, the RFC can be merged if there's consensus that the change is important and that the details of the syntax/semantics presented are workable. The decision to merge the RFC is made by the Luau team.

When revisions on the RFC text that affect syntax/semantics are suggested, they need to be incorporated before a RFC is merged; a merged RFC represents a maximally accurate version of the language change that is going to be implemented.

Each RFC will be assigned a shepherd.  This person will support you in getting feedback, request necessary changes, and ultimately either accept (and merge) the RFC or turn it away.

In some cases RFCs may contain conditional compatibility clauses. E.g. there are cases where a change is potentially not backwards compatible, but is believed to be substantially beneficial that it can be implemented if, in practice, the backwards compatibility implications are minimal. As a strawman example, if we wanted to introduce a non-context-specific keyword `globallycoherent`, we would be able to do so if our analysis of Luau code (based on the Roblox platform at the moment) informs us that no script in existence uses this keyword. In cases like this an RFC may need to be revised after the initial implementation attempt based on the data that we gather.

In general, RFCs can also be updated after merging to make the language of the RFC more clear, but should not change their meaning. When a new feature is built on top of an existing feature that has an RFC, a new RFC should be created instead of editing an existing RFC.

When there's no consensus that the feature is broadly beneficial and can be implemented, an RFC will be closed. The decision to close the RFC is made by the Luau team.

Note that in some cases an RFC may be closed because we don't have sufficient data or believe that at this point in time, the stars do not line up sufficiently for this change to be worthwhile, but this doesn't mean that it may never be considered again; an RFC PR may be reopened if new data is available since the original discussion, or if the PR has changed substantially to address the core problems raised in the prior round.

Implementation
===

When an RFC gets merged, the feature *can* be implemented; however, there's no set timeline for that implementation. In some cases implementation may land in a matter of days after an RFC is merged, in some it may take months.

To avoid having permanently stale RFCs, in rare cases Luau team can *remove* a previously merged RFC when the landscape is believed to change enough for a feature like this to warrant further discussion.

When an RFC is implemented and the implementation is enabled via feature flags, RFC should be updated to include "**Status**: Implemented" at the top level (before *Summary* section).
