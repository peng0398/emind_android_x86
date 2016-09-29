- 系统log位置：
/data/system/dropbox/

- 更改framework/base下的代码后，要执行 make update-api,否则API不会更新，将导致别的模块无法引用

- error: unpack failed: error Missing tree
解决方法：git push --no-thin origin HEAD:refs/for/release

- repo upload  Permission denied(publickey)  其原因是ssh账号默认使用的是email账号的id部分，而如果username和email的id部分不同的话就不对了，执行 git confit --global review.review_url.username MYNAME 即可解决问题。
