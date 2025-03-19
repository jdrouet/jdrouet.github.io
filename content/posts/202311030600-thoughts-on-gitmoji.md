+++
title = "My thoughts on gitmoji"
description = "Just use conventional commits, keep things simple."
date = 2023-11-03

[extra]
emoji = "🙂"

[taxonomies]
tags = ["git", "good practices"]
+++

First, **I have nothing against conventional commits.** Enforcing conventions when working on a project or codebase is the best way to maintain a clean and understandable project. While it may take time for newcomers to join and adopt these conventions, once they do, it becomes easier for them to contribute.

I have been accustomed to using conventional commits in most of my personal projects or previous work projects. It helps to keep commits atomic, making it easier to read the changes in pull/merge requests. When you merge these commits into your main branch, it becomes easier to understand the complete history of the product. Additionally, it allows you to automatically generate your changelog.

Lately, gitmoji has emerged as a replacement for the traditional keywords like `feat`, `build`, `fix`, etc. in commit messages. Instead, it uses emojis like ✨, 👷, 🐛 or other keywords like `sparkles`, `construction_worker`, `bug`, which are then automatically replaced by emojis on GitHub. This has resulted in a history full of emojis on GitHub, which can be deciphered by those familiar with them. You can check out an example of this [here](https://github.com/carloscuesta/gitmoji/commits/master).

I personally believe that this is slowing people down without adding real value. First, you need to maintain a mind map of what each emoji represents. For example, when you write your commit, you might think, "It's a feature, so I have to write `sparkles` or use that emoji." Similarly, when you read someone else's commit, you have to remember the meaning behind each emoji.

Additionally, I'm curious about the accessibility implications of using this method to mark commits. How does it appear to individuals with color blindness, or how does it function with tools used by blind individuals? Are we unintentionally creating a gap that will require additional tools to address?

But what if I really love emojis and want to see them everywhere? Why not develop a browser extension that, when you visit a Github repository, replaces all instances of `feat` with a ✨, and so on? This way, I can have the same user experience without adding any barriers to the existing ones.

As a conclusion: use conventional commits, it’s great for you, great for your project and great for people that will contribute to it. Keep it simple. Avoid unnecessary gaps.
