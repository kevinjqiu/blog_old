---
layout: post
title: "Enable Auto-complete in a PDB session"
date: 2009-06-12 09:25
comments: true
categories: python, debugging
---

EDIT: 2012-06-19 There are so many other much better options now that renders recipe obsolete. See [pdb++](https://bitbucket.org/antocuni/pdb/src) or [ipdb](https://github.com/gotcha/ipdb)

Pretty simple actually…Just put the following code in `~/.pdbrc` and then you can use the Tab key during a PDB session to see the available attributes of the current context.

```python
import rlcompleter
pdb.Pdb.complete=rlcompleter.Completer(locals()).complete
```
