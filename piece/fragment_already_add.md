# Fragment already added

在使用 DialogFragment的时候经常遇到的错误：
```
Fatal Exception: java.lang.IllegalStateException: Fragment already added: n{61c4418}
```
产生的原因就是同一个`DialogFragment`多次`show(fragmentManager,tag)`的导致


#### 这个场景不可能
在我们的BaseActivity中有一个常用的加载进度弹框FragmentDialog，我们需要在网络请求的时候显示，
```
open class BaseActivity() : AppCompatActivity(){
    private val fragmentDialog: FragmentDialog by lazy {
        FragmentDialog()
    }
}
```
假设`ActivityA`***有且只有一个网络请求***，进入`ActivityA`的时候请求并显示进度条，
```
override fun onCreate(savedInstanceState: Bundle?) {
  super.onCreate(savedInstanceState)
  loadData()
  fragmentDialog.show(supportFragmentManager, this.toString())
}
```
是不是觉的代码没有问题？但是Crash工具收集的issue这个问题还不少，想不通原因，于是我们去`StackOverFlow`寻找解决办法，关键字搜索`fragment already added`，于是我们迅速的找到了答案：

> [IllegalStateException: Fragment already added in the tabhost fragment](https://stackoverflow.com/a/29393015/6293442)

于是乎我们根据指引在`BaseDialogFragment`中添加了如下代码：
```
override fun show(manager: FragmentManager, tag: String?) {
    manager.findFragmentByTag(tag)?:return
    super.show(manager, tag)
}
```
信心满满，发布！！周报中`fixBug+1`，几天后，what？怎么还有？我们返回去继续看该问题的[详细解答](https://stackoverflow.com/a/29393015/6293442),🈶️一个哥们儿这么说：
> i had already implemented to check if fragment already added then return (the code of add fragment will not be executed) , as the my application is live on the play store still i got the crash fragment is already added from the some of the devices not all the devices i am trying to figure it out from the month ago, i can not over come it, i had applied numbers of solutions and update the build on the live play store, still i am receiving the error from the some of the devices like Galaxy S20+ 5G, huawei , there fewer devices that have the issue i got, any help would be appreciated thanks

其实看似150+的vote但是没有解决所有人的问题，究竟是什么原因的呢？
最根本的原因其实就是fragment的 `show()` 和 `dismiss()` 都是***异步***的，是异步就有延迟，如果没有一个同步判断节点，这个问题就会永远存在，但是数量不并不是很多。虽然`manager.findFragmentByTag(tag)`是同步代码，但是此时可能DialogFragment还在添加的过程中，你的代码已经run到了`findFragmentByTag` 所以还是会有问题。接下来我们对解决方案进行改进：
```
class MyDialogFragment: DialogFragment() {
    override fun show(manager: FragmentManager, tag: String?) {
        manager.findFragmentByTag(tag)?:return
        super.show(manager, tag)
    }
}
class ActivityA: AppCompatActivity() {
    private var mShowing = false
    private val loadingDialog: MyDialogFragment by lazy { MyDialogFragment() }

    fun showLoading() {
        if (!mShowing) {
            mShowing = true
            loadingDialog.show(supportFragmentManager, this.toString())
        }
    }

    fun dismissLoading() {
        if (mShowing) {
            loadingDialog.dismiss()
            mShowing = false
        }
    }
}
```
经过改进，我们再次发布上线以后，？？？？？ what？这个issue依然存在😭😭😭，一口老血吐了出来。上面我们说了`show()`和`dismiss()`都是异步的，上面的代码只处理了`show()`,`dismiss()`如何处理？
如果是java我们通常利用回调处理，但是kotlin我们直接用高阶函数更方便(这里不谈性能的问题)：
```
class MyDialogFragment : DialogFragment() {
    var onClose: (() -> Unit)? = null

    override fun onDismiss(dialog: DialogInterface) {
        super.onDismiss(dialog)
        onClose?.invoke()
    }

    override fun onCancel(dialog: DialogInterface) {
        super.onCancel(dialog)
        onClose?.invoke()
    }
}

class ActivityB : AppCompatActivity() {
    private var mShowing = false
    private val loadingDialog: MyDialogFragment by lazy {
        MyDialogFragment().also {
            it.onClose = { mShowing = false }
        }
    }

    fun showLoading() {
        if (!mShowing) {
            mShowing = true
            loadingDialog.show(supportFragmentManager, this.toString())
        }
    }

    fun dismissLoading() {
        if (mShowing) {
            loadingDialog.dismiss()
        }
    }
}
```
到这里我们就解决了我们本章所要解决的问题！🎉
(感谢你的阅读，不一定能解决你的问题，但是如果能帮到你我会很高兴😊)
