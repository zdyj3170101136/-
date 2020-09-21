#### shell

#### bench.sh

通过nsq等的shell学习语法。

**#!** 是一个约定的标记，它告诉系统这个脚本需要什么解释器来执行，即使用哪一种 Shell

readongly表示只读变量。

![截屏2020-09-10 下午6.10.32](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午6.10.32.png)

![截屏2020-09-10 下午6.14.31](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午6.14.31.png)

![截屏2020-09-10 下午6.14.43](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午6.14.43.png)



echo是屏幕显示，使用$把表示使用变量的实际值

![截屏2020-09-10 下午6.15.39](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午6.15.39.png)



dirs和pushdhttps://www.jianshu.com/p/53cccae3c443

目录栈。，而dirs是显示目录栈的内容。而目录栈就是一个保存目录的栈结构，该栈结构的顶端永远都存放着当前目录（这里点从下面可以进一步看到.



pushd后面如果直接跟目录使用，会切换到该目录并且将该目录置于目录栈的栈顶。(时时刻刻都要记住，目录栈的栈顶永远存放的是当前目录。如果当前目录发生变化，那么目录栈的栈顶元素肯定也变了；反过来，如果栈顶元素发生变化，那么当前目录肯定也变了。)下面是一个例子：

/dev/null 是空设备，以及标准输出。

https://www.cnblogs.com/xiao02fang/p/10314921.html



题中为把标准输出和标准错误都挂到/de v/null 并且挂在后台执行。

![截屏2020-09-10 下午6.32.10](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午6.32.10.png)

![截屏2020-09-10 下午6.33.12](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午6.33.12.png)



https://www.cnblogs.com/viviane/p/11101643.html



获取nsqd进程的id号加回到当前目录。



trap 在指接受到的信号后执行函数。



sleep表示进程休眠0.3s。



### **curl文件下载**

curl将下载文件输出到stdout，将进度信息输出到stderr，不显示进度信息使用–silent 选项。

1 . `curl URL --silent` 
这条命令是将下载文件输出到终端，所有下载的数据都被写入到stdout。

2 . `curl URL --silent -O` 
使用选项 -O 将下载的数据写入到文件，必须使用文件的绝对地址。

3 . `curl URL -o filename --progress` 
\######################################### 100.0% 
选项 -o 将下载数据写入到指定名称的文件中，并使用 –progress 显示进度条。



echo -n不换行输出



curl -o将数据保存成文件

curl -s不输出错误和进度信息。

![截屏2020-09-10 下午7.04.30](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午7.04.30.png)

这是输出。

首先得用chmod赋予文件执行权限。

![截屏2020-09-10 下午7.09.37](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午7.09.37.png)

```shell
#!/bin/bash
readonly messageSize="${1:-200}"
readonly batchSize="${2:-200}"
readonly memQueueSize="${3:-1000000}"
readonly dataPath="${4:-}"
set -e
set -u

echo "# using --mem-queue-size=$memQueueSize --data-path=$dataPath --size=$messageSize --batch-size=$batchSize"
echo "# compiling/running nsqd"
pushd apps/nsqd >/dev/null
go build
rm -f *.dat
./nsqd --mem-queue-size=$memQueueSize --data-path=$dataPath >/dev/null 2>&1 &
nsqd_pid=$!
popd >/dev/null

cleanup() {
    kill -9 $nsqd_pid
    rm -f nsqd/*.dat
}
trap cleanup INT TERM EXIT

sleep 0.3
echo "# creating topic/channel"
curl --silent 'http://127.0.0.1:4151/create_topic?topic=sub_bench' >/dev/null 2>&1
curl --silent 'http://127.0.0.1:4151/create_channel?topic=sub_bench&channel=ch' >/dev/null 2>&1

echo "# compiling bench_reader/bench_writer"
pushd bench >/dev/null
for app in bench_reader bench_writer; do
    pushd $app >/dev/null
    go build
    popd >/dev/null
done
popd >/dev/null

echo -n "PUB: "
bench/bench_writer/bench_writer --size=$messageSize --batch-size=$batchSize 2>&1

curl -s -o cpu.pprof http://127.0.0.1:4151/debug/pprof/profile &
pprof_pid=$!

echo -n "SUB: "
bench/bench_reader/bench_reader --size=$messageSize --channel=ch 2>&1

echo "waiting for pprof..."
wait $pprof_pid
```



#### fmt.sh

https://blog.csdn.net/csyuanA/article/details/76408836

xargs把标准输出转换成命令行参数。

```go
#!/bin/bash
find . -name "*.go" | xargs goimports -w
```

![截屏2020-09-10 下午7.21.10](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午7.21.10.png)

