---
title: "アプリのバージョンアップとRedux Persistの共存"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "redux-persist", "redux" ]
published: false
---
# 概要
Redux Persistとは、ブラウザにReduxのストアをキャッシュすることで、アプリから離れても状態を保持することができます。
Redux Persistについては以下の記事がわかりやすいかと思います。
https://qiita.com/yasuhiro-yamada/items/bd86d7c9f35ebb1c1e7c

この記事では、アプリをアップデートした際のストアの引き継ぎ方、あるいはストアのリセットについて解説します。
