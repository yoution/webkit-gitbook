#浏览器js事件研究

## Add Event
**EventListenerMap::add**  WebCore/dom/EventListenerMap.cpp

### Dispatch Event
- **dispatchEventInDOM**   WebCore/dom/EventDispatcher.cpp
- *eventPath* = [target, ...,HTMLBodyElement,HTMLHtmlElement,HTMLDocument,HTMLDocument],第2个HTMLDocument的currentTarget为DomWindow
- Event:CAPTURING_PHASE   *eventPath*  [end,...,head -1] => handleEvent
- Event:AT_TARGET   *eventPath*  [head] =>  handleEvent
- Event:BUBBLING_PHASE   *eventPath*  [head -1,...,end] =>  handleEvent

#### Handle Event 中的特例
当at_target过程中的dom上同时绑定capture和bubbling事件时，事件是按照绑定顺序触发
```cpp
  if (event.eventPhase() == Event::CAPTURING_PHASE && !registeredListener->useCapture())
      continue;
  if (event.eventPhase() == Event::BUBBLING_PHASE && registeredListener->useCapture())
      continue;
```