也即是在本目录下找到名字是*.name的文件。

#### test.sh

```go
#!/bin/bash
set -e

GOMAXPROCS=1 go test -timeout 90s $(go list ./... | grep -v /vendor/)
GOMAXPROCS=4 go test -timeout 90s -race $(go list ./... | grep -v /vendor/)

# no tests, but a build is something
for dir in apps/*/ bench/*/; do
    dir=${dir%/}
    if grep -q '^package main$' $dir/*.go 2>/dev/null; then
        echo "building $dir"
        go build -o $dir/$(basename $dir) ./$dir
    else
        echo "(skipped $dir)"
    fi
done
```

go list ./... 会找出当前目录下所有的go文件

而$()就是变量替换，对每个文件使用go test命令来运行test文件。

GOMAXPROCS是环境变量的意思。



图中for循环的dir打印出来时这个，说明只包括目录不包括文件。

然后${%/}表示去除/。

![截屏2020-09-10 下午8.03.16](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午8.03.16.png)



grep -q表示逻辑判断，如果找到返回1，没有0.

^和$表示精确匹配package main，也就是有main这些目录下，有main文件的时候，进行go build编译，然后-o是指定输出文件。



basename表示目录的文件名。

![截屏2020-09-10 下午8.12.46](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午8.12.46.png)



#### travis

![截屏2020-09-10 下午8.40.49](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午8.40.49.png)



awk就是把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行各种分析处理

awk工作流程是这样的：读入有'\n'换行符分割的一条记录，然后将记录按指定的域分隔符划分域，填充域，$0则表示所有域,$1表示第一个域,$n表示第n个域。默认域分隔符是"空白键" 或 "[tab]键"



https://www.cnblogs.com/hepeilinnow/p/10331095.html



第二个-F以.作为分割符打印出版本号。



https://blog.csdn.net/u012351051/article/details/78939551

比较符



https://www.cnblogs.com/lit10050528/p/4915001.html

[[]]和[]的区别。



```go
#!/bin/bash

set -e

go_minor_version=$(go version | awk '{print $3}' | awk -F. '{print $2}')
if [[ $go_minor_version -gt 10 ]]; then
    export GO111MODULE=on
else
    wget -O dep https://github.com/golang/dep/releases/download/v0.5.0/dep-linux-amd64
    chmod +x dep
    ./dep ensure
fi

./test.sh
./coverage.sh --coveralls
```



#### dist.sh

注意$()是表达式的结果做变量替换，而${}是直接的变量替换。

另外可以直接在""里头执行命令。

![截屏2020-09-10 下午10.08.09](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午10.08.09.png)





![截屏2020-09-10 下午10.14.38](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午10.14.38.png)直接使用的话BASHSOURCE[0]表示当前执行的脚本所在的目录以及文件名，而dirname打印出来是., cd .表示进入当前目录，pwd则是显示值。

不直接pwd，显示值，是因为根本没有进入所以无法显示。



awk的NF表示最后一个字符

而<表示输入重定向。



![截屏2020-09-10 下午10.29.37](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午10.29.37.png)

这显示的是系统类型。



mktemp表示创建临时文件-d表示目录。TMPDIR就是环境变量

![截屏2020-09-10 下午10.25.16](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午10.25.16.png)



Tar压缩文件

https://blog.csdn.net/u014642915/article/details/86579575



![截屏2020-09-10 下午10.49.27](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-10 下午10.49.27.png)

使用环境变量作为make的传递参数。



```go
#!/bin/bash

# 1. commit to bump the version and update the changelog/readme
# 2. tag that commit
# 3. use dist.sh to produce tar.gz for all platforms
# 4. upload *.tar.gz to bitly s3 bucket
# 5. docker push nsqio/nsq
# 6. push to nsqio/master
# 7. update the release metadata on github / upload the binaries there too
# 8. update nsqio/nsqio.github.io/_posts/2014-03-01-installing.md
# 9. send release announcement emails
# 10. update IRC channel topic
# 11. tweet

set -e

DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
rm -rf   $DIR/dist/docker
mkdir -p $DIR/dist/docker

GOFLAGS='-ldflags="-s -w"'
arch=$(go env GOARCH)
version=$(awk '/const Binary/ {print $NF}' < $DIR/internal/version/binary.go | sed 's/"//g')
goversion=$(go version | awk '{print $3}')

echo "... running tests"
./test.sh

for os in linux darwin freebsd windows; do
    echo "... building v$version for $os/$arch"
    BUILD=$(mktemp -d ${TMPDIR:-/tmp}/nsq-XXXXX)
    TARGET="nsq-$version.$os-$arch.$goversion"
    GO111MODULE=on GOOS=$os GOARCH=$arch CGO_ENABLED=0 \
        make DESTDIR=$BUILD PREFIX=/$TARGET BLDFLAGS="$GOFLAGS" install
    pushd $BUILD
    sudo chown -R 0:0 $TARGET
    tar czvf $TARGET.tar.gz $TARGET
    mv $TARGET.tar.gz $DIR/dist
    popd
    make clean
    sudo rm -r $BUILD
done

docker build -t nsqio/nsq:v$version .
if [[ ! $version == *"-"* ]]; then
    echo "Tagging nsqio/nsq:v$version as the latest release."
    docker tag nsqio/nsq:v$version nsqio/nsq:latest
fi
```

