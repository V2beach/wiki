# Makefile

1.makefile的使用上，功能不多，不足以出一本书，直接看文档可以满足需求，https://www.gnu.org/software/make/manual/make.html ，趁这个机会学着非工具书式地看英文文档，系统地快速过一遍这种能力。（只看了chapter1和2[An Introduction to Makefiles](https://www.gnu.org/software/make/manual/make.html#toc-An-Introduction-to-Makefiles)，作为general introduction已经够了，然后看chapter4 **Writing Rules**里的通配符Wildcards、路径检索Directory Search、伪造目标Phony Targets、Multiple Targets和Multiple Rules，chapter7的Conditional Parts of Makefiles也可以看）

2.小项目的makefile可以手写，大项目的用其他工具生成，都需要理解makefile结构（其实就是用一个脚本代替shell里的gcc xxx一系列命令）。

3.cmake、automake等都是简化的生成makefile的工具。

下面几乎都在翻译原文档。https://www.gnu.org/software/make/manual/make.html ，很可能最后发现狗屁不通。

我当然可以选择更简单的方式用cmake管理项目，但底层最终都是用make，我认为不学会makefile的处理逻辑就不能完全理解各种构建工具。

### doc, GNU `make`

下面这样Makefile里的一个功能block称为一个Rule，一条规则。

#### 2.1 What a Rule Looks Like

```makefile
target … : prerequisites …（前提；先决条件；必要条件）[priˈrekwəzɪt]
        recipe（方法；配方；诀窍）[ˈresəpi]
        …
        …
```

target通常是程序生成的文件，可执行文件或者目标文件(object files)。但也可以是要实施carry out的某个行动action，比如clean（phony假的、伪造的[ˈfoʊni]  targets）。

prerequisites是作为输入来创造target的那些文件，一个target通常要依赖于多个prerequisites文件。

A recipe是一个make**要carry out的action（要执行的动作）**，一个方法recipe可以有多个命令，可以同行也可以不同行，但recipe前一定要加一个tab（可以设置tab长度，但不能替换成(4个或8个)空格），.RECIPEPREFIX可以修改prefix不为tab。recipe所在的rule可以没有prerequisites，只有target，比如包含一系列删除命令的clean就没有prerequisites。

rule解释的是什么时候、如何生成target files。make就在指定的prerequisites上执行recipe来创建和更新target files。rule也会解释什么时候、如何执行一个action。

#### 2.2 A Simple Makefile

还是直接看例子吧。

```makefile
edit : main.o kbd.o command.o display.o \
       insert.o search.o files.o utils.o
        cc -o edit main.o kbd.o command.o display.o \
                   insert.o search.o files.o utils.o

main.o : main.c defs.h
        cc -c main.c
kbd.o : kbd.c defs.h command.h
        cc -c kbd.c
command.o : command.c defs.h command.h
        cc -c command.c
display.o : display.c defs.h buffer.h
        cc -c display.c
insert.o : insert.c defs.h buffer.h
        cc -c insert.c
search.o : search.c defs.h buffer.h
        cc -c search.c
files.o : files.c defs.h buffer.h command.h
        cc -c files.c
utils.o : utils.c defs.h
        cc -c utils.c
clean :
        rm edit main.o kbd.o command.o display.o \
           insert.o search.o files.o utils.o
```

上面是一个简单直接的(straightforward)makefile，描述了edit这个可执行文件的生成过程，依赖于8个目标文件，这八个目标文件又依次(in turn)依赖于8个C源文件和3个头文件。

所有C文件里都引入了defs.h，定义编辑命令的几个C文件引入了command.h，底层的改变editor buffer缓冲的C source引入了buffer.h。

输入make就能生成可执行文件edit，输入make clean就能调用上面的clean命令。

recipe是cc -c和cc -o，cc就是gcc的简写，GNU Compiler Collection GNU编译器套装。

prefix tab(.RECIPEPREFIX)在每一个recipe前是用来区分recipe跟其他行的。

关于rule解释什么时候生成target files和执行recipe——**执行make命令时，只有修改过的prerequisites(或不存在的targets)会进行recompile和relink，而那些没有改变的文件不用再次生成。**



clean这个target不是文件只是个action，因为通常不会想执行clean，所以clean不是任何rule的prerequisite。也就是说，除非明确指定，make不会执行clean。clean没有prerequisites所以只是用来执行target对应的recipe（言下之意只有有prerequisites的rule才会生成文件）。

