+++
date = '2026-07-03T17:51:04-07:00'
draft = false
title = 'Information architecture is the foundation of good documentation'
summary = ""
+++

Good docs begin with architecture, that is, the invisible structure that determines whether someone can find what they need in the first place.

I've spent over a decade writing technical documentation for startups, Angular, and Bazel. The pattern holds everywhere: the best-written page in the world is useless if no one can find it, and a merely adequate page becomes (gasp) valuable if it sits exactly where a reader expects it to.

## Discoverability and design

Teams often try to solve discoverability with better search. Search helps, but it's a patch over a deeper issue. The real fix is upstream: decide on your categories, your naming conventions, your hierarchy, and stick to them. If you have to pivot later, that's ok, just be clear and intentional about it.

When a reader lands on any page in your docs, they should be able to predict, roughly, what else exists and where it lives. A "Concepts" section should always mean the same thing. A "Reference" page should always follow the same shape. That predictability is what lets people skim instead of lose time (and patience) hunting.

If you're documenting a product, the company's messaging and positioning, along with the personas that you're writing for must all be very clear. Documentation is a reflection of the product. And in a subtle way it is part of the marketing. If docs are easy to understand and navigate with key points being discoverable, then people will like your product even more.

## Consistent paradigms reduce cognitive load

The same principle applies at the page level. If one guide presents steps as numbered lists and the next presents nearly identical steps as prose paragraphs, the reader has to re-learn how to read your docs every time they switch pages. Pick a paradigm — how you present prerequisites, how you format warnings, how you structure a getting-started guide versus a how-to versus a reference page — and apply it everywhere. The goal is for the _shape_ of the documentation to become as familiar as the content itself, so readers spend their attention on the material, not on decoding your formatting choices.

## Code snippets need to compile

There's one non-negotiable in my own work: every code snippet in a doc has to actually run. It's tempting to write "illustrative" code that's close enough, especially under deadline pressure. But a snippet with a typo, an outdated API call, or a missing import actively erodes trust. A reader who copies broken code and hits an error will doubt every other snippet in your docs, even the correct ones. Testing snippets is part of what gives the snippet the authority of documentation rather than a suggestion.

For example, when documenting a Python library, I always include working code:

```python
from mylib import configure

# Initialize the library with production settings
config = configure(
    debug=False,
    timeout=30,
    retries=3
)

# Use it immediately
result = config.run()
print(f"Success: {result}")
```

Every line has to compile, every import has to exist, every variable has to be defined.

## Key takeaway

Architecture, consistency, and tested code all point to the same goal of reducing the distance between "I have a question" and "I have a working answer." It's worthwhile work. It's the difference between documentation people tolerate and documentation people trust.