#### coverage.sh



```go
#!/bin/bash
# Generate test coverage statistics for Go packages.
#
# Works around the fact that `go test -coverprofile` currently does not work
# with multiple packages, see https://code.google.com/p/go/issues/detail?id=6909
#
# Usage: coverage.sh [--html|--coveralls]
#
#     --html      Additionally create HTML report
#     --coveralls Push coverage statistics to coveralls.io
#

set -e

workdir=.cover
profile="$workdir/cover.out"
mode=count

generate_cover_data() {
    rm -rf "$workdir"
    mkdir "$workdir"

    for pkg in "$@"; do
        f="$workdir/$(echo $pkg | tr / -).cover"
        go test -covermode="$mode" -coverprofile="$f" "$pkg"
    done

    echo "mode: $mode" >"$profile"
    grep -h -v "^mode:" "$workdir"/*.cover >>"$profile"
}

show_html_report() {
    go tool cover -html="$profile" -o="$workdir"/coverage.html
}

show_csv_report() {
    go tool cover -func="$profile" -o="$workdir"/coverage.csv
}

push_to_coveralls() {
    echo "Pushing coverage statistics to coveralls.io"
    # ignore failure to push - it happens
    $GOPATH/bin/goveralls -coverprofile="$profile" \
                          -service=travis-ci       \
                          -ignore="nsqadmin/bindata.go" || true
}

generate_cover_data $(go list ./... | grep -v /vendor/)
show_csv_report

case "$1" in
"")
    ;;
--html)
    show_html_report ;;
--coveralls)
    push_to_coveralls ;;
*)
    echo >&2 "error: invalid option: $1"; exit 1 ;;
esac
```

1. $@

- 传递给脚本或函数的所有参数。



Tr为字符处理函数。

这里是把字符处理的时候把文件名从/变换成-。

https://www.cnblogs.com/wuyuetan/p/3778015.html



case 中$1表示第一个参数 

;; 表示break退出循环

*）表示默认 esac表示结束。



#### test-go-fmt

![截屏2020-09-12 下午8.42.01](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-12 下午8.42.01.png)



Gofmt

```bash
  -l    list files whose formatting differs from gofmt's
```



-z表示如果为空

![截屏2020-09-12 下午8.51.56](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-12 下午8.51.56.png)

```go
#!/usr/bin/env bash
set -euo pipefail
T="$(mktemp)"
find . -name '*.go' | xargs gofmt -l > "$T"

if [ -n "$(cat $T)" ]; then
 echo "Following Go code is not formatted."
 echo "-----------------------------------"
 cat "$T"
 echo "-----------------------------------"
 echo "Run 'go fmt ./...' in your source directory"
 rm -f "$T"
 exit 1
fi
rm -f "$T"
```

#### release.sh

```go
#!/bin/bash

# Updates the Version variables, commits, tags and signs

set -e
set -x

version="$1"

if [ -z $version ]; then
   echo "Need a version!"
   exit 1  
fi

make clean
sed -i "s/Version = semver\.MustParse.*$/Version = semver.MustParse(\"$version\")/" version/version.go
sed -i "s/const Version.*$/const Version = \"$version\"/" cmd/ipfs-cluster-ctl/main.go
git commit -S -a -m "Release $version"
lastver=`git tag -l | grep -E 'v[0-9]+\.[0-9]+\.[0-9]+$' | tail -n 1`
echo "Tag for Release ${version}" > tag_annotation
echo >> tag_annotation
git log --pretty=oneline ${lastver}..HEAD >> tag_annotation
git tag -a -s -F tag_annotation v$version
rm tag_annotation
```

sed -i表示直接修改读取的文件内容，而不是输出到终端。

而s动作https://www.cnblogs.com/xiaoliu66007/p/11059527.html，表示替换。

中间的*就表示开头

![截屏2020-09-12 下午9.08.39](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-12 下午9.08.39.png)



#### collect-profiles.sh

通常shell 搜索

一，shell内置，$1

二，shell函数

三，shell内置命令

四，环境变量



