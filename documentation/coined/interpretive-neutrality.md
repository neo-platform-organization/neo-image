# Interpretive neutrality

Coined in the neo design discussion on 2026-11-27.

A structure's authoring form does not permanently determine its meaning.
Programs may inspect the same underlying graph through object-oriented,
functional or structural interpretations, regardless of how it was authored.

Warp is interpretively neutral; Weft and Knight provide particular
interpretations. A default interpretation is not an irreversible classification.
The mechanism for selecting an interpretation remains an implementation choice.

Inspection does not automatically execute, mutate or grant authority over the
data. Viewing an object as a function does not by itself establish purity:
functional execution must account for state dependencies through its contract,
such as explicit inputs or a snapshot. Active interpretations should be visible
and inspectable. Universal alternate interpretation is an intended design
principle, not a claim of completed VM support.

Related ideas include reflection, metaprogramming, multiple views and extensible
languages. This term does not establish that the combination is historically
unique. See also [generative coherence](generative-coherence.md).
