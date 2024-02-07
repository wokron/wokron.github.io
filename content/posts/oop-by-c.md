+++
title = "面向对象的 C 语言"
tags = ["C", "C++", "C#", "Java", "Python", "Golang"]
categories = ["编程语言"]
aliases = ["/posts/583d5770"]
date = "2023-10-01T22:53:53+08:00"
+++
## 一、前言——对象与过程
> 碎碎念：这篇文章里提到的语言是真的多：c、c++、c#、java、python、golang

c 语言怎么能面向对象呢？c 语言的设计当然并非为面向对象做出考虑，但是其拥有的语法却足以使我们写出具有面向对象味道的代码了。因为无论是面向过程或面向对象，其背后的本质思想都是相同的，那就是这样一个著名的公式：
$$
    程序 = 数据结构 + 算法
$$

<!-- more -->

面向过程无非是强调其算法的一面；面向对象无非是强调其数据结构的一面。当我们使用面向过程的思想编写代码时，我们所想的是数据是函数中的参数和变量，数据在过程中流动和变化。而在面向对象中情况则反了过来：方法成为了类的成员，被类型所划分，并从属于一定数据的集合。

了解了这一点之后，再看程序语言从面向过程到面向对象的发展过程也能有新的认识。这一发展背后实际上是程序的关注点由机器向人的转变。在面向过程的时代，人们所关注的是如何**操纵**数据。那时的机器还没有蒙在名为抽象的面纱之下，呈现在操作者面前的依旧是赤裸裸的整个内存空间，数据与数据之间没有清晰的边界，是操作者自己组织起整个系统，为各个空间划分边界，定下名称。而在这一构筑起来的系统之上，数据本身就没有那么重要了，因为更底层已经为其提供了随时取用的接口。这时，**管理**流程成了另一个关键问题。因为在底层的支持下构建起来的日益庞大的应用，其自身的结构却往往不能支持其质量。于是人们以数据为界，将面条一般的数据流切割成彼此独立却又相互关联的部分。这样对象才得以诞生。

## 二、c语言的面向对象何以可能
说回 c 语言，当其以结构体的方式组织起数据的时候，就已经有了对象的雏形了。如果我们将函数视为所属于其第一个参数类型的方法，那么对象的方法也可以表示。但是只有这两点并非真正的面向对象，因为面向对象的三大特征——封装、继承和多态，其中的后两者还未实现。

让我们来详细分析一下继承和多态到底在表明什么。继承是两个类型间的关系，类型 A 继承了类型 B，则类型 A 具有类型 B 所具有的一切属性和方法，这意味着对于 A 和 B 这两个不同的类型，都具有所属于 B 的部分。从这一点来说，两者是**相同**的（也因为这种相同，子类才能不加转换的赋给父类变量）。而多态（在这里指方法的重写而不包括重载）则指子类 A 对从 B 所继承的方法的重写，使得虽是相同方法，其表现却能有所**不同**。

明确了继承和多态，接下来我们从数据的角度分析 c 语言为何可以面向对象。所谓的一个对象，即在地址空间中的一段连续区域。此时继承中所谓的相同，即对两个不同类型的对象，其内存空间中相同位置所表达的含义相同。如果对于 B 类型来说，偏移 4 个字节之后的 4 个字节表示一个 int 字段，那么对于继承 B 类型的 A 类型来说，偏移 4 个字节之后的 4 个字节应同样表示一个 int 字段。类似的，多态中所谓的不同，可以表达为类型中**相同**的方法名对应的具体过程**不同**。由于过程在机器码中表现为地址，那么本质上来说，多态的这种不同不过是指相同字段中的值不同罢了。

此时 c 语言中实现继承和多态的方法呼之欲出，那就是使用指针。地址指示了变量所处空间的起始位置，却不表明按何种方式解释这块区域，而指针完成了这份工作。对于所有赋给指针的地址值，其都如实翻译其中的数据，那么如果想要子类与父类按照同样的方式进行翻译，就需要子类在组织其结构时保持和父类一致。而对于多态来说，事情则更简单了，函数指针同样是指针，只要使其指向不同的函数即可。

这样也可以理解为什么 c++ 中只能使用指针实现多态（引用本质还是指针）。
```cpp
Child c;
Father *f = &c;  // 正确
Father f2 = c;   // 错误
```

