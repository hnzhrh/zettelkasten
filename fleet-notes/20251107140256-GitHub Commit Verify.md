---
"type:": fleet-note
"title:": 20251107140256-GitHub Commit Verify
"id:": 20251107140301
"created:": 2025-11-07T14:03:01
url:
tags:
  - fleet-note
  - git
"processed:": false
"archived:": false
---

[Generating a new GPG key - GitHub Docs](https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key)

用 RSA 4096

[使用 GPG 签名 commit 记录 - 什么是 DevOps? DevOps 介绍 ｜ CODING DevOps](https://coding.net/help/docs/repo/security/gpg-sign.html)


```shell
gpg: directory '/Users/ryan/.gnupg' created
gpg: WARNING: no command supplied.  Trying to guess what you mean ...
gpg: Go ahead and type your message ...
```

[Sign commits with GPG keys \| IntelliJ IDEA Documentation](https://www.jetbrains.com/help/idea/2025.2/set-up-GPG-commit-signing.html?Set_up_GPG_commit_signing&keymap=macOS#configure-the-environment)

不使用签名
```shell
git config commit.gpgSign false
```
# Reference