> Targets that do not refer to files but are just actions are called *phony targets*.

不涉及任何文件只有执行动作的targets叫phony targets。

#### 2.3 How `make` Processes a Makefile

make咋处理makefile呢？

default goal（默认目标），默认地，make从第一个target（名字不以'.'为前缀的target）开始。goals就是make实际要更新的那些targets，可以通过.DEFAULT_GOAL改变default goal，或者在命令行make参数里控制。

这下明白了，make -j4/-j6/-j8那些（5.4 Parallel Execution，--jobs，多进程）都是按这个default goal执行的。

还是以上面那个simple Makefile为例，我们（主观上）的default goal就是更新可执行程序edit，所以把这条rule放在最开始。

运行`make`之后。

make读取当前路径下的Makefile并从第一个rule开始处理，在simple Makefile里，第一条rule是重链接relink目标edit，但是在make完全处理完edit这条rule之前，make必须处理edit依赖的文件，也就是prerequisites里的目标文件，每一个对应自己的一条rule，这些rule都在通过编译他们的源文件更新每个.o目标文件。只有在目标文件不存在或者源文件的修改时间比目标文件的修改时间更晚（近）时才会执行重编译recompilation的这些rule。

其他规则rule被处理只因为他们的targets在default goal的prerequisite里出现了，如果default goal不依赖于某条rule的target，那么就不会处理那条rule，除非在make里用一个类似于make clean的指令指定了。

但是在处理每一个目标文件时，make还是会考虑更新他们的prerequisites，在这里就是那些source files和header files。但是在Makefile里，这些源文件头文件并不是任何rule的target，所以make就不会对这些.c和.h文件做任何事了。但是！**假如有，make会按照这些源文件对应的规则，更新那些用Bison或Yacc之类的自动生成的代码。**

只有当edit不存在或者任何prerequisites比他新的时候，在执行完上面的目标文件更新之后，make开始重链接edit。比如我们改变了command.h，会重编译kbd.o，command.o和files.o，然后链接edit。

#### 2.4 Variables Make Makefiles Simpler

main.o kbd.o command.o display.o insert.o search.o files.o utils.o这种长串容易出错，一般需要用变量。用法是，'='定义完用$()调用。

```makefile
objects = main.o kbd.o command.o display.o \
          insert.o search.o files.o utils.o

edit : $(objects)
        cc -o edit $(objects)
main.o : main.c defs.h
        cc -c main.c
kbd.o : kbd.c defs.h command.h
        cc -c kbd.c
command.o : command.c defs.h command.h
        cc -c command.c
display.o : display.c defs.h buffer.h
        cc -c display.c
insert.o : insert.c defs.h buffer.h
        cc -c insert.c
search.o : search.c defs.h buffer.h
        cc -c search.c
files.o : files.c defs.h buffer.h command.h
        cc -c files.c
utils.o : utils.c defs.h
        cc -c utils.c
clean :
        rm edit $(objects)
```

#### 2.5 Letting `make` Deduce(推断；演绎)[dɪˈdus] the Recipes

详细清楚地说明spell out编译源文件的recipes不是必须的。

```makefile
objects = main.o kbd.o command.o display.o \
          insert.o search.o files.o utils.o

edit : $(objects)
        cc -o edit $(objects)

main.o : defs.h
kbd.o : defs.h command.h
command.o : defs.h command.h
display.o : defs.h buffer.h
insert.o : defs.h buffer.h
search.o : defs.h buffer.h
files.o : defs.h buffer.h command.h
utils.o : defs.h

.PHONY : clean
clean :
        rm edit $(objects)
```

有个隐式implicit[ɪmˈplɪsɪt]规则是，更新一个.o文件的时候，make会自动关联同名的.c文件，并自动用cc -c命令，所以可以直接省略。

比如recipe ‘cc -c main.c -o main.o’ to compile main.c into main.o，可以直接omit [oʊˈmɪt]忽略掉只写main.o。

#### 2.6 Another Style of Makefile

如果用上面的隐式规则写makefile，就可以用下面这种通过prerequisites而不是targets组织的形式。

```makefile
objects = main.o kbd.o command.o display.o \
          insert.o search.o files.o utils.o

edit : $(objects)
        cc -o edit $(objects)

$(objects) : defs.h
kbd.o command.o files.o : command.h
display.o insert.o search.o files.o : buffer.h
```

