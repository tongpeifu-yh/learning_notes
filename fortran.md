# Fortran相关记录

作者：通配符

Email：tongpeifu.yh@gmail.com

## Fortran与C混合编程

Fortran与C混合编程时一般对Fortran代码进行修改以适配C语言ABI，因为Fortran的ABI各家编译器并不统一（比如Compaq、英特尔的编译器会把函数名转为大写，gfortran则会使用小写并后缀下划线）。从Fortran 2003以后，Fortran标准引入了用于和C语言混合编程的`BIND(C)`以及内置模块`ISO_C_BINDING`，让Fortran兼容C语言ABI，使混合编程方法更加统一。然而，不是所有的特性都可以用于混合编程，比如C语言的可变参数、unsigned类型等。

本质上，混合编程要求两方对同一个函数原型能够各自实现定义或声明（只有函数原型相同，链接器才能把它们链接到一起），同时进行正确的数据交互。`BIND(C)`以及内置模块`ISO_C_BINDING`正是完成了这两项工作。

在Fortran 2003出现以前，Compaq编译器（也就是后来的Intel编译器）使用一些扩展语法实现混合编程，这些扩展大多以`!DEC$` 、`!DIR$`开头，在其他编译器中会被认为是注释，这种写法已经过时了，应当弃用。

### BIND(C)

`BIND(C)`用于指定与C语言混合编程时当前函数的C语言名称。它的语法是：

```
BIND(C [, NAME=scalar-default-char-constant-expr])
```

例如，在C语言中名称为`print_C`的函数：

```c
void print_C(char *string);
```

需要在Fortran中如下定义或者声明：

```fortran
subroutine print_c(string) bind(C, name="print_C")
      !定义或者声明
      use iso_c_binding, only: c_char
      character(kind=c_char) :: string(*)
      end subroutine print_c
```

其中，`bind(C, name="print_C")`指定当前函数在C语言中的符号（Symbol）为`print_C`。使用linux的`nm`命令可以查看二进制程序中所有的符号，使用`bind(C, name="print_C")`指定函数名称后再查看编译出的二进制程序中的符号，会发现程序中只有`name="print_C"`指定的符号`print_C`。

若`BIND(C)`不指定`NAME=`，默认使用转为小写的Fortran名称符号，在上例中就是`print_c`。

`BIND(C)`还用于结构体（即派生类型）的声明。例如：

```fortran
USE ISO_C_BINDING
 TYPE, BIND(C) :: myType
   INTEGER(C_INT) :: i1, i2
   INTEGER(C_SIGNED_CHAR) :: i3
   REAL(C_DOUBLE) :: d1
   COMPLEX(C_FLOAT_COMPLEX) :: c1
   CHARACTER(KIND=C_CHAR) :: str(5)
 END TYPE
```

对应C语言结构体：

```c
 struct {
   int i1, i2;
   /* Note: "char" might be signed or unsigned.  */
   signed char i3;
   double d1;
   float _Complex c1;
   char str[5];
 } myType;
```

### ISO_C_BINDING

这是一个内置的模块，用于提供和C语言一致的数据类型、常量（如`C_NULL_CHAR`表示`'\0'`）、转换函数。

