---
layout: post
title: Selectively Locking An Android UI
maintitle: Selectively Locking An Android UI
tags: [android, java, thread-safe, september, 2018]
---

I found that sometimes I want to disable most parts of my UI, but have certain elements remain locked in their state when this happens.  As an example, I want a stop button to be still clickable when my start button is disabled when a simulation running.

The Android `View` doesn't seem to have an easy way to do this, so I decided to build my own.
Basically is a wrapper class that provides a thread safe way of preventing an certain `Views` to be tampered with while changing the whole UI's state.

Here is a small class that will get us started.

```
public class LockableElement {

    public View view;
    public AtomicBoolean locked;

    LockableElement(View view, boolean locked) {
        this.view = view;
        this.locked.set(locked);
    }

    void toggleEnabled(boolean isEnabled) {
        view.setEnabled(isEnabled);
        view.setClickable(isEnabled);
        view.invalidate();
    }
   
    void setLocked() {
        locked.set(true);
        toggleEnabled(false);
    }

    void setUnlocked() {
        locked.set(false);
        toggleEnabled(true);
    }

    boolean isLocked() {
        return locked.get();
    }
}
```

So to set a UI element to be disabled before, during, and after disabling the entire UI, we can create a locked element like this from an activity:
```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    
    // Where PhoneNumberInput is an EditText.
    LockableElement lockablePhoneNumber = new LockableElement(R.id.PhoneNumberInput, true);
}
```
Simple enough.  This element is currently locked.

Now, in our activity we need an easy way to take care of locking/unlocking all of the UI items.  We can achieve this by using a simple list, and checking if the element should be disabled.

```
boolean disableUI = false;
List<LockableElement> uiList = new ArrayList<>();
uiList.add(lockablePhoneNumber);
uiList.add(lockableEmailAddress);
uiList.add(lockableFirstName);

public void toggleUIState(List<LockableElement> uiList) {
    for(LockableElement lockableElement : uiList) {
        if(!lockableElement.isLocked()) {
            lockableElement.toggleEnabled(disableUI);
        }
    }
    disableUI = !disableUI;
}
```

If you see the screenshots below, here is how it will appear:

Example of an unlocked UI.
![Unlocked UI](/assets/img/EnabledUI.jpg)

Example of the UI locked with just the stop button enabled.
![Locked UI](/assets/img/DisabledUI.jpg)

Example of other elements being toggled by locking.
![Enabled email](/assets/img/EmailEnabled.jpg)

Based on certain events, the elements can be unlocked easily, like so:
```
lockablePhoneNumber.setUnlocked();
```

Hope this was helpful.

Keep it real,

Matthew
