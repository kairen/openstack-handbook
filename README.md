# OpenStack 相關技術整理
本書為 OpenStack 相關技術整理與部署操作實例，也持續針對新加入套件進行介紹與說明，未來會進一步針對套件的程式碼進行窺探，除了透過該書紀錄所學，也一併推廣 OpenStack 相關技術。


# 協同合作方式
1. 在 ```Github``` 上 ```fork``` 到自己的 Repository，例如：```<User>/openstack-book.git```，然後 ```clone```到 local 端，並設定 Git 使用者資訊。

 ```sh
git clone https://github.com/kairen/openstack-book.git
cd openstack-book
git config user.name "User"
git config user.email user@email.com
```
2. 修改程式碼或頁面後，透過 ```commit``` 來提交到自己的 Repository：

 ```sh
git commit -am "Fix issue #1: change helo to hello"
git push
```
3. 在 GitHub 上提交一個 Pull Request。
4. 持續的針對 Project Repository 進行更新內容：

 ```sh
 git remote add upstream  https://github.com/kairen/openstack-book.git
 git fetch upstream
 git checkout master
 git rebase upstream/master
 git push -f origin master
 ```
