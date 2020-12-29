# 生成 Gitlab EE 许可证

基于 gitlab-ee:13.4.4-ee

## 安装ruby环境

[Ruby](https://www.ruby-lang.org/zh_cn/)

## 安装模块依赖

安装ruby的gitlab-license程序包

```bash
gem install gitlab-license
```

如果gem无法访问仓库，可以将gitlab-license.gem下载下来进行安装。

## 生成许可证

参照[gitlab-license](https://github.com/mockingbot/gitlab-license)的readme编写生成license的ruby脚本

```bash
cat > gitlab-ee-license.rb
```

```ruby
require 'openssl'
require 'gitlab/license'

# Generate a key pair. You should do this only once.
key_pair = OpenSSL::PKey::RSA.generate(2048)

# Write it to a file to use in the license generation application.
File.open("license_key", "w") { |f| f.write(key_pair.to_pem) }

# Extract the public key.
public_key = key_pair.public_key
# Write it to a file to ship along with the main application.
File.open("license_key.pub", "w") { |f| f.write(public_key.to_pem) }

# In the license generation application, load the private key from a file.
private_key = OpenSSL::PKey::RSA.new File.read("license_key")
Gitlab::License.encryption_key = private_key

# Build a new license.
license = Gitlab::License.new

# License information to be rendered as a table in the admin panel.
# E.g.: "This instance of GitLab Enterprise Edition is licensed to:"
# Specific keys don't matter, but there needs to be at least one.
license.licensee = {
  "Name"    => "none",
  "Company" => "none",
  "Email"   => "chou@gitlab.com"
}

# The date the license starts. 
# Required.
license.starts_at         = Date.new(2015, 4, 24)
# The date the license expires. 
# Not required, to allow lifetime licenses.
license.expires_at        = Date.new(2099, 4, 23)

# The below dates are hardcoded in the license so that you can play with the
# period after which there are "repercussions" to license expiration.

# The date admins will be notified about the license's pending expiration. 
# Not required.
license.notify_admins_at  = Date.new(2099, 4, 19)

# The date regular users will be notified about the license's pending expiration.
# Not required.
license.notify_users_at   = Date.new(2099, 4, 23)

# The date "changes" like code pushes, issue or merge request creation 
# or modification and project creation will be blocked.
# Not required.
license.block_changes_at  = Date.new(2099, 5, 7)

# Restrictions bundled with this license.
# Not required, to allow unlimited-user licenses for things like educational organizations.
license.restrictions  = {
  # The maximum allowed number of active users.
  # Not required.
  active_user_count: 10000

  # We don't currently have any other restrictions, but we might in the future.
}

puts "License:"
puts license

# Export the license, which encrypts and encodes it.
data = license.export

puts "Exported license:"
puts data

# Write the license to a file to send to a customer.
File.open("GitLabBV.gitlab-license", "w") { |f| f.write(data) }


# In the customer's application, load the public key from a file.
public_key = OpenSSL::PKey::RSA.new File.read("license_key.pub")
Gitlab::License.encryption_key = public_key

# Read the license from a file.
data = File.read("GitLabBV.gitlab-license")

# Import the license, which decodes and decrypts it.
$license = Gitlab::License.import(data)

puts "Imported license:"
puts $license

# Quit if the license is invalid
unless $license
  raise "The license is invalid."
end

# Quit if the active user count exceeds the allowed amount:
if $license.restricted?(:active_user_count)
  active_user_count = 10000
  if active_user_count > $license.restrictions[:active_user_count]
    raise "The active user count exceeds the allowed amount!"
  end
end

# Show admins a message if the license is about to expire.
if $license.notify_admins?
  puts "The license is due to expire on #{$license.expires_at}."
end

# Show users a message if the license is about to expire.
if $license.notify_users?
  puts "The license is due to expire on #{$license.expires_at}."
end

# Block pushes when the license expired two weeks ago.
module Gitlab
  class GitAccess
    # ...
    def check(cmd, changes = nil)
      if $license.block_changes?
        return build_status_object(false, "License expired")
      end

      # Do other Git access verification
      # ...
    end
    # ...
  end
end

# Show information about the license in the admin panel.
puts "This instance of GitLab Enterprise Edition is licensed to:"
$license.licensee.each do |key, value|
  puts "#{key}: #{value}"
end

if $license.expired?
  puts "The license expired on #{$license.expires_at}"
elsif $license.will_expire?
  puts "The license will expire on #{$license.expires_at}"
else
  puts "The license will never expire."
end
```

其中要修改的部分有：

1. `license.licensee`的`Name`、`Company`、`Emal`。
2. `license.starts_at`和`license.expires_at`等一系列时间。
3. `active_user_count = User.active.count`修改为`active_user_count = 1`。

## 执行脚本

```ruby
ruby license.rb
```

如果打印出License相关信息即成功。

会在脚本目录下生成生成 `GitLabBV.gitlab-license` `license_key` `license_key.pub` 这三个文件。

> *license_key 为私钥，license_key.pub 为公钥，GitLabBV.gitlab-license 为 license 文件*

> PS: 这三个文件以及gitlab-ee-license.rb脚本我都放在gitlab-ee-license目录中

## 使用许可证

- 用 `license_key.pub` 文件替换 `/opt/gitlab/embedded/service/gitlab-rails/.license_encryption_key.pub`。
- 重启 `gitlab-ctl restart`
- `GitLabBV.gitlab-license` 即是许可证，打开设置的gitlab的网页，如：`http://gitlab.example.com/`，使用root账号登录，填入 `http://gitlab.example.com/admin/license` 地址，将许可证上传即可。

## 修改订阅计划

将`/opt/gitlab/embedded/service/gitlab-rails/ee/app/models/license.rb`中的计划修改成`ULTIMATE_PLAN`。

```diff
--- /opt/gitlab/embedded/service/gitlab-rails/ee/app/models/license.rb
+++ /opt/gitlab/embedded/service/gitlab-rails/ee/app/models/license.rb
@@ -382,7 +382,7 @@
  end

  def plan
-    restricted_attr(:plan).presence || STARTER_PLAN
+    restricted_attr(:plan).presence || ULTIMATE_PLAN
  end

  def edition
```

修改完成后使用 `gitlab-ctl restart` 重新启动。