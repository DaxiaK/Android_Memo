# UiScrollable bug

## what's wrong ?

`UiScrollbale.getChildByText()` sometime will scroll a little times that it can't find  
the uiobject in screen.

The root cause may be from system before Android 6.0.


## flow

When you call the `getChildByText` , its flow is :

`getChildByText -> scrollIntoView -> scrollForward -> scrollSwipe`

In `scrollSwipe` , system will run swipe command , wait , than find all `AccessibilityEvent` and `filter AccessibilityEvent.TYPE_VIEW_SCROLLED` in current screen. This step make UiScrollable scroll again or check it in the end.

But the bug is here , system can't return right `AccessibilityEvent` before Android 6.0 , especially in Android 4.4 .

```
public boolean scrollSwipe(final int downX, final int downY, final int upX, final int upY, final int steps) 
 { 
Runnable command = new Runnable() { 
    @Override public void run() 
   {swipe(downX, downY, upX, upY, steps);} 
  }; 

   // Collect all accessibility events generated during the swipe command and get the last event
   ArrayList<AccessibilityEvent> events = new ArrayList<AccessibilityEvent>(); 

   runAndWaitForEvents(command, new
   EventCollectingPredicate(AccessibilityEvent.TYPE_VIEW_SCROLLED, events), 
   Configurator.getInstance().getScrollAcknowledgmentTimeout());

    //**Root cause**
    //Some times the events list size will be 0 ,
    //but in fact it can scroll again. 
    Log.e(LOG_TAG,"events size = " + events.size()); 

    AccessibilityEvent event = getLastMatchingEvent(events,
         AccessibilityEvent.TYPE_VIEW_SCROLLED); 

    if (event == null) { 
     // end of scroll since no new scroll events received
     recycleAccessibilityEvents(events);
     return false; 
     } 
...
..
. 
}
```

## solve

I think the best way is that Overriding the code and defining the suitable MaxSearchSwipes , and if the item is over this size , make the test fail.


```
public class UiScrollableFix extends UiScrollable {
public UiScrollableFix(UiSelector container) 
{ super(container); }

 @Override
 public boolean scrollIntoView(UiSelector selector) throws UiObjectNotFoundException { 
  Tracer.trace(selector);
  // if we happen to be on top of the text we want then return here UiSelector 
  childSelector = getSelector().childSelector(selector); 
  if (exists(childSelector)) { 
  return (true); 
   } else {  
  // we will need to reset the search from the beginning to start search     
  scrollToBeginning(getMaxSearchSwipes());
  if (exists(childSelector)) { 
  return (true); }

  for (int x = 0; x < getMaxSearchSwipes(); x++) { 
  boolean scrolled = scrollForward(); 
  if (exists(childSelector)) { 
   return true; 
   }

    //For avoiding the scroll fail. 
   if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) { 
      if (!scrolled) { 
      return false; 
       } 
     } 
   }
 } 
return false; 
}
}
```





