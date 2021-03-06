.. _signaturegeneration-chapter:

====================
Signature Generation
====================

Introduction
============

The :ref:`processor-chapter` creates an overall signature for a crash based on
the signatures of the stack frame of the crashing thread. It walks the stack
from the frame with the lowest number (the top of the stack) applying rules and
accumulating a list of signatures found to be relevant. Once the rules are done,
the list of signatures is concatenated into a single string. That single string
become the crash's overall signature.


Normalization
=============

Before any frame signatures are considered, they are normalized. This is just a
string formating change. Runs of spaces are compressed to just one space. Commas
are insured to always be followed by a space, integer values are replaced by
'int'. Signatures that match the signaturesWithLineNumbersRegEx regular
expression are combined with their source code line. Frames that have no
function information are written as sourcecode/line number pairs. If no source
code is available, it tries to find a module/address pair. Failing that, it
falls back to just an address.


The SkipList Rules
==================

The signature is generated by walking through each stack frame considering its
'name' (as normalized above). Frames / names are skipped or added to the
signature list according to the rules. When a signature list is complete, it is
converted to string by concatenating the frame names with spaces and a vertical
bar between each name, for example: objc_msgSend | IdleTimerVector is the
signature for a stack that contained (irrelevant frames), "objc_msgSend",
"IdleTimerVector" which matched neither prefix nor irrelevant regular
expressions and possibly other frames which did not become part of the
signature.


regular expressions
===================

Each SkipList rule is a regular expression. Typically, it takes the form of an
alternation of frame names, but any legal regular expression can be used.
Regular expression alternation syntax is a|b|c: Match on 'a' or 'b' or 'c'. This
work is done in Python, so use Python Regular Expression Syntax


signatureSentinels
==================

A typical rule might be: "_purecall".

This is the first rule to be applied. The code iterates through the stack frame,
throwing away everything it finds until it encounters a match to this regular
expression or the end of the stack. If it finds a match, it passes all the
frames after the match to the next step. If it finds no match, it passes the
whole list of frames to the next step.


irrelevantSignatureRegEx
========================

A typical rule might be::

    @0x0-9a-fA-F{2,}|@0x1-9a-fA-F|RaiseException|CxxThrowException

A frame which matches this regular expression will be appended to the signature
only if a prefix frame has already been seen (see next rule).


prefixSignatureRegEx
====================

A typical rule might be::

    @0x0|strchr|strstr|strlen|PL_strlen|strcmp|wcslen|memcpy|memmove|memcmp|malloc|realloc|objc_msgSend

though at Mozilla it has grown much longer.

This is the rule that generates compound signatures. A frame that matches this
regular expression changes the state of the machine to 'seen prefix'. In 'seen
prefix' state, irrelevant or prefix frames are appended. As soon as a frame is
neither, it is appended and the signature list is complete.

Once the signature list is complete, the signature is generated as mentioned
above
