# OpenStack 雲端開源平台技術整理
本書是整理 OpenStack 雲端開源平台相關技術與資訊整理，其中包含部署工具、手動一步一步安裝、各套件簡介等。本書也會隨著 OpenStack 的發展而新增新的章節來進行描述，也希望進一步針對套件的核心程式碼進行窺探，除了記錄期間所學過程，也希望一起推廣 OpenStack 雲端開源平台。

# 參與貢獻
如果您想一起參與貢獻的話，您可以協助以下項目：
* 幫忙校正、挑錯別字、語病等等
* 提供一些修改建議
* 提出對某些術語翻譯的建議
* 提供 OpenStack 新套件教學、簡介或者是程式碼分析。

# 透過 Github 進行協作
1. 在 ```Github``` 上 ```fork``` 到自己的 Repository，例如：```<User>/openstack-book.git```，然後 ```clone```到 Local 端，並設定 Git 使用者資訊。
```sh
$ git clone https://github.com/<User>/openstack-book.git
$ cd openstack-book
$ git config user.name "User"
$ git config user.email user@email.com
```

2. 修改程式碼或內容後，透過 ```commit``` 來提交到自己的 Repository：
```sh
$ git commit -am "Fix issue #1: change helo to hello"
$ git push
```
> 若新增採用一般文字訊息，如 ```Add neutron vlan tag intro```。

3. 在 GitHub 上提教一個 Pull Request。
4. 持續針對專案的 Repository 進行更新：
```sh
$ git remote add upstream  https://github.com/imac-cloud/openstack-book.git
$ git fetch upstream
$ git checkout master
$ git rebase upstream/master
$ git push -f origin master
```

# License
