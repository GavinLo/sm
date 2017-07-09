# 关于客户端开发的状态机架构

## 无序
还记得当年刚出来工作的时候，看到那冗长的switch-case代码段，一个cpp文件5w多行代码的情景历历在目。从那时候开始，我觉得，一个软件的架构真TM重要！即使不为性能考虑，也得为开发考虑。不然项目做大了，谁管得了这不羁的代码。
	
## 规范
在做了10年的开发之后，更确切的说，是客户端开发。其中涵盖了PC软件、之后的原生app、和时下最流行的webapp，当然还有很火的微信小程序。我发现，其实整个编程思路基本上在这10年里并没有太大的变化。在PC时代，没有Activity，没有ViewController，也没有Page的概念。在这个背景下，软件没有一个固定框架，所以代码的可维护性，主要取决于架构师的架构能力。在有了固定框架的现在，很多东西都变得正规起来，这对于业界来说，是一件好事。
	
## 沉淀
我第一次接触状态机，是在做游戏AI的时候。当年发现了这个东西，如获至宝。后来我发现，其实很多的开发模式都可以套用状态机模式，所谓“万物皆状态”。如果把程序在某个时间段运行的情况抽象成一个个状态的话，这样对于代码的可维护性和团队中不同程序员之间的模块解耦，有很大的帮助。
	
## 早期实现
我所见过的状态机的早期实现，其实只有简单的状态切换概念，用到的是万恶的switch。

```C++
enum State {
    State1 = 0, 
    State2, 
    State3, 
};
	
int state;
	
void Run(int initState) {
    state = s;
    switch(state) {
        case State1: {
            printf(“State1 Running...”);
            changeState(State2);
            break;
        }
        case State2: {
            printf(“State2 Running...”);
            changeState(State3);
            break;
        }
        case State3: {
            printf(“State3 Running...”);
            break;
        }
    }
}
	
int main() {
    Run(State1);
    return 0;
}
```

假如每个case日渐剧增，那么必然使得单个文件体积的变大，对编译器不友好，维护成本也很高。
	
## 优化实现
对于case体积的问题，可以把每个case都封装成为一个函数来解决问题。或者使用函数指针，将一堆函数（函数定义也可以使用模板特化实现）放入指定数组，然后通过下标（其实就是当前状态常量，一般为int）来调用指定函数，这样可以消灭switch和case，是代码变得更优雅。

## 进阶实现
但毕竟，上述的方法是个偏方，对于某些编程语言来说，这样的方式还是不优雅。怎么才能优雅，还是得靠面向对象。
	
```C++
class State;
class StateMachine;
typedef map<string, State*> StateMap;
typedef std::vector<State*> StateStack;
typedef ulong StateMessage;
	
class State {
  friend class StateMachine;
	
  public:
    State(const string &name);
    virtual ~State();
	
  public:
    const string &Name();
    virtual void OnStateEnter();
    virtual void OnUpdate();
    virtual void OnStateLeave();
	
  public:
    StateMachine *ParentMachine();
    StateMachine *SubMachine();
	
  private:
    string mName;
    StateMachine *mParentMachine;
    StateMachine *mSubMachine;
};
	
class StateMachine {
	
  public:
    StateMachine(const string &name);
    virtual ~StateMachine();
	
  public:
    template <typename T> T *AddState(const string &name);
    State *CurrentState();
    template <typename T> T *GetState(const string &name);
    bool ChangeState(const string &name, bool pushState = true);
    template <typename T> bool ChangeState(bool pushState = true);
    bool RevertState(int step = 1, bool pushState = false);
	
  private:
    string mName;
    StateMap mStates;
    State *mCurrentState;
    StateStack mHistory;
};
```

这是一个基本的状态机的定义。状态机呈树状结构，也就是说，一个状态它同时也可以是一个状态机，包含若干个状态。但是在某一时刻，一个状态机只有处于一个当前状态。在状态机切换状态时，会调用当前的OnStateLeave函数，并改变当前状态，在调用新的当前状态的OnStateEnter函数，完成状态切换。还有，在程序运行的过程当中，状态机会负责在不断的调用当前状态的OnUpdate函数，以便让当前状态不断的运转。这样，每一个状态都是一个State的实现，互相独立，使得功能区分更加明确，团队分工也更加的明确。
__如果把状态，看成是一个页面和它所有的逻辑的话，状态机模型适用于所有的客户端程序开发。__

## 思考
在实际的开发当中，上述的状态机实现还是过于简单。它缺少错误检查，缺少模块间的通信，缺少UI互动。

```C++

class State;
class StateMachine;
typedef map<string, State*> StateMap;
typedef std::vector<State*> StateStack;
typedef ulong StateMessage;
typedef ulong View; // 窗口标识/控件标识

class State {
    friend class StateMachine;

  public:
    State(const string &name);
    virtual ~State();

  public:
    const string &Name();
    virtual View OnCreateView();
    virtual bool OnPreCondition();
    virtual void OnStateEnter();
    virtual void OnUpdate();
    virtual void OnStateLeave();
    virtual bool OnPostCondition();

  public:
    void SendMessage(StateMessage message, StateData data = NULL); // 发送消息到状态机
    virtual void OnStateMessage(StateMessage message, StateData data); // 接受状态机的消息

  public:
    StateMachine *ParentMachine();
    StateMachine *SubMachine();

  private:
    string mName;
    StateMachine *mParentMachine;
    StateMachine *mSubMachine;
};

class StateMachine {

  public:
    StateMachine(const string &name);
    virtual ~StateMachine();

  public:
    template <typename T> T *AddState(const string &name);
    State *CurrentState();
    template <typename T> T *GetState(const string &name);
    bool ChangeState(const string &name, bool pushState = true);
    template <typename T> bool ChangeState(bool pushState = true);
    bool RevertState(int step = 1, bool pushState = false);

  public:
    void SendMessage(StateMessage message, StateData data = NULL); // 发送消息到当前状态
    virtual void OnStateMessage(StateMessage message, StateData data); // 接受当前状态的消息

  private:
    string mName;
    StateMap mStates;
    State *mCurrentState;
    StateStack mHistory;
};
```

通过OnCreateView控制状态所在的控件或者窗口。通过OnPreCondition（前置条件，不符合不能进入该状态）和OnPostCondition（后置条件，不符合不能离开当前状态）实现条件控制。通过SendMessage实现状态和状态、状态和状态机、状态机和状态机直接的模块通信。至此，状态机模型已满足几乎所有客户端的开发要求。

## 总结
我一直以为，写代码是一件十分快乐的事情。我不希望看到工程师们夜以继日的沉没在难以维护的代码当中而对开发失去了兴趣。花费了半天的精力才找出一个低级的语法错误。

## TODO
之后我会建立一个简单的开源项目，各种语言各种平台上的状态机模型实现，希望感兴趣的朋友一起交流