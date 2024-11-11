---
title: "Deploy Publish blog with Github Actions"
lang: en
date: 2024-11-10
draft: false
description: "Use GitHub deploy method and Github Actions to deploy in your server."
tags: ["publish", "swift"]
---

## Intro

[Publish](https://github.com/JohnSundell/Publish) is an awesome static site generator created by [John Sundell](https://x.com/johnsundell). This website is built with this tool.

One of the steps is to upload to the server all the file generated, the first approach I used was to upload all the files in the output folder, via FTP manually but, then I had an idea.

## Deploy

For deployment, I use the following code:

```swift
.publish(
        withTheme: .nsstudent,
        deployedUsing: DeploymentMethod.gitHub("NSStudent/NSStudent"),
...
}
```

This is the implementation of the `github` method:

```swift
/// Deploy a website to a given GitHub repository.
    /// - parameter repository: The full name of the repository (including its username).
    /// - parameter useSSH: Whether an SSH connection should be used (preferred).
    static func gitHub(_ repository: String, useSSH: Bool = true) -> Self {
        let prefix = useSSH ? "git@github.com:" : "https://github.com/"
        return git("\(prefix)\(repository).git")
    }
```

This method searches the repository with the name `NSStudent/NSStudent`.  By default, the implementation tries to clone the repository via SSH with this code inside the `git` method.

```swift
static func git(_ remote: String) -> Self {
        DeploymentMethod(name: "Git (\(remote))") { context in
            let folder = try context.createDeploymentFolder(withPrefix: "Git") { folder in
                if !folder.containsSubfolder(named: ".git") {
                    try shellOut(to: .gitInit(), at: folder.path)

                    try shellOut(
                        to: "git remote add origin \(remote)",
                        at: folder.path
                    )
                }

                try shellOut(
                    to: "git remote set-url origin \(remote)",
                    at: folder.path
                )

                _ = try? shellOut(
                    to: .gitPull(remote: "origin", branch: "master"),
                    at: folder.path
                )

                try folder.empty()
            }

            let dateFormatter = DateFormatter()
            dateFormatter.dateFormat = "yyyy-MM-dd HH:mm"
            let dateString = dateFormatter.string(from: Date())

            do {
                try shellOut(
                    to: """
                    git add . && git commit -a -m \"Publish deploy \(dateString)\" --allow-empty
                    """,
                    at: folder.path
                )

                try shellOut(
                    to: .gitPush(remote: "origin", branch: "master"),
                    at: folder.path
                )
            } catch let error as ShellOutError {
                throw PublishingError(infoMessage: error.message)
            } catch {
                throw error
            }
        }
    }
```

The magic happens when the `context` calls `context.createDeploymentFolder ` function. This method creates a new folder with the name `GitDeploy`  inside to a hidden folder `.publish`. 

Right now, all the codes are in your repository.

To create the action to upload the website in the server via FTP, you need to add the following `main.yml` file:

```bash
name: Deploy website

on: [push]

jobs:
  FTP-Deploy-Action:
    name: FTP-Deploy-Action
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@master
    - name: FTP-Deploy-Action
      uses: SamKirkland/FTP-Deploy-Action@2.0.0
      env:
        FTP_SERVER: ${{ secrets.FTP_NAME }}
        FTP_USERNAME: ${{ secrets.FTP_USER }}
        FTP_PASSWORD: ${{ secrets.FTP_PASSWORD }}
        ARGS: --delete --exclude-glob=.git*/** --exclude-glob=.git** --delete-excluded
```

This file use the [FTP-Deploy-Action](https://github.com/SamKirkland/FTP-Deploy-Action). It has three secrets variables `secrets.FTP_NAME`, `secrets.FTP_USER`, `secrets.FTP_PASSWORD` to not expose the real values. The `ARGS` parameter tries to remove all unused files and it prevents the upload off all the files inside the `.git` or `.github` folders.

> You need to pull this change in the current `GitDeploy` repository. Right now, every time you use `Publish deploy` in the terminal and push a new version in this repository, the action uploads all the files to the FTP.

I will try to write with new things about [Publish](https://github.com/JohnSundell/Publish) and how I use them in this blog.
