# Golang Unit Test Practical Experience

# 1. Technologies Selection
Go Test and Gomonkey(mock).

# 2. Gomonkey Failed with "panic: permission denied" on Mac
I found this problem when I ran unit test on my local Mac. I search the reason on the Internet and found this:
> Monkey won't work on some security-oriented operating system that don't allow memory pages to be both write and execute at the same time.

So there has two methods to solve the problem:
## a. Bypass System Permissions(Validated)
```shell
cd `go env GOPATH`
git clone https://github.com/eisenxp/macos-golink-wrapper.git
mv `go env GOTOOLDIR`/link `go env GOTOOLDIR`/original_link
cp `go env GOPATH`/macos-golink-wrapper/link `go env GOTOOLDIR`/link
chmod +x `go env GOTOOLDIR`/link
```
## b. Use Docker Image(Not Verified)
```shell
brew cask install docker
docker pull golang
docker run -it --rm -v /Users/admin/go:/go --privileged docker.io/golang  bash
```

# 3. Project Path of Test Files Matters Coverage
For example, my project organized as follow.
```
main.go
handler/
    handler.go
service/
    service.go
rpc/
...
```
And I want to build unit test for handler.go and service.go. Usually, I would like to organize all test files in just one path for easily manage (eg. Java Maven). Also, if I needed to split test code from bussiness code and build an independent project for test code, it would be easily deployed. The project's organzise like this.
```
main.go
handler/
    handler.go
service/
    service.go
rpc/
test/
  handler/
    handler_test.go
  service/
    service_test.go
...
```
I will run all test and generate coverage report with commands like this.
```shell
go test -gcflags=all=-l -count=1 -coverprofile=c.out ./... # -gcflags=all=-l forbid compiler to generate inline function, -count=1 forbid go test with cache, ./... means this run test covers current path and all sub paths.
go tool cover -html=c.out -o coverage.html
```
We may get inaccurate coverage for this test. Because golang doesn't support calculating coverage with multiple package.
> The reason for this was that go test -cover is per default only recording code coverage for the package that is currently tested, not for all the packages from the project.

How to solve this problem? There has two ideas for this dilemma. Write test file within the path containing related buissiness code file. You can find this phenomenon in many golang project. Also, this pattern is good for testing private methods in the same package. Another idea is recursively handle tests in all packages and manually calculate coverage with script. I have not looked in detail to this idea and just list references here.
<br/>[Go多个pkg的单元测试覆盖率](https://singlecool.com/2017/06/11/golang-test/)
<br/>[Get Accurate Code Coverage in Golang (Go) Across Multiple Packages](https://www.ory.sh/golang-go-code-coverage-accurate/)

# 4. Run Test for Single Test File Reports "command-line-arguments [build failed]"
Project organized as follow.
```
service/
    service.go
    service_test.go
...
```
I ran go test with commond:
```shell
go test -gcflags=all=-l -count=1 service/service_test.go
```
But console echos:
```shell
service/service_test.go:xx:xx: undefined: fun
FAIL    command-line-arguments [build failed]
```
This means compiler cannot find ```fun()``` in ```service.go```. But ```service_test.go``` and ```service.go``` are in same path and it should work.
The reason for this failure is that go test will generate a virtual package "command-line-arguments" for the designated test file ```service_test.go``` and it makes ```service.go``` and ```service_test.go``` in different packages. The function ```fun()``` is private and therefore ```service_test.go``` in "command-line-arguments" pacakge cannot refer to it.
How to solve this problem? Told compiler what are other source files. The commond is this.
```shell
go test -gcflags=all=-l -count=1 service/service_test.go service/service.go
```

