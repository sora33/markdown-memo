
## 参考にした
下記がwindows向けだったので、Mac向けの記事を作成

https://qiita.com/hidekatsu-izuno/items/3743ea426c7ffa0f3878


## クライアントへの設定


``` 
vim ~/.ssh/config                 
```

```
Host my-development-environment-ubuntu-auto
    HostName i-07512a9534fff6084
    Port 22
    User ubuntu
    IdentityFile ~/pemFile/aws-and-infra-ssh-key.pem
    ProxyCommand /usr/local/bin/ssm_connect.sh %h %p default
```


``` 
vim /usr/local/bin/ssm_connect.sh
```


``` 
#!/bin/bash

INSTANCE_ID="$1"
PORT="$2"
PROFILE="$3"

# Set default profile if none provided
if [ -z "$PROFILE" ]; then
  PROFILE="default"
fi

# Fetch instance state
INSTANCE_STATE=$(aws ec2 describe-instance-status --include-all-instances --instance-ids "$INSTANCE_ID" --query "InstanceStatuses[*].InstanceState.Name" --output text --profile "$PROFILE")

case "$INSTANCE_STATE" in
  "pending")
    # No handle
    ;;
  "running")
    # No handle
    ;;
  "rebooting")
    # No handle
    ;;
  "stopping")
    echo "Waiting for the instance to stop..." 1>&2
    aws ec2 wait instance-stopped --instance-ids "$INSTANCE_ID" --profile "$PROFILE"
    if [ $? -ne 0 ]; then
      echo "Failed to wait for instance to stop: $INSTANCE_ID" 1>&2
      exit 1
    fi
    aws ec2 start-instances --instance-ids "$INSTANCE_ID" --profile "$PROFILE"
    if [ $? -ne 0 ]; then
      echo "Failed to start instance: $INSTANCE_ID" 1>&2
      exit 1
    fi
    ;;
  "stopped")
    aws ec2 start-instances --instance-ids "$INSTANCE_ID" --profile "$PROFILE"
    if [ $? -ne 0 ]; then
      echo "Failed to start instance: $INSTANCE_ID" 1>&2
      exit 1
    fi
    ;;
  "shutting-down")
    echo "Instance is shutting down: $INSTANCE_ID" 1>&2
    exit 1
    ;;
  "terminated")
    echo "Instance is terminated: $INSTANCE_ID" 1>&2
    exit 1
    ;;
  *)
    echo "Failed to describe instance status: $INSTANCE_ID" 1>&2
    exit 1
    ;;
esac

echo "Waiting for the instance status to be OK..." 1>&2
aws ec2 wait instance-status-ok --include-all-instances --instance-ids "$INSTANCE_ID" --profile "$PROFILE"
if [ $? -ne 0 ]; then
  echo "Failed to wait for instance status to be OK: $INSTANCE_ID" 1>&2
  exit 1
fi

aws ssm start-session --target "$INSTANCE_ID" --document-name AWS-StartSSHSession --parameters "portNumber=$PORT" --profile "$PROFILE"
```

$1, $2, $3 などはシェルスクリプトが受け取るコマンドライン引数を指しています。

$1 は第一引数で、このスクリプトの文脈ではAmazon EC2のインスタンスID（例：i-1234567890abcdef0）が入ります。
$2 は第二引数で、SSH接続のポート番号（通常は22）が入ります。
$3 は第三引数で、AWS CLIのプロファイル名が入ります。プロファイルが指定されていない場合は、デフォルトのプロファイルが使用されます。
これらの引数はSSHのconfigファイル内でProxyCommandの部分で指定されます。例えば：

ssh
Copy code
ProxyCommand /path/to/ssm_connect.sh %h %p default
この設定では、%h はホスト名（ここではインスタンスID）、%p はポート番号、default はAWS CLIのプロファイル名としてシェルスクリプトに渡されます。

SSHコマンドを実行すると、SSHがこのProxyCommandを内部で呼び出し、ssm_connect.sh スクリプトはこれらの引数（$1, $2, $3）を受け取って処理を行います。

### 実行権限を付与
スクリプトに実行権限を与えます。
``` 
chmod +x /path/to/ssm_connect.sh
```


