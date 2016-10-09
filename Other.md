- 系统log位置：
/data/system/dropbox/

- 更改framework/base下的代码后，要执行 make update-api,否则API不会更新，将导致别的模块无法引用

- error: unpack failed: error Missing tree
解决方法：git push --no-thin origin HEAD:refs/for/release

- repo upload  Permission denied(publickey)  其原因是ssh账号默认使用的是email账号的id部分，而如果username和email的id部分不同的话就不对了，执行 git confit --global review.review_url.username MYNAME 即可解决问题。

		
- Enabling full system-level freeform window support

	If you have access to a computer with the Android SDK installed, 
	you can enable support for freeform mode at the system level by running the following adb shell command:
	settings put global enable_freeform_support 1
	
	Reboot your device after running the above command, and a new button will appear for app entries in your device's recent apps page to enter/exit freeform mode for a given app.