而 c++ 中使用 `new` 关键字申请内存这一点也被 java、c# 等面向对象语言学了过去。java、c# 等中的类变量，实际上也和指针或引用没有区别
```cpp
// c++
SomeClass *obj = new SomeClass();
```
```java
// java
SomeClass obj = new SomeClass();
```
```csharp
// c#
SomeClass obj = new SomeClass();
```

现在让我们结束一些形而上的讨论，看一下 c 语言到底应当如何实现面向对象。

## 三、封装的实现
所谓封装就是数据成员与方法结合，隐藏类内成员，对外暴露方法接口。这一点很是简单。

假设现在有一个简单的 `Dog` 类，里面有一个字段 `name`。
```c
struct Dog {
    char *name;
};
```
一般来说，想要访问 `name` 字段只需要直接指明字段名即可。
```c
int main() {
    struct Dog dog;
    dog.name = "scooby";
    printf("the dog's name is %s\n", dog.name);

    return 0;
}
```

不过为了封装，可以使用 `getter` `setter` 对字段进行读写。
```c
char *dog_get_name(struct Dog *self) {
    return self->name;
}

char *dog_set_name(struct Dog *self, char name[]) {
    self->name = name;
}
```

这里 `Dog` 作为了函数的第一个参数，以此表明该函数是属于 `Dog` 的方法。那么上面的程序就可以改写为下面的形式。
```c
int main() {
    struct Dog dog;
    dog_set_name(&dog, "scooby");
    printf("the dog's name is %s\n", dog_get_name(&dog));

    return 0;
}
```

当然对于简单的字段都要用程序封装实在是一种古板的做法，尤其是在使用 c 语言模拟面向对象的时候，但这只是一个例子而已。除此之外，golang 将首字母大写的属性公开的方法也有可取之处，c# 的 property 也模拟了类似的读写语法。
```go
// golang
type Dog struct {
    Name string
}
```
```csharp
// csharp
class Dog
{
    public string Name { get; set; }
}
```

另外，各位应该能看出这样的写法和 python 实在类似，我们可以用 python 写出类似的代码。从中也可以稍微体会到一些 python 特有的设计哲学。
```py
class Dog:
    name: str

    def get_name(self):
        return self.name
    
    def set_name(self, name):
        self.name = name

if __name__ == "__main__":
    dog = Dog()
    Dog.set_name(dog, "scooby")
    print(f"the dog's name is {Dog.get_name(dog)}")
```

## 四、继承的实现
在第二小节时说过，继承体现在子类与父类具有相同的内存结构，举个例子，假如有两个类 `Pet` 和 `Dog`，其中 `Pet` 具有属性 `name`，`Dog` 继承了 `Pet`，同时增加了属性 `color`。那么对于 `Dog` 中的 `name` 来说，其所在的位置应该与 `Pet` 中的位置相同。

也就是说，对于 `Pet`
```c
struct Pet {
    char *name;
};

void pet_describe(struct Pet *self) {
    printf("the pet's name is %s\n", self->name);
}
```

那么 `Dog` 也应当具有相同的结构，`name` 应当位于最开始的位置
```c
// 正确
struct Dog { // : Pet
    char *name;
    char *color;
};

// 错误
struct DogFalse { // : Pet
    char *color;
    char *name;
};
```

对于错误的结构，会导致错误的输出。
```c
int main() {
    struct Dog dog1 = {"scooby", "brown"};
    struct DogFalse dog2 = {"white", "kitty"};
    struct Pet *p1 = &dog1;
    struct Pet *p2 = &dog2;
    pet_describe(p1);
    pet_describe(p2);
}
```
```text
the pet's name is scooby
the pet's name is white
```

除此之外，继承还有另一种实现方式，就是使用嵌套的结构体，因为最终在内存中，嵌套的结构都会被展平。此时 `Pet` 的定义不变，`Dog` 的定义变为如下形式
```c
struct Dog {
    struct Pet super;
    char *color;
}
```

但是这样存在一个问题，就是 `Pet` 类中的字段我们不再能直接访问，这违反了继承的原则，尽管这种写法和之前的写法中 `Dog` 结构体的结构都是一样的。