## サーバー環境への設定
### スクリプトの作成
1. まず、~/bin ディレクトリを作成します。ディレクトリが存在しない場合は以下のコマンドで作成します。
```
mkdir -p ~/bin
```
2. 次に、stop_instance.sh ファイルを ~/bin ディレクトリ内に作成します。nano や vim などのテキストエディタを使用できます。
```
vim ~/bin/stop_instance.sh
```
3. エディタが開いたら、スクリプトの内容を貼り付けて保存します。
```
#/bin/bash -e

UPTIME=`/usr/bin/uptime -s`
B5TIME=`date +"%Y-%m-%d %T" --date "-5 minutes"`
if [ "$UPTIME" \> "$B5TIME" ]; then
    echo "Skip the instance stop for initializing."
    exit 0
fi

SESSION_COUNT=`ps -A x | grep " sshd:" | grep -v -e grep -v -e /usr/sbin/sshd | wc -l`
if [ $SESSION_COUNT -gt 0 ]; then
    echo "Skip the instance stop because sessions exist."
    exit 0
fi

echo "Stopping the instance."
sudo /sbin/shutdown -h now > /dev/null 2>&1

```


### 実行権限を付与
スクリプトに実行権限を与えます。
``` 
chmod +x ~/bin/stop_instance.sh
```

### Cronの設定
1. crontab を編集するには、以下のコマンドを実行します。
```
crontab -e
```
2. エディタが開いたら、以下の行を追加して保存します。
```
*/5 * * * * ~/bin/stop_instance.sh | logger -t stop_instance
```

これで、5分ごとに stop_instance.sh スクリプトが実行されるようになります。

これで、Session Manager のセッションが存在しない場合にインスタンスが自動で停止するようになるはずです。この設定を行った後には、しばらく様子を見て確認してみてください。




## スクリプトの解説
特定の条件に基づいてAWS EC2インスタンス（Ubuntuサーバー）を自動的に停止するためのものです。詳細について説明します。

#/bin/bash -e: Bashシェルを使用することを宣言しています。-eオプションは、スクリプト内のコマンドがエラーを返したらスクリプトを終了するという意味です。ただし、この行に#が付いているため、コメントとして扱われ実行されません。

UPTIME=/usr/bin/uptime -s: システムが起動してからの時間を取得します。

B5TIME=date +"%Y-%m-%d %T" --date "-5 minutes": 現在時刻から5分前の時刻を取得します。

if [ "$UPTIME" \> "$B5TIME" ]; then ... fi: システムの起動時間が5分以内であれば、インスタンスを停止しないでスクリプトを終了します。

SESSION_COUNT=ps -A x | grep " sshd:" | grep -v -e grep -v -e /usr/sbin/sshd | wc -l: 現在開いているSSHセッションの数をカウントします。

if [ $SESSION_COUNT -gt 0 ]; then ... fi: SSHセッションが開いていれば、インスタンスを停止しないでスクリプトを終了します。

sudo /sbin/shutdown -h now > /dev/null 2>&1: 上記の条件に一致しない場合（すなわち、システムの起動から5分以上経過し、SSHセッションも開いていない場合）、インスタンスを停止します。出力とエラーは無視されます。

このスクリプトは、通常はcronジョブとして定期的に実行されます。このように設定すると、不必要にインスタンスが稼働し続けることなく、コストを削減できます。

## Cronの実行確認
1. ログを確認する: logger -t stop_instanceが追加されているため、/var/log/syslog（Ubuntu）にログが出力されるはずです。次のコマンドで確認できます。
```
grep 'stop_instance' /var/log/syslog
```
2. ログの中身
```
@ip-10-0-5-254:~$ grep 'stop_instance' /var/log/syslog
Sep 17 16:45:01 ip-10-0-5-254 CRON[3487]: (ubuntu) CMD (~/bin/stop_instance.sh | logger -t stop_instance)
Sep 17 16:45:01 ip-10-0-5-254 stop_instance: Skip the instance stop because sessions exist.
```

ログによると、stop_instance.shスクリプトは確かに実行されており、Skip the instance stop because sessions exist.というメッセージも出力されています。これは、スクリプトがSSHセッションが存在すると認識したため、インスタンスの停止をスキップしたということを意味します。

この情報から、cronジョブ自体は正常に動作していると言えます。また、stop_instance.shスクリプトも期待通りに動作しているようです。

スクリプトが「セッションが存在するため停止をスキップする」と判断しているので、これは多分にSSHセッションが開いている（あるいは他の何らかのセッションが開いている）ためだと考えられます。

この状態が期待した動作であれば問題ありませんが、何らかの不具合や誤動作が疑われる場合は、スクリプトのロジックや条件式を再確認する価値があります。