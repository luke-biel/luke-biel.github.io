---
layout: post
title: "Writing a trello client"
categories: misc
comments: true
---

#### Or how I picked up typescript

I pick up new projects quite often. To balance the chaos I have numerous [trello] boards. Each of them represents project or idea that I work on, or intend to work on. To organize them, I've created a tool that shows you (from across all of your boards) what tasks have you **In Progress** and some of those that you can pick up from **To Do**.

#### Why typescript?

Other languages I've considered were *bash*, *python* and *ruby*. Apart for my personal desire to learn *JS* and *nodejs*, *typescript* stood out by being type safe **AND** by providing non-nullability by default. I got used to explicit "null" types when I started writing my applications in *Rust*.  
Additionally typescript was supplying me with whole *nodejs* ecosystem and libraries.

#### Setting up

*TypeScript* can be seamlessly integrated with *npm* environment.  
First we create new npm package `npm init`. This should prompt us with new package creator:
```text
└─[0] <> npm init
This utility will walk you through creating a package.json file.
It only covers the most common items, and tries to guess sensible defaults.

See `npm help json` for definitive documentation on these fields
and exactly what they do.

Use `npm install <pkg>` afterwards to install a package and
save it as a dependency in the package.json file.

Press ^C at any time to quit.
package name: (test)
version: (1.0.0) 0.1.0
description: A dummy ts package
entry point: (index.js) lib/index.js
test command:
git repository: git@github.com:/luke-biel/test
keywords:
author: Łukasz Biel <lukasz.p.biel@gmail.com>
license: (ISC) MIT
```
Next thing we need to add typescript as a dependency - `npm install typescript @types/node --save-dev` - and alter *package.json* to include build command which will compile our code into javascript library. The `@types/node` module is necessary for *TypeScript* to know what function signature does node export.
```json
"scripts": {
    "build": "tsc -p .",
    "clean": "rm -rf ./lib",
    "refresh-pkg": "rm -rf ./node_modules ./package-lock.json && npm install",
    "app": "node ./lib/index.js"
}
```
We add `tsconfig.json` so *TypeScript* compiler knows how to build our code:
```json
{
    "compilerOptions": {
        "target": "es5",
        "module": "commonjs",
        "lib": ["es6", "es2019", "dom"],
        "declaration": true,
        "outDir": "lib",
        "rootDir": "src",
        "strict": true,
        "types": ["node"],
        "esModuleInterop": true,
        "resolveJsonModule": true
    }
}
```
With all those things set up, we can create `src/index.ts` file:
```typescript
console.log("Hello, world!");
```
Running it with `npm run build && npm run app` results in:
```text
└─[0] <> npm run build && npm run app

> test@0.1.0 build /home/lbiel/zone/test
> tsc -p .


> test@0.1.0 app /home/lbiel/zone/test
> node ./lib/index.js

Hello, world!
```

#### Integrating with trello

Trello provides REST api via url <https://api.trello.com/1>.

In order to make any request to their service we need their API key and token, first one we get at <https://trello.com/app-key>, latter at <https://trello.com/1/authorize?expiration=never&scope=read,write,account&response_type=token&name=Server%20Token&key=$TRELLO_KEY>, where we should replace **$TRELLO_KEY** with our app-key. Further reading is available at <https://developers.trello.com/docs/api-introduction>.

In order to get list of user boards, we need to **GET** `/members/me/boards`. We can freely limit json fields to only contain *name* and *id* by adding `?fields=name,id`. We won't use other fields now. An example of correct response is:
```json
[
  {
    "name": "bts",
    "id": "5d1dd88d8a013d245d94bf79"
  },
  {
    "name": "dir-assert",
    "id": "5e1e1813e86c19040b01120a"
  },
  {
    "name": "ion-fmt",
    "id": "5e1afde5d3055a80b84bfe5c"
  }
]
```

I've used *axios* for http client (`npm install axios --save`).
```typescript
axios.get(
    ${MY_BOARDS_URL}?key=${this.config.apiKey}&token=${this.config.apiToken}&fields=name,id`
);
```
and prepared interface beforehand, to function as a DTO:
```typescript
interface BoardDTO {
    name: string;
    id: string;
}
```
this is huge benefit of **TypeScript** as we can have typed DTOs opposed to vague and uncertain data structures that some other languages would provide.

Having board *id* we can now query each of the boards to get names of lists (columns) available. We do that on endpoint `/boards/$ID/lists`. We can again limit ourselves only to download *name* and *id* fields. Here's an excerpt from *dir-assert* board:
```json
[
  {
    "id": "5e1e481819a7f78868014c9e3",
    "name": "To Do"
  },
  {
    "id": "5e1e481c41772d7c02ddf38b",
    "name": "In Progress"
  },
  {
    "id": "5e1e4811276d9c1b3a6f1210",
    "name": "Done"
  }
]
```

We are only interested with boards that have lists named **To Do** and **In Progress**, so we can filter out other ones.  

We need cards from each **To Do** list and each **In Progress** list grouped in one place.
This purpose we can fulfill with `/lists/$ID/cards` endpoint. Example response:
```json
[
  {
    "id": "5e440f40baa1f3516713b575",
    "name": "assert should println! or debug! all paths it went through"
  },
  {
    "id": "5e440f8939420000a178e545",
    "name": "unit test error new_* functions"
  },
  {
    "id": "5e4410c2c00a4a49e3540929",
    "name": "Cleanup code"
  }
]
```

With all that information collected in DTOs we can now start drawing it!

#### Drawing on terminal

I've tested some of the available [tui]s but none was feeling right, therefore I've settled down with plain *chalk*. *Chalk* is a library that allows you to paint text in easy way, by using template strings.
```typescript
chalk`{red ...there's nothing being worked on currently}`
```

The final result looks like this:

![Trollo](https://luke-biel.github.io/trollo/usage.png)

#### Summary

[repo]

Okay, it works. It even displays useful information, so that's a success. But it's a bit slow. I can use it once a while, but if I'd start each bash session with [trollo][repo] run, as I intended, I'd get annoyed quite quickly for how slow the app is. And the bottleneck is that I'm making *1* request for boards, *X* requests for each boards list, *2\*X* requests for each list cards. So when X is 10, I'm making effectively 31 http requests. That's a lot.  
Well, it's time to create board for trollo and put caching in **To Do**.

[trello]: https://trello.com/
[tui]: https://en.wikipedia.org/wiki/Text-based_user_interface
[repo]: https://luke-biel.github.io/trollo
