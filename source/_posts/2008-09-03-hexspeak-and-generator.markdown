---
layout: post
title: "Hexspeak and Generator"
date: 2008-09-03 00:03
comments: true
categories: python
---

EDIT: 2011-04-21 While browsing my old blog posts, I found the code here can be greatly improved. Here it is:

```python
def words():
    with open('/usr/share/dict/words', 'r') as f:
        return (x.strip().upper() for x in f.readlines())

MAPPING = {'A':'A', 'B':'B', 'C':'C', 'D':'D',
           'E':'E', 'F':'F', 'O':'0', 'S':'5', 'I':'1'}

def main():
    is_hexword = lambda word: all(ch in MAPPING for ch in word)
    for word in filter(is_hexword, words()):
        print word, "\t", ''.join(MAPPING.get(ch, ch) for ch in word)

if __name__ == '__main__':
    main()
```



So, it’s a relatively slow day at work, and I’ve been “stumbling upon” on Wikipedia when I found this:

[http://en.wikipedia.org/wiki/Hexspeak](http://en.wikipedia.org/wiki/Hexspeak)

It’s pretty interesting, because I remember [OS161](http://www.eecs.harvard.edu/~syrah/os161/) from my operating system course where they use 0xDEADBEEF as a value for uninitialized pointers. So I decided to write a small Python program that finds me all the “Hexspeak” words from a regular English dictionary.

Beginning by finding a plain text English dictionary. I know Linux has a file ‘words’ in the file system somewhere. A little search gives me its location: /usr/share/dict/words

Now onto Python coding:

Define a character-to-hex map:

```python
HEXSPEAK_DICT={'A':'A', 'B':'B', 'C':'C', 'D':'D', 'E':'E', 'F':'F', 'O':'0', 'L':'1', 'S':'5', 'I':'1', }
```

Define a method to open the dictionary and return a list of words:

```python
DICT = '/usr/share/dictdef words(file=DICT):
    f = open(file, 'r')
    retval = f.readlines()
    f.close()
    retval = [ x.strip().upper() for x in retval ]
    return retval
```

Define a method to print out the collection of hexspeaks

```python
def print_dict(dict):
    for key in dict.keys():
        print "%-20s\t%s"%(key, dict[key])
```

The program itself is pretty clear. We go through every character in every word in the dictionary. If the character is not in a permitted Hexspeak character, we through out the word. Otherwise, we take the character and translate it into a hexspeak character if necessary. Finally we print out the whole thing.

Running it, it gives me a list of hexspeak words. Everything is cool.

Now, the idea of generator has been around for a while. If you’re operating on a list, instead of loading the list into memory at once, you can use a generator and return one piece at a time, so it can save some resources. Granted, I’ve never used generators before, so I decided to experiement using a generator.

Here’s the code. Everytime xwords() is called, it returns the next processed value in the list.

```python
def xwords(file=DICT):
    f = open(file, 'r')
    retval = f.readlines()
    f.close()

    for rv in retval:
        yield rv.strip().upper()
```

Now we need to modify WORDS to make it point to xwords

```python
WORDS = xwords()
```

Running it, it gives the same result. So my experiment with generator succeeded! Hooray!

However, a little profiling on the program contradicts the intuition that using a generator is faster. I used Python’s timeit module for profiling:

```bash
python -m timeit -n 3 'import hexspeak; hexspeak.main()'
```

Using generator:

```
3 loops, best of 3: 565 msec per loop
```

Using list:

```
3 loops, best of 3: 549 msec per loop
```

Hmmm, so not only using a generator doesn’t save me any time, it actually got beaten by a tad bit by the plain ol’ list implementation…I’m sure I’m missing some points here. A good topic for a blog post for another day.

Here's a list of [Hexspeak words](/2008/09/03/hexspeak-word-list)
