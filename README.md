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

Before opening an RFC, consider how your change interacts with other existing code. Luau has a fairly generous backwards compatibility guarantee: there are no tools for versioning such as epochs, language editions, or standards. Some common backwards compatibility concerns include: how might this feature interact with paren-less function calls in Luau? How might it interact with code using your new keyword as an identifier?

### Is your feature compositional?

We often propose new features to solve a problem users currently have. However, language changes do not exist in a vacuum, and not all features will compose nicely with each other. We consider removing features impossible: features that do not compose well together will be stuck in that state _forever_.

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

## Process

To open an RFC, a Pull Request must be opened which creates a new Markdown file in the `docs/` folder. The RFCs should follow the template `TEMPLATE.md`, and should have a file name that is a short human readable description of the feature (using lowercase alphanumeric characters and dashes only). They should use the general area of the RFC as a prefix, e.g. `syntax-generic-functions.md` or `function-debug-info.md`.

RFCs follow an explicit set of stages heavily inspired by the [TC39 process for evolving ECMAScript (JavaScript)](https://tc39.es/process-document/). Unless otherwise specified, decisions referenced in this document can only be made by consensus amongst the “Luau reviewers”, which is the set of people who are members of the [luau-reviewer GitHub team](https://github.com/orgs/luau-lang/teams/luau-reviewer).

### (1) Proposal

**Status**: All RFCs start out in this state. The proposer, whoever opened the RFC pull request, will provide a problem and solution sketch following the template.

**What Happens**: Participants discuss the RFC, ask design questions, note any design issues or alternate solutions, etc.

**Next Stage**: A Luau reviewer must champion this RFC.

**Off Track**: A proposal without a champion may sit indefinitely, or may be closed if it is proposing something covered by an anti-RFC (an RFC stating why Luau won’t implement a class of features, see below).

### (2) Draft

**Status**: A proposed RFC that has a reviewer champion.

**What Happens**: The implementor or champion begins work on a prototype and the RFC is brought to Luau reviewers.

**Next Stage**:

1. Luau reviewers reach a consensus that this feature is a positive addition to Luau.
1. There exists a prototype implementation: it does not need to be feature complete, but it should demonstrate core functionality.

**Off Track**: Luau reviewers may fail to reach a consensus for a reason that will be recorded on the RFC, or may require further elaboration of the design constraints / goals. Proposals that cannot reach a consensus may remain in **Draft** indefinitely.

### (3) Implementing

**Status**: This feature has a prototype implementation either merged into Luau behind a `Debug*` flag or as an open PR against the Luau repo.

**What Happens**: The implementation is updated to be feature complete with respect to the design in the RFC, any edge cases that arise are addressed, discussed, and then added to the RFC.

**Next Stage**:

1. The implementor (generally the proposer) and a reviewer agree that the implementation meets the RFC specification.
1. At this point, Luau reviewers will decide whether the RFC needs additional vetting. For otherwise “simple” changes, they can move straight to Stable instead of going through Experimental.
1. If going through **Experimental**, the prototype implementation should be merged into the Luau repo and guarded by a flag starting with `LuauExperimental*`. If it has already been merged into the repo behind a `Debug*` flag, then it will be renamed to `LuauExperimental*`.

**Off Track**: Luau reviewers may decide that, in the process of building the feature, the implementor or champion has uncovered design flaws which may mean that the RFC does not represent a coherent addition to Luau. This would move the feature back to Draft assuming its champion still supports it, or may entail an anti-RFC being opened explaining why the feature has been rejected from Luau in perpetuity.

### (4) Experimental

**Status**: This feature is ready for undirected community testing, but is not yet officially stable.

**What Happens**: Until moving to the next stage, the feature is available as a `LuauExperimental*` flag. The champion and proposer collect feedback to ensure that the design as-is is suitable for moving out of experimental.

**Next Stage**: Luau reviewers decide that the RFC has been sufficiently tested and can elect to stabilize it.

**Off Track**: community feedback as received by the Luau reviewers indicates that this RFC has some sort of issue, for example:

* It solves the problems stated, but otherwise feels awkward to use.
* Some other aspect of the feature that was not considered is brought to light.

This would move the feature back to **Implementing**, assuming that there are minor design revisions, or **Draft** in the case of major revisions. If the Luau reviewers are not able to reach a consensus on moving this to Stable, and the scope of changes are unclear, then this moves back to **Draft**.

### (5) Stable

Status: This feature is now a part of the Luau language and specification

What Happens: The RFC pull request is now merged in, indicating that this RFC process has been finished. The flag can now be clipped.

## Anti-RFCs

Once in a blue moon, Luau reviewers may decide to release an "anti-RFC." This is to document why Luau doesn’t implement a commonly requested feature. A recent example of this is ["Do not implement JSX"](https://github.com/luau-lang/rfcs/pull/276).

These are like RFCs in that they are technical documents, but otherwise:

1. They are only authored by Luau reviewers (anti-RFCs from non-reviewers will not be accepted);
1. They follow an expedited process (essentially just proposal and stable, per the above);
1. They follow a different format:

```markdown
# Summary

A brief overview of the feature and why it will not be added.

# Motivation

Details as to why the feature will not be added.

# Conditions to Reconsider

Any reasons that this decision may be revisited, to allow for the anti-RFC to be responded to with an anti-anti-RFC.
```
