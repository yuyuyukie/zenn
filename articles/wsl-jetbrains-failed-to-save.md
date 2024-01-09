---
title: "JetbrainsIDE環境で「Failed to save settings」に対処する"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wsl2", "linux", "jetbrains", "ide"]
published: true
---
# 症状
WSL2環境でWebStorm(PhpStormなど他Jetbrains製IDEでしたら同じ症状が起きます)で発生。
GitHubからcloneしたプロジェクトで作業をしようとしたところ、セーブしようとすると
```
Unable to save settings : Failed to save settings . Please restart IntelliJ IDEA
```
というエラーのポップアップが出てファイルのセーブが出来なかった。

# 解決
`ls -l`でプロジェクトの権限を見てみると、
```shell
drwxr-xr-x 18 root root 4096 Jan  9 10:19 projectA
```
所有者がrootユーザーになっていてIDEがパーミッションエラーになっていたようです。

そういえばrootユーザーでcloneしてたな・・・
```shell
chown userName:userName -R projectA
```
としてディレクトリを再帰的にユーザーの所有にすればOKです。

# 調査の軌跡
この原因以外にも同様の事象が発生することもあるようです。
https://stackoverflow.com/questions/36479123/unable-to-save-settings-intellij-idea
https://stackoverflow.com/questions/65532547/failed-to-save-settings-please-restart-intellij-idea-2020-1-linux-pc

2番目の記事を参考に、IDEのツールバーから`Help => Show Log in Explorer`でログを確認してみましたが、特別エラーの表示はありませんでした。
今回のケースでは`Commit`タブで
```
Unable to create a backup file (.gitignore~). The file left unchanged.
```
という旨のエラーが表示されており、権限関係だと分かった次第です。
