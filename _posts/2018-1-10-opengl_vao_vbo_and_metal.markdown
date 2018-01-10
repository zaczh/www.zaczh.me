# 顶点属性
在OpenGL着色过程中，输入顶点和输出顶点是1:1关系。对于同一个Shader，同样的顶点输入意味着同样的输出。因此，OpenGL缓存了每个顶点的绘制结果。当检查到重复的顶点绘制的时候，直接返回缓存的结果。
OpenGL是如何检测到重复顶点绘制的呢？如果要逐个比对原始的输入数据，那样显然太慢了。OpenGL采用了顶点属性（vertex attributes)的机制将所有相关的输入变量放在一起。
顶点属性不仅包括了用户手动指定的每个顶点的变量（通过attribute变量修饰符申明），而且还包含上一个着色器阶段输出的中间结果。譬如，在分段着色器中申明一个变量为out，然后在顶点着色器中声明同样名称的变量为in。这种中间结果的变量也称为全局变量，只有全局变量才能用接口块（Interface Block，其实就是一个C结构体)来定义格式。

用于OpenGL不支持变量名，所以顶点属性都是通过数组传递到OpenGL的，名为Buffer Object。Buffer Object是一个类似C数组一样的数据结构，它存储了顶点index和该index对应的属性（attributes）值。Shader从Buffer Object中获取顶点属性有三种方式：
+ 着色器里面指定

    通过在着色器中设置in变量的location值来指定：
 ```glsl
layout(location = 2) in vec4 a_vec;
```
    这行语句将VAO中index为2的变量赋值给a_vec。
+ 预链接（Pre-link）阶段指定

    也可以不在着色器里面指定，而是在链接着色器代码前通过OpenGL指令指定：
 ```glsl
void glBindAttribLocation(GLuint program, GLuint index, const GLchar *name);
```
+ 自动指定

    如果上述两种方法都没有使用，那么shader中用到的变量会在**链接**的时候自动指定，其顺序是完全任意的（即使是同样的着色器代码）。应该避免自动指定，如果某个变量不再使用了，就应该把它从着色器代码中删除。

# VBO和VAO
VAO和VBO是OpenGL里面很容易弄混的两个概念，这里的讲解希望能区分出来它们。
## VAO
VAO（Vertex Array Object）是OpenGL中用来放置所有顶点绘制数据的地方。VAO包含了很多子变量，你可以把它想象成一个客户端变量的集合体。修改VAO绑定关系的代码如下：
```glsl
glGenVertexArrays(1, &vao);
glBindVertexArray(vao);
```
上面提到的Buffer Object也是放在VAO里面的。由于一个着色过程可能会绘制很多个顶点，所以VAO里面放置的Buffer Object数据也是放在一个数组里面，可以把它想像成一个二维数组。
VAO有一个重要特性：它不会对里面的缓冲数据进行拷贝或者保存操作。这意味着，任何对里面数据的修改，都会影响到所有当前正在使用该VAO的用户。这说明VAO并不是一个OpenGL Buffer Object，实际上OpenGL的实现只是把VAO设置为了一个全局变量。另外，当修改OpenGL绑定的VAO之后，之前的VAO下面的那些缓冲变量并不会受到影响。一个着色器可能会用不同的VAO来绘制，相互之间不干涉。
VAO是特殊的OpenGL对象，每个OpenGL context只有一个VAO对象（注意：不同的OpenGL context拥有各自独立的VAO绑定关系，相互不影响），所以这里毋需指定绑定的目标。
当修改任何VAO子变量的时候，都会反映到最上层的VAO上面来，这样当下次重新绑定到此VAO的时候，之前的修改都会保留。这就是OpenGL设计VAO的初衷：将所有变化的部分汇总到一处。

## VBO
VBO(Vertex Buffer Object)是一个标准的OpenGL 缓冲对象。既然是缓冲对象，说明是用来在CPU和GPU之间共享数据的。其使用方法也比较简单：
```glsl
glBindBuffer(GL_ARRAY_BUFFER, buf1);
```
当绑定了VBO之后，通过以下方法来设置vertex attributes数据：
```glsl
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, 0, 0);
```
注意这里有个OpenGL的兼容问题：glVertexAttribPointer函数的最后一个参数在不同情况下有不同的意义。如果调用此方法之前绑定了VBO，那么该参数表示相对于绑定VBO的偏移量，否则（低版本OpenGL模式）表示实际的buffer数据复制地址指针。

## Index Buffer
顶点索引数据是通过Index Buffer传递的。 Index Buffer是一种具有特殊用途的buffer对象，它绑定在GL_ELEMENT_ARRAY_BUFFER目标上。另外，因为Index Buffer是客户端的状态，所以OpenGL将Index Buffer的绑定关系放置在VAO里面，这意味着修改Index buffer需要先绑定VAO。由于Index Buffer和VBO类型相似，又都是属于顶点数据，所以有些人也将其称为VIBO（Vertex Index Buffer Object，非官方名称）。
```glsl
glBindVertexArray(vao);
GLuint elementbuffer;
glGenBuffers(1, &elementbuffer);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, elementbuffer);
```
当指定了Index buffers之后，后续调用glDrawElements或者glDrawArray就可以使用该Index buffers来索引index了。

由于OpenGL的兼容性问题，VAO只有在OpenGL3.0以上版本才能使用。OpenGL ES 2.0无法获取和修改VAO绑定关系，意味着OpenGL ES 2.0只能使用传统的glDrawElements方法来绘制原型(primitive)。OpenGL ES
2.0里面的顶点属性数据是如何传递的呢？

通过通用顶点数据数组（generic vertex attribute array），代码如下：
```glsl
glEnableVertexAttribArray(GLKVertexAttribTexCoord0);
glVertexAttribPointer(GLKVertexAttribTexCoord0, 3, GL_FLOAT, GL_FALSE, 0, &buf);
...
```
这里，先打开了一个通用的顶点属性数组通道，然后向其传递CPU侧的数据指针，复制数据。

**要特别需要注意的是，虽然低版本的OpenGL不支持修改VAO，但是顶点属性数组依然是客户端状态，这意味着修改可能会影响同一个OpenGL context里面的其他程序。**

# 与Metal的比较
使用过Metal之后发觉比OpenGL好了太多：
- Metal不区分Vetex Buffer Object和其他buffer对象，统一为MTLBuffer
- Metal可以在客户端和GPU之间共享头文件，头文件里面定义的Buffer Index名称以及数据结构可以在两端共享
- API简洁明了，没有坑爹的兼容性问题

iOS新项目如果不需要兼容其他平台的话，建议使用Metal绘制。
