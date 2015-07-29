# Horizon 儀表板套件
Horizon提供了```Dashboard```管理介面，

![架構圖](images/openstack-conceptual-arch-folsom.jpg)

編輯```/usr/share/openstack-dashboard/openstack_dashboard/templates/_stylesheets.html```：
```html
<link href='{{ STATIC_URL }}dashboard/css/custom.css' media='screen' rel='stylesheet' />
```
在```/usr/share/openstack-dashboard/openstack_dashboard/static/dashboard/css```加入一個```custom.css```：
```css
.topbar {
background: #33495e;
}
.topbar h1.brand {
background: #33495e repeat-x top left;
padding-left: 0px;
}
.topbar h1.brand a {
padding-left: 0px;
background: url(../img/NUTC_cloud_small.png) no-repeat center 0px;
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

CentOS 重新開啟 HTTP 服務：
```sh
systemctl restart httpd.service memcached.service
```