所有源文件都用了defs.h，个别文件用了另外两个头文件，重复定义targets来包含所有的prerequisites，按prerequisites来组织，但是书写顺序不变，多个targets对应一个prerequisites。

哪种更好见仁见智，这个写起来紧凑但不清晰，不那么好读。

#### 3.3 Including Other Makefiles

include指令(directive[d**aɪ**ˈrektɪv])指示符告诉make暂停(suspend[səˈspend])挂起读取当前的makefile，转而先行读取include指定的makefiles。

```makefile
include filenames…
```

行尾可以添加#注释，前缀或者说行首不能是tab，否则会被视为recipe。另外，filenames里如果有$()变量或者函数会被扩展。

假如现在有这样一些文件要引入，有foo，a.mk, b.mk, and c.mk, and `$(bar)` expands to `bish bash`，

```makefile
include foo *.mk $(bar)
is equivalent to
include foo a.mk b.mk c.mk bish bash
```

一种需要用include的情形occasion是有很多个程序，分别被在不同的路径下独立的makefiles负责处理，他们需要通用的变量定义的集合或者pattern模式规则（共用部分用include引入）。

另一种情形是，想从源文件自动生成prerequisites的时候，prerequisites可以放在一个文件里，主要的makefile把他引入。这种实践通常比把prerequisites附加在makefile末尾简洁。这里没懂，也没继续读chapter4的对应部分，https://www.gnu.org/software/make/manual/make.html#Automatic-Prerequisites。

