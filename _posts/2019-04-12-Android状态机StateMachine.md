---
layout:     post                    # 使用的布局（不需要改）
title:     Android状态机StateMachine    # 标题 
subtitle:  StateMachine源码分析以及使用           #副标题
date:       2019-04-12              # 时间
author:     Jack                      # 作者
header-img: img/post-bg-einstein.jpg  #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
    - 源码
    - 状态机 


---

# Android状态机StateMachine

## 1.StateMachine 源码分析

### 1.1 StateMachine 简介

> The state machine defined here is a hierarchical state machine which processes messages and can have states arranged hierarchically.

### 1.2 StateMachine 适用范围
1. 一个对象的行为取决于它的状态，并且它必须在运行时候根据状态来改变行为；

2. 一个操作中含有庞大的多分支条件语句，且这些分支依赖于该对象的状态。

    在Android系统源码中，蓝牙，WIFI的状态处理就使用的StateMachine

### 1.3 StateMachine 作用
1. 将状态封装成对象，在特定的状态下执行不同的操作，得到不同的行为模式；
2. 将状态和行为封装在一起，解决庞大分支语句带来的阅读性差和不方便扩展问题，降低程序的复杂性和提升灵活性

### 1.4 StateMachine 各个模块的作用
![](https://ws1.sinaimg.cn/large/b5ec746bgy1g24lwi2s0ij20k60jl0to.jpg)

#### 1.4.1 State

```java
public class State implements IState
{
　　protected State() {}
　　public void enter() {}
　　public void exit() {}
　　public boolean processMessage(Message msg) {}
　　public String getName() {}

}
```
状态的基类，StateMachine中的状态都是由State派生而来。

#### 1.4.2 StateMachine
StateMachine的构造函数都是protected类型，不能实例化，都是由其子类进行初始化操作。
StateMachine有三个重载的构造函数，一个是通过指定消息循环对象来构造状态机，一个是通过指定Handler对象来构造状态机，另一个则是直接启动一个消息循环线程来构造一个状态机；他们均通过initStateMachine函数来初始化该状态机

```java
   /**
     * Initialize.
     *
     * @param looper for this state machine
     * @param name of the state machine
     */
    private void initStateMachine(String name, Looper looper) {
        mName = name;
        mSmHandler = new SmHandler(looper, this);
    }

    /**
     * Constructor creates a StateMachine with its own thread.
     *
     * @param name of the state machine
     */
    protected StateMachine(String name) {
        mSmThread = new HandlerThread(name);
        mSmThread.start();
        Looper looper = mSmThread.getLooper();

        initStateMachine(name, looper);
    }

    /**
     * Constructor creates a StateMachine using the looper.
     *
     * @param name of the state machine
     */
    protected StateMachine(String name, Looper looper) {
        initStateMachine(name, looper);
    }

    /**
     * Constructor creates a StateMachine using the handler.
     *
     * @param name of the state machine
     */
    protected StateMachine(String name, Handler handler) {
        initStateMachine(name, handler.getLooper());
    }

    /**
     * Add a new state to the state machine
     * @param state the state to add
     * @param parent the parent of state
     */
    protected final void addState(State state, State parent) {
        mSmHandler.addState(state, parent);
    }
```

#### 1.4.3 SmHandler
SmHandler是消息处理派发和状态控制切换的核心，运行在单独的线程上。

```java
    private static class SmHandler extends Handler {
        ...
        /** State used when state machine is halted */
        private HaltingState mHaltingState = new HaltingState();

        /** State used when state machine is quitting */
        private QuittingState mQuittingState = new QuittingState();

        /** Reference to the StateMachine */
        private StateMachine mSm;

        /**
         * Information about a state.
         * Used to maintain the hierarchy.
         */
        private class StateInfo {
            /** The state */
            State state;

            /** The parent of this state, null if there is no parent */
            StateInfo parentStateInfo;

            /** True when the state has been entered and on the stack */
            boolean active;
        }

        /** The map of all of the states in the state machine */
        private HashMap<State, StateInfo> mStateInfo = new HashMap<State, StateInfo>();

        /** The initial state that will process the first message */
        private State mInitialState;

        /** The destination state when transitionTo has been invoked */
        private State mDestState;

        /** The list of deferred messages */
        private ArrayList<Message> mDeferredMessages = new ArrayList<Message>();
        /**
         * Constructor
         *
         * @param looper for dispatching messages
         * @param sm the hierarchical state machine
         */
        private SmHandler(Looper looper, StateMachine sm) {
            super(looper);
            mSm = sm;
            addState(mHaltingState, null);
            addState(mQuittingState, null);
        }
        ...
```
SmHandler 中有三个内部类

- StateInfo（存储当前State和其parentState，以及是否是激活状态，用来构建树形结构）

  在StateMachine类中，StateInfo就是包装State组成一个Node，建立State父子关系。

  ```java
  private HashMap<State, StateInfo> mStateInfo = new HashMap<State, StateInfo>();
  ```

- HaltingState和QuittingState都是从State派生，重写了processMessage方法，用于状态为停止后的逻辑处理

```java
       /**
         * State entered when transitionToHaltingState is called.
         */
        private class HaltingState extends State {
            @Override
            public boolean processMessage(Message msg) {
                mSm.haltedProcessMessage(msg);
                return true;
            }
        }

        /**
         * State entered when a valid quit message is handled.
         */
        private class QuittingState extends State {
            @Override
            public boolean processMessage(Message msg) {
                return NOT_HANDLED;
            }
        }
```

下面列出SmHandler核心接口实现：

```java
/**
         * Handle messages sent to the state machine by calling
         * the current state's processMessage. It also handles
         * the enter/exit calls and placing any deferred messages
         * back onto the queue when transitioning to a new state.
         */
        @Override
        public final void handleMessage(Message msg) {
           ...
        }

        /**
         * Do any transitions
         * @param msgProcessedState is the state that processed the message
         */
        private void performTransitions(State msgProcessedState, Message msg) {
            ...
        }
         /**
         * Complete the construction of the state machine.
         */
        private final void completeConstruction() {
            ...
        }
        /**
         * Process the message. If the current state doesn't handle
         * it, call the states parent and so on. If it is never handled then
         * call the state machines unhandledMessage method.
         * @return the state that processed the message
         */
        private final State processMsg(Message msg) {
            ...
        }
        /**
         * Initialize StateStack to mInitialState.
         */
        private final void setupInitialStateStack() {
            ...
        }
        /**
         * Add a new state to the state machine. Bottom up addition
         * of states is allowed but the same state may only exist
         * in one hierarchy.
         *
         * @param state the state to add
         * @param parent the parent of state
         * @return stateInfo for this state
         */
        private final StateInfo addState(State state, State parent) {
             ...
        }
        /** @see StateMachine#setInitialState(State) */
        private final void setInitialState(State initialState) {
           ...
        }

        /** @see StateMachine#transitionTo(IState) */
        private final void transitionTo(IState destState) {
            mDestState = (State) destState;
            if (mDbg) mSm.log("transitionTo: destState=" + mDestState.getName());
        }

        /** @see StateMachine#deferMessage(Message) */
        private final void deferMessage(Message msg) {
            if (mDbg) mSm.log("deferMessage: msg=" + msg.what);

            /* Copy the "msg" to "newMsg" as "msg" will be recycled */
            Message newMsg = obtainMessage();
            newMsg.copyFrom(msg);

            mDeferredMessages.add(newMsg);
        }

        /** @see StateMachine#quit() */
        private final void quit() {
            if (mDbg) mSm.log("quit:");
            sendMessage(obtainMessage(SM_QUIT_CMD, mSmHandlerObj));
        }

        /** @see StateMachine#quitNow() */
        private final void quitNow() {
            if (mDbg) mSm.log("quitNow:");
            sendMessageAtFrontOfQueue(obtainMessage(SM_QUIT_CMD, mSmHandlerObj));
        }
```
可以看到SmHandler是消息转发和处理的核心。

接下来根据StateMachine的标准使用流程一步一步分析：

```java
class HelloWorld extends StateMachine {
    HelloWorld(String name) {
        super(name);
        addState(mState1);
        setInitialState(mState1);
    }

    public static HelloWorld makeHelloWorld() {
        HelloWorld hw = new HelloWorld("hw");
        hw.start();
        return hw;
    }

    class State1 extends State {
        @Override 
        public boolean processMessage(Message message) {
            log("Hello World");
            return HANDLED;
        }
    }
    State1 mState1 = new State1();
}

void testHelloWorld() {
    HelloWorld hw = makeHelloWorld();
    hw.sendMessage(hw.obtainMessage());
}
```

1.在StateMachine初始化时，创建SmHandler对象

```java
  private void initStateMachine(String name, Looper looper) {
        mName = name;
        mSmHandler = new SmHandler(looper, this);
    }
```

2.在SmHandler构造时，默认添加了mHaltingState、mQuittingState

```java
        private SmHandler(Looper looper, StateMachine sm) {
            super(looper);
            mSm = sm;

            addState(mHaltingState, null);
            addState(mQuittingState, null);
        }
```

3.调用SmHandler的addState，格式化树形层次结构的mStateInfo

```java
       /**
         * Add a new state to the state machine. Bottom up addition
         * of states is allowed but the same state may only exist
         * in one hierarchy.
         *
         * @param state the state to add
         * @param parent the parent of state
         * @return stateInfo for this state
         */
        private final StateInfo addState(State state, State parent) {
            if (mDbg) {
                mSm.log("addStateInternal: E state=" + state.getName() + ",parent="
                        + ((parent == null) ? "" : parent.getName()));
            }
            StateInfo parentStateInfo = null;
            if (parent != null) {
                parentStateInfo = mStateInfo.get(parent);
                if (parentStateInfo == null) {
                    // Recursively add our parent as it's not been added yet.
                    parentStateInfo = addState(parent, null);
                }
            }
            StateInfo stateInfo = mStateInfo.get(state);
            if (stateInfo == null) {
                stateInfo = new StateInfo();
                mStateInfo.put(state, stateInfo);
            }

            // Validate that we aren't adding the same state in two different hierarchies.
            if ((stateInfo.parentStateInfo != null)
                    && (stateInfo.parentStateInfo != parentStateInfo)) {
                throw new RuntimeException("state already added");
            }
            stateInfo.state = state;
            stateInfo.parentStateInfo = parentStateInfo;
            stateInfo.active = false;
            if (mDbg) mSm.log("addStateInternal: X stateInfo: " + stateInfo);
            return stateInfo;
        }
```

那么目前的树形层次结构(不计mHaltingState、mQuittingState，这两个状态是负责内部逻辑处理的)中就只有mState1.

4.设置状态机状态setInitialState

```java
        /** @see StateMachine#setInitialState(State) */
        private final void setInitialState(State initialState) {
            if (mDbg) mSm.log("setInitialState: initialState=" + initialState.getName());
            mInitialState = initialState;
        }
```

5.StateMachine准备完毕，开始启动，调整mStateStack中各种State堆栈的次序

```java
    /**
     * Start the state machine.
     */
    public void start() {
        // mSmHandler can be null if the state machine has quit.
        SmHandler smh = mSmHandler;
        if (smh == null) return;

        /** Send the complete construction message */
        smh.completeConstruction();
    }
```

smh.completeConstruction()这个方法用来构建状态机运行模型

```java
 /**
         * Complete the construction of the state machine.
         */
        private final void completeConstruction() {
            if (mDbg) mSm.log("completeConstruction: E");

            /**
             * Determine the maximum depth of the state hierarchy
             * so we can allocate the state stacks.
             */
            int maxDepth = 0;
            for (StateInfo si : mStateInfo.values()) {
                int depth = 0;
                for (StateInfo i = si; i != null; depth++) {
                    i = i.parentStateInfo;
                }
                if (maxDepth < depth) {
                    maxDepth = depth;
                }
            }
            if (mDbg) mSm.log("completeConstruction: maxDepth=" + maxDepth);

            mStateStack = new StateInfo[maxDepth];
            mTempStateStack = new StateInfo[maxDepth];
            setupInitialStateStack();

            /** Sending SM_INIT_CMD message to invoke enter methods asynchronously */
            sendMessageAtFrontOfQueue(obtainMessage(SM_INIT_CMD, mSmHandlerObj));

            if (mDbg) mSm.log("completeConstruction: X");
        }

         /**
         * Initialize StateStack to mInitialState.
         */
        private final void setupInitialStateStack() {
            if (mDbg) {
                mSm.log("setupInitialStateStack: E mInitialState=" + mInitialState.getName());
            }

            StateInfo curStateInfo = mStateInfo.get(mInitialState);
            for (mTempStateStackCount = 0; curStateInfo != null; mTempStateStackCount++) {
                mTempStateStack[mTempStateStackCount] = curStateInfo;
                curStateInfo = curStateInfo.parentStateInfo;
            }

            // Empty the StateStack
            mStateStackTopIndex = -1;

            moveTempStateStackToStateStack();
        }
```

6.StateMachine消息处理sendMessage

```java
    /**
     * Enqueue a message to this state machine.
     *
     * Message is ignored if state machine has quit.
     */
    public final void sendMessage(int what) {
        // mSmHandler can be null if the state machine has quit.
        SmHandler smh = mSmHandler;
        if (smh == null) return;

        smh.sendMessage(obtainMessage(what));
    }
```

SmHandler是从Handler派生的，因此对应于sendMessage的是handleMessage:

```java
        /**
         * Handle messages sent to the state machine by calling
         * the current state's processMessage. It also handles
         * the enter/exit calls and placing any deferred messages
         * back onto the queue when transitioning to a new state.
         */
        @Override
        public final void handleMessage(Message msg) {
            if (!mHasQuit) {
                if (mDbg) mSm.log("handleMessage: E msg.what=" + msg.what);

                /** Save the current message */
                mMsg = msg;

                /** State that processed the message */
                State msgProcessedState = null;
                if (mIsConstructionCompleted) {
                    /** Normal path */
                    msgProcessedState = processMsg(msg);
                } else if (!mIsConstructionCompleted && (mMsg.what == SM_INIT_CMD)
                        && (mMsg.obj == mSmHandlerObj)) {
                    /** Initial one time path. */
                    mIsConstructionCompleted = true;
                    invokeEnterMethods(0);
                } else {
                    throw new RuntimeException("StateMachine.handleMessage: "
                            + "The start method not called, received msg: " + msg);
                }
                performTransitions(msgProcessedState, msg);

                // We need to check if mSm == null here as we could be quitting.
                if (mDbg && mSm != null) mSm.log("handleMessage: X");
            }
        }
```

实际的消息处理调用了processMsg：

```java
/**
         * Process the message. If the current state doesn't handle
         * it, call the states parent and so on. If it is never handled then
         * call the state machines unhandledMessage method.
         * @return the state that processed the message
         */
        private final State processMsg(Message msg) {
            StateInfo curStateInfo = mStateStack[mStateStackTopIndex];
            if (mDbg) {
                mSm.log("processMsg: " + curStateInfo.state.getName());
            }

            if (isQuit(msg)) {
                transitionTo(mQuittingState);
            } else {
                while (!curStateInfo.state.processMessage(msg)) {
                    /**
                     * Not processed
                     */
                    curStateInfo = curStateInfo.parentStateInfo;
                    if (curStateInfo == null) {
                        /**
                         * No parents left so it's not handled
                         */
                        mSm.unhandledMessage(msg);
                        break;
                    }
                    if (mDbg) {
                        mSm.log("processMsg: " + curStateInfo.state.getName());
                    }
                }
            }
            return (curStateInfo != null) ? curStateInfo.state : null;
        }

        /** Validate that the message was sent by quit or quitNow. */
        private final boolean isQuit(Message msg) {
            return (msg.what == SM_QUIT_CMD) && (msg.obj == mSmHandlerObj);
        }
```

在这个函数中可以看到，如果是显示调用了quit和quitNow接口，就会把Status状态转换为mQuittingState；其他正常调用时，首先会判断当前Status是否HANDLED了消息，如果是NOT_HANDLED状态，就会递归到父类Status中处理消息。

processMessage方法中当前state先处理消息，不处理则丢给parent state，这些StateMachine对象通过同一个Handler和Loop，实现了Mesage的传递和责任链式的处理。



7.消息分发后同步状态机performTransitions

```java
 /**
         * Do any transitions
         * @param msgProcessedState is the state that processed the message
         */
        private void performTransitions(State msgProcessedState, Message msg) {
            /**
             * If transitionTo has been called, exit and then enter
             * the appropriate states. We loop on this to allow
             * enter and exit methods to use transitionTo.
             */
            State orgState = mStateStack[mStateStackTopIndex].state;

            /**
             * Record whether message needs to be logged before we transition and
             * and we won't log special messages SM_INIT_CMD or SM_QUIT_CMD which
             * always set msg.obj to the handler.
             */
            boolean recordLogMsg = mSm.recordLogRec(mMsg) && (msg.obj != mSmHandlerObj);

            if (mLogRecords.logOnlyTransitions()) {
                /** Record only if there is a transition */
                if (mDestState != null) {
                    mLogRecords.add(mSm, mMsg, mSm.getLogRecString(mMsg), msgProcessedState,
                            orgState, mDestState);
                }
            } else if (recordLogMsg) {
                /** Record message */
                mLogRecords.add(mSm, mMsg, mSm.getLogRecString(mMsg), msgProcessedState, orgState,
                        mDestState);
            }

            State destState = mDestState;
            if (destState != null) {
                /**
                 * Process the transitions including transitions in the enter/exit methods
                 */
                while (true) {
                    if (mDbg) mSm.log("handleMessage: new destination call exit/enter");

                    /**
                     * Determine the states to exit and enter and return the
                     * common ancestor state of the enter/exit states. Then
                     * invoke the exit methods then the enter methods.
                     */
                    StateInfo commonStateInfo = setupTempStateStackWithStatesToEnter(destState);
                    invokeExitMethods(commonStateInfo);
                    int stateStackEnteringIndex = moveTempStateStackToStateStack();
                    invokeEnterMethods(stateStackEnteringIndex);

                    /**
                     * Since we have transitioned to a new state we need to have
                     * any deferred messages moved to the front of the message queue
                     * so they will be processed before any other messages in the
                     * message queue.
                     */
                    moveDeferredMessageAtFrontOfQueue();

                    if (destState != mDestState) {
                        // A new mDestState so continue looping
                        destState = mDestState;
                    } else {
                        // No change in mDestState so we're done
                        break;
                    }
                }
                mDestState = null;
            }

            /**
             * After processing all transitions check and
             * see if the last transition was to quit or halt.
             */
            if (destState != null) {
                if (destState == mQuittingState) {
                    /**
                     * Call onQuitting to let subclasses cleanup.
                     */
                    mSm.onQuitting();
                    cleanupAfterQuitting();
                } else if (destState == mHaltingState) {
                    /**
                     * Call onHalting() if we've transitioned to the halting
                     * state. All subsequent messages will be processed in
                     * in the halting state which invokes haltedProcessMessage(msg);
                     */
                    mSm.onHalting();
                }
            }
        }

        /** @see StateMachine#transitionTo(IState) */
        private final void transitionTo(IState destState) {
            mDestState = (State) destState;
            if (mDbg) mSm.log("transitionTo: destState=" + mDestState.getName());
        }

        /** @see StateMachine#deferMessage(Message) */
        private final void deferMessage(Message msg) {
            if (mDbg) mSm.log("deferMessage: msg=" + msg.what);

            /* Copy the "msg" to "newMsg" as "msg" will be recycled */
            Message newMsg = obtainMessage();
            newMsg.copyFrom(msg);

            mDeferredMessages.add(newMsg);
        }

        /**
         * Move the deferred message to the front of the message queue.
         */
        private final void moveDeferredMessageAtFrontOfQueue() {
            /**
             * The oldest messages on the deferred list must be at
             * the front of the queue so start at the back, which
             * as the most resent message and end with the oldest
             * messages at the front of the queue.
             */
            for (int i = mDeferredMessages.size() - 1; i >= 0; i--) {
                Message curMsg = mDeferredMessages.get(i);
                if (mDbg) mSm.log("moveDeferredMessageAtFrontOfQueue; what=" + curMsg.what);
                sendMessageAtFrontOfQueue(curMsg);
            }
            mDeferredMessages.clear();
        }
```

这个阶段是Status的enter/exit逻辑处理：

```java
 private final StateInfo setupTempStateStackWithStatesToEnter(State destState) {
            /**
             * Search up the parent list of the destination state for an active
             * state. Use a do while() loop as the destState must always be entered
             * even if it is active. This can happen if we are exiting/entering
             * the current state.
             */
            mTempStateStackCount = 0;
            StateInfo curStateInfo = mStateInfo.get(destState);
            do {
                mTempStateStack[mTempStateStackCount++] = curStateInfo;
                curStateInfo = curStateInfo.parentStateInfo;
            } while ((curStateInfo != null) && !curStateInfo.active);

            if (mDbg) {
                mSm.log("setupTempStateStackWithStatesToEnter: X mTempStateStackCount="
                        + mTempStateStackCount + ",curStateInfo: " + curStateInfo);
            }
            return curStateInfo;
        }
        /**
         * Call the exit method for each state from the top of stack
         * up to the common ancestor state.
         */
        private final void invokeExitMethods(StateInfo commonStateInfo) {
            while ((mStateStackTopIndex >= 0)
                    && (mStateStack[mStateStackTopIndex] != commonStateInfo)) {
                State curState = mStateStack[mStateStackTopIndex].state;
                if (mDbg) mSm.log("invokeExitMethods: " + curState.getName());
                curState.exit();
                mStateStack[mStateStackTopIndex].active = false;
                mStateStackTopIndex -= 1;
            }
        }
```

从setupTempStateStackWithStatesToEnter可以看出查找的是当前子类State层级中的未激活的顶层父类State，而invokeExitMethods可以看出首先会调用子类State的exit()，递归调用父类State的exit()，直至匹配到指定State（注意!curStateInfo.active这个限制条件）；

```java
	   /**
         * Move the contents of the temporary stack to the state stack
         * reversing the order of the items on the temporary stack as
         * they are moved.
         *
         * @return index into mStateStack where entering needs to start
         */
        private final int moveTempStateStackToStateStack() {
            int startingIndex = mStateStackTopIndex + 1;
            int i = mTempStateStackCount - 1;
            int j = startingIndex;
            while (i >= 0) {
                if (mDbg) mSm.log("moveTempStackToStateStack: i=" + i + ",j=" + j);
                mStateStack[j] = mTempStateStack[i];
                j += 1;
                i -= 1;
            }

            mStateStackTopIndex = j - 1;
            if (mDbg) {
                mSm.log("moveTempStackToStateStack: X mStateStackTop=" + mStateStackTopIndex
                        + ",startingIndex=" + startingIndex + ",Top="
                        + mStateStack[mStateStackTopIndex].state.getName());
            }
            return startingIndex;
        }

        /**
         * Invoke the enter method starting at the entering index to top of state stack
         */
        private final void invokeEnterMethods(int stateStackEnteringIndex) {
            for (int i = stateStackEnteringIndex; i <= mStateStackTopIndex; i++) {
                if (mDbg) mSm.log("invokeEnterMethods: " + mStateStack[i].state.getName());
                mStateStack[i].state.enter();
                mStateStack[i].active = true;
            }
        }
```

从setupInitialStateStack和moveTempStateStackToStateStack接口可知，mTempStateStack是子类在栈底模式； mStateStack是顶层父类在栈底模式；

举例一幅层级图描述如下：



![img](https:////upload-images.jianshu.io/upload_images/3613947-d72d08819a0c9ffa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/282/format/webp)

State树结构层次图

那么在sendMessage前mTempStateStack的堆栈情况：



![img](https:////upload-images.jianshu.io/upload_images/3613947-085830a3a9d9b4d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/169/format/webp)

mTempStateStack堆栈状态

mStateStack的堆栈情况：



![img](https:////upload-images.jianshu.io/upload_images/3613947-468aebc01b9894a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/169/format/webp)

mStateStack堆栈状态

因此在此阶段是先调用顶层父类State的enter()接口，然后再递归到子类的enter()逻辑处理。

## 2. StateMachine 的使用

Google源码中已经将使用案例添加到源码注释中

路径如下:

frameworks\base\core\java\com\android\internal\util\StateMachine.java

```
/* <p>When a state machine is created <code>addState</code> is used to build the
 * hierarchy and <code>setInitialState</code> is used to identify which of these
 * is the initial state. After construction the programmer calls <code>start</code>
 * which initializes and starts the state machine. The first action the StateMachine
 * is to the invoke <code>enter</code> for all of the initial state's hierarchy,
 * starting at its eldest parent. The calls to enter will be done in the context
 * of the StateMachines Handler not in the context of the call to start and they
 * will be invoked before any messages are processed. For example, given the simple
 * state machine below mP1.enter will be invoked and then mS1.enter. Finally,
 * messages sent to the state machine will be processed by the current state,
 * in our simple state machine below that would initially be mS1.processMessage.</p>
<code>
        mP1
       /   \
      mS2   mS1 ----> initial state
</code>
 * <p>After the state machine is created and started, messages are sent to a state
 * machine using <code>sendMessage</code> and the messages are created using
 * <code>obtainMessage</code>. When the state machine receives a message the
 * current state's <code>processMessage</code> is invoked. In the above example
 * mS1.processMessage will be invoked first. The state may use <code>transitionTo</code>
 * to change the current state to a new state</p>
 *
 * <p>Each state in the state machine may have a zero or one parent states and if
 * a child state is unable to handle a message it may have the message processed
 * by its parent by returning false or NOT_HANDLED. If a message is never processed
 * <code>unhandledMessage</code> will be invoked to give one last chance for the state machine
 * to process the message.</p>
 *
 * <p>When all processing is completed a state machine may choose to call
 * <code>transitionToHaltingState</code>. When the current <code>processingMessage</code>
 * returns the state machine will transfer to an internal <code>HaltingState</code>
 * and invoke <code>halting</code>. Any message subsequently received by the state
 * machine will cause <code>haltedProcessMessage</code> to be invoked.</p>
 *
 * <p>If it is desirable to completely stop the state machine call <code>quit</code> or
 * <code>quitNow</code>. These will call <code>exit</code> of the current state and its parents,
 * call <code>onQuiting</code> and then exit Thread/Loopers.</p>
 *
 * <p>In addition to <code>processMessage</code> each <code>State</code> has
 * an <code>enter</code> method and <code>exit</exit> method which may be overridden.</p>
 *
 * <p>Since the states are arranged in a hierarchy transitioning to a new state
 * causes current states to be exited and new states to be entered. To determine
 * the list of states to be entered/exited the common parent closest to
 * the current state is found. We then exit from the current state and its
 * parent's up to but not including the common parent state and then enter all
 * of the new states below the common parent down to the destination state.
 * If there is no common parent all states are exited and then the new states
 * are entered.</p>
 *
 * <p>Two other methods that states can use are <code>deferMessage</code> and
 * <code>sendMessageAtFrontOfQueue</code>. The <code>sendMessageAtFrontOfQueue</code> sends
 * a message but places it on the front of the queue rather than the back. The
 * <code>deferMessage</code> causes the message to be saved on a list until a
 * transition is made to a new state. At which time all of the deferred messages
 * will be put on the front of the state machine queue with the oldest message
 * at the front. These will then be processed by the new current state before
 * any other messages that are on the queue or might be added later. Both of
 * these are protected and may only be invoked from within a state machine.</p>
 *
 * <p>To illustrate some of these properties we'll use state machine with an 8
 * state hierarchy:</p>
<code>
          mP0
         /   \
        mP1   mS0
       /   \
      mS2   mS1
     /  \    \
    mS3  mS4  mS5  ---> initial state
</code>
 * <p>After starting mS5 the list of active states is mP0, mP1, mS1 and mS5.
 * So the order of calling processMessage when a message is received is mS5,
 * mS1, mP1, mP0 assuming each processMessage indicates it can't handle this
 * message by returning false or NOT_HANDLED.</p>
 *
 * <p>Now assume mS5.processMessage receives a message it can handle, and during
 * the handling determines the machine should change states. It could call
 * transitionTo(mS4) and return true or HANDLED. Immediately after returning from
 * processMessage the state machine runtime will find the common parent,
 * which is mP1. It will then call mS5.exit, mS1.exit, mS2.enter and then
 * mS4.enter. The new list of active states is mP0, mP1, mS2 and mS4. So
 * when the next message is received mS4.processMessage will be invoked.</p>
 *
 * <p>Now for some concrete examples, here is the canonical HelloWorld as a state machine.
 * It responds with "Hello World" being printed to the log for every message.</p>
<code>
class HelloWorld extends StateMachine {
    HelloWorld(String name) {
        super(name);
        addState(mState1);
        setInitialState(mState1);
    }

    public static HelloWorld makeHelloWorld() {
        HelloWorld hw = new HelloWorld("hw");
        hw.start();
        return hw;
    }

    class State1 extends State {
        @Override public boolean processMessage(Message message) {
            log("Hello World");
            return HANDLED;
        }
    }
    State1 mState1 = new State1();
}

void testHelloWorld() {
    HelloWorld hw = makeHelloWorld();
    hw.sendMessage(hw.obtainMessage());
}
</code>
 * <p>A more interesting state machine is one with four states
 * with two independent parent states.</p>
<code>
        mP1      mP2
       /   \
      mS2   mS1
</code>
 * <p>Here is a description of this state machine using pseudo code.</p>
 <code>
state mP1 {
     enter { log("mP1.enter"); }
     exit { log("mP1.exit");  }
     on msg {
         CMD_2 {
             send(CMD_3);
             defer(msg);
             transitonTo(mS2);
             return HANDLED;
         }
         return NOT_HANDLED;
     }
}

INITIAL
state mS1 parent mP1 {
     enter { log("mS1.enter"); }
     exit  { log("mS1.exit");  }
     on msg {
         CMD_1 {
             transitionTo(mS1);
             return HANDLED;
         }
         return NOT_HANDLED;
     }
}

state mS2 parent mP1 {
     enter { log("mS2.enter"); }
     exit  { log("mS2.exit");  }
     on msg {
         CMD_2 {
             send(CMD_4);
             return HANDLED;
         }
         CMD_3 {
             defer(msg);
             transitionTo(mP2);
             return HANDLED;
         }
         return NOT_HANDLED;
     }
}

state mP2 {
     enter {
         log("mP2.enter");
         send(CMD_5);
     }
     exit { log("mP2.exit"); }
     on msg {
         CMD_3, CMD_4 { return HANDLED; }
         CMD_5 {
             transitionTo(HaltingState);
             return HANDLED;
         }
         return NOT_HANDLED;
     }
}
</code>
 * <p>The implementation is below and also in StateMachineTest:</p>
<code>
class Hsm1 extends StateMachine {
    public static final int CMD_1 = 1;
    public static final int CMD_2 = 2;
    public static final int CMD_3 = 3;
    public static final int CMD_4 = 4;
    public static final int CMD_5 = 5;

    public static Hsm1 makeHsm1() {
        log("makeHsm1 E");
        Hsm1 sm = new Hsm1("hsm1");
        sm.start();
        log("makeHsm1 X");
        return sm;
    }

    Hsm1(String name) {
        super(name);
        log("ctor E");

        // Add states, use indentation to show hierarchy
        addState(mP1);
            addState(mS1, mP1);
            addState(mS2, mP1);
        addState(mP2);

        // Set the initial state
        setInitialState(mS1);
        log("ctor X");
    }

    class P1 extends State {
        @Override public void enter() {
            log("mP1.enter");
        }
        @Override public boolean processMessage(Message message) {
            boolean retVal;
            log("mP1.processMessage what=" + message.what);
            switch(message.what) {
            case CMD_2:
                // CMD_2 will arrive in mS2 before CMD_3
                sendMessage(obtainMessage(CMD_3));
                deferMessage(message);
                transitionTo(mS2);
                retVal = HANDLED;
                break;
            default:
                // Any message we don't understand in this state invokes unhandledMessage
                retVal = NOT_HANDLED;
                break;
            }
            return retVal;
        }
        @Override public void exit() {
            log("mP1.exit");
        }
    }

    class S1 extends State {
        @Override public void enter() {
            log("mS1.enter");
        }
        @Override public boolean processMessage(Message message) {
            log("S1.processMessage what=" + message.what);
            if (message.what == CMD_1) {
                // Transition to ourself to show that enter/exit is called
                transitionTo(mS1);
                return HANDLED;
            } else {
                // Let parent process all other messages
                return NOT_HANDLED;
            }
        }
        @Override public void exit() {
            log("mS1.exit");
        }
    }

    class S2 extends State {
        @Override public void enter() {
            log("mS2.enter");
        }
        @Override public boolean processMessage(Message message) {
            boolean retVal;
            log("mS2.processMessage what=" + message.what);
            switch(message.what) {
            case(CMD_2):
                sendMessage(obtainMessage(CMD_4));
                retVal = HANDLED;
                break;
            case(CMD_3):
                deferMessage(message);
                transitionTo(mP2);
                retVal = HANDLED;
                break;
            default:
                retVal = NOT_HANDLED;
                break;
            }
            return retVal;
        }
        @Override public void exit() {
            log("mS2.exit");
        }
    }

    class P2 extends State {
        @Override public void enter() {
            log("mP2.enter");
            sendMessage(obtainMessage(CMD_5));
        }
        @Override public boolean processMessage(Message message) {
            log("P2.processMessage what=" + message.what);
            switch(message.what) {
            case(CMD_3):
                break;
            case(CMD_4):
                break;
            case(CMD_5):
                transitionToHaltingState();
                break;
            }
            return HANDLED;
        }
        @Override public void exit() {
            log("mP2.exit");
        }
    }

    @Override
    void onHalting() {
        log("halting");
        synchronized (this) {
            this.notifyAll();
        }
    }

    P1 mP1 = new P1();
    S1 mS1 = new S1();
    S2 mS2 = new S2();
    P2 mP2 = new P2();
}
</code>
 * <p>If this is executed by sending two messages CMD_1 and CMD_2
 * (Note the synchronize is only needed because we use hsm.wait())</p>
<code>
Hsm1 hsm = makeHsm1();
synchronize(hsm) {
     hsm.sendMessage(obtainMessage(hsm.CMD_1));
     hsm.sendMessage(obtainMessage(hsm.CMD_2));
     try {
          // wait for the messages to be handled
          hsm.wait();
     } catch (InterruptedException e) {
          loge("exception while waiting " + e.getMessage());
     }
}
</code>
 * <p>The output is:</p>
<code>
D/hsm1    ( 1999): makeHsm1 E
D/hsm1    ( 1999): ctor E
D/hsm1    ( 1999): ctor X
D/hsm1    ( 1999): mP1.enter
D/hsm1    ( 1999): mS1.enter
D/hsm1    ( 1999): makeHsm1 X
D/hsm1    ( 1999): mS1.processMessage what=1
D/hsm1    ( 1999): mS1.exit
D/hsm1    ( 1999): mS1.enter
D/hsm1    ( 1999): mS1.processMessage what=2
D/hsm1    ( 1999): mP1.processMessage what=2
D/hsm1    ( 1999): mS1.exit
D/hsm1    ( 1999): mS2.enter
D/hsm1    ( 1999): mS2.processMessage what=2
D/hsm1    ( 1999): mS2.processMessage what=3
D/hsm1    ( 1999): mS2.exit
D/hsm1    ( 1999): mP1.exit
D/hsm1    ( 1999): mP2.enter
D/hsm1    ( 1999): mP2.processMessage what=3
D/hsm1    ( 1999): mP2.processMessage what=4
D/hsm1    ( 1999): mP2.processMessage what=5
D/hsm1    ( 1999): mP2.exit
D/hsm1    ( 1999): halting
</code>
*/
```

若想在项目中使用StateMachine需要将源码（路径需保持一致）导入到你所需要的项目中

![](https://ws1.sinaimg.cn/large/b5ec746bgy1g26ka6v8x7j20al05bglk.jpg)



## 3.总结

StateMachine内部有个HandlerThread，SmHandler用于处理自有消息loop的消息，还有一项重要的任务就是管理StateInfo，StateInfo包含parentState的信息，Message产生的时候，交给当前的栈顶的State处理，然后交由它的parentState继续处理，parentState处理完毕后，没有更多的parentState，Message就处理完毕了。





## 参考链接

[Android StateMachine分析](https://www.jianshu.com/p/a5435a05ae48)

[关于StateMachine的那些事儿](https://blog.csdn.net/a8403025/article/details/80666056)

[Android 状态机stateMachine的应用](https://www.2cto.com/kf/201706/651378.html)