golang 中采用后一种写法表示“继承”关系，规定当只使用类型名而不添加字段名的时候，自动使“子类”获得“父类”的属性。这就以一种很简单的方式解决了继承问题。Ken Thompson 在设计 golang 时确实保留了一些 c 的风格。
```go
type Pet struct {
    Name string
}

type Dog struct {
    Pet
    Color string
}

func main() {
	dog := Dog{"scooby", "brown"};
    fmt.Println(dog.name);  // 正确
}
```

## 五、多态的实现
除了属性之外，还有方法需要处理。为了实现多态，我们需要让方法也具有继承关系。具体来说，也应当有一个类似于属性的结构体，其中存储该类型所用到的所有函数，而当出现一个子类的时候，也应当有一个属于该子类的，继承父类函数结构体的结构体。该结构体称为虚表。

我们举 `Pet`、`Dog` 和 `Cat` 作为例子。`Pet` 具有属性 `name` 和方法 `describe`；`Dog` 和 `Cat` 继承 `Pet`。`Dog` 有额外属性 `color` 和方法 `bark`，`Cat` 有方法 `meow`。

那么三个类型各有其虚表
```c
struct PetVTable {
    char *(*describe)(struct Pet *, char []);
};

struct DogVTable {
    char *(*describe)(struct Pet *, char []);
    void (*bark)(struct Dog *);
};

struct CatVTable {
    char *(*describe)(struct Pet *, char []);
    void (*meow)(struct Cat *);
};
```
此时我们若想调用某一对象的方法，就不再直接通过函数的全局名称访问，而是通过虚表间接访问。由于虚表之间有着结构上的继承关系，所以子类必定拥有父类的方法；又由于虚表存储着函数的地址而非函数本身，因此对于有着不同虚表的对象，调用相同的方法也可能实际执行不同的函数。

对于面向对象来说，每个类的对象有着相同的虚表，因此只需要一个类使用一个全局的虚表即可。我们可以在每个类的字段中添加一个指向虚表的指针。由于需要保证对于父类和子类，都能访问其虚表，因此该指针必须放在结构体的首部。
```c
struct Pet {
    struct PetVTable *f;
    char *name;
};

struct Dog {
    struct DogVTable *f;
    char *name;
    char *color;
};

struct Cat {
    struct CatVTable *f;
    char *name;
};
```

接下来需要将虚表与对象关联起来，也就是说，在初始化时，将类所对应的虚表的值赋给 `f`。这就需要引入构造函数了。

```c
struct Pet *create_Pet(struct Pet *self, char name[]) {
    static struct PetVTable v = {NULL};
    self->f = &v;
    self->name = name;
}

struct Dog *create_Dog(struct Dog *self, char name[], char color[]) {
    self = create_Pet(self, name);
    static struct DogVTable v = {dog_describe, dog_bark};
    self->f = &v;
    self->color = color;
}

struct Cat *create_Cat(struct Cat *self, char name[]) {
    self = create_Pet(self, name);
    static struct CatVTable v = {cat_describe, cat_meow};
    self->f = &v;
}
```
这里分别定义了三个类型的构造函数，都传入了一个指针表示要初始化的对象的地址，为什么不在构造函数内部调用 `malloc` 呢？因为还存在栈上变量的可能。因此应该传入指针，把内存的申请留给用户。另外在 `Dog` 和 `Cat` 的构造函数中先调用了 `Pet` 的构造函数，这也是面向对象语言中的初始化流程。还有，我们将虚表定义为了函数内的 `static` 变量，这也保证了同一类只有同一虚表。

最后还请注意 `create_Pet` 函数中的虚表，这里将其值设定为 NULL，说明 `Pet` 类不存在一个对应 `describe` 方法的实际函数。这意味着 `Pet` 类不应调用 `describe` 函数，或更进一步的，`Pet` 类不应存在实例。也就是说 `Pet` 类应该是一个抽象类。这一点实际上很像 c++ 中对于虚函数的定义。以 c++ 来描述 `Pet` 类的话应该就是这样的：
```cpp
struct Pet {
    char *name;
    virtual char *describe(char buf[]) = 0;
}
```
将 `describe`“赋值”为 0 和将虚表中的 `describe` 项赋值为 NULL（也是0）完全是一样的。