此模块提供的所有符号列表参见gfortran文档[9.2节`ISO_C_BINDING`](https://gcc.gnu.org/onlinedocs/gfortran/ISO_005fC_005fBINDING.html)，或者Intel Fortran Compiler文档中Compiler Reference->Mixed Language Programming->Standard Tools for Interoperability中的内容。

#### 数据类型

由于Fortran和C类型格式的不同（Fortran使用`type(kind)`的方式区分同一类但不同长度的类型），`ISO_C_BINDING`采用一系列**命名常量**作为kind值来与C语言的变量类型对标：例如，`C_INT`对标C语言中的`int`类型在Fortran中所属类型的kind，因此Fortran中的`integer(C_INT)`与C语言的`int`等价；同理Fortran中`real(C_FLOAT)`与C语言`float`等价。

| **ISO_C_BINDING中的命名常量**  <br/> (kind type parameter if value is positive) | **C类型** | **等价Fortran类型** |
|:---|:---|:---|
| C_SHORT<br/>C_INT<br/>C_LONG<br/>C_LONG_LONG | short int<br/>int<br/>long int<br/>long long int | INTEGER(KIND=2)<br/>INTEGER(KIND=4)<br/>INTEGER (KIND=4 or 8)<br/>INTEGER(KIND=8) |
| C_SIGNED_CHAR| signed charunsigned char| INTEGER(KIND=1)|
| C_SIZE_T| size_t| INTEGER(KIND=4 or 8)|
| C_INT8_T<br/>C_INT16_T<br/>C_INT32_T<br/>C_INT64_T | int8_t<br/>int16_t<br/>int32_t<br/>int64_t | INTEGER(KIND=1)<br/>INTEGER(KIND=2)<br/>INTEGER(KIND=4)<br/>INTEGER(KIND=8) |
| C_FLOAT<br/>C_DOUBLE<br/>C_LONG_DOUBLE | float<br/>double<br/>long double | REAL(KIND=4)<br/>REAL(KIND=8)<br/>REAL(KIND=8 or 16) |
| C_FLOAT_COMPLEX <br/>C_DOUBLE_COMPLEX <br/>C_LONG_DOUBLE_COMPLEX | float \_Complex <br/>double \_Complex <br/>long double \_Complex | COMPLEX(KIND=4) <br/>COMPLEX(KIND=8) <br/>COMPLEX(KIND=8 or 16) |
| C_BOOL | \_Bool |LOGICAL(KIND=1)|
| C_CHAR| char| CHARACTER(LEN=1)|

部分类型的kind值因操作系统和CPU架构而异。在32位x86平台中，由于gcc的`long double`类型是80bit的（因为x87浮点单元是80bit），而Intel编译器不支持这种浮点数，因此`C_LONG_DOUBLE` 值为-1，表示不可用。

Fortran处理`LOGICAL`类型的机制与C不同，Intel使用-1（即所有位都为1，0xff）表示true、0表示false，需要指定`fpscomp[:]logicals`选项（使用`standard-semantics`选项时自动包含此选项）使之和C一致；而gfortran采用Fortran标准规定的，要和C99标准保持一致（1代表true，0代表false，其他值未定义）。

C语言没有字符串类型，只有字符数组，如果要与Fortran交互，只能如下定义：

```fortran
character(len=1), dimension(*), intent(in) :: string
```

必须定义成长度为一的字符串（也就是一个单个字符）的数组。

#### 指针

`ISO_C_BINDING`提供派生类型`C_PTR` 和`C_FUNPTR`分别表示C指针和C函数指针。Intel文档声称：由于Fortran数组在内存中不一定连续，所以不能传递给C；但可以传递已分配的`allocatable` 数组到C中，也可以把C中的数组传递到Fortran。

C和Fortran的指针需要两个来自`ISO_C_BINDING`的内部函数`C_LOC`与`C_F_POINTER`进行转换。gfortran文档给出了这个例子：

```fortran
	use iso_c_binding
  	type(c_ptr) :: cptr1, cptr2
  	integer, target :: array(7), scalar
  	integer, pointer :: pa(:), ps
  	cptr1 = c_loc(array(1)) ! 程序员要确保这个数组在内存中是连续的
  	cptr2 = c_loc(scalar)
  	call c_f_pointer(cptr2, ps)
  	call c_f_pointer(cptr2, pa, shape=[7])
```

其中，第二行`type(c_ptr)`表示使用C指针类型，定义两个C指针；第三行`target`表示当前定义的变量需要被指针指向（Fortran中只有具有`target`属性的变量可以用指针指向）；第四行定义了两个Fortran指针。第五行调用`c_loc`函数取变量的C语言地址，得到C指针。最后调用`c_f_pointer`函数把C指针转换为Fortran指针。在转换C指针为Fortran数组类型指针时，必须传递`shape`参数指定数组元素个数。

注意Fortran使用引用传递，也就是说传递参数时`TYPE(C_PTR)`相当于`void**`，`TYPE(C_PTR), VALUE`相当于`void*`。

Intel文档同样给出了一个例子：

```fortran
program demo_c_f_pointer
       use, intrinsic :: iso_c_binding
       implicit none
    
       interface
         function make_array(n_elements) bind(C)
           import ! Make iso_c_binding visible here
           type(C_PTR) :: make_array
           integer(C_INT), value, intent(IN) :: n_elements
         end function make_array
       end interface
    
       type(C_PTR) :: cptr_to_array
       integer(C_INT), pointer :: array(:) => NULL()
       integer, parameter :: n_elements = 3 ! Number of elements
    
       ! Call C function to create and populate an array
       cptr_to_array = make_array(n_elements)
       ! Convert to Fortran pointer to array of n_elements elements
       call C_F_POINTER (cptr_to_array, array, [n_elements])
       ! Print value
       print *, array
    
end program demo_c_f_pointer
```

```c
#include <stdlib.h>
int  *make_array(int n_elements) {
    int *parray;
    int i;
	parray = (int*) malloc(n_elements * sizeof(int));
	for (i = 0; i < n_elements; i++) {
		parray[i] = i+1;
	}
	return parray;
 }
```

首先在`interface`中声明一个接口（即函数原型），返回值为`type(C_PTR)`，参数为值传递的`integer(C_INT)`。在函数中，首先调用C函数获取一个C指针，再使用`C_F_POINTER`函数把C指针转换为Fortran指针，转换时传入数组元素个数。

#### 函数指针

对函数指针，`ISO_C_BINDING`提供`C_F_PROCPOINTER` 把C函数指针转换为Fortran函数指针，`C_FUNLOC`把Fortran函数转换为C函数指针。使用派生类型`C_FUNPTR`表示C函数指针。gfortran文档给出了这个例子：

```c
/* Procedure implemented in Fortran.  */
void get_values (void (*)(double));

/* Call-back routine we want called from Fortran.  */
void
print_it (double x)
{
  printf ("Number is %f.\n", x);
}

/* Call Fortran routine and pass call-back to it.  */
void
foobar ()
{
  get_values (&print_it);
}
```

函数`foobar`调用了`get_values`，并传入函数指针`print_it`作为参数。因此在Fortran代码中需要实现函数`get_values`，并且需要正确接收C函数指针作为参数。

```fortran
MODULE m
  IMPLICIT NONE

  ! Define interface of call-back routine.
  ABSTRACT INTERFACE
    SUBROUTINE callback (x)
      USE, INTRINSIC :: ISO_C_BINDING
      REAL(KIND=C_DOUBLE), INTENT(IN), VALUE :: x
    END SUBROUTINE callback
  END INTERFACE

CONTAINS

  ! Define C-bound procedure.
  SUBROUTINE get_values (cproc) BIND(C)
    USE, INTRINSIC :: ISO_C_BINDING
    TYPE(C_FUNPTR), INTENT(IN), VALUE :: cproc

    PROCEDURE(callback), POINTER :: proc

    ! Convert C to Fortran procedure pointer.
    CALL C_F_PROCPOINTER (cproc, proc)

    ! Call it.
    CALL proc (1.0_C_DOUBLE)
    CALL proc (-42.0_C_DOUBLE)
    CALL proc (18.12_C_DOUBLE)
  END SUBROUTINE get_values

END MODULE m
```

在Fortran的`get_values`函数中，使用`TYPE(C_FUNPTR), VALUE`类型接收C语言函数指针（注意`VALUE`不能少），然后按照Fortran的方法定义了Fortran的函数指针`proc`，最后调用`C_F_PROCPOINTER`把C函数指针转换为Fortran的函数指针。

然后是如何把Fortran函数指针转换为C的。gfortran文档中的例子是：

```c
int
call_it (int (*func)(int), int arg)
{
  return func (arg);
}
```

这个C函数需要`int (*func)(int)`类型的参数，因此在Fortran中调用该函数时需要准备好这个类型的函数指针：

```fortran
MODULE m
  USE, INTRINSIC :: ISO_C_BINDING
  IMPLICIT NONE

  ! Define interface of C function.
  INTERFACE
    INTEGER(KIND=C_INT) FUNCTION call_it (func, arg) BIND(C)
      USE, INTRINSIC :: ISO_C_BINDING
      TYPE(C_FUNPTR), INTENT(IN), VALUE :: func
      INTEGER(KIND=C_INT), INTENT(IN), VALUE :: arg
    END FUNCTION call_it
  END INTERFACE

CONTAINS

  ! Define procedure passed to C function.
  ! It must be interoperable!
  INTEGER(KIND=C_INT) FUNCTION double_it (arg) BIND(C)
    INTEGER(KIND=C_INT), INTENT(IN), VALUE :: arg
    double_it = arg + arg
  END FUNCTION double_it

  ! Call C function.
  SUBROUTINE foobar ()
    TYPE(C_FUNPTR) :: cproc
    INTEGER(KIND=C_INT) :: i

    ! Get C procedure pointer.
    cproc = C_FUNLOC (double_it)

    ! Use it.
    DO i = 1_C_INT, 10_C_INT
      PRINT *, call_it (cproc, i)
    END DO
  END SUBROUTINE foobar

END MODULE m
```

首先在Fortran对`call_it`C函数的声明中，把第一个参数的类型声明为`TYPE(C_FUNPTR), VALUE`，注意`VALUE`不能少。然后定义一个`int (*func)(int)`类型的函数，再用`C_FUNLOC`函数把这个函数转换为`TYPE(C_FUNPTR)`类型（即C函数指针类型），最后把转换后的C指针传给`call_it`C函数。

### 函数原型声明

配合`BIND(C)`以及`ISO_C_BINDING`，可以在Fortran中声明与C等价的函数原型，具体格式如下：

```fortran
  integer(c_int) function func(i,j) bind(c,name="func")
    use iso_c_binding, only: c_int
    integer(c_int), VALUE :: i
    integer(c_int) :: j
```

对应C语言函数原型为：

```c
int func(int i, int *j)
```

要注意的是，Fortran默认使用地址传递（即参数为指针），因此需要加上`value`指定值传递，或者在C语言中声明为指针类型。

### 参数传递

把参数从C语言传递到Fortran，有以下几种情况。

#### 标量类型

使用`类型(kind), value`的方式声明即可。

#### 指针

对指向一个变量的指针，在Fortran中直接声明为`类型(kind)`即可，因为Fortran采用地址传递，默认传递变量用的就是指针。

对指向多个变量的指针，在Fortran需要声明为`type(C_PTR), value`类型，然后再定义一个Fortran数组指针`类型(kind), pointer :: array(:)`，最后用`C_F_POINTER`函数把C指针转换到`array`，注意转换为数组指针要传入`shape`来指定数组长度。

另外，虽然Fortran不允许VLA（变长数组，即使用变量作为数组长度来定义一个数组），但它允许在函数传递参数时用变量声明数组长度，同时支持`allocatable`可变长度的动态数组。例如：

```fortran
subroutine print_array(num,size)
	implicit none
	integer :: size
	integer :: num(size)
	
	write(*,*) num
end
```

而且Fortran传数组也是用地址传递，因此对指向多个变量的指针，也就是C语言中的数组，也可以在Fortran中直接声明为数组。

#### 字符串

C语言字符串是char类型数组。Intel文档中，直接使用数组：

```fortran
character(len=1), dimension(*), intent(in) :: string
```

来接受字符指针，而不会特意声明为`type(C_PTR), value`类型。但是要把这个字符串数组转换为单个字符串，需要用指向字符串的指针：

```fortran
integer s_len ! Length of string
character(100), pointer :: string1

call C_F_POINTER(C_LOC(string),string1)
s_len = INDEX(string1,C_NULL_CHAR) - 1
print *, string1(1:s_len)
```

也就是先把它转换到C指针，再转换到Fortran指针。但实际上直接以`type(C_PTR), value`类型接收，然后`C_F_POINTER`转换更加方便。

#### 函数指针

使用`TYPE(C_FUNPTR), VALUE`声明，然后定义Fortran的函数指针`proc`，最后调用`C_F_PROCPOINTER`把C函数指针转换为Fortran的函数指针。



## Fortran传递数组参数

似乎没有任何教程明确指出Fortran传递数组参数的多种方式，除了IBM的XL Fortran Language Reference。在[这里](https://www.cenapad.unicamp.br/parque/manuais/Xlf/lr38.HTM#HDRARRAYS)可以查看关于此的信息。

这里给出几个实例。

```fortran
    program testfortran20240820

    implicit none

    ! Variables
    integer(kind=4) ::a=5,b=6,c
	integer,dimension(4,5) ::arr
    ! Body of testfortran20240820
	arr=3
	call sub1(arr)
	call sub2(arr,size(arr))
	call sub3(arr)
	contains
		! Assumed-Shape Arrays
		! 只需要指定形参的维度（多少个冒号），每个维度的长度Fortran自动通过实参确定
		! 还可以指定某一维度的lower bound，会根据长度自动确定upper bound
		! 默认lower bound是1
		! 使用这种声明方式必须要有接口定义，可以是interface，也可以是module内函数，或者过程内函数
	    subroutine sub1(a)
		    implicit none
			integer,dimension(:,:) ::a
			print *,shape(a)
		end subroutine sub1
		! Explicit-Shape Arrays
		! 显式通过传递的参数确定数组的维度、长度。
		! 该方式还可以在子过程内定义新的局部数组（一般情况下是不允许使用变量作为数组长度的）。
		subroutine sub2(a,length)
		    implicit none
			integer length
			integer,dimension(length) ::a
			print *,shape(a)
		end subroutine sub2
		! Assumed-Size Arrays
		! 使用`*`声明数组维度，Fortran将不传递任何数组维度信息
		! 因此不能整体操作数组，也不能使用`size()`或者`shape()`计算维度信息！
		! 以下子程序将编译错误
		subroutine sub3(a)
		    implicit none
			integer length
			integer,dimension(*) ::a
			print *,size(a) !在此处产生错误
		end subroutine sub3
    end program testfortran20240820
```