make首先会在include的指定路径找included makefiles，然后会在‘-I’ or ‘--include-dir’指定的路径里找，最后会从prefix/include (normally /usr/local/include [1](https://www.gnu.org/software/make/manual/make.html#FOOT1)) /usr/gnu/include, /usr/local/include, /usr/include.里面找。

##### 这里要注意，参见[Summary of Options](https://www.gnu.org/software/make/manual/make.html#Options-Summary)，这里的‘-I’ or ‘--include-dir’是make的参数而不是gcc的，跟CXXFLAGS简记里的区分开，另外这里引入的无论什么后缀都是makefile。

如果引入的makefile在上面的路径里找不到，make首先会给一个warning，会继续处理整个makefile，尝试remake这个makefile，如果还找不到就报fatal error。

如果就算include找不到某些makefile也不想让他报错，就需要用-include或sinclude，其他的完全一样，除了-include找不到makefile甚至makefile的prerequisites的时候都完全不会报错。

#### 4 Writing Rules

##### 					9.2 Arguments to Specify the Goals

goals是make最终要更新的那些targets。其他targets只有出现在prerequisites里才会更新。

规则的顺序不重要，上面说了，从第一个不是以'.'为前缀的target开始，以'.'为前缀如果target里有'/'也可以作为default goal。如果第一条rule里面有多个targets，那就选择第一个target作为default，.DEFAULT_GOAL可以修改选择default goal的规则。

或者在make的参数里指定，比如make clean，make edit，make会按照顺序挨个处理这些goals。

**除非以'-'前缀开始或者包含'='**，makefile的任何target都可能是goal，甚至不在makefile里的target，如果有隐式规则make，也可以作为goal。

make参数里指定的goals会被存在MAKECMDGOALS这个变量variable里，一般用不到。

```makefile
sources = foo.c bar.c

ifneq ($(MAKECMDGOALS),clean)
include $(sources:.c=.d)
endif
```

$(sources:.c=.d)这个语法的意思是，sources变量所有.c替换成.d。

这个condition语句的意思是，当执行clean rules的时候就不要引入.d的makefile了，用不上。这个还挺有用的。（include按顺序载入文件，最先载入的.d makefile中的目标会成为默认目标。）

指定goal的一个使用场景是，只想编译一个程序的其中一部分，或者只想编译多个程序中的一个。

```makefile
.PHONY: all
all: size nm ld ar as
```

比如假如只想用上面的size程序，就可以指定make size，只有size这条rule对应的程序会被重编译。

另一个指定goal的使用场景是，make一些不常用的文件，比如debugging output用的，或者testing使用的程序，这些在makefile里写了但并不是default goal的prerequisites。

还有一个指定goal的使用场景就是执行关联着phony target或者empty target的recipe，比如ma ke clean，很多makefiles都有clean这个phony target，只有你显式指定make xxx才会执行这些phony target。

##### 16.6 [Standard Targets for Users](https://www.gnu.org/software/make/manual/make.html#Standard-Targets)

这里是一些上面提到的，需要指定goal的，通常用得到的targets。

> All GNU programs should have the following targets in their Makefiles:

###### all

Compile the entire program. This should be the default target.

这个不是天然的默认目标，一般要把all放在开头（includes、conditional part和variables之后）。

###### clean，mostlyclean，distclean，realclean，clobber

###### install

Copy the executable file into a directory that users typically search for commands; copy any auxiliary(辅助的) files that the executable uses into the directories where it will look for them.

###### print

Print listings of the source files that have changed.

###### tar

Create a tar file of the source files.

###### shar

Create a shell archive (shar file) of the source files.

###### dist

Create a distribution file of the source files. This might be a tar file, or a shar file, or a compressed version of one of the above, or even more than one of the above.

###### TAGS，check，test

#### 4.4 Using Wildcard Characters in File Names

make里的通配符可以是***，?，[...]**，跟正则像。～如果单独出现或者跟着一个slash /就代表home路径，～代表/home/you。如果～跟着词，比如~might/bin，代表用户名是这个词，扩展为/home/might/bin。如果是MS-DOS or MS-Windows这种没有home路径的系统，这个功能用环境变量HOME模拟。

在recipes里，shell负责wildcard expansion通配符扩展，在其他环境里，比如makefile里，通配符扩展只有在使用wildcard函数的时候发生。（下面的target-pattern、prereq-pattern、automatic variables不算他说的通配符扩展，这里说的是无限制的、随时用的）

#### 4.4.1 Wildcard Examples

通配符可以用在recipe里，比如，

```makefile
clean:
        rm -f *.o
```

也可以用在prerequisites里，比如，

```makefile
print: *.c
        lpr -p $?
        touch print
```

make print可以打印所有改变了的.c文件。这个是emoty target，会生成一个print文件，可以是空的，只需要记录时间，这个rule目的是获取任何新于print的文件。

定义变量的时候不能直接用通配符，比如，

```makefile
objects = *.o
```

objects就会变成'\*.o'这个字符串，但是在target, prerequisite, recipe里使用这种形式的objects，这个\*也会作为通配符被扩展。

```makefile
objects := $(wildcard *.o)
```

用这种形式才能获得所有匹配通配符的文件list变量objects。

#### 4.4.2 Pitfalls of Using Wildcards

现在想实现，用路径下所有.o文件生成foo，

```makefile
objects = *.o

foo : $(objects)
        cc -o foo $(CFLAGS) $(objects)
```

跟上文提到的一样，这种写法可以匹配到所有路径下的.o，但跟我的目标不一致，这里只是因为在recipe里被shell调用到了\*.o，prerequisites实际上是'\*.o'，意思是，假如路径下所有的.o目标文件都被删除了，这种写法会报错而不是根据prerequisite作为target的rule重新生成.o文件。

所以这种情况还是要用wildcard function。

> MS下面路径用\分割，Unix用/，最好用slash/而不是backslash\。

#### 4.4.3 The Function `wildcard`

跟上面*.o匹配不到也会逐字verbatim使用不一样，$(wildcard, pattern...)匹配不到就被忽略omitted。

```makefile
$(patsubst %.c,%.o,$(wildcard *.c))
```

跟最下面8.1的subst一样，para3中的所有para1被para2替换。可以把路径里所有的.c文件后缀变为.o存到list里。

```makefile
objects := $(patsubst %.c,%.o,$(wildcard *.c))

foo : $(objects)
        cc -o foo $(objects)
```

> (This takes advantage of the implicit rule for compiling C programs, so there is no need to write explicit rules for compiling the files. See [The Two Flavors of Variables](https://www.gnu.org/software/make/manual/make.html#Flavors), for an explanation of ‘:=’, which is a variant of ‘=’.)

意思是这种写法也能自动按隐式规则编译所有.o的同名.c源文件。

#### 4.5 Searching Directories for Prerequisites

vpath，可能用得到，用到再学。

#### 4.6 Phony Targets

一个phony target是非文件名的target，rather更确切地说，就是个当你显式指定的时候执行的recipe的名字。有两个用phony的原因，避免这个recipe的名字跟文件名有冲突（比如这个文件名刚好是某个goal相关的prerequisite）；还用来提高性能。

```makefile
#.PHONY: clean
clean:
        rm *.o temp
```

这个例子里，如果不加.PHONY，当前路径出现一个文件叫clean的话，clean这个target对应的recipe无论如何都不会被执行到，因为clean没有任何prerequisites，clean这个target永远会被认为是up to date最新的。所以需要通过一个特殊的target .PHONY，让clean这个target作为.PHONY的prerequisite，指明这是个假的target。

这样无论路径下有没有clean文件都可以用make clean执行clean这个rule了。



Phony targets在跟make结合的递归调用时也很有用。这种情况里makefile通常有一个子路径列表的变量，一种简单方法是定义一条rule，在recipe里用一个loop处理所有的子路径。

```makefile
SUBDIRS = foo bar baz

subdirs:
        for dir in $(SUBDIRS); do \
          $(MAKE) -C $$dir; \
        done
```

\$在make和shell里用法都很丰富。

\$在shell里也是调用变量，只用一个dollar\$的话，make预处理会用makefile里定义的变量取代\$，所以需要用俩dollar\$\$调用shell里的变量。

为什么这里调用的是shell里的变量而不是makefile里定义的变量？因为recipe里的loop在make预处理后会转为shell指令，这里的dir是在shell命令里才会执行到的。

这种写法有很多问题。

1.任何检测出来的error都会被这条rule忽略。

2.



Phony targets也可以有prerequisites。

#### 7 Conditional Parts of Makefiles

Conditional，SyntaxTesting Flags。。。

Makefile的条件部分，比如ifdef, ifeq，，endif。

假如有xxx的定义或者假如相等就执行某个命令，比如include $()，处理Makefile脚本的也是一种编译器，只不过相对简单。

#### 10 Using Implicit Rules

边用边记吧。

---

### syntax相关，很重要

#### 4.2 [Rule Syntax](https://www.gnu.org/software/make/manual/make.html#Rule-Syntax)

```makefile
targets : prerequisites
        recipe
        …
targets : prerequisites ; recipe
        recipe
        …
```

通常一条rule就是上面这两种样子，因为dollar符号\$用于变量引用，所以如果prerequisite里用到了，就需要转义写两个dollar符号\$\$代替一个。

最开始包括上文中提到的，一条rule会解释when和how更新targets，这里再次详细解释，prerequisites决定targets是否out of date过期了，如果targets比任一prerequisites更旧，就要更新；至于如何更新targets由recipe决定，recipe包含多条在shell里执行的命令。(see [Writing Recipes in Rules](https://www.gnu.org/software/make/manual/make.html#Recipes)).

#### 4.12.1 [Syntax of Static Pattern Rules](https://www.gnu.org/software/make/manual/make.html#Static-Usage)，最重要

```makefile
targets …: target-pattern: prereq-patterns …
        recipe
        …
```

这个是静态模式的rule，targets确定rule作用的对象，跟普通的rule一样可以包含wildcard通配符。

注意target-pattern只有一个，prereq-pattern**s**可以有多个。

target-pattern和prereq-pattern说明怎么计算每个target的prerequisites，每个target跟target-pattern匹配来得到一部分target name，叫做stem（梗；茎；干），stem被替代进所有prereq-patterns来构造prerequisite name。

每个pattern都normally（通常；正常情况下）只包含一个%，当target-pattern匹配到一个target的时候，%可以匹配target name的任意部分，这个部分就是上面说的STEM，而stem之外剩下的target name的部分必须跟target-pattern完全匹配。每个目标的先决条件名称是通过用**词干**stem替换每个先决条件模式中的“%”来生成的。（很好理解，**就是target-pattern用通配符提取target name里的前缀之类的东西，填充到prereq-pattern作为prerequisites。**）

%的转义用\，backslashes。

```makefile
objects = foo.o bar.o

all: $(objects)

$(objects): %.o: %.c
        $(CC) -c $(CFLAGS) $< -o $@
```

以这个为例，用所有的相关.c文件即foo.c和bar.c编译得到foo.o和bar.o。

这里的\$<和​\$@是automatic variable自动变量，\$<指prerequisite名字即%.c，​\$@指target名字即%.o。(see [Automatic Variables](https://www.gnu.org/software/make/manual/make.html#Automatic-Variables).)

<u>注意</u>，这里**每个targets都必须能匹配上target pattern，如果不是全部匹配，就要用filter函数**，函数相关在下面chapter8。

filter可以**排除**掉target里不匹配target-pattern的target file names。

```makefile
files = foo.elc bar.o lose.o

$(filter %.o,$(files)): %.o: %.c
        $(CC) -c $(CFLAGS) $< -o $@
$(filter %.elc,$(files)): %.elc: %.el
        emacs -f batch-byte-compile $<
```

这个例子里\$(filter %.o,​\$(files))的结果是bar.o lose.o，用bar.c和lose.c编译。而$(filter %.elc,\$(files))的结果是foo.elc，由foo.el生成。

```makefile
bigoutput littleoutput : %output : text.g
        generate text.g -$* > $@
```

%可以提取出big和little，这里的*可以填充big和little，分别生成到bigoutput和littleoutput里。

几个automatic variables的不同点，**\$@和\$<都是取全名，分别是target和prerequisite，而\$*只取%通配符匹配到的部分。**

没时间整理了，这个直接查下面文档吧。

##### [Automatic Variables](https://www.gnu.org/software/make/manual/make.html#Automatic-Variables)

---

```makefile
build/%.o: src/%.cpp
    @mkdir -p $(@D)
    $(CXX) $(INCPATH) $(CXXFLAGS) -MM -MT build/$*.o $< >build/$*.d
    $(CXX) $(INCPATH) $(CXXFLAGS) -c $< -o $@
```

上面这段是familia的makefile，作为example，要把他学到我的项目里。

build/%.o: src/%.cpp这个写法我猜是匹配所有上面的prerequisites，这样写合法，学到了。

首先$(@D)这个，开始以为是shell的语法，recipe不知道怎么区分，for:$$var那种写法又会到shell里执行，总之查到了https://stackoverflow.com/questions/42646316/what-does-d-mean-in-shell-script 。makefile里\$@指的是target name也就是build/%.o，\$(@D)取的是前面的路径build，如果不存在slash/，\$(@D)的值就是`.`。

然后这里第二句的意思是，打印所有\$<源文件的依赖，把目标名改为build/%.o，并把执行结果>输出到build/\$*.d里。

最后这句就是编译了。

```makefile
build/libfamilia.a: $(OBJS)
    ar crv $@ $(filter %.o, $?)
```

而这条rule的作用是用OBJS的所有目标文件生成库文件，ar用于操作archive file，用来创建静态链接库.a文件，动态链接库.so文件也参考他的做法，

```makefile
python/familia.so : python/cpp/familia_wrapper.cpp familia
	$(CXX) $(INCPATH) $(CXXFLAGS) -c $< -o python/cpp/familia_wrapper.o
	$(CXX) $(INCPATH) $(CXXFLAGS) -shared python/cpp/familia_wrapper.o $(LDFLAGS_SO) -l$(PYTHON_VERSION) -o $@
```

首先用\$<取到第一个prerequisite，生成目标文件，然后用-shared生成$@的.so文件。

-shared的作用写在下面了。

---

#### 5.1 [Recipe Syntax](https://www.gnu.org/software/make/manual/make.html#Recipe-Syntax)

@echo，echo用于打印、回显，@控制命令不在shell里输出结果，

@echo一般用于把内容输出到别的地方比如文件里，而不再shell里显示。

#### 7.2 [Syntax of Conditionals](https://www.gnu.org/software/make/manual/make.html#Conditional-Syntax)

上面写过了。

#### 8.1 [Function Call Syntax](https://www.gnu.org/software/make/manual/make.html#Syntax-of-Functions)

```makefile
$(function arguments)
or
${function arguments}
```

比如，

```makefile
comma:= ,
empty:=
space:= $(empty) $(empty)
foo:= a b c
bar:= $(subst $(space),$(comma),$(foo))
# bar is now ‘a,b,c’.
```

这里subst是用comma替换foo的每个space，para3中的所有para1被para2替换。

#### 9.7 [Summary of Options](https://www.gnu.org/software/make/manual/make.html#Options-Summary)

**make的参数。**

### CXXFLAGS(gcc options)简记

familia+marisa里面用到的记录在这。

##### -fpic or -fPIC

在动态库中生成位置无关的代码，通过全局偏移表GOT(global offset table)访问所有常量地址。程序启动时动态加载程序解析GOT条目。https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html

man gcc查看manual，/something之后用n和N查找下一个或上一个。

##### -pipe

Use pipes rather than temporary files for communication between the various -Istages of compilation.  This fails to work on some systems where the assembler is unable to read from a pipe; but the GNU assembler has no trouble.

编译优化操作，不加的话/tmp下会产生中间文件.s汇编文件或.o，用管道避免文件IO。

##### -W

man gcc里用/-W$搜不到，所以不用！相对地，-w是关闭编译warning。

##### -Wall

Turns on all optional warnings which are desirable for normal code.  At present this is -Wcomment, -Wtrigraphs, -Wmultichar and a warning about integer promotion causing a change of sign in "#if" expressions.  Note that many of the preprocessor's warnings are on by default and have no options to control them.

开启所有正常情况下代码需要的warnings。

##### -std=c++11

Determine the language standard.   This option is currently only supported when compiling C or C++. The compiler can accept several base standards.

c90, c89, iso9899:1990, c9x, c11, c1x, gnu90, gnu89, gnu99, gnu9x, gnu11, gnu1x, c++98, c++03, gnu++98, gnu++03, c++11, c++0x, gnu++11, gnu++0x, c++1y, gnu++1y

##### -fno-omit-frame-pointer

-O, -O2, -O3, -Os编译优化会开启-fomit-frame-pointer，需要用-fno-关掉。

> frame pointer指向本函数栈帧顶，通过它可以找到本函数在进程栈中的位置。有专门的寄存器保存该值。主要是backtrace用，每个函数的frame pointer保存在其后调用的函数的栈帧中。 因此可以得到调用层级里面的每个函数的栈帧，从而可以打印出back trace。

##### -fpermissive

Downgrade some diagnostics about nonconformant code from errors to warnings.  Thus, using -fpermissive allows some nonconforming code to compile.

将一些关于不合格代码的诊断从错误降级为警告。 因此，使用 -fpermissive 允许编译一些不符合要求的代码。

这个感觉还挺有用？

##### -O3

Optimization Option，-O  -O0  -O1  -O2  -O3  -Os -Ofast -Og。

Optimize yet more.  -O3 turns on all optimizations specified by -O2 and also turns on the -finline-functions, -funswitch-loops, -fpredictive-commoning, -fgcse-after-reload, -ftree-vectorize, -fvect-cost-model, -ftree-partial-pre and -fipa-cp-clone options.

除了开启O2的优化外还开启后面这些编译优化。

##### -ffast-math

Sets -fno-math-errno, -funsafe-math-optimizations, -ffinite-math-only, -fno-rounding-math, -fno-signaling-nans and -fcx-limited-range. This option causes the preprocessor macro "\_\_FAST_MATH\_\_" to be defined. This option is not turned on by any -O option besides -Ofast since it can result in incorrect output for programs that depend on an exact implementation of IEEE or ISO rules/specifications for math functions. It may, however, yield faster code for programs that do not require the guarantees of these specifications.

定义fast math那个预处理器宏，理解的是降低精度，可能会导致精确实现的代码输出错误，如果不需要数学函数的IEEE/ISO规范，开启会产生更快代码。我们这里显然完全不需要，采样用的全是INT，最后推理算分布用的也不需要精度很高。

这个选项只有-Ofast会打开。

##### -pthread

This is a synonym for -**pthreads**. 即-pthread和-pthreads同义。

Add support for multithreading using the POSIX threads library.  This option sets flags for both the preprocessor and linker.  This option does not affect the thread safety of object code produced  by the compiler or that of libraries supplied with it.

在编译时增加多线程的支持。该选项同时对“预处理器”和“链接器”产生作用。

不影响编译器产生的目标代码的线程安全性，或代码里用到的多线程库的线程安全性。

man gcc已经找不到-lpthread就说明弃用了，忘了他吧。

##### -I <u>dir</u>

Add the directory dir to the list of directories to be searched for header files. Directories named by -I are searched before the standard system include directories.  If the directory dir is a standard system include directory, the option is ignored to ensure that the default search order for system directories and the special treatment of system headers are not defeated .  If dir begins with "=", then the "=" will be replaced by the sysroot prefix; see --sysroot and -isysroot.

跟上面3.3 Including Other Makefiles的-I别搞混。

用-I将dir添加到搜索的**头文件**路径列表里，如果dir路径是系统自动引入的路径则被忽略以防对系统头文件特殊处理被破坏。如果dir以'='为前缀则自动替换为sysroot prefix。

##### -I<u>dir</u>

> Add the directory dir to the head of the list of directories to be searched for header files.  This can be used to override a system header file, substituting your own version, since these directories are searched before the system header file directories.  However, you should not use this option to add directories that contain vendor-supplied system header files (use -isystem for that).  If you use more than one -I option, the directories are scanned in left-to-right order; the standard system directories come after. 
>
> If a standard system include directory, or a directory specified with -isystem, is also specified with -I, the -I option is ignored.  The directory is still searched but as a system directory at its normal position in the system include chain.  This is to ensure that GCC's procedure to fix buggy system headers and the ordering for the "include_next" directive are not inadvertently changed.  If you really need to change the search order for system directories, use the -nostdinc and/or -isystem options.

这个可用于覆盖系统头文件？从左往右扫描，好像跟上面没区别啊。

##### -L<u>dir</u>

Add directory dir to the list of directories to be searched for -l.

###### 注意1.这里是L上面是i；2.这俩跟dir或者library连着的时候注意有无空格，不加空格似乎总是对的。

> -l<u>library</u> or -I <u>library</u>，Search the library named library when linking.  (The second alternative with the library as a separate argument is only for POSIX compliance and is not recommended.) It makes a difference where in the command you write this option; the linker searches and processes libraries and object files in the order they are specified.  Thus, foo.o -lz bar.o searches library z after file foo.o but before bar.o.  If bar.o refers to functions in z, those functions may not be loaded. The linker searches a standard list of directories for the library, which is actually a file named liblibrary.a.  The linker then uses this file as if it had been specified precisely by name. The directories searched include several standard system directories plus any that you specify with -L. Normally the files found this way are library files---archive files whose members are object files.  The linker handles an archive file by scanning through it for members which define symbols that have so far been referenced but not defined.  But if the file that is found is an ordinary object file, it is linked in the usual fashion.  The only difference between using an -l option and specifying a file name is that -l surrounds library with lib and .a and searches several directories.

链接的时候搜索名为library的库。（**加空格的第二种形式仅用于 POSIX 合规性，不推荐！**）

要按顺序，不知为啥（估计跟编译原理相关），似乎一般放在目标文件之后。

另外把xxx.a或者xxx.so文件放进/usr/lib（sudo权限）/usr/local/lib（一般权限）之类的地方，链接时用-l或-Llibxxx.a或libxxx.so引入。

比较方便的方式还是make时直接输出到项目路径下，然后export LD_LIBRARY_PATH=./third_party/lib:$LD_LIBRARY_PATH，只需要替换其中项目路径的./third_party/lib。

##### -c <u>file</u>

-c 生成.o目标文件，-S 生成.s汇编文件。

##### -o <u>file</u>

Write output to file.  This is the same as specifying file as the second non-option argument to cpp.  gcc has a different interpretation of a second non-option argument, so you must use -o to specify the output file.

将输出写入file。

##### -M

> Instead of outputting the result of preprocessing, output a rule suitable for make describing the dependencies of the main source file.  The preprocessor outputs one make rule containing the object file name for that source file, a colon, and the names of all the included files, including those coming from -include or -imacros command line options.
>
> Unless specified explicitly (with -MT or -MQ), the object file name consists of the name of the source file with any suffix replaced with object file suffix and with any leading directory parts removed.  If there are many included files then the rule is split into several lines using \-newline.  The rule has no commands.
>
> This option does not suppress the preprocessor's debug output, such as -dM.  To avoid mixing such debug output with the dependency rules you should explicitly specify the dependency output file with -MF, or use an environment variable like DEPENDENCIES_OUTPUT.  Debug output will still be sent to the regular output stream as normal.
>
> Passing -M to the driver implies -E, and suppresses warnings with an implicit -w.

不输出与处理的结果，而是输出（如到shell）一个描述当前make的源文件所有依赖的规则。预编译器输出这个make规则包含名字跟源文件相同的目标文件，冒号，和所有被引入的文件。

比如`$ gcc -M main.c ` 之后，输出`main.o: main.c /usr/include/stdc-predef.h /usr/include/stdio.h`。

##### -MF <u>file</u>

将依赖关系写到file里。

##### -MM

与-M相似，只是不包含系统头文件。

##### -MT <u>target</u>

重新定义目标对象名，比如上面那个main.o？

##### -shared

> Produce a shared object which can then be linked with other objects to form an executable.  Not all systems support this option.  For predictable results, you must also specify the same set of options used for compilation(-fpic, -fPIC, or model suboptions) when you specify this linker option.

生成一个共享对象，然后可以将其与其他对象链接以形成可执行文件。 并非所有系统都支持此选项。 为了获得可预测的结果，还必须在指定此链接器选项时指定用于编译的同一组选项（-fpic、-fPIC 或子选项）。