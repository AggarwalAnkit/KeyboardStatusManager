# KeyboardStatusManager
# Observer changes in keyboard status in android (OPEN, CLOSED)

    import android.app.Activity
    import android.arch.lifecycle.LiveData
    import android.graphics.Rect
    import android.view.View
    import android.view.ViewTreeObserver
    import java.lang.ref.WeakReference

    enum class KeyboardStatus {
        OPEN, CLOSED
    }

    class KeyboardStatusLiveData private constructor() : LiveData<KeyboardStatus>() {
        private var activityWeakReference: WeakReference<Activity>? = null
        private var rootView: View? = null
        private var globalLayoutListener: ViewTreeObserver.OnGlobalLayoutListener? = null

        constructor(activity: Activity) : this() {
            activityWeakReference?.get()?.let {
                return
            }
            activityWeakReference = WeakReference(activity)
        }

        override fun onActive() {
            super.onActive()
            addOnGlobalLayoutListener()
        }

        override fun onInactive() {
            super.onInactive()
            removeOnGlobalLayoutListener()
        }

        private fun addOnGlobalLayoutListener() {
            activityWeakReference?.get()?.let {
                rootView = rootView ?: it.findViewById(android.R.id.content)
                globalLayoutListener = ViewTreeObserver.OnGlobalLayoutListener {

                    val rect = Rect().apply { rootView!!.getWindowVisibleDisplayFrame(this) }

                    val screenHeight = rootView!!.height

                    // rect.bottom is the position above soft keypad or device button.
                    // if keypad is shown, the rect.bottom is smaller than that before.
                    val keypadHeight = screenHeight - rect.bottom

                    // 0.15 ratio is perhaps enough to determine keypad height.
                    if (keypadHeight > screenHeight * 0.15) {
                        postValue(KeyboardStatus.OPEN)
                    } else {
                        postValue(KeyboardStatus.CLOSED)
                    }
                }
                rootView!!.viewTreeObserver.addOnGlobalLayoutListener(globalLayoutListener)
            }
        }

        private fun removeOnGlobalLayoutListener() {
            rootView?.viewTreeObserver?.removeOnGlobalLayoutListener(globalLayoutListener)
            rootView = null
            globalLayoutListener = null
        }
    }

    class KeyboardManager private constructor() {
        private var keyboardStatusLiveData: KeyboardStatusLiveData? = null

        private constructor(activity: Activity) : this() {
            if (keyboardStatusLiveData == null) {
                keyboardStatusLiveData = KeyboardStatusLiveData(activity)
            }
        }

        companion object {
            private var instance: KeyboardManager? = null
            fun init(activity: Activity): KeyboardManager {
                if (instance == null) {
                    instance = KeyboardManager(activity)
                }
                return instance!!
            }

            fun status(): KeyboardStatusLiveData? {
                if (instance == null) {
                    throw IllegalAccessException("Call init with activity reference before accessing status")
                }
                return instance!!.keyboardStatusLiveData
            }
        }
    }
