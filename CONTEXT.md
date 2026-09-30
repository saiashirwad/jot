# jot

A macOS menu-bar tool for collecting notes on text you are reading, by voice or by typing, and handing them to an agent in one paste.

## Language

**Annotation**:
One Excerpt, one Note and one Source, captured together.
_Avoid_: Highlight, clip, entry

**Excerpt**:
The text selected in any app when an Annotation starts. May be empty.
_Avoid_: Selection, quote, snippet

**Note**:
What the user says or types about the Excerpt.
_Avoid_: Comment, memo, transcript

**Source**:
Where the Excerpt came from: app, window title, URL when there is one, and time.
_Avoid_: Origin, context

**Voice annotation**:
An Annotation whose Note starts as speech, transcribed live, then edited before saving.

**Text annotation**:
An Annotation whose Note is typed.

**Stack**:
The ordered set of Annotations not yet handed off. There is one Stack.
_Avoid_: Queue, list, buffer, session

**Hand-off**:
Pasting the whole Stack into the focused app in one step, which then empties the Stack.
_Avoid_: Export, flush, dump
