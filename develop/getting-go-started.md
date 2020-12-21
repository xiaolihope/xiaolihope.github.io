---
title: Getting Go Started
date: 2016-11-30T11:26:51+08:00
---

## Build Go From Source Code

### Git clone

```bash
cd ~
git clone https://github.com/golang/go
cp -r go go1.4
```
### Build Go1.4

```bash
cd ~/go1.4
git checkout origin/release-branch.go1.4
cd src
./make.bash
```
### Build Go 1.7

```bash
cd ~/go1.7
git checkout origin/release-branch.go1.7
cd src
./make.bash
```

### Setup Go Env ###

```
cat > /etc/profile.d/golang.sh << EOF
export GOPATH=~/gource
export GOROOT=~/go
export GOROOT_BOOTSTRAP=~/go1.4
PATH=$GOROOT/bin:$GOPATH/bin:$PATH
EOF
mkdir ~/gource
```

## Go Get ##

```bash
go get github.com/spf13/hugo
```
So `go get` does:

1. Get source code,
2. Get dependeicies,
3. Build and install.

Create `~/.netrc`:

```
machine github.ibm.com login nanjj password <secret>
```

```bash
go get github.ibm.com/edge/redstone
```

## Test Driven Study Pattern ##

Test driven study pattern.

### Go Test Example ###

```bash
go test\$
```

{{< code "m01_test.go" >}}

### Go Test ###

```bash
go test -run TestStrings\$
```

{{< code "m02_test.go" >}}

### Go Run ###

```# go run m03.go```

{{< code "m03.go" >}}
