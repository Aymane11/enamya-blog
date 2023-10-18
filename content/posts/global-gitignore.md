---
title: "One .gitignore to rule them all"
date: 2023-10-18T18:49:25+01:00
author: "Aymane BOUMAAZA"
draft: false
description: "How to create a global .gitignore file for all your projects"
tags:
 - TIL
 - gitignore
 - git
 - version-control
---

Today I learned that I can have a global `.gitignore` file, so I don't have to create one for every project.

According to [git documentation](https://git-scm.com/docs/gitignore#_configuration), some of the sources that `git` uses to find patterns to ignore are:
- The `.gitignore` file we all know and love.
- And patterns in the file specified by `core.excludesFile` configuration variable.

> **Note** that the local `.gitingore` file will always be **more prioritized** than the global one. _(i.e. if a file is ignored in the global `.gitignore` file, but explicitly tracked in the local `.gitignore` file, it will be tracked, and vice versa)_

{{< figure src="/img/global_gitignore.png" align="center" alt="The local .gitignore is always more prioritized than the global one" caption="The file `my.file` is ignored globally but explicitly tracked locally, so it will be tracked." border="#f8f4f0" width="80%" >}}

Using the global git config, we can create a global `.gitignore` file that will be used by `git` to ignore files and directories across all projects.

```bash
touch ~/.gitignore # or any other file name
```

Then we can add patterns to ignore in this file, for example:

```bash
echo "*.log" >> ~/.gitignore # Ignore all log files
```

Finally, we need to set the `core.excludesFile` configuration variable to point to the file we just created:

```bash
git config --global core.excludesfile ~/.gitignore
```

And voilà, we have a global `.gitignore` that will be used by `git` across all projects.

> Bonus: The [gitignore.io](https://gitignore.io/) website is a great resource to generate `.gitignore` files for different projects and languages.