而command是shell内置变量，用来检验对应的命令是否存在。

并且可以让脚本函数于常规命令不同时候，执行常规命令。



![截屏2020-09-19 上午10.45.05](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-19 上午10.45.05.png)



uname -n返回当前主机名

![截屏2020-09-19 上午10.43.24](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-19 上午10.43.24.png)



Tar -C则是把压缩文件的目标的目录替换。



代码中把当前目录换成了.。



IFS是改变什么为分割符在变量替换的时候。

比如我们设置一个变量

Var = ab::cd

如果是echo $var ，输出就是ab::cd 

但是如果设置IFS为：

则会输出ab cd

```GO
IFS=$'\n\t'
```

```go
#!/usr/bin/env bash

# collect-profiles.sh
#
# Collects go profile information from a running `ipfs` daemon.
# Creates an archive including the profiles, profile graph svgs,
# ...and where available, a copy of the `ipfs` binary on the PATH.
#
# Please run this script and attach the profile archive it creates
# when reporting bugs at https://github.com/ipfs/go-ipfs/issues

set -euo pipefail
IFS=$'\n\t'

SOURCE_URL="${1:-http://127.0.0.1:5001}"
tmpdir=$(mktemp -d)
export PPROF_TMPDIR="$tmpdir"
pushd "$tmpdir" > /dev/null

if command -v ipfs > /dev/null 2>&1; then // 把ipfs命令移到当前目录下
  cp "$(command -v ipfs)" ipfs
fi

echo Collecting goroutine stacks
curl -s -o goroutines.stacks "$SOURCE_URL"'/debug/pprof/goroutine?debug=2'

echo Collecting goroutine profile
go tool pprof -symbolize=remote -svg -output goroutine.svg "$SOURCE_URL/debug/pprof/goroutine"

echo Collecting heap profile
go tool pprof -symbolize=remote -svg -output heap.svg "$SOURCE_URL/debug/pprof/heap"

echo "Collecting cpu profile (~30s)"
go tool pprof -symbolize=remote -svg -output cpu.svg "$SOURCE_URL/debug/pprof/profile"

echo "Enabling mutex profiling"
curl -X POST "$SOURCE_URL"'/debug/pprof-mutex/?fraction=4'

echo "Waiting for mutex data to be updated (30s)"
sleep 30
curl -s -o mutex.txt "$SOURCE_URL"'/debug/pprof/mutex?debug=2'
go tool pprof -symbolize=remote -svg -output mutex.svg "$SOURCE_URL/debug/pprof/mutex"

echo "Disabling mutex profiling"
curl -X POST "$SOURCE_URL"'/debug/pprof-mutex/?fraction=0'

OUTPUT_NAME=ipfs-profile-$(uname -n)-$(date +'%Y-%m-%dT%H:%M:%S%z').tar.gz
echo "Creating $OUTPUT_NAME"
popd > /dev/null
tar czf "./$OUTPUT_NAME" -C "$tmpdir" .
rm -rf "$tmpdir"
```







#### maketarball.sh





chmod u表示所有的用户，g表示所有的组，o表示其他人。



cp表示复制，把当前目录文件的所有文件复制到当临时目录下。



git discribe 将会输出离当前提交最近的分支的版本号。

 The command finds the most recent tag that is reachable from a commit.

![截屏2020-09-19 上午11.41.51](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-19 上午11.41.51.png)

其中第二个表示此分支的提交次数以及前九个commit id。

而dirty表示是否与本地的不同并在输出加上dirty后缀

 Describe the state of the working tree. When the working tree matches HEAD, the output is the same as "git describe HEAD". If the working
           tree has local modification "-dirty" is appended to it. 



![截屏2020-09-19 上午11.24.43](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-19 上午11.24.43.png)

```go
#!/usr/bin/env bash
# vim: set expandtab sw=2 ts=2:

# bash safe mode
set -euo pipefail
IFS=$'\n\t'

# readlink doesn't work on macos
OUTPUT="${1:-go-ipfs-source.tar.gz}"
if ! [[ "$OUTPUT" = /* ]]; then
  OUTPUT="$PWD/$OUTPUT"
fi

GOCC=${GOCC=go}

TEMP="$(mktemp -d)"
cp -r . "$TEMP"
( cd "$TEMP" &&
  echo $PWD &&
  $GOCC mod vendor &&
  (git describe --always --match=NeVeRmAtCh --dirty 2>/dev/null || true) > .tarball &&
  chmod -R u=rwX,go=rX "$TEMP" # normalize permissions
  tar -czf "$OUTPUT" --exclude="./.git" .
  )

rm -rf "$TEMP"
```



六七月份数据 ostime 移动。2020

