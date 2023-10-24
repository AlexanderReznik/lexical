# Lexical's useHistory adds support for Undo and Redo

Part of undo and redo requires supporting coalescing of certain dynamic operations such as continuous typing.
The below provides background information regarding the continuous typing undo coalescing feature.

**Undo Coalescing of Typed Text**

**TL;DR**
When working with text editors, users expect a well polished undo experience. A key part of this is a smooth, well thought out workflow when undoing continuous typing events. For this post, strict typing events may be limited to typing characters, backspacing and forward deleting. Users generally expect that continuous typing should fully undo with a single undo gesture and that they don’t require separate undo gestures for each character.
The algorithm that covers this is “undo coalescing of typed text” and there’s a certain degree of nuance involved. This post covers that nuance while attempting to avoid prescriptive implementation details.

**What Kind of Edits Require Coalescing?**
As mentioned above, continuous typing should be coalesced into a single undo action. Continuous typing is a loaded phrase and we’ll get into some details below.
There are other types of edits that require undo coalescing such as rapidly resizing an image or rapidly changing the font size of selected text. These latter two examples fall under what might be termed “dynamic operations” and their implementation might vary quite a bit from coalescing of typing.

**What Constitutes Continuous Typing?**
For the purposes of undo coalescing, continuous typing events are safely limited to these three actions:

1. Forward typing characters (heads up, composed character typing discussed later).
2. Deleting backwards, e.g. backspace gestures including backspace key presses.
3. Forward deletion, e.g. forward delete gestures including Delete key presses.

Any interruption to the above breaks the notion of continuous typing. Note that switching between either of these three actions may itself be considered an interruption of continuous typing or it may not. This ends up being a usability decision. In my opinion, when examining various word processors, switching between these three actions should maintain continuity, because users often make typos or corrections in the flow of typing and therefore would lean towards the whole sequence of typing, backspacing and deleting to be within a single undoable session.
Note that Google Docs and Quip treat a switch between typing, backspacing and deleting as disruptions to the continuous typing sessions.

**Start with Selected Text and Followup with Typing**
Let’s start with some selected text. The user may have selected the text directly, as a result of pasting text or through some other sequence of steps. When followed by continuous typing, the initial selection should be included along with the following typing events as part of a single coalesced undo session.

**Timeouts**
Text editing is generally a modeless user experience and so just about any event beyond the above typing actions will disrupt continuous typing. The first and foremost is a timeout.
Quip seems to require continuous typing to occur within 1 second intervals, whereas Google Docs requires about 2 seconds. In my opinion, the timeout may stretch to as long as 3 to 5 seconds. The more time between typing events, the more chance the user has to intentionally input a block of text, a thought or concept.
The use of a timeout is not necessarily required, and the amount of time is certainly up for debate. Collaborative co-editing also makes use of timeouts when grouping text that is merged across clients. Longer timeouts tend to make collaboration a smoother, less disruptive experience, because receiving other client input while expressing one’s thoughts tends to be unhelpful.
Implementers should consider the correct timeout based on their customer expectations and related collaborative features that might share this timeout.

**Hard and Soft Returns**
Hard returns as generated by pressing the Return key tend to disrupt and thereby reset continuous typing.
For Google Docs and Quip, soft returns as generated by pressing shift+Return key do not terminate continuous typing. This makes sense in that the soft return ends text wrap but does not end the paragraph.

**Changing Selections and Key Focus**
Any change to the selection by the user should terminate coalescing. This includes:

- Arrow gesture navigation.
- Gestures to select a word, a sentence or a paragraph.
- Any other selection change that does not move the blinking caret to the next or previous character by way of the above mentioned typing events.

Additionally, ending keyboard focus on the editing window, including moving focus to another editing window or a different view within the same window should also terminate coalescing. This also includes setting focus to modal popovers, such as, for example, a hyperlink property editor.

**Composed Characters**
Character composition occurs when typing diacritics, CJK and even emojis. The process usually involves at least these three phases:

- Start composition.
- Replace composed text with other text and remain composing. This step may occur many times within a single composition session.
- Stop composition either through accepting the text or canceling the latest input.

Coalescing should terminate at start composition. Replacing composed text should generally be coalesced. Stopping composition may or may not reset coalescing depending upon the implementation. Quip appears to try to coalesce multiple composed characters, but disrupts coalescing when switching between composed and non-composed characters. Google Docs appears to disrupt coalescing more frequently. Both coalesce the text replacements.

**Auto Correct, Spelling Corrections and Text Substitutions**
Text substitution behaves much like composed characters. When the text is transformed, the coalescing should terminate and the transformed text becomes the first coalesced entry in the next undo session.

**Copy & Pasting**
Any actions like copy and paste typically trigger events that are not part of the 3 continuous typing actions described above. So, copy and paste also terminates coalescing.

**In Conclusion**
Text undo coalescing greatly improves the usability of undo in that it allows the user to bypass all the intermediate events that went into forming or re-forming text within a paragraph. Given the modeless nature of text editing, limiting continuous typing to strictly three events allows for a robust and clean implementation. Implementers will need tight bottlenecks to trap selection changes, key focus changes as well as changes from collaborative clients. While the above is not complete by any means, I hope readers will find it helpful.