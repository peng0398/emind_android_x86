系统log位置：
/data/system/dropbox/

更改framework/base下的代码后，要执行 make update-api,否则API不会更新，将导致别的模块无法引用

error: unpack failed: error Missing tree
解决方法：git push --no-thin origin HEAD:refs/for/release