另外，是不是觉得 `create_Pet` 这样的函数名太奇怪了？这其实是为预处理宏来准备的。接下来上一点宏魔法 `:)`
```c
#define NEW(cls, ...) create_##cls(malloc(sizeof(struct cls)), __VA_ARGS__)  // alloc on heap
#define DECL(cls, name, ...) struct cls name; create_##cls(&name, __VA_ARGS__)  // alloc on stack
```

这样分别定义了两个宏用来在堆和栈上创建变量。使用起来就如下面这样：
```c
int main() {
    struct Pet *pet = NEW(Cat, "Kitty");  // Pet *pet = new Cat("Kitty");
    DECL(Dog, dog, "Beauty", "white");  // Dog dog("Beauty", "white");
}
```
是不是有一点 c++ 的感觉了？

最后，我们实现一下 `Dog` 和 `Cat` 类的函数，并用一个小程序测试一下。
```c
char *dog_describe(struct Pet *self, char buf[]) {
    struct Dog *dself = self;
    sprintf(buf, "the %s dog is %s", dself->color, dself->name);
    return buf;
}

void *dog_bark(struct Dog *self) {
    printf("%s said: \"wooof!!!\"\n", self->name);
}

char *cat_describe(struct Pet *self, char buf[]) {
    struct Cat *cself = self;
    sprintf(buf, "this cat named %s", cself->name);
    return buf;
}

void *cat_meow(struct Cat *self) {
    printf("%s said: \"meow...\"\n", self->name);
}

int main() {
    struct Pet *pet1 = NEW(Cat, "Kitty");
    struct Pet *pet2 = NEW(Dog, "Scooby", "brown");

    char desc_buf[30];
    printf("desc cat: %s\n", pet1->f->describe(pet1, desc_buf));
    printf("desc dog: %s\n", pet2->f->describe(pet2, desc_buf));

    struct Cat *cat = pet1;
    struct Dog *dog = pet2;
    cat->f->meow(cat);
    dog->f->bark(dog);

    return 0;
}
```

输出结果如下，可以看出 `describe` 函数的多态性。
```text
desc cat: this cat named Kitty
desc dog: the brown dog is Scooby
Kitty said: "meow..."
Scooby said: "wooof!!!"
```

## 六、函数重载？
最后还想讲一点关于函数重载的内容，函数重载也是面向对象的一部分。c 语言没有函数重载，这对于程序员来说十分不友好，因为众所周知，取名字是一项十分耗费时间精力的工作。可如果我说能够让 c 语言也有函数重载，你是否相信呢？

当然这一点是夸大了，我们不可能改变 c 语言的语法，我的意思是，如何在没有函数重载的情况下以类似的方式解决函数重载所解决的问题。对于这一点，实际上我们可以完全参考 python 的解决方法。python 没有函数重载，实际上我在使用 python 很长时间之后还没有意识到这一点，当然这一定上源于用 python 进行的工作大部分不过是调用 api 而已。但是这还是能一定程度上说明 python 的解决方法是较为合理的。

python 的实现方法实际上就是参数表。通过设定默认值使得调用者只需要设定自己需要的参数，从而实现了“伪重载”。举一个例子
```py
def add(
    a: int = 0,
    b: int = 0,
    arr: list[int] = None,
):
    if arr is None:
        return a + b
    else:
        return sum(arr)


print(add(a=1, b=2))
print(add(arr=[1, 2, 3, 4, 5]))
```
输出
```text
3
15
```
当然 python 不需要重载还有部分鸭子类型的原因。c 语言虽然没有，但是依旧可以使用参数表模拟多态的方式更加灵活地调用函数。

```c
#include <stdio.h>

struct AddParam {
    int a;
    int b;
    int *arr;
    int size;
};

int add(struct AddParam *param) {
    if (param->arr == NULL) {
        return param->a + param->b;
    } else {
        int sum = 0;
        for (int i = 0; i < param->size; i++) {
            sum += param->arr[i];
        }
        return sum;
    }
}

int main() {
    struct AddParam param1;
    param1.a = 1;
    param1.b = 2;
    printf("%d\n", add(&param1));
    struct AddParam param2;
    int arr[5] = {1, 2, 3, 4, 5};
    param2.arr = arr;
    param2.size = 5;
    printf("%d\n", add(&param2));
}
```
