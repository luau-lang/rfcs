# Feature name

## Summary

Allow `table.clone` to copy tables with locked metatables.

## Motivation

Currently `table.clone` cannot create shallow copies of a table if it has a locked metatable. The dictionary definition for shallow copying is to copy all the fields of a table to a new table but not doing so recursively. There is no mention of metatables in the first place so most users expect said behavior just to work, if they haven't taken a look at the manual of course.

## Design

When `table.clone` attempts to clone a table with a locked metatable the table is shallow copied with the exception of not assigning the metatable to the shallow copy.

## Drawbacks

There are no drawbacks to this. You can already create a shallow copy of a table with a locked metatable (without assigning the metatable to the copy) so there is no downside in allowing `table.clone` to do the same.

## Alternatives

An alternate mode of action would be to also clone the locked metatable to the new copy. This would however come with a few downsides including potential security and usability issues.
