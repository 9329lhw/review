<h1 align="center">依赖注入，控制反转</h1>

## 解释一
    一、什么是依赖注入和控制反转
    1.依赖注入（DI）— Dependecy Injection
    为了更方便的理解，我们把依赖注入分开理解，首先什么是依赖？顾名思义，
    依赖就是各组件之间的一种关系。一般来说，在面向对象编程中，我们在类A
    中 使用到了 类B的实例，我们就可以说A依赖B，B是A的依赖。传统的写法就
    是在A类中直接调用B的实例。这种写法会形成强耦合，不能保证A类的纯洁性，
    所以并不是我们理想的设计模式。
    那什么是注入什么，由谁注入呢？注入就是我们实现解耦的一种方法，我们把
    A对B这种依赖关系，通过各种方式由第三方注入进来，而不是A直接对B进行操作，
    这个第三方就是IOC容器。下边是两种常用的注入方式
        a. 由构造函数注入：
        class A
        {
            private $objectB;//由构造函数注入
        
            public function __construct($objectB)
            {
                $this->objectB = $objectB;
            }
        
            public function getAData()
            {
                $data = $this->objectB->getBData(); //... return $data;
            }
        }
        b. 由setter方法注入：
        class A {
            private $objectB; //由setter方法注入
            public function setObjectB($objectB){
                $this->objectB = $objectB;
            }
            public function getAData() {
                $data = $this->objectB->getBData(); //... return $data;
            }
        }
        
    2.控制反转（IOC）— Inversion Of Control
    说完依赖注入我们可以发现，本来是两个对象A\B之间的事情，现在变成了三个对象。
    没错，多出来的这个就是容器，它就是控制AB依赖关系的。现在我们理解一下控制反
    转这个概念，就会变得非常容易，控制就是对依赖关系的控制权，反转是原本这个控
    制权是在A内部代码实现的，现在变成由外部容器实现然后注入进来，这就是所谓的
    反转。依赖注入是控制反转的具体实现，这两个概念说的其实一回事，只是角度不同。
        二、IOC容器的具体实现
        在讲IOC容器实现之前，我们先说一下解耦。实际应用中，大多时候A类依赖的不
        仅仅是B，还有C\D\E等等好多的类，这些被依赖的类，会在A类中的各种各样的方
        法中使用
        class A {
            public function getAData() {
        
                $objectB = new B();
                $dataB = $objectB->getBData();
                $objectC = new C();
                $dataC = $objectC->getCData(); //...
            }
        
            public function getAData2() {
                $objectB = new B(); //...
            }
        }
        class B {
            public function getBData() {
                //...
            }
        }
        class C {
            public function getCData() {
                //...
            }
        }
        这样就会出现一个问题，假设某一天我们洒脱的B类不喜欢自己的名字了，它想
        改名为2B，毕竟2B更2。这个时候A类心里苦，但是没有办法，那就都改了吧。我
        们当然知道这样肯定是不合理，那这种情况怎么解决呢？
        工厂模式：
        很简单，工厂类的作用就是帮你批量的实例化那些依赖的类，这样你再也不用在
        你的主类中使用new了。
        class Factory { 
            static public function getObjectB() { 
                return new B(); 
            } 
            static public function getObjectC() { 
                return new C(); 
            }
        }
        class A { 
            private $objectB; 
            private $objectC; 
            public function getAData() { 
                $this->objectB = Factory::getObjectB(); 
                $dataB = $this->objectB->getBData(); 
                $this->objectC = Factory::getObjectC(); 
                $dataC = $this->objectC->getCData(); 
                //...
            }
        }
        这样我们就把B、C两个类从我们的主类A中净化了出去。这样的过程就是解耦，
        我们把A和B\C\D..的耦合，转变成了跟Factory一个类之间的耦合。以后不管B想
        改成什么，我们都不要管，我们只要找Factory就行了。
        似乎解决了大量耦合的问题，到此为止了吗？当然不是，毕竟在我们的主类A中
        ，还存在着一个感觉很突兀的Factory，换名字的问题仍然存在，这是一个很大
        的隐患。所以为了防止Factory换名字，我们得做点防护措施。
        class A { 
            private $objectB; 
            private $objectC; 
            public function setObjectB($instance) { 
                $this->objectB = $instance; 
            } 
            public function setObjectC($instance) { 
                $this->objectC = $instance; 
            } 
            public function getAData() { 
                $dataB = $this->objectB->getBData(); 
                $dataC = $this->objectC->getCData(); 
                //...
            }
        }
        这样A类终于变的很纯洁了，看不到B\C，也没有了Factory。我们使用A类的时候
        可以这么写
        $a = new A();
        $a->setObjectB(Factory::getObjectB());
        $a->setObjectC(Factory::getObjectC());
        $a->getAData();
        现在再来看A类，干干净净，没有依赖，Factory这个原本被依赖的对象是通过参
        数，从A类外部注入进来的，这个过程就是依赖注入的实现。这种方式虽然实现了
        依赖注入，但是会有个问题，A类中要写多个setObject方法，每次外部调用A 的
        时候，也必须要调用setObject方法好多次。如果A类依赖了十多个外部类，那我
        们每次使用A的时候岂不是很恶心。
        class A { 
            private $objectB; 
            private $objectC; 
            private $objectD; 
            //...
            public function setObjectB($instance) { $this->objectB = $instance; } 
            public function setObjectC($instance) { $this->objectC = $instance; } 
            public function setObjectD($instance) { $this->objectD = $instance; } 
            //...
            public function getAData() { 
                $dataB = $this->objectB->getBData(); 
                $dataC = $this->objectC->getCData(); 
                $dataD = $this->objectD->getDData(); 
                //...
            }
        }
        $a = new A();
        $a->setObjectB(Factory::getObjectB());
        $a->setObjectC(Factory::getObjectC());
        $a->setObjectD(Factory::getObjectD());
        这样太不合理了，貌似我们可以再加个工厂类封装一下，帮助使用者把setObject
        的工作封装一下。
        //把A类再封装一下
        class HeadFactory { 
            public static function allSet() { 
                $a = new A(); 
                $a->setObjectB(Factory::getObjectB()); 
                $a->setObjectC(Factory::getObjectC()); 
                $a->setObjectD(Factory::getObjectD()); 
                return $a; 
            }
        }
        //所有使用A类的地方，就可以直接这么使用，不在每次都setObject
        $app = HeadFactory::allSet();
        $app->getAData();
        $app->getAData2();
        这下我们使用A的时候，也变得很方便了，依赖注入总算是完成了。
        说了这么多工厂类，那么容器呢？好像并没有讲到容器是怎么实现的。对比一下
        工厂类，工厂是把依赖的类注册到静态方法上的，而IOC容器是把这些依赖类注
        册到静态数组中的。
        class Di {
            protected static $objectArr;
            static public function set($k,$obj) {
                self::$objectArr[$k] = $obj;
            }
            static public function get($k){
                return self::$objectArr[$k];
            }
        }
        class A {
            private $di; public function __construct($di) {
                $this->di = $di;
            }
            public function getAData() {
                $dataB = $this->di->get("B")->getBData();
                $dataC = $this->di->get("C")->getCData();
                //...
            }
        }
        使用的方式是这样
        Di::set("B", Factory::getObjectB());
        Di::set("C", Factory::getObjectC());
        $app = new A($DI);
        $a->getAData();
        我们一般会在项目初始化的时候，把需要注入的类全部都set好，这样我们可以
        在整个项目中都可以直接使用类似$this->di->get()这样的方式来获取到依赖的
        对象。phalcon,lavaral等框架就是类似的依赖注入实现。为了最后的优雅，我们
        要消灭所有的new,我们把主类A也加入到Factory类静态方法中
        class Factory { 
            static public function getDi(){ return new Di(); } 
            static public function getObjectA($di) { 
                return new A($di);
            } 
            static public function getObjectB() { 
                return new B(); 
            } 
            static public function getObjectC() { 
                return new C(); 
            }
        }
        这样我们最后的使用A类的方式就变成了这样
        class Di {
            protected static $objectArr;
            static public function set($k,$obj) {
                self::$objectArr[$k] = $obj;
            }
            static public function get($k){
                return self::$objectArr[$k];
            }
        }
        class Factory {
            static public function getDi(){
                return new Di();
            }
            static public function getObjectA($di) {
                return new A($di);
            }
            static public function getObjectB() {
                return new B();
            }
        }
        class A {
            private $di; public function __construct(Di & $di) {
                $this->di = $di;
            }
            public function getAData() {
                $dataB = $this->di->get("B")->getBData();
                //...
            }
        }
        class B { public function getBData() {
            //... print_r("getBData"); }
        }
        $di = Factory::getDi();
        $di::set("B",Factory::getObjectB());
        
        $app = Factory::getObjectA($di);
        $app->getAData();
        
    参考链接：https://www.huaweicloud.com/articles/13757251.html
        
## 解释二        
    什么是依赖
    不是我自身的，却是我需要的，都是我所依赖的。一切需要外部提供的，都是需要
    进行依赖注入的。
    小结
    因为大多数应用程序都是由两个或者更多的类通过彼此合作来实现业务逻辑，这使
    得每个对象都需要获取与其合作的对象（也就是它所依赖的对象）的引用。如果这
    个获取过程要靠自身实现，那么将导致代码高度耦合并且难以维护和调试。
    依赖注入解决了以下问题：
    依赖之间的解耦
    单元测试，方便 Mock
    控制反转 
    是面向对象编程中的一种设计原则，可以用来减低计算机代码之间的耦合度。其中最
    常见的方式叫做依赖注入（Dependency Injection, DI）, 还有一种叫 "依赖查找"
    （Dependency Lookup）。通过控制反转，对象在被创建的时候，由一个调控系统内所
    有对象的外界实体，将其所依赖的对象的引用传递给它。也可以说，依赖被注入到对
    象中。
    也就是说，我们需要一个调控系统，这个调控系统中我们存放一些对象的实体，或者
    对象的描述，在对象创建的时候将对象所依赖的对象的引用传递过去。
    参考链接：https://learnku.com/laravel/t/2104/understanding-dependency-injec
    tion-and-inversion-of-control

## 解释三    
    依赖注入就是将实例变量传入到一个对象中去(Dependency injection means giving 
    an object its instance variables)。
    什么是依赖
    如果在 Class A 中，有 Class B 的实例，则称 Class A 对 Class B 有一个依赖。
    例如下面类 Human 中用到一个 Father 对象，我们就说类 Human 对类 Father 有
    一个依赖。
        public class Human {
            ...
            Father father;
            ...
            public Human() {
                father = new Father();
            }
        }
        仔细看这段代码我们会发现存在一些问题：
        1.如果现在要改变 father 生成方式，如需要用new Father(String name)初始化 
        father，需要修改 Human 代码；
        2.如果想测试不同 Father 对象对 Human 的影响很困难，因为 father 的初始化
        被写死在了 Human 的构造函数中；
        3.如果new Father()过程非常缓慢，单测时我们希望用已经初始化好的 father对
        象 Mock 掉这个过程也很困难。
    依赖注入
        上面将依赖在构造函数中直接初始化是一种 Hard init 方式，弊端在于两个类不
        够独立，不方便测试。我们还有另外一种 Init 方式，如下：
        自己主动初始化依赖，而通过外部来传入依赖的方式，我们就称为依赖注入。
        public class Human {
            ...
            Father father;
            ...
            public Human(Father father) {
                this.father = father;
            }
        }
        上面代码中，我们将 father 对象作为构造函数的一个参数传入。在调用 Human 的
        构造方法之前外部就已经初始化好了 Father 对象。像这种非自己主动初始化依赖，
        而通过外部来传入依赖的方式，我们就称为依赖注入。
        现在我们发现上面 1 中存在的两个问题都很好解决了，简单的说依赖注入主要有两个好处：
        解耦，将依赖之间解耦。
        因为已经解耦，所以方便做单元测试，尤其是 Mock 测试。
    控制反转和依赖注入的关系
        1.控制反转是一种思想
        2.依赖注入是一种设计模式
    参考链接：https://www.jianshu.com/p/07af9dbbbc4b
        

	
