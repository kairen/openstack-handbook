# 客制化 Horizon
Horizon 可以透過修改原始碼，或者是加入```CSS```來改變樣式，以下簡單的介紹如何透過```CSS```客製化一個屬於自己的登入頁面，首先編輯```/usr/share/openstack-dashboard/openstack_dashboard/templates/_stylesheets.html```：
```html
<link href='{{ STATIC_URL }}dashboard/css/custom.css' media='screen' rel='stylesheet' />
```
> P.S 這邊可能會因為 Horizon 安裝方式不同，而目錄位置有所差異。

在```/usr/share/openstack-dashboard/openstack_dashboard/static/dashboard/css```加入一個```custom.css```：
```css
.navbar {
background-color: #33495e;
border-color:#33495e;
}
.navbar-default .navbar-nav > li > a {
color: #f5f5f5;
}
.topbar h1.brand {
background: #33495e repeat-x top left;
padding-left: 0px;
}
.navbar-header {
padding-left: 0px;
background: url(../img/NUTC_cloud_small.png) no-repeat center 0px;
}
.navbar-brand img {
visibility: hidden;
}
.topbar .switcher_bar .btn.btn-topnav {
background-color: #33495e;
color:#ffffff;
border-color:#33495e;
}
#splash .login {
background: #33495e url(../img/NUTC_cloud.png) no-repeat center 25px;
}
#splash .login .modal-header {
border-top: 1px solid #BAD3E1;
background-color: #ffffff;
}
.modal-title {
margin: 0;
line-height: 1.42857;
}
.btn-primary {
background-image: none !important;
background-color: #2896C8 !important;
border: none !important;
box-shadow: none;
}
.btn-primary:hover,
.btn-primary:active {
border: none;
box-shadow: none;
background-color: #BAD3E1 !important;
text-decoration: none;
}

form.ng-pristine.ng-valid.ng-scope {
background-color: #ffffff !important;
}
```
在```/usr/share/openstack-dashboard/openstack_dashboard/static/dashboard/img```放入圖片。

Ubuntu 重新開啟 HTTP 服務：
```sh
sudo service apache2 restart
```