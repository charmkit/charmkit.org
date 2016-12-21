---
layout: default
---

# CharmKit is a Juju charm development framework for easing the burden on creating charms for your applications.

## SYNOPSIS

```ruby
require 'charmkit'

namespace :znc do
  desc "Configure ZNC"
  task :config_changed do

    admin_user = config 'admin_user'
    admin_password = config 'admin_password'
    if (admin_user.nil? or admin_user.empty?) or
     (admin_password.nil? or admin_password.empty?)
      status :blocked, "Waiting for an admin username and password " +
                       "to be set, see juju config znc"
      exit 0
    end

    # Need to parse admin password into proper variables for znc
    admin_password_salt = gen_salt
    admin_password_hash = Digest::SHA256.hexdigest(
      admin_password + admin_password_salt)

    inline_template 'znc.conf',
                    '/var/lib/znc/configs/znc.conf',
                    port: config('port'),
                    admin_user: config('admin_user'),
                    admin_password: config('admin_password'),
                    admin_password_salt: admin_password_salt,
                    admin_password_hash: admin_password_hash

    chown_R "znc", "znc", "/var/lib/znc"
    run 'systemctl daemon-reload'
    run 'systemctl restart znc'

    out, err = run 'znc --version'
    if match = out.match(/^ZNC\s(\d\.\d\.\d)/i)
      version = match.captures
      run "application-version-set #{version.first}"
    else
      run 'application-version-set Unknown'
    end

    status :active, "ZNC is ready"
  end
end

__END__

@@ znc.service
[Unit]
Description=znc, and advanced IRC bouncer
After=network-online.target

[Service]
ExecStart=/usr/bin/znc -f --datadir=/var/lib/znc
User=znc
Group=znc

[Install]
WantedBy=multi-user.target

@@ znc.conf
Version = 1.6.3
<Listener l>
	  Port = <%= port %>
          IPv4 = true
          IPv6 = false
          SSL = false
</Listener>
LoadModule = webadmin

<User <%= admin_user %>>
	Admin      = true
	Nick       = <%= admin_user %>
	AltNick    = <%= admin_user %>_
	Ident      = <%= admin_user %>
	RealName   = ZNC Managed by Juju
	LoadModule = chansaver
	LoadModule = controlpanel
        <Pass <%= admin_password %>>
          hash = <%= admin_password_hash %>
          method = sha256
          salt = <%= admin_password_salt %>
        </Pass>
</User>
```

This touches on just a small portion of the DSL to help you write charms quickly and effeciently.
