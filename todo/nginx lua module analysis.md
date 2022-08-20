**Nginx Lua Module****分析**

**一****:Lua****基础语法**

本文档以lua5.1为基础进行学习,涉及内容主要是lua基础语法,lua与c/c++之间的交互,lua协同工作模型以及涉及少部分垃圾回收等.

**1.lua类型与值**

lua中变量的类型一共有八种,分别是nil、boolean、number、string、table、function、userdata以及thread.

 

nil、boolean、number(双精度浮点型)、string介绍略.

 

**table**:lua中的table为关联数组(如果不存在key则索引为数字),该数组可以用除了nil的任何值作为lua的索引.table中可以包含任意除了nil类型的值,任何带nil的值都不是table的一部分( 即使是{ test=nil } ),相应的,任何不属于table的key都关联一个nil值.

**function**:function可以是c函数也可以是lua函数

**userdata**:userdata用来将任意c/c++数据保存到lua变量中,userdata分为full userdata和light userdata两种.full userdata的内存空间由lua负责释放,而后者的内存空间由c/c++程序负责管理.另外就是lua程序是无法直接访问userdata的,因此需要将userdata传递给c函数才可以访问.操作的方式有两种,一种是单独的c函数,另一种是类似于c++对象的成员函数(如何提供这两种方法,我们后续的会提到).

如:将userdata中的成员以字符串的形式输出到终端，通常该操作有如下两种方式:

print_str(userdata)----->普通函数

userdata:print_str()---->成员函数

**thread**:代表独立的执行流程,用来实现lua的coroutines(协同程序). 该部门在后文会着重叙述.

**2.变量的位置**

lua变量从存放位置来看,有以下几种(可能还有其他...这里是我的理解)

 

环境变量:位于_ENV table的所有变量

全局变量:位于_G table中的所有变量.在lua中如果一个变量不是使用local限定的变量,那么这个变量就是全局变量,这个变量一旦创建就会存放到_G table中.在C/C++中使用lua_getglobal和lua_setglobal访问全局变量.

Lua虚拟栈中的变量,用于c/c++和lua之间的交互.

注册表:位于注册表中的变量可以被C/C++访问到,lua脚本访问不到(通过LUA_REGISTRYINDEX进行访问).

**3.lua运算符**

\+ - * / % ^

== ~= > >= < <=

and or not

.. 连接字符串

\# 返回字符串或者表的长度(返回从1号索引之后连续的数组的长度)

如arr = { 1='a', 3='b', 4='c', 5='d' }  #arr = 1

**4.流程控制**

**判断语句****:**

if 条件语句 then

代码块

elseif 条件语句 then ---注意,elseif绝对不能使用else if代替,否则会报奇怪的错误.

代码块

end

 

需要注意的是,lua中两个变量的比较即比较内容也比较类型.也就意味着nil、0都不等于false.

local str = "";

if #str then

print("aaaa");

end

该程序是无意义的.

 

**循环语句****:**

while循环

break语句

continue,在lua中是不存在continue的,需要使用其他手段实现.

for循环:lua for循环中分为数字型和泛型两种.

for v=exp1,exp2,exp3 do --- exp3可选,不存在exp3时,默认为1.

代码块

end

 

for i, v in pairs(arr) do

代码体

end

 

pairs和ipairs函数的区别：

pairs可以遍历整个数组,ipairs只能遍历到表中出现第一个不是整数的key.

 

lua迭代器实现:略.... 不会,pairs已经满足我们的开发要求了...

**5.string、table和函数**

string.upper(arg):返回参数的大写字符串

string.lower(arg):返回参数的小写字符串

string.gsub(mainstr, findstr, replacestr):将第一个查找到的字符串使用replacestr进行替换,并返回替换后的字符串.

string.reverse(arg):反转字符串

 

对于table来说,如果想要删除某个键,只需要将这个key对应的value设置为nil即可.

另外table在函数中传递是通过引用方式传递的,所以有些时候我们需要对table进行深度拷贝.

 

在lua中函数就是变量.比如如下申明

local function test (arg)

function body

end

等价于

local test = function(arg)

function body

end

另外需要注意的是,在lua的函数中可以返回多个值.当然返回值的接收可以是任意个数(多出来的被赋值为nil)

**6.元表**

Lua中每个值都能拥有一个metatable(元表),这个metatable就是普通的lua table.通常我们为table或者是userdata分配一个元表,以期望实现对应的功能.

 

lua使用元表提供了类似于c/c++中赋值运算符重载的手段.借助该手段可以扩充table、userdata的功能以及实现与c++对象类似的功能.

 

常见的元方法:

__index:在访问阶段,当key不在table中时,才会使用该元方法.__index可以是一个方法也可以是一个table.如果是一个table,则会进入这个table中查找key对应的value.

__newindex:在赋值时,当key不存在table中时,才会使用该元方法.

因为上述的元方法,才有了rawset/rawget和settable/gettable的操作. rawset/rawget在操作时将不会触发元方法.

__call:该方法的意义不清楚

__tostring/__add/__sub/__lt/__gt/....

 

为userdata提供类成员函数

上述我们知道了在lua中是不可以直接访问userdata的内部成员的,那么如何为userdata提供一系列用于操作的成员函数?这里显然要借助元表了.在lua中任意的值都可以拥有一个元表.

为了便于描述,用下文类似于table的方式描述该结构

userdata{

metatable{

__index = metatable,

method1 = func1,

method2 = func2

}

}

使用方式:userdata:method1()即可.

当使用:运算符是,第一个传入的参数为userdata.相当于执行了method1(userdata)操作.

在userdata中查找method1函数时,没有找到则调用__index元方法.发现__index方法指向了一个table,那么就到该table中查找寻找key对应的value.

 

在C语言中先创建类声明,也就是创建上述的metatable结构.当创建了一个userdata时,使用lua_setmetatable将这个metatable设置到userdata上,这个userdata就具备了对象的操作方式.

**7.协同程序**

协同是我们学习lua模块最重要的一环,也是最难的一环.这里我们介绍的是在lua中如何使用协同程序.

 

协同程序在lua中也被称之为thread.实际上这个thrad的概念并不是线程,这里简单介绍一下,在之后遇到lua中所有的thread都看成协程即可.

 

lua程序的执行有两种方式,一种是lua程序作为一个函数模块被c/c++程序调用.另一种是lua作为一个脚本直接运行.在lua脚本中是可以直接调用c/c++的.so库的.绝大多数情况下,lua是作为嵌入式语言被宿主语言调用的.这种情况下lua所属的工作线程是宿主语言的工作线程.

 

对于lua来说,这个工作线程是单线程模型(这里可以想象成一个reactor模型),但是它内部可能运行很多的任务.而一个协程即可看成在这个单线程模型中运行的一个任务.

这些任务有自己的状态,它们在执行期间可能遇到阻塞,因此这些任务可以在"预感"到阻塞之前通过yile让出处理器.此时处理器(thread)可以驱动其他的任务执行.在其他任务让出处理器后,处理器发现当前任务执行所需要的条件已经满足了,继续驱动当前任务的执行.

 

将一个http请求的处理看做一个协程.在当前线程中可能处理很多的http请求.每个http请求的执行流程都是相同的.

发起数据读取操作,处理数据,做出响应,回收资源.

在发起数据读取操作时,来自网络的数据可能没有到达(或者没有完全到达).所以

先执行do_read操作, 检查数据是否足够,数据不够则yield.

