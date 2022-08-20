# GitLabEE 安装

> 参考博客: https://blog.csdn.net/wangpaiblog/article/details/122264366

## 下载安装包

> 下载地址: https://packages.gitlab.com/gitlab/gitlab-ee
>
> 需选择系统架构所对应的安装包

![image-20220820185420765](https://img-file.lemonzuo.com/img/202208201854816.png)

![image-20220820185534861](https://img-file.lemonzuo.com/img/202208201855891.png)

## 执行安装

~~~shell
yum localinstall gitlab-ee-15.2.2-ee.0.el7.x86_64.rpm
~~~

# GitLab配置

~~~shell
cd /etc/gitlab/
vim gitlab.rb
~~~

## 访问URL配置

```shell
external_url 'https://abc.com'
```

## 邮件配置

~~~shell
# 邮件配置(腾讯企业邮箱)
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "邮箱用户名"
gitlab_rails['smtp_password'] = "邮箱授权密码"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['gitlab_email_from'] = '邮箱用户名'
gitlab_rails['smtp_domain'] = "exmail.qq.com"
~~~

## 内部Nginx配置

```shell
# Nginx配置
# nginx监听端口
nginx['listen_port'] = 50000
# ssl证书配置
nginx['ssl_certificate'] = "/opt/module/nginx/cert/cert.pem"
nginx['ssl_certificate_key'] = "/opt/module/nginx/cert/key.pem"
```

## 数据目录配置

~~~shell
git_data_dirs({
  "default" => {
    "path" => "数据目录"
   }
})
~~~

## 内存优化配置

~~~shell
puma['worker_processes'] = 0
sidekiq['max_concurrency'] = 10
gitaly['ruby_max_rss'] = 200_000_000
gitaly['concurrency'] = [
  {
    'rpc' => "/gitaly.SmartHTTPService/PostReceivePack",
    'max_per_repo' => 3
  }, {
    'rpc' => "/gitaly.SSHService/SSHUploadPack",
    'max_per_repo' => 3
  }
]

gitaly['cgroups_count'] = 2
gitaly['cgroups_mountpoint'] = '/sys/fs/cgroup'
gitaly['cgroups_hierarchy_root'] = 'gitaly'
gitaly['cgroups_memory_enabled'] = true
gitaly['cgroups_memory_limit'] = 500000
gitaly['cgroups_cpu_enabled'] = true
gitaly['cgroups_cpu_shares'] = 512

gitaly['env'] = {
  'GITALY_COMMAND_SPAWN_MAX_PARALLEL' => '2'
}

prometheus_monitoring['enable'] = false

gitlab_rails['env'] = {
  'MALLOC_CONF' => 'dirty_decay_ms:1000,muzzy_decay_ms:1000'
}

gitaly['env'] = {
  'LD_PRELOAD' => '/opt/gitlab/embedded/lib/libjemalloc.so',
  'MALLOC_CONF' => 'dirty_decay_ms:1000,muzzy_decay_ms:1000'
}
~~~

> 使配置生效： gitlab-ctl reconfigure

# Ruby安装

> 参考博客: https://blog.csdn.net/yuyue_999/article/details/116754103
>
> 最好下载最新版本,下载地址: https://repo.huaweicloud.com/ruby/ruby/

~~~shell
# 卸载当前版本
yum remove ruby
# 解压下载包
tar -xzvf ruby-2.3.0.tar.gz
cd ruby-2.3.0
# 配置安装目录
./configure --prefix=/opt/module/ruby
# 执行安装
make && make install
# 安装完成后需要配置环境变量
vim /etc/profile

export RUBY_HOME=/opt/module/ruby
export PATH=$PATH:$RUBY_HOME/bin

source /etc/profile

# 配置镜像加速
gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
gem sources -l
~~~

# GitLab授权配置

> 参考博客: https://www.lizhiqiang.name/archives/2190.html

## 替换gitlabRsa公钥

> 原始数据文件路径: /opt/gitlab/embedded/service/gitlab-rails/.license_encryption_key.pub
>
> 建议先做数据备份, 后替换为如下内容

~~~txt
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtPGu+l7+6NPtSPEArqDT
QG6AO1QlTlOoLZxi5fqZGiMZRktvTPCrnBJoHoVDFAZ9OfISOYPWPqJ9rl//q6HP
HNJ/uxCccG4vqPo8frqFS7N9/RLZZdPY4BIeJe2ivR0lDgbC35yxBwsXOdA8mWu/
g0Y6Wui6L42wdbrXAS6cRl6tamAR8fh4n8lGTHjEk8W5f56behwrvZgq32nOPK0w
nGUxrPXu8XyRcx7QBYH+1EQHstjOsiIiPTvI9VkMIBl0XfmgbR5f2UDs5wl3nVRd
vKSnTiWOgd/wbmhZhq9t+AxiC+WcUnJTSuZdHsr7MvOoGUWrkSEc+gouEGVoFSym
kQIDAQAB
-----END PUBLIC KEY-----
~~~

## 生成license程序

~~~shell
vim license.rb
~~~

> license.rb 文件内容
>
> ~~~txt
> require "openssl"
> require "gitlab/license"
> 
> # RSA 公钥
> public_key_str = "-----BEGIN PUBLIC KEY-----
> MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtPGu+l7+6NPtSPEArqDT
> QG6AO1QlTlOoLZxi5fqZGiMZRktvTPCrnBJoHoVDFAZ9OfISOYPWPqJ9rl//q6HP
> HNJ/uxCccG4vqPo8frqFS7N9/RLZZdPY4BIeJe2ivR0lDgbC35yxBwsXOdA8mWu/
> g0Y6Wui6L42wdbrXAS6cRl6tamAR8fh4n8lGTHjEk8W5f56behwrvZgq32nOPK0w
> nGUxrPXu8XyRcx7QBYH+1EQHstjOsiIiPTvI9VkMIBl0XfmgbR5f2UDs5wl3nVRd
> vKSnTiWOgd/wbmhZhq9t+AxiC+WcUnJTSuZdHsr7MvOoGUWrkSEc+gouEGVoFSym
> kQIDAQAB
> -----END PUBLIC KEY-----"
> public_key = OpenSSL::PKey::RSA.new(public_key_str)
> 
> # RSA 私钥
> private_key_str = "-----BEGIN RSA PRIVATE KEY-----
> MIIEpQIBAAKCAQEAtPGu+l7+6NPtSPEArqDTQG6AO1QlTlOoLZxi5fqZGiMZRktv
> TPCrnBJoHoVDFAZ9OfISOYPWPqJ9rl//q6HPHNJ/uxCccG4vqPo8frqFS7N9/RLZ
> ZdPY4BIeJe2ivR0lDgbC35yxBwsXOdA8mWu/g0Y6Wui6L42wdbrXAS6cRl6tamAR
> 8fh4n8lGTHjEk8W5f56behwrvZgq32nOPK0wnGUxrPXu8XyRcx7QBYH+1EQHstjO
> siIiPTvI9VkMIBl0XfmgbR5f2UDs5wl3nVRdvKSnTiWOgd/wbmhZhq9t+AxiC+Wc
> UnJTSuZdHsr7MvOoGUWrkSEc+gouEGVoFSymkQIDAQABAoIBAQCUy65Nm7LZufUG
> J5GdCQnPkU8H+tFW0Pqaz2CQqHwgfz54jO3xAnTMumI+vu2DWTa/YO5Vt7GF/k+G
> BtGTzVMo6304Upei6SluNqFqwW197BOt+kMmNojA8oUyQXGzPHVNTIgSJKN7HEa0
> NyauL2nkxOqV+Y2qL0Ut+0B1a2P9hNntxx48vaR7le7gngG4f21o5qwsxcOg4lmb
> O02KCGugMJDxzYw4VbxBMfbWC7QwtmPHCjqZRQPJFwZP1EXo5g9tZSrRXHntcPEx
> VSCTlnDI+G3s71JvVAKE7i2hZAjN3zwe1+6FfJnESvcjnyh3TNxEtNNP40vM0LPS
> oyH1USIBAoGBANluGaILWZsQYsT8Bs+3Qq5tcRJga4wBh/tjBGyCNr8uvuqGtrMk
> vVI8LWGTU4/Nefd+tJlOLs9IiJq57T3m9mQ5Mn1Vi7byRCy6wQqjZnSJ+TupJ3ox
> Tm6Tsq0ciAQwoS0HqEd6LoHmczcG8dl7nDyRsUsAe9QFtuTz8X0Kk8uxAoGBANUK
> r9fKyQWi2Ha/TgUdLzyaopfcmTfSauRfrVG0WqpyPfRVXvWylVt20Ac4a/ThDSSD
> ZwfaZIPi/gRw4tTVp5c9mDgDlTWQcKzWq1zASUEtpc5ppwRMgngPTz5Hl+tuGzDY
> dkAy2YlsXpyhZGapl84/1TULpKoIYgIOhoufeKDhAoGAUOolReWdahR1/UKhMknL
> 2efGjYUuYMLtHQNjURJAV3OI/vQ1J4PDpMfaR5axITHhctZHVUoAJ4mhtJr+i+vY
> w8F5ZaUhQmr0LgUt88yNQ09ZXfd8Rn/05Te35a5Ze92xDXXtDPSOPC9Lry25cSsM
> IIpDhVrfui6KOrgBpXv7NnECgYEAoGWBatjUbJfkndL+rL8CV4CdNfTyrqKPtA2M
> 8lz1fiqxFopICnhAFzLnAOir7xyZxongQntc/ici1LkhLtkFassHFfUsm71598dQ
> EW78OERj93p4MrZf7ICqStugN7MYabgvn7opKlwbB5ZDfz/keXZ50YxIl3PkRmQl
> TG3uZkECgYEA09zBvu48DPz3SMNYqlSHYxOHjeyyM2cgZ+PwKbk1o8zlnzUum6zu
> S73COjn/mDXco46Cx3HIo04HKydEL4sWHvhAOsg3BnDq6ZAVbBydFfN9Qa22NOh+
> aNW2kiPdY6AODt4w/KNF9F0Yq9C+SKLglPyM4R1jNPJbJB693iNK3Y0=
> -----END RSA PRIVATE KEY-----"
> private_key = OpenSSL::PKey::RSA.new(private_key_str)
> 
> Gitlab::License.encryption_key = private_key
> 
> license = Gitlab::License.new
> 
> license.licensee = {
>   "Name"    => "xxxx",
>   "Company" => "xxxx",
>   "Email"   => "xxx@qq.com"
> }
> 
> # 秘钥开始时间
> license.starts_at         = Date.new(2022, 8, 20)
> # 秘钥到期时间
> license.expires_at        = Date.new(2100, 12, 31)
> license.notify_admins_at  = Date.new(2100, 12, 31)
> license.notify_users_at   = Date.new(2100, 12, 31)
> license.block_changes_at  = Date.new(2100, 12, 31)
> 
> license.restrictions  = {
>   active_user_count: 10000
> }
> 
> puts "License:"
> puts license
> 
> data = license.export
> 
> puts "Exported license:"
> puts data
> 
> File.open("xxx.gitlab-license", "w") { |f| f.write(data) }
> 
> 
> Gitlab::License.encryption_key = public_key
> 
> 
> data = File.read("xxx.gitlab-license")
> 
> $license = Gitlab::License.import(data)
> 
> puts "Imported license:"
> puts $license
> 
> unless $license
>   raise "The license is invalid."
> end
> 
> if $license.restricted?(:active_user_count)
>   active_user_count = User.active.count
>   if active_user_count > $license.restrictions[:active_user_count]
>     raise "The active user count exceeds the allowed amount!"
>   end
> end
> 
> if $license.notify_admins?
>   puts "The license is due to expire on #{$license.expires_at}."
> end
> 
> if $license.notify_users?
>   puts "The license is due to expire on #{$license.expires_at}."
> end
> 
> module Gitlab
>   class GitAccess
>     def check(cmd, changes = nil)
>       if $license.block_changes?
>         return build_status_object(false, "License expired")
>       end
>     end
>   end
> end
> 
> puts "This instance of GitLab Enterprise Edition is licensed to:"
> $license.licensee.each do |key, value|
>   puts "#{key}: #{value}"
> end
> 
> if $license.expired?
>   puts "The license expired on #{$license.expires_at}"
> elsif $license.will_expire?
>   puts "The license will expire on #{$license.expires_at}"
> else
>   puts "The license will never expire."
> end
> ~~~

## 生成license

> 执行命令 ruby license.rb
>
> 最终得到 xxx.gitlab-license 为秘钥文件

##  修改授权类型

~~~shell
vim /opt/gitlab/embedded/service/gitlab-rails/ee/app/models/license.rb
restricted_attr(:plan).presence || STARTER_PLAN
替换为
restricted_attr(:plan).presence || ULTIMATE_PLAN
~~~

# 上传license

> 启动gitlab
>
> gitlab-ctl start
>
> 管理员账户登录后上传文件
>
> ![image-20220820193204108](https://img-file.lemonzuo.com/img/202208201932152.png)
>
> ![image-20220820193239777](https://img-file.lemonzuo.com/img/202208201932815.png)
>
> ![image-20220820193321086](https://img-file.lemonzuo.com/img/202208201933120.png)
>
> ![image-20220820193401692](https://img-file.lemonzuo.com/img/202208201934727.png)
>
> ![image-20220820193527931](https://img-file.lemonzuo.com/img/202208201935987.png)