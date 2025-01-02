---
layout: post
title:  "Package design"
author: kpapp
categories: [ tutorial, package, design ]
image: assets/images/design.jpg
featured: false
hidden: true
---

Package design often doesn’t get the attention it deserves, yet it has a huge impact on the maintainability of our codebases.

Take, for example, contract packages—those that contain interfaces, DTOs, or other foundational elements. These are frequently shared across different parts of the system, becoming the glue that holds everything together. But here's the catch:

🔗 The more dependencies a package has, the more stable it needs to be.

When we depend heavily on a package, any change to it can propagate like ripples through a pond, requiring updates across multiple layers of the codebase. This is why it's crucial to pause and evaluate:

- Is this package as stable as it needs to be?
- Are we minimizing the need for change here?
- Does the design follow key principles, like ensuring high cohesion and low coupling?

By putting extra thought into designing and maintaining stable packages, especially those with many dependents, we can avoid unnecessary churn and keep our systems easier to evolve.
