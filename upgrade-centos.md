# update otrs5 to znuny 6.0 in CentOS7
## VMの複製
現在のOTRSサーバのVMをクローンしておく

ネットワークの設定
```
# nmcli con show ens33
# nmcli con mod ipv4.address XXX.XXX.XXX.XXX
# nmcli con mod ipv4.gateway XXX.XXX.XXX.XXX
# nmcli con down ens33
```

## 前準備
各種サービスのストップ
  ```
  # systemctl stop crond
  # sudo -u otrs /opt/otrs/bin/Cron.sh stop
  # sudo -u otrs /opt/otrs/bin/otrs.Daemon.pl stop
  # systemctl stop postfix
  # systemctl stop httpd

  # systemctl disable crond
  # systemctl disable postfix
  # systemctl disable httpd

# systemctl disable crond
  # systemctl disable postfix
  # systemctl disable httpd
  ```
内容の確認
```
  # systemctl status crond
  # systemctl status postfix
  # systemctl status httpd
  ```
cpanmのインストールと環境変数（znunyのPerlライブラリインストール用。各種ライブラリは/usr/local/share/cpanm/にインストール予定）
```
# cd
# curl -L -O http://xrl.us/cpanm > cpanm
# chmod 755 cpanm
# mv cpanm /usr/local/bin
# vi /etc/profile
```

最終行に下記を追加し保存
```
export PERL5LIB="/usr/local/share/cpanm/lib/perl5/";
```

httpdの環境変数追加
```
# systemctl edit httpd
```
下記を追加して保存
```
[Service]
Environment=PERL5LIB=/usr/local/share/cpanm/lib/perl5
```
サーバのの再起動

```
# reboot
```



otrs5のバックアップ
```
# cd /opt/otrs
# ./scripts/backup.pl -d /tmp/backup
# sudo yum install -i wget findutils git screen
```

マイグレーションツールのダウンロードおよび実行
```
# cd /opt
# git clone https://github.com/itgovernanceportal/otrs-znuny-helpers.git
# cd /opt/otrs-znuny-helpers/migrationHelpers/otrs5_2_znuny6
# ./10_migrationPrepare-otrs2znuny.sh /opt/otrs /tmp
```

ファイルバックアップ
```
# ./20_migrationFilesOnly-otrs2znuny.sh /tmp
```

Znunyの依存パッケージ、Perlライブラリをインストール
```
# yum -y install gcc epel-release httpd-devel perl-Convert-BinHex "perl(DBD::mysql)" "perl(Text::CSV_XS)" procmail perl-core "perl(Authen::NTLM)" "perl(Authen::SASL)" "perl(XML::Parser)" "perl(XML::LibXSLT)" "perl(Net::LDAP)" "perl(Encode::HanExtra)" "perl(ModPerl::Util)" "perl(Net::SMTP::SSL)" "perl(Net::DNS)"

# cpanm -L /usr/local/share/cpanm/ DateTime Moo Mail::IMAPClient Crypt::Eksblowfish::Bcrypt JSON::XS YAML::XS Template CSS::Minifier::XS JavaScript::Minifier::XS Jq Spreadsheet::XLSX
```

Znunyインストール、DBマイグレーション
```
# ./30_migrationZnuny-otrs2znuny.sh /tmp/otrs /opt/otrs
# cp /opt/otrs/Kernel/Config.pm.dist /opt/otrs/Kernel/Config.pm
# /opt/otrs/bin/otrs.SetPermissions.pl
# cp /opt/otrs/scripts/apache2-httpd.include.conf /etc/httpd/conf.d/zzz_otrs.conf
# cd /etc/httpd/conf.modules.d/
# cp -ip 00-mpm.conf 00-mpm.conf.org 
#　sed -i '/^LoadModule mpm_event_module modules\/mod_mpm_event.so/s/^/#/' /etc/httpd/conf.modules.d/00-mpm.conf
# sed -i '/^#LoadModule mpm_prefork_module modules\/mod_mpm_prefork.so/s/^#//' /etc/httpd/conf.modules.d/00-mpm.conf
```

httpd再起動
```
# systemctl restart httpd
# systemctl enable httpd
# systemctl list-unit-files -t service | grep enabled
```

http://\[IP-addr\]/otrs/index.pl にWebブラウザでログイン。（ユーザパスワードは既存のものを使用可能）