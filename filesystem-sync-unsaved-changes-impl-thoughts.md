---
title: filesystem-sync-unsaved-changes-impl-thoughts
date: 2023-01-26
---

So I'm struggling right now with implementing the backlog writing in a way that feels even _remotely_ clean...

Of course, the technical side of just writing the serialized backlog is fairly trivial;
personally I'd think just keep the file open, hold an io.Writer and then serialize and write on `w`.

The first question I have to wrestle with is

> am I happy having the filesystem interaction code in the `model` package?
> I don't think so, I think I'd like to separate it out and be able to generify above day, backlog, ...

Now let's try to implement a check for unwritten changes.
This also does not necessarily belong in the `model`, so I figure it should be abstracted as well.
Suddenly our generic code has to know when a mutating operation takes place on the day, backlog, ..., each of which might have a very different interface; and we're working in a language that doesn't offer any mechanism for denoting (im)mutability...

So I figure we might have a pure _data_ and _operations_ model in `internal/model` and then orthogonal to it a "less pure" filesystem handling, change tracking, ... _operations_ model e.g. in `internal/control/fsmodel` or something?
There then the model-components could be replicated _operationally_ such that each _mutating_ operation notes internally that a change has occurred (I'd say to use a `sync/atomic.Bool`);
it would further have to offer e.g. a `Write() error` member.

What does a ...-wrapper need?
 - Serialize(v any) []byte
 - NoteChange()
 - Write(w io.Writer, []byte) error

---

Let's think about this another way...
What if we had some central filesystem-syncing component (FSSC), i.E. a thing that kept track of necessary and performed writes?
Anytime I do a mutating operation on a day or the backlog (or ...) I register with FSSC that a write to <thing> is in order (to be synced on the FS-side).
When I write a day to file the FSSC gets notified too, and is able to remove the write-necessity it might have stored.

Consider:

```
   ME                                FSSC

                                        []

   change 2022-12-14                 -> [ 2022-12-14 ]

   change backlog and 2023-01-12     -> [ 2022-12-14 , backlog , 2023-01-12 ]

   write backlog                     -> [ 2022-12-14 , 2023-01-12 ]

   try to quit                       HOLD UP, you still have these unsaved:
                                        [ 2022-12-14 , 2023-01-12 ]
                                     (It could even have handles to the
                                      necessary write operations stored such
                                      that it could be used to perform all
                                      necessary writes if the user requests to
                                      do that.)
```


