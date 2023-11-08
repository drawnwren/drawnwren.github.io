+++
title = "the perils of the in memory filesystem"
slug = "in-memory-filesystem"
+++

## Problem Specification

An extremely common interview question is the "in memory filesystem",

Here is an example prompt that I received in the past: 

> Design an in-memory file system class, which supports these operations:
>
> ls : takes as a parameter a string. If the string is a file path, return the file name. If the string is a directory path, return a list of file & child directory names in the directory.
>
>
> mkdir: takes a path, and creates a directory at it if it does not already exist. Intermediate directories that don’t exist should also be created.
>
> addContentsToFile: Given a file path & file content (string), if the file exists then append to its content, otherwise create a new file and store the contents there.
>
> getContentsFromFile: Given a file path, return file contents in string format.


and here is the [relevant LeetCode problem](https://leetcode.com/problems/design-in-memory-file-system/description/).

The first thing we notice is that there a few subtleties which make these
prompts different. We'll explore these subtleties by exploring a python
solution to the leetcode problem and translating it to rust. 

## Basic Python Implementation

Generally, interviewers expect you to write something like this in python and
produce code that looks like the following:

```python
from collections import defaultdict
class Node:
    def __init__(self):
        self.child=defaultdict(Node)
        self.content=""
        
class FileSystem(object):

    def __init__(self):
        self.root=Node()
        
    def find(self,path):#find and return node at path.
        curr=self.root
        if len(path)==1:
            return self.root
        for word in path.split("/")[1:]:
            curr=curr.child[word]
        return curr
        
    def ls(self, path):
        curr=self.find(path)
        if curr.content:#file path,return file name
            return [path.split('/')[-1]]
        return sorted(curr.child.keys())
		
    def mkdir(self, path):
        self.find(path)

    def addContentToFile(self, filePath, content):
        curr=self.find(filePath)
        curr.content+=content

    def readContentFromFile(self, filePath):
        curr=self.find(filePath)
        return curr.content
```

# Node 

So, let's suppose we want to do something similar and translate this code into
Rust.

We start with the `Node` class. A direct translation would look something like
the following.

```rust
use std::collections::HashMap;
struct Node {
    child: HashMap<String,Node>,
    content: String

}

impl Node {
    pub fn new() -> Self {
        Self {
            child: HashMap::new(),
            content: "".to_string()
        }
    }
}
```

Immediately, we run into our first problem. Rust doesn't allow recursive data
types like this because there's no way to know the size of `Node` at compile
time. Luckily, that's pretty straightforward. We can just fix that by inserting
a `Box`.

```rust
use std::collections::HashMap;

struct Node {
    child: HashMap<String,Box<Node>>,
    content: String

}

impl Node {
    pub fn new() -> Self {
        Self {
            child: Box::new(HashMap::new()),
            content: "".to_string()
        }
    }
}
```

This is only a slight change. Now, `child` contains a `HashMap` which contains
a `Box`. The `Node`s while be stored on the heap, and this change will flow
through to create a bit more complexity later.

## FileSystem class

Let's continue to look at the `FileSystem` class.

```python
class FileSystem(object):

    def __init__(self):
        self.root=Node()

    ...
        
```

becomes,

```rust
struct FileSystem {
    root: Node
}

impl FileSystem {
    pub fn new() -> Self{
        Self {
            root: Node::new()
        }
    }
}
```

And this is probably the simplest conversion of the whole post.

### Find 

```python
    def find(self,path):#find and return node at path.
        curr=self.root
        if len(path)==1:
            return self.root
        for word in path.split("/")[1:]:
            curr=curr.child[word]
        return curr
```

This is where things start to get tricky. We'll come back to this
implementation a few times.

```rust  
impl FileSystem {
       pub fn find(&mut self, path: String) -> &mut Node {
            let mut curr = &mut self.root; // &mut Node
            if path.len() == 1 {
                return &mut self.root // &mut Node
            }
            for word in path.split("/") {
                if let Some(result) = curr.child.get(word) {
                    curr = result.as_mut(); // &mut Node
                }
            }
            return curr
       }
}
```

### Mkdir

Now, let's look at mkdir. The python version relies on the behavior of
defaultdict to insert unfound paths.

```rust
impl FileSystem {
    pub fn mkdir(&mut self, path: String) {
        self.find(path);
    }
}
```

Let's go back and update `find` so that it uses the `entry` api to get the same
behavior.


```rust  
impl FileSystem {
       pub fn find(&mut self, path: String) -> &mut Node {
            // we need to move the path check first to avoid a double borrow
            if path.len() == 1 {
                return &mut self.root // &mut Node
            }

            let mut curr = &mut self.root; // &mut Node
            
            // advance the iterator once to get past the first null split
            for word in path.split("/").skip(1) {
                curr = curr.child.entry(word).or_insert(Box::new(Node::new())); // &mut Node
            }
            return curr
       }
}
```

That makes the rest of the implementation flow fairly straightforwardly, 

## The rest

```rust
impl FileSystem {
    pub fn ls(&mut self, path: String) -> Vec<String> {
        let curr = self.find(path);
        if curr.content != "" {
            return vec![path.split("/").last().unwrap()]
        }
        let mut ans = curr.child.keys().collect();
        ans.sort();
        ans
    }
    pub fn add_content_to_file(&mut self, path: String, content: String) {
        let curr = self.find(path);
        curr.is_file = true;
        curr.content.push_str(&content);
    }
    pub fn read_content_from_file(&mut self, path: String) -> String {
        let curr = self.find(path);
        curr.content
    }
}
```

## Uh-oh

But wait, now whenever we call `FileSystem::ls(path)` we allocate the memory
for a directory structure. This makes the following test fail, 

```rust
    let mut fs = FileSystem::new();
    fs.ls("/a/b/c");
    fs.ls("/a") // => outputs "b"
```

The solution also doesn't allow for the existence of files with empty strings,

```rust
    let mut fs = FileSystem::new();
    fs.add_content_to_file("/a/b/c","");
    fs.ls("/a/b/c"); // => outputs "" instead of "c"
```

Both of these can be solved with relative ease by introducing a flag on `Node`, 

```rust
use std::collections::HashMap;
struct Node {
    child: HashMap<String,Node>,
    content: String,
    is_file: false
}
```

And then handling the flag in `add_content_to_file` and `ls`.
```rust 
impl FileSystem {
    pub fn add_content_to_file(&mut self, path: String, content: String) {
        let curr = self.find(path);
        curr.is_file = true;
        curr.content.push_str(&content);
    }

    pub fn ls(&mut self, path: String) -> Vec<String> {
        let curr = self.find(path);
        if curr.is_file {
            return vec![path.split("/").last().unwrap()]
        }
        let mut ans = curr.child.keys().collect();
        ans.sort();
        ans
    }
}
```

Now, let's look at `find`

```rust
impl FileSystem {
    pub fn find(&mut self, path: String) -> &mut Node {
            // we need to move the path check first to avoid a double borrow
            if path.len() == 1 {
                return &mut self.root // &mut Node
            }

            let mut curr = &mut self.root; // &mut Node
            
            // advance the iterator once to get past the first null split
            for word in path.split("/").skip(1) {
                curr = curr.child.entry(word).or_insert(Box::new(Node::new())); // &mut Node
            }
            return curr
       }
}
```

What we actually want to do here is abstract out the `curr` update.

```rust
    pub fn find(&mut self, path: String, f: ????) -> &mut Node {
            // we need to move the path check first to avoid a double borrow
            if path.len() == 1 {
                return &mut self.root // &mut Node
            }

            let mut curr = &mut self.root; // &mut Node
            
            // advance the iterator once to get past the first null split
            for word in path.split("/").skip(1) {
                //curr.child.entry(word).or_insert(Box::new(Node::new()))
                curr = f(word); // &mut Node
            }
            return curr
       }


```

We want `f`'s type to be `|curr: &mut Node, word: String| -> &mut Node`, but
that really begs the question -- does passing a function as a parameter make
sense here? We've just defined the function signature of what really just looks
like a method to me. So, let's try implementing `create` and `find` on the node
itself.


```rust
impl Node {
    pub fn new() -> Self {
        Self {
            child: HashMap::new(),
            content: String::new()
        }
    }
    pub fn create(&mut self, name: &str) -> &mut Node {
        self.child.entry(name.to_string()).or_insert_with(|| Box::new(Node::new())).as_mut()
    }

    pub fn find(&mut self, name: &str) -> &mut Node {
        self.child.get_mut(name).unwrap()
    }
}

}
impl FileSystem {
    pub fn find(&mut self, path: String, create: bool) -> &mut Node {
            // we need to move the path check first to avoid a double borrow
            if path.len() == 1 {
                return &mut self.root // &mut Node
            }

            let mut curr = &mut self.root; // &mut Node
            
            // advance the iterator once to get past the first null split
            for word in path.split("/").skip(1) {
                //curr.child.entry(word).or_insert(Box::new(Node::new()))
                curr = match create {
                    true => curr.create(word),
                    false => curr.find(word)
                }; // &mut Node
            }
            return curr
       }
}
```

That works pretty well. But the return type of `HashMap::get_mut` is `Option`
which really highlights another shortcoming of the problem. How should we deal
with `ls /b/cee/d/a` when `b/cee/` doesn't exist? Neither of the original
prompts address the case. So, our current implementation will just panic
(thanks to the `unwrap` call on `Node::find`).

Here's what the final rust implementation looks like: 
```rust
use std::collections::HashMap;

struct Node {
    is_file: bool,
    child: HashMap<String,Box<Node>>,
    content: String
}

impl Node {
    pub fn new() -> Self {
        Self {
            child: HashMap::new(),
            content: String::new(),
            is_file: false
        }
    }
    pub fn create(&mut self, name: &str) -> &mut Node {
        self.child.entry(name.to_string()).or_insert_with(|| Box::new(Node::new())).as_mut()
    }

    pub fn find(&mut self, name: &str) -> &mut Node {
        self.child.get_mut(name).unwrap()
    }
}

struct FileSystem {
    root: Node
}

impl FileSystem {
    pub fn new() -> Self {
        Self {
            root: Node::new()
        }
    }

    pub fn find(&mut self, path: String, create: bool) -> &mut Node {
            if path.len() == 1 {
                return &mut self.root // &mut Node
            }

            let mut curr = &mut self.root; // &mut Node
            // advance the iterator once to get past the first null split
            for word in path.split("/").skip(1) {
                curr = match create {
                    true => curr.create(word),
                    false => curr.find(word)
                };
            }
            return curr
        }



    pub fn ls(&mut self, path: String) -> Vec<String> {
        let curr = self.find(path.clone(),false);
        if curr.is_file {
            return vec![path.split("/").last().unwrap().to_owned()]
        }
        let mut ans = curr.child.keys().cloned().collect();
        ans.sort();
        ans
    }

    pub fn add_content_to_file(&mut self, path: String, content: String) {
        let curr = self.find(path,true);
        curr.is_file = true;
        curr.content.push_str(&content);
    }

    pub fn mkdir(&mut self, path: String) {
        self.find(path,true);
    }

    pub fn read_content_from_file(&mut self, path: String) -> String {
        let curr = self.find(path,false);
        curr.content.to_string()
    }
}
```
