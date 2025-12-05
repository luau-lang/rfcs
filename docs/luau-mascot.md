# An official mascot for Luau

**Status**: Implemented

## Summary

Luau's branding is currently entirely based around a rotated square logo in a particular shade of blue.
This works well for professional presentation on the website and so forth, but there is a gap for
elements of the brand identity that are more personable and easier for people to identify with. Toward
that end, this RFC proposes Luau adopt an official mascot for the language.

## Motivation

The primary motivation is that Luau's current branding consists entirely of a logo, which is relatively
cold and harder for people to identify with. In the Rust community, there's been a lot of success with
[Ferris the crab](https://www.rustacean.net/) as a language mascot, leading to a variety of merchandise
and fanart from the language community. Ferris serves as an icon that Rust users are much happier to
identify with and portray than just the language's logo on its own, especially in less formal settings.
We'd like to be able to provide a mascot that can serve a similar role for Luau!

## Design

This RFC proposes that Luau adopt the Hawaiian monk seal as a mascot. There's a few reasons for this
choice. First, we want to have a brand and identity that is more distinct from Lua itself, and 
therefore we want to choose a mascot that is _not_ based on the "moon" theme that is central to
Lua's brand identity. Second, Luau's name itself draws from Hawaiian culture since it is named after
a traditional Hawaiaan feast. The Hawaiian monk seal is the state mammal of Hawaii, and one of only
two mammals endemic to the islands. Third, the Hawaiian monk seal is an endangered species and we
can, at a minimum, use the choice of mascot to direct energy towards conservation efforts for this
adorable creature.

For a concrete mascot design, I commissioned [Alex Mullins (@AlexMarieFina)](https://github.com/AlexMarieFina)
to produce a cartoon-style drawing of a Hawaiian monk seal in Luau's brand blue color. 
This mascot is Hina the Seal.

![Hina the Seal](mascot.png)

We propose naming her Hina, after the Hawaiian goddess of the moon. The word moon itself in Hawaiian
(_mahina_) is derived from this name, and it pays a small homage to Luau's "moon" roots in Lua
while staying within the existing theme and not being too heavy-handed.

## Drawbacks

We may prefer to remain "neutral" and avoid expanding the scope of Luau's branding to leave room for
more organic adoption of a mascot within the community. 

## Alternatives

In early discussions leading up to my commission, the two most serious suggestions for alternative
animals were rabbits (owing to the eastern cultural tradition of "moon rabbits") and a beluga
(thanks to the pun, "beluauga"). We could get some alternative art proposals for either mascot, but
an obvious downside to the beluga, in particular, is that we would have a blue whale mascot which
conflicts with a certain [already well-known blue whale mascot](https://docker.com).

Additionally, there was some discussion about naming options. The main thrusts for name suggestions
were broadly in three categories: names starting with L (referencing Lua), names starting with S
(for alliteration with Seal), and names with alignment around the moon and Hawaiian and Polynesian
culture. Notable suggestions from these groups included Luna and Lucy, Sera and Selene, 
and Mahina and Hina. This proposal chose `Hina` on the basis of its thematic fit, brevity (single
syllable), and distinctiveness (e.g. Luna is very close to Lune and Luau, Selene is already the name
of a community tool, etc.).