​    ![0](https://note.youdao.com/yws/public/resource/965c9f034a82ffb0f8b4de6ca81f3e73/xmlnote/452B54CFBD7949DF851A055546E593EB/33443)

 

 

实际上可以把协程看做是一个状态机.假设一个协程的代码块需要执行100行语句,在第20行的时候发现将要陷入阻塞了,所以主动yield,此时lua底层模块记录了这个协程的执行状态.等待下次驱动这个协程(状态机)运行时,先获取这个协程的执行状态,然后恢复上次的执行状态继续执行(也就是从第20行代码处开始执行剩下的80行代码).

 

协程可以由C/C++程序创建并驱动执行. 也可以由C/C++程序对lua程序发起调用,在lua程序中创建协程并启动协程运行.

同样在lua程序中可以检测当前状态机的运行状态,然后主动yield.也可以调用C函数,在C函数执行yield.

当宿主程序上新的事件被触发时,可以继续驱动协程的运行,也可以调用lua函数由lua函数驱动协程的继续运行.

 

其实可以没必要区分到底是C/C++还是lua创建或者是控制协程的运行, 完全可以把lua程序看做是C/C++中的一个函数.至于控制协程的执行方法,因为区分C/C++代码还是lua代码,因此就产生了两套API接口.只是他们的功能和意义是差不多的.

 

Lua协程相关API:

1.coroutine.create(f):使用函数f创建一个协程，返回创建的协程对象.类型为thread,c类型为lua_State.

2.coroutine.resume(co[, v1,...]):重新开始运行协程,返回操作状态和yield传递的参数.

3.coroutine.yield(...):暂停协程,yield中的参数会作为resume函数的返回值.

4.coroutine.running():返回当前正在运行的协程.

5.coroutine.status(co):查看协程当前的运行状态.一个协程有三种状态,suspended、running、dead.

6.coroutine.wrap(f):与create功能相同,但是它返回一个函数.一旦调用该函数即进入协程.

 

EventLoop驱动Coroutine的执行:

这里只描述一个简单的回射服务器.业务处理函数为:

function{

READ:

do_read;

if(readData == 1000 bytes)

write(readData)

else

coroutine.yield()

goto READ;

print("task done")

}

​    ![0](https://note.youdao.com/yws/public/resource/965c9f034a82ffb0f8b4de6ca81f3e73/xmlnote/B943F712EC024F8789C6A9BACCE5C3DB/33444)

 

**8.异常处理**

程序运行过程中,不可避免的出现一些错误(语法错误、运行错误),如果lua是以嵌入的方式执行的,那么一旦出现错误,则lua程序立即返回宿主程序.

 

不管是lua还是c api中都可以触发一个错误操作,供调用者捕获.在lua程序中可以使用error函数触发.在C中可以使用lua_error来触发.

 

有时为了捕获和处理异常,在lua中有两处操作.

第一处：在lua中安全调用另一个lua函数,避免另一个lua函数出错时导致程序返回到宿主程序.方法是通过pcall安全的调用另一个lua函数.

第二处：C/C++在调用一个Lua程序时需要知道它运行正确还是异常,顺便捕获异常信息.通过lua_pcall函数调用lua相关API.

**9.模块包含**

loadfile:加载文件,编译文件,然后返回一个函数,但是不运行.可以理解为将另一个文件下的所有内容load进一个str,需要对这个str进行解析操作才可以执行.

local test = loadfile("./2.lua")  test()

 

dofile:实际上就是loadfile + f()操作.加载文件,并且将loadfile返回的函数执行一遍.

 

require:与dofile相同,但是require有路径自动搜索和避免重复包含的功能.

**二****:lua****与****C/C++****交互**

**1.lua虚拟栈**

lua和C/C++交互是通过一个虚拟的lua栈来实现的.在C语言中可以将一些参数、返回值放入栈,在lua中对堆栈中的数据进行出栈处理.或者在lua中将一些返回值入栈,在c语言中对这些返回值进行出栈.

 

这个栈的索引可以是正数也可以是负数.另外,在lua中数组是的索引是从1开始的.因此在lua栈中,1表示栈低,-1表示栈顶.

​    ![0](https://note.youdao.com/yws/public/resource/965c9f034a82ffb0f8b4de6ca81f3e73/xmlnote/33F4A13C178649FEA9110C453A1BEAEB/33448)

 

**2.C/C++与lua调用方式**

**在****C/C++****中调用****lua****模块**

1.通过luaL_loadfile(L, "test.lua")加载test.lua到L中.

2.通过lua_pcall(L, nargs, nresults, errfunc)运行test.lua.此时test.lua中的所有全局变量都被加入到L的global中.

3.通过lua_getglobal(L, "变量名")获取test.lua中变量名对应的全局变量压栈.

4.通过lua_toxxx(L, -1)可以获取到这个变量的内容.

5.通过lua_getglobal(L, "函数名")获取到一个函数对象.

6.通过lua_pushxxx(L, "arg") 将函数所需要的一些参数压入栈中.

7.调用lua_pcall(L, 1, 1, 0)弹出栈顶的一个元素,将其作为一个参数传递给func.并将返回的结果压栈.

 

**lua****中调用****C****提供的函数**

1.C/C++中将要提供的函数push到lua栈中

2.C/C++中将要提供的资源添加到lua的全局变量表中

 创建一个table, { "method1"=func1, "method2"=func2, "method3"=func3 }

 调用lua_setglobal将这个table添加到Lua的_G table中.

 在lua中,可以通过tablename.func1直接调用该函数.

3.将C/C++程序编译成一个动态库,然后由lua脚本进行require

 创建一个luaL_Reg的结构体数组,将所需要提供的函数放到这个数组中

 然后调用luaL_newlib将这个结构体数组注册到L上

 最后将这个C/C++程序编译成动态库.

 在lua程序中即可通过require将包含这个动态库,并调用这个动态库提供的函数,

**3.C/C++与Lua之间的交互**

C/C++调用lua提供的函数.

int lua_pcall (lua_State *L, int nargs, int nresults, int msgh);

nargs表示这个lua函数要传递多少个参数,nresult表示有多少个返回值.errfunc表示错误函数的索引.

lua_pcall函数在调用时,将从lua栈上pop nargs个元素,接下来再pop lua函数并发起lua函数的调用.最后在lua_pcall函数返回时将lua函数的返回值push到lua栈上.同样返回值也是以压栈的方式存放到lua的虚拟栈中的.

lua_pcall函数出错时,如果没有指定错误处理函数,会将出错消息push到lua的虚拟栈并返回一个非0的值.

需要注意的是,C/C++和lua始终操作的是同一个lua栈,需要注意避免lua栈被污染.

 

C/C++可以将一系列的变量(整形、table、userdata)等保存到Lua栈上,也可以直接保存到Lua的全局表中(_G),在Lua中可以直接使用这个table等变量(等于是自己的全局变量).C/C++可以通过相关函数读取或者设置(修改)全局表中对应变量的值,也可以通过伪索引LUA_GLOBALSINDEX获取整个全局表.

 

除此之外,C/C++还可以将一些变量或者资源绑定到某个LuaVM的注册表上,这些资源对Lua是不可见的.注册表的访问也是通过伪索引LUA_REGISTRYINDEX.

**4.lua C相关API**

**lua_State****对象**

创建lua_State对象: lua_State *L = luaL_newstate();

销毁lua_State对象: lua_close(L)

 

**lua****虚拟栈操作****:**

lua_gettop(L):获取Lua栈中元素的个数,也是Lua栈栈顶的索引.

lua_settop(L):设置Lua栈的大小,多的去除,少的填nil. (去掉的是栈顶多出来的元素.)

lua_pushboolean(L, int b):把b作为一个boolean类型的变量压栈

lua_pushcclosure(L, fn, n):将fn作为闭包函数压入栈中,n告诉函数,有多少个值需要关联到函数上.详细操作见下文.

lua_pushcfunction(L, fn):实际上就是lua_pushcclosure(L, fn, 0)

lua_pushfstring(L, const char *format, ....):将一个格式化的字符串入栈,只是其格式化规则与sprintf不大相同.

lua_pushstring(L, str):将一个字符串入栈.

lua_pushglobaltable(L):将一个全局变量压栈.待见下文

lua_pushinterger(L, n):将整数n压栈.

lua_pushlightuserdata(L, void *p):将一个轻量级的userdata压栈.仅保存C中userdata的一个指针.

lua_pushnil(L):将nil压栈

lua_pushnumber(L, n):将一个double n压栈.

lua_pushstring(L, s):将字符串s压栈.(识别\0结尾)

lua_pushvalue(L, index):将栈上index位置处的值拷贝一份压栈.

lua_tonumber(L, index):获取栈上指定索引的数据,然后转换为浮点型返回.注意,必须赋值给一个浮点型(这里以%d输出number值是错误的,即使插入的是整形)

lua_toboolean,lua_tointege,lua_tostring,lua_totable,lua_tocfunction,lua_touserdata等函数.

lua_isboolean(L, index):判断指定位置上的值是否是boolean类型.

同样的函数有lua_istable、lua_iscfunciton、lua_isstring、lua_isfunction等.

 

lua_pop(L, n):从栈中弹出多少个元素.

lua_insert和lua_remove操作

 

**C****语言中的****table****操作****:**

创建一个新表: void lua_createtable(lua_State *L, int narr, int nrec)

lua_createtable(L, 0, 0)：创建一个narr行nrec列的table并把table压入栈中.

lua_newtable(L),创建一个空table并将其压入栈中.实际上它就是lua_createtable(L,0,0).如果知道预先的table的详细信息可以使用第一种方式创建以提高性能.

为已经压栈的table添加元素:

首先通过lua_newtable(L)创建一个空的table压入栈中.

然后将要放入table中的元素也压入lua栈.

接下来,通过lua_setfield(L, index, key)将该table栈顶的元素弹出,以key-value的方式插入到table中.其中index为table在lua栈的索引.

lua_newtable(L); // 创建一个table然后将其压入栈中

lua_pushnumber(L, 10); // 将number压入栈中

lua_setfield(L, -2, "number"); // 弹出栈顶元素然后以number=10的方式压入table中

获取table中的元素

首先通过lua_getfield(L, index, key)从index位置的table中获取key对应的value值.并将其压入栈顶

接下来访问栈顶的元素,这个元素就是table中key对应的值.

lua_getfield(L, -1, "number"); // 此时table是栈顶元素,table["number"]的值将压入栈顶

lua_tonumber(L, -1);// 将栈顶的元素转换为number返回.

注意：如果在当前table中没有找到与之对应的key,该操作会触发table的_index方法.（如果重写过table的_index方法的话）

 

lua_settable

首先将table压入栈中.

接下来将key压入栈

接下来将value压入栈

使用settable将压入栈中的key和value弹出然后组合成key-value插入table中.

lua_gettable

首先将要从table中查找的key压入栈中

然后调用gettable,此时会将key弹出来,然后从table中找到key对应的value,将value压入栈中.

 

lua_rawget和lua_rawset.

上述操作都会触发table中的元方法.而这两个方法不触发元方法,并且效率更高

它们的操作个settable和gettable相同.都是先将key入栈.然后进行操作.

 

**C****语言中操作****userdata**

创建userdata:

void *lua_newuserdata(lua_State *L, size_t size)

创建一个指定大小的内存块,并把这个内存块的首地址返回. 这个指定大小的内存块对于lua脚本来说就是userdata.

userdata可以是一个结构体，或者是一个类对象.但是目前对于lua脚本来说就是一个无用的内存块.另外可以通过lua_setmetatable为lua栈中的userdata绑定一个元表.

 

void *lua_pushlightuserdata(lua_State *L, void *p):与newuserdata函数功能相同,只是void *p所指向的内存空间由宿主程序自己完成释放.

 

**other API**

lua_CFunction: typedef int (*lua_CFunction) (lua_State *L);

 

lua_newthread：创建一个协程,该函数返回一个lua_State指针.并且会将这thread类型的值压入栈中.

L1 = lua_newthread(L)

L1 ---> 空栈

L ---> 栈顶元素为L1

除了主线程外,其他thread对象和lua的其他对象一样都是垃圾回收的对象.当新建一个线程时,线程会压入栈中,这样就确保新线程不会成为垃圾.

当然为了避免新的thread成为垃圾,我们可以使用下文中的luaL_ref函数

lua_yield:暂停当前的协程

lua_resume:重启当前协程

 

lua_xmove:

void lua_xmove(lua_State *from, lua_State *to, int n)

从from的stack中pop n个value,然后将他们push到to的stack中

 

luaL_ref/ luaL_unref:

int luaL_ref(lua_State *L, int t);

在当前栈索引t处的元素是一个table, 在该table中创建一个对象.对象是当前栈顶的元素,并返回创建对象在表中的索引值.之后会pop栈顶的对象.(即将栈顶元素放到t对应的table中,并返回栈顶元素在t中所处的index)

int luaL_uref(lua_State *L, int t, int ref): 释放索引t指向的table中的ref引用的元素.(将table中对应的元素删除掉)

 

luaL_loadbuffer/luaL_loadfile/luaL_loadstring

与luaL_load函数类似,将加载后的内容（只加载不运行）,作为一个lua函数压在栈顶.

 

luaL_tolstring: char *luaL_tolstring(lua_State *L, int idx, size_t *len)

将指定索引处的值转换为C字符串返回.如果len不为NULL,则将字符串的长度通过len传出.

**三****:****延续与协程编程模型**

其实可以把延续和协程都看做是一个状态机.当然有的书上将协程描述为一个独立的堆栈.

 

不管延续还是协程,既然作为一个状态机,那么就具备描述当前状态的能力.只是它们用来保存这些状态数据的方式不同罢了.

 

使用延续或协程编程的好处是,底层的事件系统不需要关系状态机当前处于什么状态,它只管回调就够了,而状态机在被事件系统驱动时根据自身的当前状态及执行策略迁移到下个状态.

 

所以，延续和协程都只是为了描述状态机的起始状态、临时状态、终止状态以及各个状态间的转换方式,实现事件系统预上层业务之间的解耦.

**1.延续和协程对状态的描述**

Continuation:通过类成员变量描述状态机的初始状态、所处状态、最终状态等,使用类成员函数描述状态机的执行流程,如何从一个状态转变到另一个状态.

如：

Class A{

int status;

void (*handler)(void);

A() { status = 0; handler = setup1; }

void steup1(){

cout << "start" << " status:"  << status << endl; status++;

 handler = setup2(); handler();

 }

void setup2(){

cout << "runing" << "status:" << status << endl; status++;

handler = setup3(); handler();

}

void setup3() {

cout << "stop" << "status:" << status << endl; status++;

}

}

在创建一个A类对象,并调用它的handler()方法之后便会驱动状态机的运行直到任务完结.(先不考虑任务调度)

 

Coroutine:是使用函数来构建的状态机,因此它使用函数的局部变量来描述状态机的一系列状态信息,使用程序执行指令(函数内部的代码)来描述状态机如何从一个状态转变为另一个状态,直到最后任务完毕.

如:

function co_task(){

  int status = 0;

  cout << "start" << " status:"  << status << endl; status++;

  status ++;

  cout << "runing" << "status:" << status << endl; status++;

  status ++;

  cout << "stop" << "status:" << status << endl; status++;

}

 

}

**2.延续和协程的任务调度**

修改上述例子,将它变为异步程序.在非阻塞网络编程中,任务的执行都需要底层EventLoop的驱动.我们这里也使用EventLoop来驱动延续和协程的执行.

 

Continuation:每次完成一个步骤, 状态机的状态由类成员属性保存.

Class A{

int status;

  int stop;

void (*handler)(void);

A() { status = 0; handler = setup1; stop = 0; }

void steup1(){

cout << "start" << " status:"  << status << endl; status++;

 handler = setup2();

 }

void setup2(){

cout << "runing" << "status:" << status << endl; status++;

handler = setup3();

}

void setup3() {

cout << "stop" << "status:" << status << endl; status++;

  stop = 1;

}

}

创建一个A对象之后,将齐投放到EventLoop的任务队列中.EventLoop第一次驱动时调用A对象的handler方法,此时A对象完成第一步任务,接下来检测A对象的stop状态是否为1来决定是否将A对象重新放入EventLoop的任务队列中.

 

Coroutine:函数栈的信息就是状态机的所有状态,在任务切出时由处理器(lua运行环境)负责将函数栈保存起来,在任务切入时由处理器恢复之前保存的函数栈信息.

function co_task(){

int status = 0;

int stop = 0;

cout << "start" << " status:"  << status << endl; status++; // setup1

coroutine.yield(stop);  // 将stop标志传给外部驱动,当发现stop变为1就不再驱动协

cout << "runing" << "status:" << status << endl; status++; // setup2

coroutine.yield(stop);

cout << "stop" << "status:" << status << endl; status++; stop = 1;

}

使用Coroutine创建一个协程,然后通过coroutine.resume(co)启动协程.协程在第一步完成之后让出处理器,根据yield返回的stop状态将协程重新放回任务队列,等待EventLoop的下次驱动.当EventLoop下次驱动协程时,只需要通过coroutine.resume(co)运行协程,协程即可从setup2开始处运行.直到最终,yield返回stop=1,外处理程序回收coroutine资源,到此任务完成.

**3.一个简单的例子(伪代码)**

整个任务为非阻塞读取100个字节,读取完毕后任务终止.

 

Continuation:当预测到条件不满足时将退出函数.

Class Task{

int readData = 0;

int stop = 0;

void event_handler() {

readData += do_read()

if(readData < 100)

return;

else{

printf("read Done");

stop = 1;

}

}

}

EventLoop: acceptEvent回调时, t = new Task,然后监听该连接的读事件.读事件触发则回调t->event_handler();在event_handler返回之后判断t->stop是否为1,如果不为1,继续等待当前连接的读事件触发.否则移除当前连接的读事件,delete t,然后关闭socket.

 

Coroutine:当预测到条件不满足时,通过yield从task返回处理器.

function co_task()

{

int readData = 0;

do{

readData += do_read();

if(readData < 100)

coroutine.yield(stop);

else{

print("read Done");

break; // 结束任务

}

}whlie(1);

}

在EventLoop中,当收到一个acceptEvent之后,以co_task()创建一个协程,然后监听该连接的读事件.读事件触发则通过co.resume(co_task)运行. 在resume函数返回之后检测coroutine的状态,如果任务已经完毕,则移除当前连接的读事件,销毁co对象,关闭socket.

 

上诉的Continuation是非阻塞的实现,阻塞的实现可以为：

function  co_task(){

char buffer[100];

do_read(buffer, 100); // 阻塞读取100个字节.

printf("read done");

}

do_read函数一旦调用,先检查当前缓冲区中是否有足够的数据,如果有则返回.否则通过yield进入内核态,由内核态负责监控读写事件,触发对应的回调函数.在回调函数的处理中,读取数据,如果数据读取够则通过resume驱动上层状态机运行.否则继续等待事件触发.

NginxLua模块借助协程的这个特性将阻塞操作转换到了异步非阻塞的框架中.

 

可以看到对于延续来说,它的所有状态信息都保存在对象的成员属性中,在事件触发时只需要调用它暴露的接口即可.而对于协程来说,需要主动告知底层将函数的当前栈空间保存起来(yield函数作用),在事件触发时需要通过resume将协程当前的函数栈重新加载并继续运行.

**四****:Nginx Lua****模块设计**

这里的Nginx Lua模块是指OpenResty.原生的Nginx模块开发需要使用C/C++来实现,并且需要程序员对异步编程模型、Nginx事件系统以及工作原理和内部处理流程.

OpenResty把Lua语言嵌入了Nginx,用Lua作为"胶水"语言粘合Nginx的各个模块和接口,以脚本的方式实现复杂的http、tcp、udp等业务.

另外OpenResty使用Lua内建的协程特性配合Nginx异步非阻塞编程模型,实现了无阻塞的处理各种复杂业务.可以说,对Lua业务代码来说,它是同步阻塞的,对NginxCore来说它是异步非阻塞的.

OpenResty相当于为Lua业务代码营造了一种操作系统之上的氛围.把以前对Nginx的开发看做是内核开发,那么现在Lua业务的开发相当于是在操作系统之上的应用程序开发.内核始终是以异步非阻塞的方式运行的,而应用程序不管是同步阻塞还是异步非阻塞,都不会到内核产生任何影响(阻碍).

可以这么看待Nginx Lua模块(下文不再提OpenResty),它将同步阻塞的Lua代码转换到异步非阻塞的NginxCore中.除此之外,Nginx Lua模块还实现了多协程调度和协作,让"应用层程序员"(Lua程序员)可以在Lua代码中创建多个"线程"(协程)共同完成一个复杂的业务.

 

**1.操作系统和OpenResty**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 

既然OpenResty是为Lua程序员营造一种操作系统的环境氛围,那么我们就配合操作系统的概念理解OpenResty.为了方便理解,我们下文都是基于单核CPU、单进程单线程来描述的.

 

NginxLua模块包含的功能比较多,为了配合我们下文的描述,这里只谈及NginxLua模块的两个部分.一个是NginxLuaAPI,配合Nginx架构提供给Lua程序使用的API.一个是NginxLuaRun（个人抽象出来的一个概念）,实现类似操作系统的进程调度、协作等功能.

 

对于操作系统来说,分为三部分:

内核 系统调用接口 应用程序

对于OpenResty来说:

NginxCore+NginxLuaRun NginxLuaAPI Lua业务代码

 

对NginxLuaAPI很好理解,就像是系统调用一般,将NginxCore的能力通过接口的方式暴露给Lua.Lua可以借助这些接口完成复杂的业务,如网络通信、多进程(协程)编程等.

 

对NginxLuaRun来说,一方面是使用协程将整个体系划分为用户态和内核态,一方面实现了Lua多协程调度和协作(相当于内核的进程调度模型),另一方面为了配合Nginx Http业务处理流程.NginxLuaRun实际上针对的是一个http请求的某个处理流程,因为Nginx的开发就是在http请求的某些阶段进行处理,当http到达某个处理流程时,会"创建"NginxLuaRun来为这个阶段的Lua代码营造一个操作系统的环境氛围,将请求交给应用程序去处理,应用程序可能以多进程的方式处理一个复杂的业务.

​    ![0](https://note.youdao.com/yws/public/resource/965c9f034a82ffb0f8b4de6ca81f3e73/xmlnote/4DE1D9B6FD8F4DC99CDAD2CC12E59295/33446)

 

 

实际上内核就是异步非阻塞的处理各种任务,它的底层实现EventLoop + Reactor,负责提供"动力",也就是回调.并且在此EventLoop的处理中包含了对多进程的调度和处理(驱动),如遍历进程的就绪队列,然后加载对应进程的上下文,恢复进程的运行等.

而NginxLuaRun是针对http请求的某个阶段.它的动力是由NginxCore提供的,自身充当一个分流器.NginxCore底层实现也是EventLoop + Reactor,负责提供"动力",也就是回调驱动NginxLuaRun的执行,而NginxLuaRun会创建协程驱动协程运行,这个协程可能创建多个用户协程,NginxLuaRun会对这些协程进行调度,并支持协程间的协作.

 

NginxCore是EventLoop + Reactor,而NginxLuaRun是一个Loop.这个NginxCore会驱动很多的NginxLuaRun运行,而NginxLuaRun会将动力流分配给用户协程.那么这个NginxLuaRun会堵塞NginCore吗(NginLuaRun是否会长时间占用NginxCore的动力流,致使其他任务得不到(或者是很长时间内得不到)处理)?

可能出现这种情况,但是这是用户没有合理的使用动力流,与此处的设计无关.作者是基于以下原来来设计这个NginxLuaRun的.

首先NginxLuaRun针对的是一个http请求中的某个阶段的处理阶段,这个阶段应该是很快就执行完的.

其次这个阶段中耗时部分是网络操作或者是条件等待等,遇到这种情况NginxLuaRun会让出动力流,由NginxCore的Reactor负责监控这些条件是否满足再分配动力流(驱动).

最后,不可避免的,这个业务是非常复杂的,它可能需要很久才能执行完毕,遇到这种情况,程序员应该有意识的让出动力流(NginxLuaAPI中提供了timer_at等方法).

 

比如很多人排队买包子,第一个人买3个包子,第二个人买2包子,第三个人买100个包子,第四个人买3个包子,第五个人第六个人....,如果第三个人一次就要买够100个包子,那么必然会导致后面的人都延迟一段时间才能买到包子.这时需要第三个人应该有点"素质",本次先买10个包子,然后主动到队尾排队,下次再买10个包子,继续到队尾排队,直到买够100个包子.

**2.深入理解进程和协程**

不管是进程、线程还是协程,从本质上来说都只是代表了一块堆栈空间（这句话的意思是,进程和线程或者协程本身不具备处理能力）.

 

把进程、线程和协程都看做是一个状态机就好理解多了.这个状态机描述了这个任务的起始状态和终止状态以及中间的各种临时状态(如程序运行中的各种变量).以及如何从一个状态迁移到另一个状态(代码逻辑,也可以说是代码指令).

 

具备处理能力的是CPU,CPU加载这个状态机(进程或者线程)的状态信息,从当前的指令位置开始一条一条指令的执行.这些指令的执行导致了状态机的状态发生变化.这就是我们看到的进程的执行.

 

对于协程来说,具备处理能力的是进程或者是线程(实际上是CPU驱动进程或者线程的执行,进程或者线程进而驱动协程执行,本质上还是CPU对指 令的顺序执行).进程/线程的驱动导致了协程这个状态机从一个状态变为另一个状态.

**3.内核和NginxCore**

为了方便理解,这里就先移除了NginxLuaRun.上文说过NginxLuaRun可以理解为一个分流器,只是为了多协程调度和协作的.(当然也有创建和释放协程资源)

 

不管是协程还是进程,中途都可能遇到阻塞.而异步编程就是避免阻塞的一种编程模型.所以kernel和Nginx Core都是采用的异步编程模型,当然也可以理解为一个EventLoop + Reactor.

 

在操作系统之上可能运行着很多的进程,而操作系统对这些进程的驱动是周期遍历(while操作)就绪队列中处于就绪状态的进程(pcb),然后加载对应进程pcb上保存的上下文,恢复它的堆栈空间以及要执行指令的位置,当这个任务处理完毕或者主动返回处理器(让出动力流,也就是退回EventLoop)时,操作系统再加载下一个就绪状态的进程的相关信息,驱动下个进程的运行.

 

而对于Nignx Core来说,它同样是一个EventLoop,周期遍历自己的任务队列,然后调用这个任务的handler函数,驱动对应函数的运行.可以看到Nginx Core跟操作系统的实现是非常相似的.

 

CPU在一段时间内处理的指令数是固定的.而EventLoop的目的就是为了保证这段时间内所执行的指令个数是处理能力的极限,这样才能保证整个系统运行的最大效率.因此EventLoop要求所有任务都是异步非阻塞的.

 

但是很多情况下,我们编写一个程序都是同步阻塞的(程序的同步阻塞和异步非阻塞,在内核看来都是顺序执行了一些代码).就以sleep为例.

我们知道操作系统提供了sleep函数,Nginx Core也提供了相应的定时器操作.当一个进程长时间执行sleep操作或Nginx Lua协程中长时间执行sleep操作时,实际上是不会对整个系统的处理能力产生影响的.也就是所,这段时间内整个系统所处理的任务数(指令数)不受sleep阻塞的影响.

 

这里大概描述一下为什么,这里涉及到的细节还是比较多的.中间的描述并不一定与操作系统的实现保持一致,实际上我对操作系统也并不了解.这里只是凭借自己的想法,如何合理的实现操作系统的相关功能.

首先是进程执行sleep操作时,在sleep中首先这样做.将当前进程的pcb放到一个操作系统提供的定时任务队列中，然后主动让出CPU（yield,从用户态切入内核态）.此时CPU将从应用程序的SP切换到OS的SP,并将应用程序的堆栈信息保存起来.接下来OS遍历就绪队列(定时任务就绪、或者其他条件瞒住而处于就绪状态的任务)驱动(resume,从内核态切回用户态)处于就绪状态的进程继续运行(CPU将从OS的SP切换到对应进程的SP,并按照SP的指示进行处理).

 

可以把resume理解为调用一个函数指针,把yield理解成一个return操作，把进程理解成一个函数(实际上有一种编程模型是Continuation,就是采用这种方式实现状态机的功能).只是这个resume操作会在之前运行的基础上继续运行(函数回调是从头开始),yield操作是将当前运行状态保存然后再返回.所以在Continuation编程中看起来是丑陋和复杂的(它需要根据中间状态制定不同的处理流程).

 

这样看Lua代码中的sleep操作对NginxCore来说也是这样,当调用sleep时,会为当前协程设置一个定时器(首先设置这个协程标识对象上的handler指针,并将当前协程的标识保存到NginxCore的定时任务队列),然后主动执行yield操作,保存当前协程的运行状态.而NginxCore在遍历就绪的定时任务队列时(时间已到),将会对这个任务出队,并回调这个任务对象提供的handler接口.在这个handler接口中,通过resume继续驱动协程的运行.

 

可以看到,进程的阻塞并不会影响操作系统的处理效率,Lua代码的阻塞不影响NginxCore的处理效率.

**4.用户态和内核态**

用户态和内核态的出现是必然的,用户态和内核态实际上是两个不同的堆栈. 系统调用实现了从阻塞到非阻塞的转换, 有些API对于用户态是阻塞的,但是对于内核态来说都是非阻塞的. 所有的用户程序都是被内核所驱动的,在这个用户程序阻塞(等待某个条件满足)期间,内核转而处理其他的用户程序(其他的用户堆栈),等待这个用户程序条件满足并获得处理器之后再继续处理当前用户程序的堆栈.而用户程序在从自己的堆栈空间切换出去,再到切换回自己的堆栈空间这段时间内,表现出的现象就是阻塞.但是对应内核,在这个用户程序"阻塞"期间,自己在非阻塞的处理其他事情.

 

在程序中使用系统调用,可以从用户态切换到内核态,经过内核处理之后,再从内核态切换到用户态.

Lua程序中通过NginxLuaAPI,也可以从协程切换到NginxLuaRun/NginxCore,经过NginxLuaRun/NginxCore处理之后,再从NginxLuaRun/NginxCore切换到用户态.

 

如上述的sleep操作,阻塞connect操作等,都会从用户态转入内核态中,由NginxLuaRun/NginxCore监控就绪状态,再从NginxLuaRun/NginxCore切换到用户态,用户态的程序才可以继续执行（进程因为等待某种条件的满足陷入阻塞状态,条件满足之后由阻塞状态变为就绪状态,在获得处理器之后从就绪状态变为运行状态）.

 

因为内核编程是异步非阻塞的,用户需要对内核架构以及异步非阻塞编程有深入的了解才可入手,而且容易出错.现在通过系统函数提供的操作,用户只能按照固定的方式做事降低了出错情况.并且将用户程序由同步阻塞(实际上对操作系统来说所有的应用程序都是同步阻塞的只是看待的角度不同)的编程方式转变为异步非阻塞,降低编程难度.

 

用户态程序在调用系统函数时,在系统函数中先将某些操作设置到相关的标识变量.然后用户态程序执行陷阱程序(trap或者syscall)进入内核态.在内核态中根据这些标识变量判断用户目的,然后进行处理.处理完毕之后将相关结果保存到某些堆栈上,然后再切换到用户态.

 

上述把内核看做了是一个EventLoop,把进程看做是一个堆栈空间.

当进程中调用sleep时,首先在sleep函数中为当前进程的标识设置了一个定时标志以及一个用来在定时器触发时继续驱动当前进程执行的handler并将当前进程的标识放到内核的定时任务队列中.接下来通过syscall从用户态切换到内核态(保存堆栈信息),在内核处理流程中,也就是EventLoop中会检测当前任务返回的一些信息(如上述的定时标志),然后转而处理其他的就绪任务(进程).等到定时任务就绪时,EventLoop驱动这个进程（任务）继续之前的操作（从内核态切换到用户态）.

由此可见,在内核中的编程(也就是EventLoop)都必须是基于异步非阻塞的编程模型,而且编程复杂,稍有不慎就会对EventLoop产生影响.而用户程序的编程就简单了,只需要调用相关的系统函数即可实现复杂的功能.

 

对比NginxCore和Lua协程(代码).

当NginxCore(EventLoop)在处理某个任务时,将会进入对应的lua_handler,在这个lua_handler中转而进入NginxLuaRun这个分流器.在分流器中,根据相应的调度原则驱动对应的协程(状态机)运行.

而在协程中调用NginxLuaAPI提供的sleep函数时,首先会为自己的上下文设置一些状态标志,然后创建一个定时器对象,为这个定时器对象初始化handler和data以便定时器回调时可以通知到自己,接下来通过将这个定时器投放到Nginx的定时任务队列中,最后通过lua_yield从用户态切换到内核态.

NginxCore(内核)在处理就绪任务时,调用这个任务的handler通知上层等待的条件已经就绪,在对应的处理函数中将"动力"(指令的执行顺序)转入NginxLuaRun,由NginxLuaRun驱动处于就绪状态的协程运行,也就是通过lua_resume操作从内核态再切换到用户态继续执行用户态程序.

 

既然谈到用户态和内核态了,那么我们来谈谈什么情况下OS不能剥夺进程的处理器.

我们编写一个程序: int main() { while(1); },当这个程序运行时,OS是不能剥夺该进程的处理器资源的.但是一个进程的运行是有时间片限制的呀!

一个用户进程只有从用户态切换到内核态之后,处理器才能被剥夺.切换到内核态之后,内核可以决定是立即再从内核态切回用户态让用户进程继续运行,还是过一段时间后再从内核态切换到用户态让用户进程继续运行.

而CPU时间片的实现是这样的,在从内核态切换到用户态之前,先为当前进程计算一个本次运行执行的时间限制,然后驱动进程运行(切入).动程序从用户态切换到内核态之后,内核态检测CPU时间片是否到期,决定立即驱动当前的用户进程,或者将当前的用户进程pcb挂载到就绪队列的尾部,等到轮到它时再驱动当前的用户进程执行.

**5.NginxLuaRun多协程协作**

我在描述的时候,可能将协程描述为进程.其实本文件中不管是进程、线程、协程都是一个概念,那就是一个堆栈空间,一个状态机而已.只是它们占用的资源多少罢了.

 

之前已经介绍过NginxLuaRun,它仅仅是在一个http请求的某个回调处理上营造一种操作系统的氛围.这个处理阶段可以创建多个线程(协程)完成一个复杂的业务.而这里描述的就是多线程(协程)的协作和同步问题. 

 

上文已经描述过NginxCore是一个大的EventLoop,负责提供动力(实际上EventLoop也是一个分流器,负责将CPU提供的动力传输给对应的任务,这里了解一下就好,叙述太多反而不易理解了).而NginxLuaRun是一个分流器.这个分流器与内核的分流器差别比较大,内核的分流器是针对所有用户进程的,而这个分流器仅仅负责一个http请求的一个handler处理.因此这个分流器是小而片面的,它的线程调度(协程)与操作系统的任务调度差别比较大.上述已经描述符NginxLuaRun一般情况下不会对NginxCore造成阻塞(长时间占用动力流),这里就不再描述了.

 

下面我们就针对NginxLuaRun片面的类操作系统协程调度器进行叙述.

 

在Linux操作系统上,我们创建一个进程之后,在一个进程中可以fork多个子进程,而子进程又可以fork新的子进程.

父进程可能会对子进行执行wait_pid()操作,由此引出两个问题.僵尸进程和孤儿进程.

僵尸进程:子进程先于父进程结束,但是父进程没有执行wait_pid()等操做,而是在自己干自己的活.可能在干完自己的活之后再处理子进程的终止状态.

孤儿进程:父进程先于子进程结束.子进程将会被init进程领养,由init进程负责收尸.而在NginxLuaModule中有点区别,它不存在永不退出的init进程,所以处于孤儿状态下的协程相当于是被http request所领养的.

 

进程的状态转换:

​    ![0](https://note.youdao.com/yws/public/resource/965c9f034a82ffb0f8b4de6ca81f3e73/xmlnote/1F4FE193EBC74EFABEC0246A61D32EC4/33447)

 

一个进程从创建之后就处于就绪状态,处于就绪状态的进程获取到处理器资源后就可以开始运行(此时处于运行状态).

处于运行状态的进程在时间片到期的情况或者是主动让出CPU资源,就会从运行状态变为就绪状态.

而处于运行状态的进程如果因为条件不满足将会从运行状态变为阻塞状态(处于阻塞状态的进程即使获得了处理器资源也无法继续运行)

处于阻塞状态的进程在条件满足时将会由阻塞状态变为就绪状态(需要在就绪队列中排队),处于就绪状态的进程在获取到处理器资源时才可以继续运行.

 

更深一步的进程状态转换

Linux中,进程是分为子进程和父进程的.父进程可能阻塞等待子进程终止,也可能子进程退出时,父进程正在处理自己的事情,等到处理完之后再获取子进程的处理结果.就由此引出两个问题,僵死进程和孤儿进程.所以我们对上述状态转换进行补充.

​    ![0](https://note.youdao.com/yws/public/resource/965c9f034a82ffb0f8b4de6ca81f3e73/xmlnote/E23FD43B9D1141DEA1EA1500192ACDEB/33445)

 

当一个进程运行结束后,如果存在父进程但是父进程此时没有阻塞等待子进程退出,此时子进程将变成僵死状态.

处于僵死状态的子进程只有在父进程主动调用wait_pid或者是父进程退出时才会变为终止状态,进行资源回收.（当然父进程有可能对子进程的返回值不关心,也就是detache操作,在NginxLuaRun中不存在该操作,我们这里也就不涉及了）

如果父进程先于子进程结束,子进程将会被Init进程领养.此时子进程在运行结束后将直接变为终止状态(不会存在僵死状态),但是孤儿进程不是一种状态,所以不在上图体现.

如果子进程处理僵死状态,那么子进程的子进程在结束时会是什么状态呢?此时子进程已经结束(也就是不再关心子子进程的终止状态),子子进程就变成了孤儿进程,在终止时直接变为终止状态,进行资源回收等操作.

 

NginxLuaRun采用上述的状态转换实现自己的协程管理.区别与OS是对所有进程(线程)的调度,而NginxLuaRun处理的只有一个实体协程,其他的都是用户子协程,当然Nginx的用户子协程是不随着实体协程的退出而退出的,他们是一种类似于多进程模型的关系. 一个NginxLuaRun中所有的协程都终止后,http请求才能进入下个阶段的处理流程.

当一个http请求到达某个处理阶段时,将会回调某个lua_handler处理函数,在该函数中加载lua代码,并以该lua代码创建lua chunk(闭包工厂)和闭包函数.接下来创建一个协程处理这个闭包函数,通过NginxLuaRun来驱动这个协程运行.

在NginxLuaModule中一个协程的状态划分与OS的划分差不多.分别是就绪状态(suspend)、运行状态(runing)、僵死状态(zombie)、终止状态(dead)、normal状态(与suspend类型,只是出于该状态的协程并不是去就绪队列中排队,而是子协程在让出CPU之后驱动父协程运行)、阻塞状态(这个NginxLuaModule并没有对应的标志,而是添加一些事件到NginxCore中,等待NginxCore的驱动NginxLuaRun,再由NginxLuaRun驱动对应的协程执行).

 

需要知道的是从NginxLuaRun返回到NginxCore时,这个NginxLuaRun中没有一个协程是处于就绪状态的.这有这样,NginxCore在监控到条件满足后回调对应的处理函数,经过处理之后再次进入NginxLuaRun驱动处于就绪状态的协程运行.

 

 

上文已经描述过系统调用过程中用户态和内核态的切换.并不是所有的系统函数都会导致用户态和内核态的切换,如获取一些资源状态等API等.

当一个协程调用某些NginxLuaAPI时,将会从用户态(Lua协程)切换到内核态(NgxinCore/NginxLuaRun)，而操作便是lua_yield.一个协程通过yield从用户态切换到内核态时可能出现几种情况.

第一种是,为发出某些请求,该请求可以立即获取到结果.此时内核（NginxLuaRun）需要在处理完毕之后立即驱动当前协程继续运行(lua_resume)

第二种是,当前协程主动执行yield让出cpu的使用权(如thread_spawn操作),这种情况下当前协程并没有阻塞,在获取到CPU资源的情况下即可继续运行.当前协程将会保存到posted_threads队列（就是就绪队列）的尾部.等待NginxLuaRun遍历posted_threads协程(就绪队列),再驱动对应协程的运行(lua_resume).

第三种是,出现阻塞等操作,如sleep操作和网络IO操作(connect).此类状况发生时,LuaModule将不会再处理当前协程,而是转而处理posted_threads中的协程.直到没有处于就绪状态的协程时才返回到NginxCore,而NginxCore负责监控上述协程等待的条件是否满足,满足时通过相应的回调函数驱动协程继续运行(lua_resume).

 

从用户态切换到内核态之后,NginxLuaRun可能执行的动作是:

第一种是:为响应上述一操作,继续驱动当前协程运行.

第二种是:驱动当前协程的父协程运行.

第三种是:遍历就绪队列(posted_threads),驱动处于就绪状态的协程运行.

 

当一个实体协程终止时NginxLuaRun可能做如下处理:

一个协程在结束时,会先处理它的所有zombie_child_thread.

如果存在用户协程(也就是子协程),说明任务没有完毕,继续处理处于就绪状态的协程,否则NginxLuaRun退出,等待NginxCore再次驱动，之后再进入NginxLuaRun.

如果没有用户协程,说明当前的handler所有任务都已经完毕,根据处理结果转入下个处理流程(handler),或者是finaly_request,此时NginxLuaRun也会销毁.

 

当一个用户协程终止时NginxLuaRun可能做出如下处理:

一个协程在结束时,会先处理它的所有zombie_child_thread.

一个用户协程必然有一个父协程,此时父协程可能存活，也可能已经终止或者僵死.

如果父协程不存在(终止或僵死),接下来查看是否还存在用户协程.如果用户协程数为0,则说明当前NginxLuaRun的任务已经完成.否则转而执行就绪队列中的协程,处理完毕之后等待NginxCore在阻塞协程的条件满足时继续驱动NginxLuaRun的运行.

如果父协程存活,那么就需要考虑他们之间是否存在同步操作.也就是父协程执行wait操作等待子协程结束.如果父协程没有wait操作,那么这个时候用户协程将会变成父协程的僵死协程挂载到父协程的zombie_child_threads队列上.如果父协程等待当前协程退出,那么在释放当前协程之后需要驱动父协程运行.

**6.CoSocket实现**

NginxLuaModule实现了CoSocket(Coroutine Socket),提供了一系列用于操作的接口(类似于系统函数).这些接口对用户(Lua程序)来说是阻塞的,对于内核(NginxCore)来说确是非阻塞的.

用户程序调用一个"阻塞操作"时,将会从用户态切换到内核态,由NginxCore/NginxLuaRun负责在条件满足时进行异步回调.在异步回调的处理过程中,将驱动协程继续运行,也就是从内核态再切回用户态.而在用户态切换到内核态和内核态切换到用户态的这段时间里对Lua程序来说就是阻塞的,而在这段时间里NginxCore/NgxinLuaRun可以在异步处理其他非阻塞任务.

 

下面来分析一下Nginx中的CoSocket的实现.这里就以tcp socket为例了.

tcp socket相关API:socket、connect、receive、receiveuntil、send、settimeout、sslhandshake等.

 

**tcp.socket()**

该函数实际上是为lua返回一个table,这个table包含了tcp的所有socket操作,也就是connect、receive、send等API.

使用方式:

local tcpsocket = ngx.tcp.socket()

tcpsocket:connect, tcpsocket:send, tcpsocket:receive, tcpsocket.receiveuntil....

 

**tcp.connect()**

该函数接收的参数是host和port,以及可选的options.另外NginxLua是支持连接复用的,所以该接口也可能从连接池中获取socket.

这个host可以是IP,也可以是域名.因此connect接口是具备域名解析功能的,当然域名解析操作对于NginxCore来说也是异步非阻塞的.

在这个函数中先获取host和port,然后先从连接池中查找是否有与之匹配的连接,然后复用连接并返回即可.

否则就需要进行DNS解析等操作,设置DNS解析的相关回调然后发起DNS解析操作(这个操作是由NginxCore提供的异步非阻塞操作),通过yield从用户态切换到内核态.

当DNS解析成功之后,将会回调对应的handler,在这个handler中根据传递的data可以知道是谁执行的DNS解析操作,接下来对再去连接池中查找一次是否有与之匹配的连接,如果没有连接就会向指定的地址发起非阻塞的connect请求.非阻塞的connect请求可能立即连接成功,也可能出现EINPROGRESS，所以继续设置回调函数等待连接建立.

当连接建立之后,设置相关的读写处理函数并监听当前连接的读写事件,并将该连接对象保存到对应的堆栈上,通过NginxLuaModule的lua_resume驱动协程继续运行,也就是从内核态再切换到用户态.此时连接已经建立.

 

**tcp.settimeout()**

可以看到,该操作只需要设置一些成员属性在对应阶段使用即可.不需要从用户态切换到内核态,因此这里也就不对它做过多的介绍.

 

**tcp.receive()**

该函数接收一个参数,但是这个参数如果是一个整形,那么它表示的是size.如果是一个字符串,那么它表示的是一个模式.

该函数返回函数调用是否成功,错误消息以及读取到的数据(出错情况下读取到的部分数据)

size:阻塞读取指定字节的数据之后返回或者遇到错误时才返回.如EOS、ERROR等.

模式:"*a"表示读取到连接结束才返回,也就是EOS. "*l"表示读取一行即返回,除非是遇到错误.

如:防止TCP粘包的数据读取操作.

local data, err = tcp:receive(4);

if(err == nil){

log("read error");

tcp:close();

return;

}

data, err = tcp:receive(tonumber(data));

if(err == nil){

log("read error");

tcp:cloe();

return;

}

log("read data:" .. data);

tcp:close();

该函数的实现涉及到了lua_tcp_upstream的设计,这个设计是非常复杂的(需要单独的一个章节进行分析),我们这里只是简单的介绍该部分逻辑如何实现.

当receive函数调用时,会告诉tcp_upstream以哪种方式读取数据,读取指定的字节数，还是读取一行或者读取到EOS.实际上就是告诉tcp_upstream在什么情况下resume当前的协程.

receive函数有几个特点,如果内核中已有的数据满足读取需要,那么会立即返回(也就是不会发生用户态和内核态的切换).如果数据不够,那么就会出现从用户态切入内核态.

首先查看upstream缓冲区上的数据是否满足读取要求,如果满足则创建一个userdata压入lua栈并返回.此时lua程序即可获取到要读取的数据.

如果数据不满足,接下来根据Nginx事件系统,也就是connection对象上的读事件是否就绪,然后对底层的fd执行读取操作,直到遇到EAGAIN或者EOS、ERROR等.接下来继续检查读取到的数据和出错状态是否满足读取要求,如果满足将数据push到lua栈然后返回.

如果不满足读取要求,则接下来设置读回调函数,如果没有监听connect的读事件,则监听该读事件，然后调用lua_yield从用户态切换到内核态.等待NginxCore的epoll_wait监听到文件描述符的读事件之后回调读处理函数,在读处理函数中对底层的fd执行读操作，并将读取到的数据挂载的upstream的buffer chain上.接下来继续监测读取到的数据是否满足要求,然后决定是否通过lua_resume驱动对应的协程运行.

补充一点,这里丢了一个read timeout的处理,这个处理是比较简单的,读者自己思考完成.如:何时添加超时定时器,何时移除超时定时器,超时定时器在NginxCore中的实现和如何通知.

 

**tcp:send()**

使用方式很简单,就是tcp:send(data)即可.

写事件在网络编程中的使用是相对复杂的,但是Nginx使用了一种巧妙的方式降低了写事件的编程难度.

通常我们对写事件的使用是,先将数据写到一个缓冲区中，然后开始监听fd的写事件,写事件触发之后从buffer中读取数据然后写到网络.完毕之后再移除写事件.因此在使用过程需要考虑对写事件的监听和移除等.

而Nginx使用的是ET模式,只有不可写变为可写时才会触发写事件,所以Nginx虽然一直监听写事件,但是写事件触发之后并不是一直通知,写事件上有一个ready标志.在需要执行写操作时,将写事件的回调函数设置为指定的处理函数,首先将数据写到缓冲区,然后根据写事件的ready标志是否为1来主动发起对写处理函数的回调.而写操作遇到EAGAIN,那么就将ready标志设置为0,等待NginxCore中的epoll_wait监控到当前连接的写事件时将ready标志设置为1,然后回调写处理函数.当不需要执行写操作时,Nginx将写事件的回调设置为"写傀儡"回调,这个"傀儡"函数并不处理任何事情,当然也不会修改写事件上的ready标志.

正是基于该事件系统的设计,极大的简化了send函数的实现难度.下面我们就看看send函数的实现.

因为send操作必须是将数据发送给网络,因此send接口也是阻塞的（切记以非阻塞的方式看待send接口）.

首先将data从Lua栈中拷贝到buffer中,接下来将buffer和数据长度设置到upstream上.然后开始检测connect上写事件的ready标志是否为1,如果为1则调用写操作函数将数据发送到网络上,这个过程中一次性发送完毕然后返回实际发送的字节数.如果发送过程中遇到EAGAIN,则需要等待写事件触发时才能继续执行写操作.因此ready不为1或者是写操作遇到EAGAIN,都需要监听当前connect的写事件(如果没有监听则监听),并且设置写事件触发之后的写处理函数,接下来调用yield主动从用户态切换到内核态,等待写事件触发.

写事件触发之后会设置ready标志,然后回调写处理函数,在写处理函数中将数据全部发送完毕之后,将写回调设置为傀儡函数,然后通过resume操作从内核态再切换到用户态.

 

**tcp:close()**

该操作就简单了,只需要从epoll_wait中移除监听的文件描述符,关闭socket,再释放掉upstream等对象即可.因为该操作不会进行异步操作,因此也不需要从用户态切换到内核态.

 

**tcp:sslhandshake()**

是否在当前连接上进行SSL/TLS握手.发生在connet调用之后.

*session, err = tcpsock:sslhandshake(reused_session?, server_name?, ssl_verify?, send_status_req?)*

从函数的原型上看,该函数支持简短握手、设置servername、对服务器端证书的验证方式等,但是不支持双向认证?

该函数返回一个session,用户可以保存这个session用于下次的简单握手(reuse_session).

由于SSL握手中的各种处理是比较复杂的,这里只是简单的介绍处理流程.

其实就是在sslhandshake函数中创建SSL对象,然后作为客户端调用ngx_ssl_handshake函数进行握手.这个函数可能返回AGAIN或者其他错误等.如果似乎AGAIN,则设置connection的读写回调函数,然后从用户态切换到内核态,等待NginxCore在事件触发之后回调对应的处理函数继续握手操作.握手完毕之后将结果压入lua栈中,通过lua_resume再从内核态切换到用户态.

**五****:Nginx lua****指令接口**

本节命名为指令接口和设计浅谈更适合一些.这里介绍一些常见的接口及一个实现的简单介绍.更详细的源码分析见下章.

**1.模块指令**

**lua_code_cache**

语法: lua_code_cache on|off

默认: on

适用上下文: http、server、location、location if

这个指令是指定是否开启lua的代码编译缓存,开发时可以设置为off,以便lua文件实时生效.如果是生产线,为了性能建议开启.

 

NginxLuaHandler在第一次为http请求创建module ctx时,如果lua_code_cache关闭,则创建一个新的Lua VM并保存到module ctx上,否则就复用全局的Lua VM.

当NginxLuaHandler中需要加载Lua代码时,会先从全局闭包工厂表中查找是否存在对应的闭包工厂,如果存在则使用闭包工厂创建对应的闭包函数(接下来一次闭包函数创建协程).否则通过lua_load加载lua代码,将lua代码以lua chunk的形式保存到全局的闭包工厂表中(这个lua chunk就是闭包工厂).

如果lua_code_cache关闭,那么每个请求拿到的Lua VM都是全新的,也就是以为着NginxLuaHandler在加载Lua代码时每次都要使用lua_load的方式进行加载.

 

**lua_package_path**

语法: lua_package_path  

默认: 由lua的环境变量决定

适用上下文: http

设置lua库代码的寻找目录.

 

**lua_package_cpath**

语法: lua_package_cpath 

默认: 空

设置c库(.so等)的寻找目录,以便在lua代码中加载c语言库.

 

**init_by_lua(_file/_block)**

适用上下文: http

作用：初始化一些lua全局变量. 它是在lua VM创建之后调用的.

示例：

 init_by_lua 'cjson = require "cjson"';

  [server](#server) {

​    [location](#location) = /api {

​      content_by_lua '

​        ngx.say(cjson.encode({dog = 5, cat = 6}))

​      ';

​    }

  }

在指令解析阶段会为lua main conf设置一个init_handler,在配置文件解析完毕之后,会调用这个init_handler.这个init_handler中开始加载(寻找)该指令对应的lua代码,然后创建协程通过NginxLuaRun驱动执行.

 

**init_worker_by_lua(_file)**

预init_by_lua的功能类似,在worker进程创建之后调用.(模块初始化)

 

**set_by_lua(_file)**

适用上下文: server、location、location if

这个指令是为了能够让nginx的变量与lua变量相互作用赋值.

示例：

 [location](#location) /foo {

​    [set](#set) $diff ''; *# we have to predefine the $diff variable here*

​    set_by_lua $sum '

​      local a = 32

​      local b = 56

​      ngx.var.diff = a - b;  -- write to $diff directly

​      return a + b;      -- return the $sum value normally

​    ';

​    echo "sum = $sum, diff = $diff";

  }

 

**content_by_lua(_file)**

适用上下文: location、location if

通过这个指令来实现根据http请求做出对应的http响应内容.

[location](#location) /nginx_var {

*# MIME type determined by default_type:*

[default_type](#default_type) 'text/plain';

*# try access /nginx_var?a=hello,world*

content_by_lua "ngx.print(ngx.var['arg_a'], '**\\**n')";

}

 

实际上就是将对应的lua local conf的handler为ngx_http_lua_content_handler. 在find local and update中会将这个handler设置到http_request->content_handler上.

在CONTENT PHASE CHECK函数中发现request->content_handler不为NULL,就会进入ngx_http_lua_content_handler处理流程,在该处理流程中寻找对应的lua闭包函数,然后以此创建协程,通过NginxLuaRun调度和驱动协程运行.

 

**rewrite_by_lua(_file)**

适用上下文: http、server、location、location if

在rewrite phase尾部添加一个handler.

示例：

**location** /foo {

   **set** $a 12; *# create and initialize $a*

   **set** $b ''; *# create and initialize $b*

   **rewrite_by_lua** '

​     ngx.var.b = tonumber(ngx.var.a) + 1

​     if tonumber(ngx.var.b) == 13 then

​       return ngx.redirect("/bar");

​     end

   ';

   **echo** "res = $b";

 }

rewrite phase handler区别于content handler, content handler是按需挂载的, 而rewrite phase handler是全局挂载的. 但是rewrite_by_lua指令是location级别的.

所以一般有rewrite_by_lua指令出现,就会在rewrite phase handler集合的尾部注册一个全局的ngx_http_lua_content_handler. 当一个http请求进入该handler之后,会获取lua module的local conf,如果存在rewrite_handler,则发起对rewrite_handler的回调,同样是加载lua代码创建闭包函数,以此闭包函数创建协程,通过NginxLuaRun进行调度和驱动运行.

 

**access_by_lua(_lua)**

语法：access_by_lua 

适用上下文：http, server, location, location if

在access phase尾部添加一个handler用来权限验证.

 [location](#location) / {

​    [deny](#deny)  192.168.1.1;

​    [allow](#allow)  192.168.1.0/24;

​    [allow](#allow)  10.1.1.0/16;

​    [deny](#deny)  all;

​    access_by_lua '

​      local res = ngx.location.capture("/mysql", { ... })

​      ...

​    ';

​    *# proxy_pass/fastcgi_pass/...*

  }

与rewrite_by_lua的实现类似.

**header_filter_by_lua(_file)**

语法：header_filter_by_lua 

适用上下文：http, server, location, location if

添加一个header filter.用来实现某些功能,如追加一个response header.

 [location](#location) / {

[proxy_pass](#proxy_pass) [http](#http)://mybackend;

header_filter_by_lua 'ngx.header.Foo = "blah"';

  }

与上述的rewrite_by_lua实现类似

 

**body_filter_by_lua(_file)**

语法：body_filter_by_lua 

适用上下文：http, server, location, location if

这个指令可以用来篡改http的响应正文的。

  [location](#location) /t {

​    echo hello world;

​    echo hiya globe;

​    body_filter_by_lua '

​      local chunk = ngx.arg[1]

​      if string.match(chunk, "hello") then

​        ngx.arg[2] = true  -- new eof

​        return

​      end

​      -- just throw away any remaining chunk data

​      ngx.arg[1] = nil

​    ';

  }

与上述的rewrite_by_lua实现类似

 

**其余指令：**

lua_need_request_body

lua_http10_buffering

lua_ssl_ciphers

lua_ssl_crl

lua_check_client_abort

....

**2.lua提供的API(全局table)**

**Table:**

| ngx.arg    | 指令参数，如跟在content_by_lua_file后面的参数 |
| ---------- | --------------------------------------------- |
| ngx.var    | 变量，ngx.var.VARIABLE引用某个变量            |
| ngx.ctx    | 请求的lua上下文                               |
| ngx.header | 响应头，ngx.header.HEADER引用某个头           |
| ngx.status | 响应码                                        |

 

**API**

| ngx.log                       | 输出到error.log                                      |
| ----------------------------- | ---------------------------------------------------- |
| print                         | 等价于 ngx.log(ngx.NOTICE, ...)                      |
| ngx.send_headers              | 发送响应头                                           |
| ngx.headers_sent              | 响应头是否已发送                                     |
| ngx.resp.get_headers          | 获取响应头                                           |
| ngx.timer.at                  | 注册定时器事件                                       |
| ngx.is_subrequest             | 当前请求是否是子请求                                 |
| ngx.location.capture          | 发布一个子请求                                       |
| ngx.location.capture_multi    | 发布多个子请求                                       |
| ngx.exec                      |                                                      |
| ngx.redirect                  |                                                      |
| ngx.print                     | 输出响应                                             |
| ngx.say                       | 输出响应，自动添加'\n'                               |
| ngx.flush                     | 刷新响应                                             |
| ngx.exit                      | 结束请求                                             |
| ngx.eof                       |                                                      |
| ngx.sleep                     | 无阻塞的休眠（使用定时器实现）                       |
| ngx.get_phase                 |                                                      |
| ngx.on_abort                  | 注册client断开请求时的回调函数                       |
| ndk.set_var.DIRECTIVE         |                                                      |
| ngx.req.start_time            | 请求的开始时间                                       |
| ngx.req.http_version          | 请求的HTTP版本号                                     |
| ngx.req.raw_header            | 请求头（包括请求行）                                 |
| ngx.req.get_method            | 请求方法                                             |
| ngx.req.set_method            | 请求方法重载                                         |
| ngx.req.set_uri               | 请求URL重写                                          |
| ngx.req.set_uri_args          |                                                      |
| ngx.req.get_uri_args          | 获取请求参数                                         |
| ngx.req.get_post_args         | 获取请求表单                                         |
| ngx.req.get_headers           | 获取请求头                                           |
| ngx.req.set_header            |                                                      |
| ngx.req.clear_header          |                                                      |
| ngx.req.read_body             | 读取请求体                                           |
| ngx.req.discard_body          | 扔掉请求体                                           |
| ngx.req.get_body_data         |                                                      |
| ngx.req.get_body_file         |                                                      |
| ngx.req.set_body_data         |                                                      |
| ngx.req.set_body_file         |                                                      |
| ngx.req.init_body             |                                                      |
| ngx.req.append_body           |                                                      |
| ngx.req.finish_body           |                                                      |
| ngx.req.socket                |                                                      |
| ngx.escape_uri                | 字符串的url编码                                      |
| ngx.unescape_uri              | 字符串url解码                                        |
| ngx.encode_args               | 将table编码为一个参数字符串                          |
| ngx.decode_args               | 将参数字符串编码为一个table                          |
| ngx.encode_base64             | 字符串的base64编码                                   |
| ngx.decode_base64             | 字符串的base64解码                                   |
| ngx.crc32_short               | 字符串的crs32_short哈希                              |
| ngx.crc32_long                | 字符串的crs32_long哈希                               |
| ngx.hmac_sha1                 | 字符串的hmac_sha1哈希                                |
| ngx.md5                       | 返回16进制MD5                                        |
| ngx.md5_bin                   | 返回2进制MD5                                         |
| ngx.sha1_bin                  | 返回2进制sha1哈希值                                  |
| ngx.quote_sql_str             | SQL语句转义                                          |
| ngx.today                     | 返回当前日期                                         |
| ngx.time                      | 返回UNIX时间戳                                       |
| ngx.now                       | 返回当前时间                                         |
| ngx.update_time               | 刷新时间后再返回                                     |
| ngx.localtime                 |                                                      |
| ngx.utctime                   |                                                      |
| ngx.cookie_time               | 返回的时间可用于cookie值                             |
| ngx.http_time                 | 返回的时间可用于HTTP头                               |
| ngx.parse_http_time           | 解析HTTP头的时间                                     |
| ngx.re.match                  |                                                      |
| ngx.re.find                   |                                                      |
| ngx.re.gmatch                 |                                                      |
| ngx.re.sub                    |                                                      |
| ngx.re.gsub                   |                                                      |
| ngx.shared.DICT               |                                                      |
| ngx.shared.DICT.get           |                                                      |
| ngx.shared.DICT.get_stale     |                                                      |
| ngx.shared.DICT.set           |                                                      |
| ngx.shared.DICT.safe_set      |                                                      |
| ngx.shared.DICT.add           |                                                      |
| ngx.shared.DICT.safe_add      |                                                      |
| ngx.shared.DICT.replace       |                                                      |
| ngx.shared.DICT.delete        |                                                      |
| ngx.shared.DICT.incr          |                                                      |
| ngx.shared.DICT.flush_all     |                                                      |
| ngx.shared.DICT.flush_expired |                                                      |
| ngx.shared.DICT.get_keys      |                                                      |
| ngx.socket.udp                |                                                      |
| udpsock:setpeername           |                                                      |
| udpsock:send                  |                                                      |
| udpsock:receive               |                                                      |
| udpsock:close                 |                                                      |
| udpsock:settimeout            |                                                      |
| ngx.socket.tcp                |                                                      |
| tcpsock:connect               |                                                      |
| tcpsock:sslhandshake          |                                                      |
| tcpsock:send                  |                                                      |
| tcpsock:receive               |                                                      |
| tcpsock:receiveuntil          |                                                      |
| tcpsock:close                 |                                                      |
| tcpsock:settimeout            |                                                      |
| tcpsock:setoption             |                                                      |
| tcpsock:setkeepalive          |                                                      |
| tcpsock:getreusedtimes        |                                                      |
| ngx.socket.connect            |                                                      |
| ngx.thread.spawn              |                                                      |
| ngx.thread.wait               |                                                      |
| ngx.thread.kill               |                                                      |
| coroutine.create              |                                                      |
| coroutine.resume              |                                                      |
| coroutine.yield               |                                                      |
| coroutine.wrap                |                                                      |
| coroutine.running             |                                                      |
| coroutine.status              |                                                      |
| ngx.config.debug              | 编译时是否有 --with-debug选项                        |
| ngx.config.prefix             | 编译时的 --prefix选项                                |
| ngx.config.nginx_version      | 返回nginx版本号                                      |
| ngx.config.nginx_configure    | 返回编译时 ./configure的命令行选项                   |
| ngx.config.ngx_lua_version    | 返回ngx_lua模块版本号                                |
| ngx.worker.exiting            | 当前worker进程是否正在关闭（如reload、shutdown期间） |
| ngx.worker.pid                | 返回当前worker进程的pid                              |

 

**常量**

| Core constants            | ngx.OK (0)ngx.ERROR (-1)ngx.AGAIN (-2)ngx.DONE (-4)ngx.DECLINED (-5)ngx.nil |
| ------------------------- | ------------------------------------------------------------ |
| HTTP method constants     | ngx.HTTP_GETngx.HTTP_HEADngx.HTTP_PUTngx.HTTP_POSTngx.HTTP_DELETEngx.HTTP_OPTIONS ngx.HTTP_MKCOL ngx.HTTP_COPY ngx.HTTP_MOVE ngx.HTTP_PROPFIND ngx.HTTP_PROPPATCH ngx.HTTP_LOCK ngx.HTTP_UNLOCK ngx.HTTP_PATCH ngx.HTTP_TRACE |
| HTTP status constants     | ngx.HTTP_OK (200)ngx.HTTP_CREATED (201)ngx.HTTP_SPECIAL_RESPONSE (300)ngx.HTTP_MOVED_PERMANENTLY (301)ngx.HTTP_MOVED_TEMPORARILY (302)ngx.HTTP_SEE_OTHER (303)ngx.HTTP_NOT_MODIFIED (304)ngx.HTTP_BAD_REQUEST (400)ngx.HTTP_UNAUTHORIZED (401)ngx.HTTP_FORBIDDEN (403)ngx.HTTP_NOT_FOUND (404)ngx.HTTP_NOT_ALLOWED (405)ngx.HTTP_GONE (410)ngx.HTTP_INTERNAL_SERVER_ERROR (500)ngx.HTTP_METHOD_NOT_IMPLEMENTED (501)ngx.HTTP_SERVICE_UNAVAILABLE (503)ngx.HTTP_GATEWAY_TIMEOUT (504) |
| Nginx log level constants | ngx.STDERRngx.EMERGngx.ALERTngx.CRITngx.ERRngx.WARNngx.NOTICEngx.INFOngx.DEBUG |

 

**六****:Nginx lua****实现**

上述已经对NginxLua模块的设计进行了较为详细的分析.而这一章从功能出发描述NginxLua模块的实现,如何参与到Nginx的业务处理流程.本章更加偏向于源码分析.

**1.lua模块承担的角色**

NginxLua模块可以看做是一个抽象层,为了简化模块的开发,将开发过程中常用的接口都暴露了出来.

 

Nginx常见的模块开发有:phase handler、content handler、load-balance、header/body filter.

 

首先Nginx lua模块支持前置处理,如在Nginx启动阶段申请共享内存,以便后续程序的使用,或者为Lua vm(虚拟机, 这个是Nginx中的叫法,实际上就是一个lua_State对象)加载第三方组件和类库等.

 

接下来,由于Nginx的phase handler和header/body filter是静态注册的(全局数组),不支持按需挂载(为某个请求挂载挂载对应的phase handler).但是Nginx lua模块中开发的phase handler和header/body filter是支持按需挂载的(根据location选择不同的处理函数)

它是这样做的:如果需要注册phase handler或者是header/body filter.那么lua module在init阶段就会在将对应类型的handler注册到全局的对应类型的phase handler集合中.如果需要注册header/body filter,则同样在全局的filter chain上注册一个lua_filter.

程序在执行到lua_xxx_handler时,会先获取当前location下的lua_conf配置信息,如果这个lua_conf上存在对应处理的handler(如access_handler、rewrite handler等),就会向该handler发起回调,否则转入下个处理流程.这样,注册一个全局的handler,在这个handler中根据不同的location将任务交给对应的handler进行处理(实现了按需挂载).

在Nginx lua模块中只提供了三种phase handler的编写,分别是rewrite模块、access模块和log模块.

 

内容处理模块, content handler与phase handler不同,因为content handler模块本身就是按需加载的(location级别),所以只需要设置loc_conf的handler,在find location and update阶段会将这个loc_conf->handler设置到http_request->content_handler上.

**2.Nginx业务流程**

为了便于我们理解NginxLua模块内部的源码,我们先介绍一下NginxLua模块与外部的交互和可能出现的情况.

这里我们只描述phase handler、content handler以及filter. (load-blance和ssl的不涉及)

 

**phase handler****处理流程**

一个http请求经过phase handler处理,在phase handler中可能直接作出响应然后进入请求结束处理流程.

一个http请求经过phase handler处理之后,可能转入下个处理流程.

一个http请求经过phase handler处理之后,通知外部处理程序做出响应,然后进入请求结束处理流程.

因此,phase handler返回的状态码是有有多种类型的,不同的返回值会执行不同的处理流程.

 

如果phase handler的返回值是NGX_DONE,则意味着再phase handler内部已经做出了http响应,并调用了ngx_http_finalize_request,表示整个任务已经完成.

如果phase handler的返回值是NGX_DECLINED,则意味着http请求需要继续处理,将会转入下个phase handler的处理流程.

如果phase handler的返回值是NGX_ERROR、NGX_HTTP_CLIENT_CLOSE_REQUEST等,意味着客户端主动断开连接,接下来在check函数中调用ngx_http_finalize_request来释放http request资源和关闭连接.

如果phase handler的返回值是NGX_HTTP_RESPOSNECODE并且存在对应的errorpage,则转入读取errorpage,并且发送response header和response body.

如果phase handler的返回值是NGX_OK、NGX_AGAIN等以及是其他响应body类型的http code,则说明在phase handler中已经发送过header,甚至发送过body数据,将剩余的body数据发送完毕之后,进入请求和连接释放的处理流程.

 

**content handler****处理流程**

在content handler中不管是发送过数据、还是遇到错误、亦或者是没有发送过响应数据但是返回了http reqsponse code,都会进入ngx_http_finalize_request的处理流程.

在ngx_http_finalize_request的处理流程中会执行这些处理:在遇到错误或者任务已经完成则关闭连接和释放request资源, 如果已经有部分数据发送出去,这里将剩余的数据也通过异步的方式发送出去然后再关闭连接和释放request资源.如果是某些http response code,则寻找他们对应的errorpage,然后发送response header和response body,最后关闭连接和释放request资源.

 

**filter****处理流程**

filter的处理就简单多了,数据流过他们时进行处理,处理完毕之后将数据流向下个filter.对它们来说只有正常和出错情况.

如果出错则返回NGX_ERROR,否则将数据传递给下个filter.

返回NGX_ERROR之后,将会进入ngx_http_finalize_request中关闭连接和释放请求对象.

**3.模块初始化**

在配置初始化完毕之后,Nginx就会调用各个http模块的postconfiguration.而NginxLuaModule对应的postconfiguration函数是ngx_http_lua_init函数.

 

在ngx_http_lua_init函数一共做几件事情.

如果需要注册全局的phase handler、header filter和body filter.

创建最顶级的Lua VM,并对它进行初始化,如注册API和一些table等.

调用init_handler加载lua代码,创建协程然后驱动NgxinLuaRun处理init_by_lua指定的lua代码.

 

API的注册:

实际上就是将一个table保存到VM的全局表中,这个table里面可以是函数,也可以是一个table,table的下面又是一堆函数.如: ngx.say、ngx.echo、ngx.exec和ngx.req.read_body等.

该模块使用了一个特殊的方式来实现面向对象.它是将http request保存到了协程的全局表中(本质上也是一个VM),这样在ngx.say函数中,只需要从全局表上根据key获取http_request对象即可.

而面向对象实际上是隐式传递一个this指针,这两种实现方式有异曲同工之处.

**4.lua代码加载**

相关函数见ngx_http_lua_access_handler_inline、ngx_http_lua_access_handler_file等函数.不管是以文件还是字符串的方式注册的lua代码,都需要通过lua_load加载的VM中.

 

Openresty提供了一个开关,用来实现lua代码的动态加载,这个开关是lua_code_cache,它允许Nginx在运行期间动态加载lua chunk.

 

Openresty会为每个http_request绑定一个ngx_http_lua_module_ctx.如果lua_code_cache开关是关闭的,那么在每次创建这个ngx_http_lua_module_ctx时都会创建一个lua VM.也就意味这个这个lua VM中注册表中的code cache table是空的.否则这个ctx将会继承全局的lua VM.

 

lua chunk的加载通过以下三个步骤实现,

首先是调用ngx_http_lua_cache_load_code函数从缓存表中查找对应的lua_chunk

如果未找到则调用ngx_http_lua_clfactory_loadbuffer函数,通过lua_load加载lua代码

最后调用ngx_http_lua_cache_store_code将lua代码的闭包工厂保存到Lua VM的全局函数表中.

 

lua代码经过lua_load之后会变为一个lua chunk(实际上就是一个函数,这个函数在lua中被称之为闭包工厂),接下来对这个lua chunk发起调用之后,才会变为一个闭包函数(实际上就是执行最外层的lua代码)

**5.handler处理流程**

在NgxinLuaModule只提供了phase handler、content handler和filter、ssl以及load-balance等处理接口.我们这里只描述phase handler和content handler的处理.

 

在ngx_http_lua_init函数中,如果有rewrite操作或者access操作,则为全局的对应阶段的phase handler集合的尾部添加对应的全局handler.

这个handler分别是ngx_http_lua_rewrite_handler和ngx_http_lua_access_handler.

由于rewrite操作和access操作,包括实现基本相同,我们就只介绍rewrite的相关操作.

 

当一个请求执行到对应phase handler处理流程时,也就是ngx_http_lua_rewrite_handler,它会获取Nginx Lua模块的local conf对象,如果local conf上存在rewrite_handler,则进入rewrite_handler的处理流程.这个rewrite_handler实际上就是按需挂载.接下来,如果http request上不存在lua module ctx,则创建lua module ctx.

 

在rewrite_handler中先加载相应的Lua代码,并以此创建闭包函数.接下来创建实体协程并将Lua VM中注册的所有函数和table等以元表的方式添加到实体协程上(将Lua VM中的_G table设置到实体协程元表的__index上),接下来将Lua VM栈顶的闭包函数转移到实体协程的栈顶,并将http request对象保存到实体协程的全局表中,以实现类似于面向对象的编程(实体协程中的函数可以直接访问http request对象,相当于函数的参数中隐式传递了一个this指针).

 

接下来通过NginxLuaRun(这个在后续会将,实际上就是run_threads + run_posted_threads)驱动协程运行,完成任务.

 

而这个"任务"可能做出的处理及动作是：

1.在"任务"中做出响应（如执行了ngx.say等操作）然后要进入请求结束的处理流程

2.这个"任务"充当一个"过滤器",执行完毕之后要进入下个处理流程

3.遇到错误,根据错误情况发送出错响应还是断开连接.

 

在NginxLuaRun这一层是不关心协程的处理动作的,它只负责驱动和协调协程的运行.

如果当前阶段所有协程都已经处理完毕,并且中途没有遇到错误.那么NginxLuaRun将返回NGX_OK.此时意味着再协程中可能做出了响应,也可能没有.在phase handler中则根据是否做出响应来转向下个处理流程(返回NGX_DECLINED),还是进入finalize_request处理流程(返回NGX_HTTP_OK, finalize_request并不以该返回值作为响应码,真正的响应码已经发送出去了).

如果当前阶段有协程存活,那就意味着当前阶段的任务没有完成,此时返回NGX_AGAIN,遇到.既然NginxLuaRun返回NGX_AGAIN,那就意味着NginxLuaRun中已经没有处于就绪状态的协程了,这种情况需要等待事件系统的再次回调,驱动NginxLuaRun的再次运行(由NginxLuaRun再驱动协程运行).

如果是业务级别的错误,比如lua程序想要返回500等,在协程的处理中设置http out_header中的status,然后将header信息发送出去即可.此时NginxLuaRun返回NGX_OK(在协程协调和驱动这一层没有发生错误),在phase handler中则返回NGX_HTTP_OK(这个返回值不代表response code),以进入finalize_request处理流程.

如果是连接断开等错误,则事件系统驱动NginxLuaRun的时候会通知该事件,此时NginxLuaRun将返回NGX_ERROR,并在check函数(phase handler的上一级),销毁http request,释放http request申请的资源,断开连接,释放connect对象等.

 

content handler与rewrite所能实现的工作差不多,这里就不介绍了.

**6.NginxLuaRun实现**

首先NginxLuaModule为http request绑定了一个ngx_http_lua_ctx对象.这个ctx上描述了这个阶段内所有协程的状态,如当正在运行的协程、那些协程处于阻塞、就绪状态等.以便NginxLuaRun根据ctx对这些协程进行调度.

 

ctx上的关键属性:

resume_handler:为了配合http处理流程使用.本来设计可以是在事件处理程序中驱动NginxLuaRun运行即可.但是http请求的处理分多个阶段,由ngx_http_core_run_phases驱动整个http处理流程的执行,相当于是一层循环. NginxCore->core_run_phases->NginxLuaRun.

当我们从NginxLuaRun返回之后,直接返回到了NginxCore,因此NginxCore驱动NginxLuaRun运行之前需要先恢复core_run_phases(否则当前阶段处理完毕后不知道如何转向下个处理阶段).

当NginxCore监听到某个事件触发之后,先回调对应的事件处理函数.事件处理函数设置当前就绪的协程(cur_coctx),然后设置resume_handler并且调用ngx_http_core_run_phase以恢复http请求的处理流程,在进入phase handler之后根据resume_handler的指示重新驱动NginxLuaRun的运行.

(多个协程可能同时阻塞,每个事件的处理函数可能都不同,事件系统会回调这些函数.事件系统同一时间只能处理一个事件,因此resume_handler只有一个,并且是按需添加的)

 

 

cur_co_ctx:当前正在处理的协程(或者正要处理的协程)

entry_co_ctx:实体协程,也就是phase handler创建的那个主协程.

posted_threads:就绪队列,存放处于就绪状态等待被处理的协程.

 

在coctx上的关键属性:

parent_co_ctx:当前协程的父协程

zombie_child_threads:僵死的子协程队列

waited_by_parent: parent在等待子进程结束后通知父进程,与parent_co_ctx配合使用.

 

NginxLuaRun的由两部分组成,一部分是ngx_http_lua_run_thread,一部分是ngx_http_lua_run_posted_threads/ngx_http_lua_content_run_posted_threads.

ngx_http_lua_run_thread负责驱动协程运行、协作和调度.

*_run_posted_threads负责遍历就绪队列,并通过ngx_http_lua_run_thread驱动协程运行.

 

当一个协程调用某些NginxLuaAPI时,将会从用户态(Lua协程)切换到内核态(NginxCore/NginxLuaRun)，而操作便是lua_yield.一个协程通过yield从用户态切换到内核态时可能出现几种情况.

第一种是,为发出某些请求,该请求可以立即获取到结果.此时内核（NginxLuaRun）需要在处理完毕之后立即驱动当前协程继续运行(lua_resume)

第二种是,当前协程主动执行yield让出cpu的使用权(如thread_spawn操作),这种情况下当前协程并没有阻塞,在获取到CPU资源的情况下即可继续运行.当前协程将会保存到posted_threads队列（就是就绪队列）的尾部.等待NginxLuaRun遍历posted_threads协程(就绪队列),再驱动对应协程的运行(lua_resume).

第三种是,出现阻塞等操作,如sleep操作和网络IO操作(connect).此类状况发生时,LuaModule将不会再处理当前协程,而是转而处理posted_threads中的协程.直到没有处于就绪状态的协程时才返回到NginxCore,而NginxCore负责监控上述协程等待的条件是否满足,满足时通过相应的回调函数驱动协程继续运行(lua_resume).

 

当一个协程终止返回到内核态,会判断这是一个实体协程还是用户协程.

如果是用户协程,则必然有与之对应的父协程.首先处理当前协程下的所有处于僵死状态的子协程,接下来如果父协程存活并且正在wait,那么释放自身资源之后驱动父协程运行.如果父协程不存在,那么回收完当前协程的资源之后转而处理posted_threads队列中的协程.如果父协程存在,但是没有进行wait操作,此时当前协程将成为对应父协程的僵死进程,挂载父协程的僵死子协程队列上,等待父协程终止之后再进行处理.

 

如果所有协程都已经终止,那么根据情况转向下个处理流程还是做出响应进入关闭状态.

如果有的协程没有关闭,那么可能有两种情况.第一种是处于就绪状态,第二种是等待某个条件满足.

 

只要有协程处于就绪状态,NginxLuaRun就不会返回. NginxLuaRun一旦返回,就意味posted_threads中没有处于就绪状态的协程了.

而阻塞等待的协程在NginxCore监控到对应的事件时,会先回调对应的处理函数,在处理函数中将满足条件的协程设置为cur_coctx,然后驱动NginxLuaRun运行.(先调用ngx_http_lua_run_thread驱动当前协程的运行和调度唤醒的操作,接下来通过*_run_posted_threads处理当前处理就绪状态的协程).

 

**7.NginxLua常用API实现**

**ngx.redirect(url, status)**

status只允许设置三个值,分别是301/302/307.默认status为302.

在该指令调用之前,是不允许有响应已经发送的.并且该指令调用之后,整个NginxLuaRun将终止（所有的协程都会被销毁）.

 

在ngx_http_lua_ngx_redirect函数中,会在headers_out.headers上添加一条location:url的header信息,接下来根据status设置headers_out.status.并设置ctx->exit标志,通过yield从用户态切换到内核态.

 

在内核态中,也就是ngx_http_lua_run_thread函数中,判断exit标志,然后进入ngx_http_lua_handle_exit处理流程.

在ngx_http_lua_handle_exit函数的处理流程中,首先将所有协程都销毁掉,然后根据情况调用ngx_http_lua_send_chain_link发送响应数据(包含header的发送和body的发送),或者进入ngx_http_finalize_request中查找对应的errorpage并以此作为响应数据进行发送.

 

**ngx.exit(status)**

如果是在ssl握手阶段进行的exit,那么就不需要关系响应数据是否被发送.

如果在exit之前已经发送了响应头,那么检测status与response status是否相同,如果不同,则表示程序设计有问题,打印错误.

接下来设置ctx->exit标志,通过yield操作从用户态切换到内核态,在NginxLuaRun中将所有当前阶段的协程销毁,并根据情况将最后的响应数据发送出去.

 

当status >= 200时,做出响应.

当status == 0时,它只会退出当前阶段的处理程序,并继续运行当前请求的后续处理阶段(返回NGX_OK).

 

**ngx.exec(url, args)**

执行内部重定向,该函数必须在发送header或者显示发送body体之前调用.（类似于rewrite操作）.

 

该函数在获取到url和args并将url和args设置到ctx->exec_uri和ctx->exec_args上,然后调用yield从用户态切入到内核态.

 

在NginxLuaRun中发现存在ctx->exec_uri,然后调用ngx_http_lua_handle_exec.在该函数中根据情况调用ngx_http_named_location或ngx_http_internal_redirect(区别是url的第一个字符是@).该指令一旦调用,就会销毁该阶段的所有协程,通常情况下返回NGX_DONE.

 

**ngx.say/ngx.echo****等**

ngx.say只是在ngx.echo的基础上追加了一个'\n'

获取参数,组织成字符串,然后获取一个空闲的chain,将组织的字符串拷贝进chain->buffer中.

接下来调用ngx_http_lua_send_chain_link将这个chain发送出去.

该操作为显示发送body,如果header没有发送,会先将header发送出去,再发送body.当然发送过程是异步的.（区别与tcp.send函数）

 

**ngx.sleep**

设置当前handler的sleep事件的handler和data.然后将这个sleep事件添加到NginxCore的定时任务管理中,接下来调用yield从用户态切换到内核态.

 

当定时器触发时,会回调sleep事件的handler.也就是ngx_http_lua_sleep_handler函数.

首先将data(添加定时器的coctx),设置为当前的cur_coctx,在接下来回驱动这个协程的运行.

这里就需要区分phase处理流程和content处理流程.

对于phase处理流程,是在core_run_phases处理流程中因为阻塞而退出的,当再次触发时应该先驱动core_run_phase运行,在进入phase handler中时再驱动NginxLuaRun的运行.(如果不这样做,在当前phase handler处理完毕之后,是不会进入下个处理流程的, core_run_phases相当于一个外层循环).而对于content handler例外,content handler支持事件系统直接回调content处理流程.

 

ctx->resume_handler就是为了保存再次进入phase_handler中时需要处理的事情.继续驱动NginxLuaRun的运行.也就是ngx_http_lua_sleep_resume函数.

 

ps:content的处理流程允许NginxCore直接驱动NginxLuaRun,由NginxLuaRun驱动协程继续运行.而phase的处理流程必须在core_run_phases中执行对应的phase handler,在phase handler中通过ctx->resume_handler驱动NginxLuaRun的执行(core_run_phase的作用是在当前阶段处理完之后转入下个处理阶段,如果事件系统直接驱动phase处理流程中的NginxLuaRun,那么在当前阶段的NginxLuaRun处理完毕之后需要显示转入下个处理阶段).

 

**ngx.thread.spawn**

创建新的协程,这个协程是一个用户协程,,并将闭包函数压栈.

新的协程抢占了父协程的处理机,所以把父协程保存到ctx的posted_threads队列中,等待NginxLuaRun的驱动.接下来将cur_coctx设置为子协程的ctx并调用lua_yield从用户态返回到内核态，在NgxinLuaRun中,则开始驱动(lua_resume)子协程的运行.

 

 

**ngx.thread.wait**

父协程等待子协程退出,只要有任一个子协程终止该函数就会返回.

首先遍历所有要wait的子协程,如果当前子协程处于僵死状态则回收当前子协程.

否则设置子协程wait_by_parent标志(当子协程终止时,由NgxinLuaRun驱动对应的父协程运行),接下来调用yield从用户态切入内核态.

在NginxLuaRun中(ctx->op = default),会遍历posted_threads队列,先处理已经就绪的协程.

 

**CoSocket****相关实现**

见第四章,第六节