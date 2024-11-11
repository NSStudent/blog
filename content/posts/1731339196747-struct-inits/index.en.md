---
title: "Create a custom struct initializer"
lang: en
date: 2020-11-11
draft: false
description: "Add a custom initializer for structs"
tags: ["struct", "swift", "basics", "tips"]
---

Structs create a default initializer when you implement one.

Sometimes you want to create another initializer to facilitate the code.

The problems appear when you try to introduce our initializer. the default constructor disappears.
To avoid this problem, the best way is to include our initializer inside an extension. 

This way, we'll keep the two builders

```
struct Person {
    let name: String
    let birthday: Date
}

extension Person {
    init(name: String) {
        self.name = name
        self.birthday = Date()
    }
}

let person = Person(name: "NSStudent", birthday: Date(timeIntervalSince1970: 426348250))
let baby = Person(name: "john")
```
